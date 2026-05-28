# CODEX — Phase 2.5 Fix + Feature Sheet
## Phase 1 Bug Fixes + Image Generation Feature

---

# ═══════════════════════════
# PHASE 1 BUGS — EXACT FIXES
# ═══════════════════════════

---

## BUG 1 — Cache type loss in codex_config
**Severity:** CRITICAL
**File:** `modules/codex_config/core.py`
**Problem:** Cache hit pe `_deserialize()` sirf `json.loads` karta hai.
`value_type` cache mein store nahi hoti. Result: `bool` config
`True` ki jagah `"true"` string return karta hai. Bool conditions
silently break hoti hain.

**BEFORE DEPLOY:** Redis flush karo ek baar:
```bash
redis-cli KEYS "config:*" | xargs redis-cli DEL
```

**DELETE this function entirely:**
```python
def _deserialize(key: str, cached_json: str) -> Any:
    """Deserialize cached string. Type not stored in cache — return str for re-cast."""
    try:
        return json.loads(cached_json)
    except (json.JSONDecodeError, ValueError):
        return cached_json
```

**REPLACE `get()` with:**
```python
async def get(key: str, default: Any = None) -> Any:
    """Get a config value by key. Type-cast per value_type."""
    redis = await get_redis()
    cache_key = f"config:{key}"

    cached = await redis.get(cache_key)
    if cached is not None:
        try:
            payload = json.loads(cached.decode())
            # payload = {"v": raw_value, "t": value_type}
            return _cast(payload["v"], payload["t"])
        except Exception:
            pass  # cache corrupt — fall through to DB

    async with get_session() as session:
        result = await session.execute(
            select(SystemConfig).where(SystemConfig.key == key)
        )
        row = result.scalar_one_or_none()

    if row is None:
        return default

    value = _cast(row.value, row.value_type)
    await redis.setex(
        cache_key,
        CACHE_TTL,
        json.dumps({"v": row.value, "t": row.value_type}),
    )
    return value
```

---

## BUG 2 — Rate limit increment before AI call
**Severity:** HIGH
**File:** `bot/handlers/chat.py`
**Problem:** `increment()` pehle, AI call baad mein. Provider fail ho
to user ka slot waste hota hai. Fix: increment + decrement pattern.

**STEP 1 — Add `decrement()` to `modules/codex_ratelimit/core.py`:**
```python
async def decrement(user_id: int) -> None:
    """Refund one request slot — called on provider failure."""
    redis     = await get_redis()
    count_key = f"ratelimit:{user_id}:count"
    raw       = await redis.get(count_key)
    if raw is None:
        return
    count = int(raw)
    if count > 0:
        await redis.decr(count_key)
    logger.info("ratelimit.decrement user_id=%s", user_id)
```

**STEP 2 — In `chat.py`, find and update:**

FIND (around line 113):
```python
    # Increment rate limit counter
    tier = _user_tier(db_user)
    await ratelimit_mod.increment(db_user.id, tier)
```

REPLACE WITH:
```python
    tier = _user_tier(db_user)
    await ratelimit_mod.increment(db_user.id, tier)  # increment first
```

FIND the success assignment:
```python
        success = True
```

REPLACE WITH:
```python
        success = True
        # success — slot correctly consumed
```

FIND the except blocks where `success` stays `False`, add refund:
```python
    except httpx.HTTPStatusError as exc:
        if exc.response.status_code == 429:
            # ... retry logic ...
            try:
                response_text, prompt_tokens, compl_tokens = await _nvidia_nim_call(...)
                success = True
            except Exception as retry_exc:
                error_code = "provider_unavailable"
                await ratelimit_mod.decrement(db_user.id)  # ← ADD THIS
        else:
            error_code = f"http_{exc.response.status_code}"
            await ratelimit_mod.decrement(db_user.id)      # ← ADD THIS

    except Exception as exc:
        error_code = "provider_timeout"
        await ratelimit_mod.decrement(db_user.id)          # ← ADD THIS
```

---

## BUG 3 — /import hardcoded plan_id=2
**Severity:** MEDIUM
**File:** `bot/handlers/admin.py`

**STEP 1 — Find where codes are stored in Redis and add plan selection:**

FIND (after codes are parsed and validated):
```python
await redis.setex(f"admin_import_codes:{callback.from_user.id}", 300, json.dumps(codes))
await callback.message.edit_text("✔ Codes parsed...")
```

REPLACE WITH:
```python
await redis.setex(
    f"admin_import_codes:{callback.from_user.id}", 300, json.dumps(codes)
)

from models import Plan
from sqlalchemy import select
from modules.codex_config import get_session

async with get_session() as session:
    result = await session.execute(
        select(Plan).where(Plan.active == True, Plan.plan_type != "free")
        .order_by(Plan.id)
    )
    plans = result.scalars().all()

plan_buttons = [
    [InlineKeyboardButton(
        text=p.name,
        callback_data=f"admin:import:plan:{p.id}"
    )]
    for p in plans
]
plan_buttons.append([
    InlineKeyboardButton(text="❌ Cancel", callback_data="admin:import:cancel")
])

await callback.message.edit_text(
    f"✔ <b>{len(codes)} codes parsed</b>\n\nKaunse plan ke liye import karein?",
    parse_mode="HTML",
    reply_markup=InlineKeyboardMarkup(inline_keyboard=plan_buttons),
)
```

**STEP 2 — Add new callback handler:**
```python
@router.callback_query(F.data.startswith("admin:import:plan:"))
async def cb_import_plan_selected(callback: CallbackQuery) -> None:
    if not await is_admin(callback.from_user.id):
        return await callback.answer("Access denied", show_alert=True)

    plan_id = int(callback.data.split(":")[-1])
    redis   = await get_redis()
    raw     = await redis.get(f"admin_import_codes:{callback.from_user.id}")
    if not raw:
        return await callback.answer("Import session expired", show_alert=True)

    codes = json.loads(raw)
    await redis.delete(f"admin_import_codes:{callback.from_user.id}")

    inserted, skipped = await coupon_mod.bulk_create(
        codes=codes,
        plan_id=plan_id,
        source="sellauth",
        created_by=callback.from_user.id,
    )

    await callback.message.edit_text(
        f"✔ <b>Import complete</b>\n\n"
        f"Inserted: <code>{inserted}</code>\n"
        f"Skipped:  <code>{skipped}</code>",
        parse_mode="HTML",
    )
    await callback.answer()
```

---

## BUG 4 — payment.py hardcoded plan_map
**Severity:** MEDIUM
**File:** `bot/handlers/payment.py`

FIND (around line 118-120):
```python
plan_slug = (await redis.get(f"buy_plan:{callback.from_user.id}") or b"pro_monthly").decode()
plan_map  = {"pro_monthly": 2, "pro_yearly": 3}
plan_id   = plan_map.get(plan_slug, 2)
```

REPLACE WITH:
```python
plan_slug = (await redis.get(f"buy_plan:{callback.from_user.id}") or b"").decode()
if not plan_slug:
    await callback.answer("Session expired. Start again with /buy.", show_alert=True)
    return

from models import Plan
from sqlalchemy import select
from modules.codex_config import get_session

async with get_session() as session:
    result = await session.execute(
        select(Plan).where(Plan.slug == plan_slug, Plan.active == True)
    )
    plan_obj = result.scalar_one_or_none()

if not plan_obj:
    await callback.answer("Plan not found. Contact support.", show_alert=True)
    return

plan_id = plan_obj.id
```

**Also fix `modules/codex_admin/app.py` line 234:**

FIND:
```python
plan_id = int(body.get("plan_id", 2))
```

REPLACE WITH:
```python
plan_id = body.get("plan_id")
if not plan_id:
    return JSONResponse({"error": "plan_id required"}, status_code=400)
plan_id = int(plan_id)
```

---

## BUG 5 — _user_tier() duplicate
**Severity:** LOW
**Files:** `bot/handlers/chat.py` + `bot/handlers/user.py`

**CREATE `bot/utils/helpers.py`:**
```python
"""
bot/utils/helpers.py
Shared utility functions for all handlers.
"""


def get_user_tier(db_user) -> str:
    """Determine rate limit tier from user plan state."""
    if db_user is None:
        return "free"
    plan_id = getattr(db_user, "plan_id", None)
    credits = getattr(db_user, "credits_balance", 0)
    has_sub   = plan_id is not None and plan_id != 1
    has_topup = credits > 0
    if has_sub and has_topup: return "both"
    if has_sub:               return "sub"
    if has_topup:             return "topup"
    return "free"


def get_plan_name(db_user) -> str:
    """Human-readable plan name for display."""
    plan_id = getattr(db_user, "plan_id", None)
    names = {None: "Free", 1: "Free", 2: "Pro Monthly", 3: "Pro Yearly"}
    return names.get(plan_id, "Free")
```

In `chat.py` and `user.py`:
- DELETE `_user_tier()` and `_plan_name()` functions
- ADD at top: `from bot.utils.helpers import get_user_tier, get_plan_name`
- Replace all calls: `_user_tier(db_user)` → `get_user_tier(db_user)`

---

## BUG 6 — boto3 sync in worker
**Severity:** LOW — not crashing, fix in Phase 2 with file ops.

Add to requirements.txt when Phase 2 file ops are built:
```
aioboto3==13.1.1
```

---

# ═══════════════════════════════════
# IMAGE GENERATION — PHASE 2.5 SPEC
# ═══════════════════════════════════

## Overview

Two-provider image generation system.
Command: `/draw` (`.draw`)
Low quality: Free external API (GET request)
High quality: Google AI Studio — Imagen 3 (`imagen-3.0-generate-001`)
All limits from system_config. Credits deducted per image.

---

## New Module: `modules/codex_imagegen/`

```
modules/codex_imagegen/
  __init__.py         ← public interface
  core.py             ← orchestrator: route → enhance → generate → deliver
  prompt_enhancer.py  ← AI prompt enhancement + NSFW filter
  providers/
    free_api.py       ← GET http://king-ai-by-harmanxking.zya.me/imagegen.php
    google_imagen.py  ← Google AI Studio Imagen 3
  exceptions.py
  config_keys.py
```

---

## New system_config Keys

Add to seeder + admin dashboard under new "Image Gen" tab:

```
# Provider enable/disable
imagegen.free_api.enabled          bool    true
imagegen.google.enabled            bool    true

# Daily limits per tier (daily reset at midnight UTC)
imagegen.free_api.daily.free       int     1
imagegen.free_api.daily.topup      int     3
imagegen.free_api.daily.sub        int     5
imagegen.free_api.daily.both       int     10

imagegen.google.daily.free         int     0     # no google for free tier
imagegen.google.daily.topup        int     2
imagegen.google.daily.sub          int     5
imagegen.google.daily.both         int     10

# Credits per image
imagegen.free_api.credits_per_img  int     0     # free tier = 0 credits
imagegen.google.credits_per_img    int     5

# Prompt enhancement
imagegen.enhance_prompt            bool    true
imagegen.nsfw_filter               bool    true
imagegen.max_prompt_len            int     500

# Google Imagen config
imagegen.google.model              string  imagen-3.0-generate-001
imagegen.google.aspect_ratio       string  1:1

# Feature flag
feature.image_gen                  bool    true
```

---

## New Redis Keys

```
imggen:{user_id}:free_api:{YYYY-MM-DD}    → count (TTL: end of day UTC)
imggen:{user_id}:google:{YYYY-MM-DD}      → count (TTL: end of day UTC)
```

Daily reset = key expires at midnight UTC automatically via TTL.

---

## New DB Table: `image_jobs`

Add to `models.py`:

```python
class ImageJob(Base):
    __tablename__ = "image_jobs"

    id: Mapped[int] = mapped_column(Integer, primary_key=True)
    user_id: Mapped[int] = mapped_column(Integer, nullable=False, index=True)
    request_id: Mapped[str] = mapped_column(String(64), nullable=False)
    provider: Mapped[str] = mapped_column(String(32))      # free_api | google
    prompt_original: Mapped[str] = mapped_column(Text)     # user's raw prompt
    prompt_enhanced: Mapped[str] = mapped_column(Text)     # after enhancement
    status: Mapped[str] = mapped_column(String(32), default="queued")
        # queued | generating | done | failed | nsfw_blocked
    credits_deducted: Mapped[int] = mapped_column(Integer, default=0)
    error: Mapped[Optional[str]] = mapped_column(Text, nullable=True)
    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), server_default=func.now()
    )
```

---

## Provider Implementations

### `providers/free_api.py`

```python
"""
Free image API provider.
GET http://king-ai-by-harmanxking.zya.me/imagegen.php?prompt={encoded}
Returns image bytes directly.
"""
from __future__ import annotations
import urllib.parse
import httpx
from modules.codex_imagegen.exceptions import ImageGenError


async def generate(prompt: str) -> bytes:
    """Generate image. Returns raw image bytes (JPEG/PNG)."""
    encoded = urllib.parse.quote_plus(prompt)
    url = f"http://king-ai-by-harmanxking.zya.me/imagegen.php?prompt={encoded}"

    async with httpx.AsyncClient(timeout=60.0) as client:
        resp = await client.get(url)
        resp.raise_for_status()

        content_type = resp.headers.get("content-type", "")
        if "image" not in content_type:
            raise ImageGenError(
                f"Free API returned non-image response: {content_type}"
            )

        return resp.content
```

### `providers/google_imagen.py`

```python
"""
Google AI Studio — Imagen 3 provider.
Model: imagen-3.0-generate-001
Auth: GOOGLE_API_KEY env var
"""
from __future__ import annotations
import os
import google.generativeai as genai
from modules.codex_config import get
from modules.codex_imagegen.exceptions import ImageGenError


async def generate(prompt: str) -> bytes:
    """Generate image via Imagen 3. Returns PNG bytes."""
    api_key = os.environ.get("GOOGLE_API_KEY", "")
    if not api_key:
        raise ImageGenError("GOOGLE_API_KEY not configured")

    model_name   = str(await get("imagegen.google.model",
                                  "imagen-3.0-generate-001"))
    aspect_ratio = str(await get("imagegen.google.aspect_ratio", "1:1"))

    genai.configure(api_key=api_key)

    model  = genai.ImageGenerationModel(model_name)
    result = model.generate_images(
        prompt=prompt,
        number_of_images=1,
        aspect_ratio=aspect_ratio,
    )

    if not result.images:
        raise ImageGenError("Imagen returned no images")

    # result.images[0]._pil_image → PNG bytes
    import io
    buf = io.BytesIO()
    result.images[0]._pil_image.save(buf, format="PNG")
    return buf.getvalue()
```

---

## Prompt Enhancer: `prompt_enhancer.py`

```python
"""
prompt_enhancer.py
Two jobs:
  1. Enhance user prompt for better image quality
  2. NSFW filter — block or sanitize bad prompts

Enhancement uses Nvidia NIM (Layer 1) — cheap, fast.
NSFW uses keyword blocklist first, then optional AI check.
"""
from __future__ import annotations
import os
import httpx
from modules.codex_config import get
from modules.codex_imagegen.exceptions import NSFWBlockedError

# Hard blocklist — always blocked regardless of context
_NSFW_KEYWORDS = {
    "nude", "naked", "nsfw", "porn", "explicit",
    "genitals", "xxx", "hentai", "gore", "violence",
}

_ENHANCE_SYSTEM_PROMPT = """You are an image prompt engineer.
Your job: take a user's short image description and expand it into
a detailed, vivid prompt optimized for AI image generation.

Rules:
- Keep the user's original intent 100%
- Add: lighting style, mood, camera angle, art style, quality tags
- Keep total length under 400 characters
- Output ONLY the enhanced prompt — no explanation, no quotes
- Do NOT add anything sexual, violent, or inappropriate

Example:
Input:  girl standing in rain
Output: A young woman standing gracefully in heavy rain at night,
        neon city lights reflecting on wet pavement, cinematic
        lighting, detailed, photorealistic, 4K, moody atmosphere
"""


def _nsfw_check(prompt: str) -> bool:
    """True = NSFW detected."""
    lower = prompt.lower()
    return any(kw in lower for kw in _NSFW_KEYWORDS)


async def enhance(prompt: str) -> str:
    """
    Enhance prompt for better image generation.
    Returns enhanced prompt string.
    Raises NSFWBlockedError if prompt is blocked.
    """
    nsfw_filter = await get("imagegen.nsfw_filter", True)
    max_len     = int(await get("imagegen.max_prompt_len", 500))

    # Truncate if too long
    prompt = prompt[:max_len].strip()

    # NSFW check on original
    if nsfw_filter and _nsfw_check(prompt):
        raise NSFWBlockedError("Prompt contains inappropriate content")

    enhance_enabled = await get("imagegen.enhance_prompt", True)
    if not enhance_enabled:
        return prompt

    # Call Nvidia NIM for enhancement
    api_key = os.environ.get("NVIDIA_API_KEY", "")
    if not api_key:
        return prompt  # fallback: return original if no key

    try:
        url = "https://integrate.api.nvidia.com/v1/chat/completions"
        headers = {
            "Authorization": f"Bearer {api_key}",
            "Content-Type":  "application/json",
        }
        payload = {
            "model":      str(await get("ai.chat.model",
                                        "meta/llama-3.1-70b-instruct")),
            "max_tokens": 200,
            "messages": [
                {"role": "system", "content": _ENHANCE_SYSTEM_PROMPT},
                {"role": "user",   "content": prompt},
            ],
        }
        async with httpx.AsyncClient(timeout=30.0) as client:
            resp = await client.post(url, json=payload, headers=headers)
            resp.raise_for_status()
            enhanced = resp.json()["choices"][0]["message"]["content"].strip()

        # NSFW check on enhanced prompt too
        if nsfw_filter and _nsfw_check(enhanced):
            raise NSFWBlockedError("Enhanced prompt contains inappropriate content")

        return enhanced

    except NSFWBlockedError:
        raise
    except Exception:
        return prompt  # fallback to original on any error
```

---

## Core Orchestrator: `core.py`

```python
"""
codex_imagegen / core.py
Main orchestrator: limit check → enhance → generate → return bytes.
"""
from __future__ import annotations
import logging
from datetime import datetime, timezone
from typing import Literal

from modules.codex_config import get, get_redis, get_session
from modules.codex_imagegen.prompt_enhancer import enhance
from modules.codex_imagegen.exceptions import (
    ImageGenError, NSFWBlockedError, DailyLimitError
)

logger = logging.getLogger(__name__)

Provider = Literal["free_api", "google"]


async def _daily_count(user_id: int, provider: Provider) -> int:
    redis = await get_redis()
    today = datetime.now(timezone.utc).strftime("%Y-%m-%d")
    key   = f"imggen:{user_id}:{provider}:{today}"
    raw   = await redis.get(key)
    return int(raw) if raw else 0


async def _daily_increment(user_id: int, provider: Provider) -> None:
    redis = await get_redis()
    today = datetime.now(timezone.utc).strftime("%Y-%m-%d")
    key   = f"imggen:{user_id}:{provider}:{today}"
    await redis.incr(key)
    # TTL = seconds until midnight UTC
    now       = datetime.now(timezone.utc)
    midnight  = now.replace(hour=23, minute=59, second=59)
    ttl       = int((midnight - now).total_seconds()) + 1
    await redis.expire(key, ttl)


async def _get_daily_limit(provider: Provider, tier: str) -> int:
    key = f"imagegen.{provider}.daily.{tier}"
    return int(await get(key, 0))


async def generate_image(
    user_id: int,
    db_user,
    user_prompt: str,
    provider: Provider,
    tier: str,
) -> tuple[bytes, str, str]:
    """
    Main entry point.
    Returns (image_bytes, enhanced_prompt, provider_used).
    Raises DailyLimitError, NSFWBlockedError, ImageGenError.
    """
    # Feature flag
    if not await get("feature.image_gen", True):
        raise ImageGenError("Image generation is currently disabled.")

    # Provider enabled?
    if not await get(f"imagegen.{provider}.enabled", True):
        raise ImageGenError(f"Provider {provider} is not enabled.")

    # Daily limit check
    daily_limit = await _get_daily_limit(provider, tier)
    if daily_limit == 0:
        raise DailyLimitError(
            f"Your plan does not have access to "
            f"{'high quality' if provider == 'google' else 'image'} generation."
        )

    count = await _daily_count(user_id, provider)
    if count >= daily_limit:
        raise DailyLimitError(
            f"Daily limit reached ({count}/{daily_limit}). "
            f"Resets at midnight UTC."
        )

    # Enhance prompt (raises NSFWBlockedError if blocked)
    enhanced = await enhance(user_prompt)

    # Generate
    if provider == "free_api":
        from modules.codex_imagegen.providers.free_api import generate
    else:
        from modules.codex_imagegen.providers.google_imagen import generate

    image_bytes = await generate(enhanced)

    # Increment daily counter
    await _daily_increment(user_id, provider)

    # Log to DB
    from models import ImageJob
    async with get_session() as session:
        session.add(ImageJob(
            user_id=user_id,
            request_id=f"img_{user_id}_{int(datetime.now().timestamp())}",
            provider=provider,
            prompt_original=user_prompt,
            prompt_enhanced=enhanced,
            status="done",
            credits_deducted=0,  # credits deducted in handler
        ))
        await session.commit()

    return image_bytes, enhanced, provider
```

---

## Handler: `bot/handlers/draw.py`

```python
"""
bot/handlers/draw.py
/draw (.draw) — image generation command.
"""
from __future__ import annotations
import logging
import io

from aiogram import Router, F
from aiogram.filters import Command
from aiogram.types import Message, BufferedInputFile, InlineKeyboardMarkup, InlineKeyboardButton

import modules.codex_imagegen as imggen_mod
from modules.codex_config import get
from modules.codex_imagegen.exceptions import (
    DailyLimitError, NSFWBlockedError, ImageGenError
)
from bot.utils.helpers import get_user_tier
from bot.utils.logger import log_event, new_request_id
import modules.codex_credits as credits_mod

logger = logging.getLogger(__name__)
router = Router(name="draw")


def _pick_provider(tier: str) -> str:
    """
    Route to provider based on tier.
    Free tier → free_api only.
    Paid tiers → google (high quality).
    Can be overridden by user in future (Phase 3).
    """
    if tier == "free":
        return "free_api"
    return "google"


@router.message(Command("draw"))
async def cmd_draw(message: Message, db_user=None) -> None:
    if db_user is None or not db_user.is_registered:
        await message.answer(
            "⊘ <b>Register first</b>\n\nUse /register to create your account.",
            parse_mode="HTML",
        )
        return

    # Check feature flag
    if not await get("feature.image_gen", True):
        await message.answer(
            "⊘ <b>Image generation is currently disabled.</b>",
            parse_mode="HTML",
        )
        return

    # Parse prompt from command
    parts = message.text.split(maxsplit=1)
    if len(parts) < 2 or not parts[1].strip():
        await message.answer(
            "🎨 <b>Draw — Image Generation</b>\n"
            "━━━━━━━━━━━━━━━\n"
            "Usage:\n"
            "<code>/draw your image description here</code>\n\n"
            "Example:\n"
            "<code>/draw a cat sitting on the moon at night</code>",
            parse_mode="HTML",
        )
        return

    user_prompt = parts[1].strip()
    rid         = new_request_id()
    tier        = get_user_tier(db_user)
    provider    = _pick_provider(tier)

    # Credits check
    credits_needed = int(await get(f"imagegen.{provider}.credits_per_img", 0))
    if credits_needed > 0:
        balance = await credits_mod.get_balance(db_user.id)
        if balance < credits_needed:
            await message.answer(
                f"⊘ <b>Insufficient credits</b>\n\n"
                f"Image generation needs <code>{credits_needed}</code> credits.\n"
                f"Your balance: <code>{balance}</code>",
                parse_mode="HTML",
                reply_markup=InlineKeyboardMarkup(inline_keyboard=[[
                    InlineKeyboardButton(text="💳 Top Up", callback_data="buy:topup:open"),
                ]]),
            )
            return

    # Show thinking animation
    quality_label = "🎨 High Quality" if provider == "google" else "🖼 Standard"
    anim_msg = await message.answer(
        f"⠙ <b>Generating image...</b>\n"
        f"<i>{quality_label}</i>",
        parse_mode="HTML",
    )

    try:
        await anim_msg.edit_text(
            "⠸ <b>Enhancing your prompt...</b>", parse_mode="HTML"
        )

        image_bytes, enhanced_prompt, provider_used = await imggen_mod.generate_image(
            user_id=db_user.id,
            db_user=db_user,
            user_prompt=user_prompt,
            provider=provider,
            tier=tier,
        )

        await anim_msg.edit_text(
            "⠼ <b>Preparing your image...</b>", parse_mode="HTML"
        )

        # Deduct credits if needed
        if credits_needed > 0:
            await credits_mod.spend(
                user_id=db_user.id,
                amount=credits_needed,
                entry_type="image_gen",
                ref_id=rid,
                note=f"Image gen via {provider_used}",
            )

        # Delete animation message
        await anim_msg.delete()

        # Send image
        photo = BufferedInputFile(image_bytes, filename="codex_image.png")
        await message.answer_photo(
            photo=photo,
            caption=(
                f"🎨 <b>Generated</b>\n"
                f"━━━━━━━━━━━━━━━\n"
                f"<i>{enhanced_prompt[:200]}{'...' if len(enhanced_prompt) > 200 else ''}</i>"
            ),
            parse_mode="HTML",
        )

        log_event("image_generated", user_id=db_user.id, request_id=rid,
                  data={"provider": provider_used, "tier": tier,
                        "credits": credits_needed})

    except NSFWBlockedError:
        await anim_msg.edit_text(
            "⊘ <b>Prompt blocked</b>\n\n"
            "Aapka prompt inappropriate content detect hua hai.\n"
            "Please different prompt use karein.",
            parse_mode="HTML",
        )

    except DailyLimitError as e:
        await anim_msg.edit_text(
            f"⊘ <b>Daily limit reached</b>\n\n{e}\n\n"
            f"Upgrade karo zyada images ke liye:",
            parse_mode="HTML",
            reply_markup=InlineKeyboardMarkup(inline_keyboard=[[
                InlineKeyboardButton(text="💎 Go Premium", callback_data="buy:plan:list"),
            ]]),
        )

    except ImageGenError as e:
        await anim_msg.edit_text(
            f"⚠ <b>Generation failed</b>\n\n"
            f"<code>{e}</code>\n\n"
            f"Dobara try karo ya /support se contact karo.",
            parse_mode="HTML",
        )
        log_event("image_gen_failed", user_id=db_user.id, request_id=rid,
                  data={"error": str(e), "provider": provider})
```

---

## Exceptions: `exceptions.py`

```python
class ImageGenError(Exception):
    """Base image generation error."""

class NSFWBlockedError(ImageGenError):
    """Prompt blocked by NSFW filter."""

class DailyLimitError(ImageGenError):
    """User has hit their daily image generation limit."""
```

---

## ENV Var to Add

```env
# Google AI Studio
GOOGLE_API_KEY=           # from aistudio.google.com
```

## requirements.txt additions

```
google-generativeai==0.8.3
Pillow==11.0.0
```

---

## Alias + Command Map additions

Add to alias table:
```
/draw    .draw    → Image generation
```

Add to `_set_bot_commands()` in `main.py`:
```python
BotCommand(command="draw", description="Generate an image"),
```

Add to `bot/router.py`:
```python
from bot.handlers import draw
dp.include_router(draw.router)
```

Add to `_EXEMPT_COMMANDS` in `ratelimit.py`:
```python
"/draw",
```
(draw has its own daily limit system — not rolling window)

---

## Admin Dashboard Additions

Settings panel — new tab: **Image Gen**
All `imagegen.*` system_config keys editable live.

Logs filter — new event type: `image_generated`, `image_gen_failed`

---

## User Experience Summary

```
User: /draw a tiger in space
  → Bot: ⠙ Generating image...
  → Bot: ⠸ Enhancing your prompt...
  → Provider call (free_api or google based on tier)
  → Bot: ⠼ Preparing your image...
  → Bot: [photo sent with enhanced prompt caption]

Free tier:
  → free_api (standard quality)
  → 0 credits deducted
  → N images/day (from system_config)

Paid tier:
  → Google Imagen 3 (high quality)
  → credits_per_img deducted
  → N images/day (from system_config)

NSFW attempt:
  → Blocked before API call
  → Clean error message

Daily limit hit:
  → Error + upgrade prompt
```

---

# ═══════════════════════════════════
# PHASE 2 BUGS — ADD HERE AS FOUND
# ═══════════════════════════════════

## FORMAT:
```
## BUG N — Short description
**Severity:** CRITICAL / HIGH / MEDIUM / LOW
**File:** path/to/file.py
**Problem:** What is broken and why.

FIND: [broken code]
REPLACE WITH: [fixed code]
```

---

*Phase 1 bugs: 6 logged — all fixes documented*
*Image gen feature: fully specced*
*Phase 2 bugs: 0 logged — add as found*

---

# ═══════════════════════════════════════════════════
# PHASE 2 BUGS — EXACT FIXES
# ═══════════════════════════════════════════════════

---

## BUG P2-1 — is_admin missing await (CRITICAL)
**Severity:** CRITICAL — Security issue. Koi bhi user group approve/reject kar sakta hai.
**File:** `bot/handlers/groups.py`
**Lines:** 193, 223
**Problem:** `is_admin()` async function hai lekin `await` nahi laga.
Coroutine object return hota hai jo Python mein truthy hota hai.
Matlab condition hamesha pass ho jaati hai — koi bhi admin ban jaata hai.

**FIND line 193:**
```python
    if not is_admin(callback.from_user.id):
        await callback.answer("⊘ Admin only.", show_alert=True)
        return
```

**REPLACE WITH:**
```python
    if not await is_admin(callback.from_user.id):
        await callback.answer("⊘ Admin only.", show_alert=True)
        return
```

**FIND line 223:**
```python
    if not is_admin(callback.from_user.id):
        await callback.answer("⊘ Admin only.", show_alert=True)
        return
```

**REPLACE WITH:**
```python
    if not await is_admin(callback.from_user.id):
        await callback.answer("⊘ Admin only.", show_alert=True)
        return
```

**Safe? YES** — Sirf `await` add ho raha hai. Logic same hai. Koi side effect nahi.

---

## BUG P2-2 — plan_id hardcode in codex_files (MEDIUM)
**Severity:** MEDIUM — Naya plan add karo to free/paid detection break hoga.
**Files:**
  - `modules/codex_files/core.py` line 112-113
  - `bot/handlers/files.py` line 263

**Problem:** `plan_id != 1` se free/paid detect karna hardcoded assumption hai.
Plan table mein free plan ka ID hamesha 1 nahi hoga agar naya structure aaye.

---

### Fix A — `modules/codex_files/core.py`

**FIND (lines 112-115):**
```python
async def check_user_file_size(user_id: int, file_size_bytes: int, db_user) -> bool:
    """Return True if file is within user's plan size cap."""
    plan_id = getattr(db_user, "plan_id", 1) or 1
    key     = "file.sub.max_mb" if plan_id != 1 else "file.free.max_mb"
    max_mb  = int(await get(key, 20))
    return file_size_bytes <= max_mb * 1024 * 1024
```

**REPLACE WITH:**
```python
async def check_user_file_size(user_id: int, file_size_bytes: int, db_user) -> bool:
    """Return True if file is within user's plan size cap."""
    from models import Plan
    from sqlalchemy import select

    is_free = True  # default to free (fail-safe)
    plan_id = getattr(db_user, "plan_id", None)

    if plan_id:
        async with get_session() as session:
            result = await session.execute(
                select(Plan.plan_type).where(Plan.id == plan_id)
            )
            plan_type = result.scalar_one_or_none()
        is_free = (plan_type is None or plan_type == "free")

    key    = "file.free.max_mb" if is_free else "file.sub.max_mb"
    max_mb = int(await get(key, 20))
    return file_size_bytes <= max_mb * 1024 * 1024
```

---

### Fix B — `bot/handlers/files.py`

**FIND (line 263):**
```python
    tier_key = "file.sub.max_mb" if (db_user.plan_id and db_user.plan_id != 1) else "file.free.max_mb"
    limit_mb = int(await get(tier_key, 20))
```

**REPLACE WITH:**
```python
    # Resolve plan type from DB — never hardcode plan_id comparison
    from models import Plan
    from sqlalchemy import select
    from modules.codex_config import get_session

    _is_free = True  # default fail-safe
    if db_user.plan_id:
        async with get_session() as session:
            _res = await session.execute(
                select(Plan.plan_type).where(Plan.id == db_user.plan_id)
            )
            _plan_type = _res.scalar_one_or_none()
        _is_free = (_plan_type is None or _plan_type == "free")

    tier_key = "file.free.max_mb" if _is_free else "file.sub.max_mb"
    limit_mb = int(await get(tier_key, 20))
```

**Safe? YES** — Logic same hai (free vs paid). Sirf DB se plan_type check karna hai. Default `is_free=True` fail-safe hai — worst case free limit milega, crash nahi.

---

## BUG P2-3 — payment.py hardcoded plan_map (MEDIUM)
**Severity:** MEDIUM — Naya plan add karo to payment fail ho jayega.
**File:** `bot/handlers/payment.py`
**Lines:** ~119-120 (inside `cb_chain_select`)

**FIND:**
```python
    plan_slug = (await redis.get(f"buy_plan:{callback.from_user.id}") or b"pro_monthly").decode()
    plan_map  = {"pro_monthly": 2, "pro_yearly": 3}
    plan_id   = plan_map.get(plan_slug, 2)

    try:
        invoice = await pay_mod.create_invoice(
            user_id=callback.from_user.id,
            plan_id=plan_id,
            chain=chain,
        )
```

**REPLACE WITH:**
```python
    plan_slug = (await redis.get(f"buy_plan:{callback.from_user.id}") or b"").decode()

    if not plan_slug:
        await callback.answer("Session expired. Start again with /buy.", show_alert=True)
        return

    # Resolve plan_id from DB by slug — never hardcode
    from models import Plan
    from sqlalchemy import select
    from modules.codex_config import get_session

    async with get_session() as session:
        result = await session.execute(
            select(Plan).where(
                Plan.slug == plan_slug,
                Plan.active == True,
            )
        )
        plan_obj = result.scalar_one_or_none()

    if not plan_obj:
        await callback.answer(
            "Plan not found. Contact support.", show_alert=True
        )
        return

    plan_id = plan_obj.id

    try:
        invoice = await pay_mod.create_invoice(
            user_id=callback.from_user.id,
            plan_id=plan_id,
            chain=chain,
        )
```

**Safe? YES** — Agar plan_slug invalid hai to early return hoga — crash nahi. Existing plans ke liye DB se same ID aayega jo pehle hardcoded tha.

---

## BUG P2-4 — admin.py /import hardcoded plan_id=2 (MEDIUM)
**Severity:** MEDIUM — Sab SellAuth coupons hamesha Pro Monthly ke liye import hote hain.
**File:** `bot/handlers/admin.py`

**Two parts fix karne hain:**

---

### Fix Part A — Confirmation message mein plan selection add karo

**FIND (lines ~465-472 — where codes stored in Redis):**
```python
    # Store codes temporarily
    import json
    await redis.setex(f"admin_import_codes:{message.from_user.id}", 300, json.dumps(codes))

    await message.answer(
        f"◈ <b>Confirm import</b>\n\n{len(codes)} codes to import\nPlan: Pro Monthly",
        parse_mode="HTML", reply_markup=kb,
    )
```

**REPLACE WITH:**
```python
    # Store codes temporarily
    import json
    await redis.setex(
        f"admin_import_codes:{message.from_user.id}", 300, json.dumps(codes)
    )

    # Fetch available plans for selection
    from models import Plan
    from sqlalchemy import select
    from modules.codex_config import get_session

    async with get_session() as session:
        result = await session.execute(
            select(Plan).where(
                Plan.active == True,
                Plan.plan_type != "free",
            ).order_by(Plan.id)
        )
        plans = result.scalars().all()

    if not plans:
        await message.answer(
            "<b>⊘ No active plans found.</b> Add plans first.",
            parse_mode="HTML",
        )
        return

    plan_buttons = [
        [InlineKeyboardButton(
            text=p.name,
            callback_data=f"import:plan:{p.id}",
        )]
        for p in plans
    ]
    plan_buttons.append([
        InlineKeyboardButton(text="✗ Cancel", callback_data="admin:cancel"),
    ])

    await message.answer(
        f"◈ <b>Select plan for import</b>\n\n"
        f"<code>{len(codes)}</code> codes parsed.\n"
        f"Kaunse plan ke liye import karein?",
        parse_mode="HTML",
        reply_markup=InlineKeyboardMarkup(inline_keyboard=plan_buttons),
    )
```

---

### Fix Part B — Old `import:confirm:` callback hatao, new `import:plan:` callback add karo

**DELETE this entire callback handler:**
```python
@router.callback_query(F.data.startswith("import:confirm:"))
async def cb_import_confirm(callback: CallbackQuery) -> None:
    if not await is_admin(callback.from_user.id):
        await callback.answer("Access denied", show_alert=True)
        return

    import json
    from modules.codex_config import get_redis
    redis = await get_redis()
    raw   = await redis.get(f"admin_import_codes:{callback.from_user.id}")

    if not raw:
        await callback.answer("Import session expired", show_alert=True)
        return

    codes = json.loads(raw)
    await redis.delete(f"admin_import_codes:{callback.from_user.id}")

    inserted, skipped = await coupon_mod.bulk_create(
        codes=codes, plan_id=2, source="sellauth", created_by=callback.from_user.id,
    )

    await callback.message.edit_text(
        f"✔ <b>Import complete</b>\n\n"
        f"Inserted: <code>{inserted}</code>\nSkipped: <code>{skipped}</code>",
        parse_mode="HTML",
    )
```

**ADD this new handler in its place:**
```python
@router.callback_query(F.data.startswith("import:plan:"))
async def cb_import_plan_selected(callback: CallbackQuery) -> None:
    if not await is_admin(callback.from_user.id):
        await callback.answer("Access denied", show_alert=True)
        return

    plan_id = int(callback.data.split(":")[-1])

    import json
    from modules.codex_config import get_redis
    redis = await get_redis()
    raw   = await redis.get(f"admin_import_codes:{callback.from_user.id}")

    if not raw:
        await callback.answer("Import session expired. Start again.", show_alert=True)
        return

    codes = json.loads(raw)
    await redis.delete(f"admin_import_codes:{callback.from_user.id}")

    inserted, skipped = await coupon_mod.bulk_create(
        codes=codes,
        plan_id=plan_id,   # ← DB se selected plan, not hardcoded
        source="sellauth",
        created_by=callback.from_user.id,
    )

    await callback.message.edit_text(
        f"✔ <b>Import complete</b>\n\n"
        f"Inserted: <code>{inserted}</code>\n"
        f"Skipped:  <code>{skipped}</code>",
        parse_mode="HTML",
    )
    await callback.answer()
```

**Safe? YES** — Old callback hata ke naya daala. Agar koi purana session Redis mein tha (import:confirm: wala) — wo expire ho jayega 5 min mein automatically (TTL 300s already set hai).

---

*Phase 2 bugs: 4 logged — all fixes documented with exact code*

---

# ═══════════════════════════════════════════════════
# PRD GAP FIXES — PHASE 2.5
# ═══════════════════════════════════════════════════

---

## GAP 2 — Multi-group owner ambiguity fix
**File:** `bot/handlers/groups.py`
**Problem:** `_resolve_owner_group()` latest group return karta hai.
Agar owner ke 2 groups hain to `/gban` kaunse group mein chalega?
Ambiguous hai. Bot PM mein koi way nahi group identify karne ka cleanly.

**Solution:** Sab owner commands GROUP mein hi chalenge.
DM se chalao to "group mein jao" message aata hai.
Group context mein `message.chat.id` se exact group identify hota hai.
Multi-group ambiguity completely solve.

---

### Step 1 — `_resolve_owner_group()` replace karo

**DELETE this entire function:**
```python
async def _resolve_owner_group(message: Message) -> object | None:
    """Get group where this user is owner. DM context."""
    from modules.codex_config import get_session
    from models import Group
    from sqlalchemy import select
    async with get_session() as session:
        res = await session.execute(
            select(Group).where(Group.owner_tg_id == message.from_user.id)
            .order_by(Group.created_at.desc())
        )
        return res.scalars().first()
```

**REPLACE WITH these two functions:**
```python
_GROUP_COMMANDS_MSG = (
    "<b>⊘ Ye command group mein use karo</b>\n\n"
    "Owner commands sirf group context mein kaam karte hain.\n"
    "Apne group mein jao aur wahan command chalao.\n\n"
    "<i>Example: Group mein /gstatus type karo</i>"
)


async def _resolve_group_from_chat(message: Message) -> object | None:
    """
    Get group record from current chat context.
    Returns Group ORM object or None.
    Handles all error cases internally.

    Rules:
      - Command must be sent IN a group/supergroup chat.
      - Sender must be the owner (owner_tg_id match).
      - Group must exist in DB.
      - Group must be active or trial.
    """
    # Must be in group context
    if message.chat.type not in ("group", "supergroup"):
        await message.answer(_GROUP_COMMANDS_MSG, parse_mode="HTML")
        return None

    tg_group_id = message.chat.id
    user_tg_id  = message.from_user.id

    from modules.codex_config import get_session
    from models import Group
    from sqlalchemy import select

    try:
        async with get_session() as session:
            res = await session.execute(
                select(Group).where(Group.tg_group_id == tg_group_id)
            )
            group = res.scalar_one_or_none()
    except Exception as e:
        await message.answer(
            "<b>⚠ DB error.</b> Dobara try karo.",
            parse_mode="HTML",
        )
        return None

    # Group not in DB
    if not group:
        await message.answer(
            "<b>⊘ Ye group registered nahi hai.</b>\n\n"
            "Bot ko hatao aur dobara add karo activation ke liye.",
            parse_mode="HTML",
        )
        return None

    # Ownership check
    if group.owner_tg_id != user_tg_id:
        await message.answer(
            "<b>⊘ Sirf group owner ye command use kar sakta hai.</b>",
            parse_mode="HTML",
        )
        return None

    # Status check
    if group.status not in ("active", "trial"):
        status_msgs = {
            "pending":   "Group abhi pending approval mein hai.",
            "expired":   "Group plan expire ho gaya hai. /gplan se renew karo.",
            "suspended": "Group suspended hai. Support se contact karo.",
            "removed":   "Bot group se remove ho gaya tha. Dobara add karo.",
            "rejected":  "Group activation reject hui thi. Support se contact karo.",
        }
        detail = status_msgs.get(group.status, f"Group status: {group.status}")
        await message.answer(
            f"<b>⊘ Group active nahi hai</b>\n\n{detail}",
            parse_mode="HTML",
        )
        return None

    return group


async def _dm_only_warning(message: Message, command: str) -> None:
    """
    For sensitive commands (/gban /gunban /glimit /gmembers)
    that should NOT be visible in group chat for privacy.
    User runs them in group, bot replies via DM.

    NOTE: Bot cannot DM a user who hasn't started it.
    So we reply in group with a soft warning and delete the command message.
    """
    # Delete the command message from group (privacy)
    try:
        await message.delete()
    except Exception:
        pass  # bot may not have delete permission — ignore

    # Try to DM the user
    try:
        await message.bot.send_message(
            chat_id=message.from_user.id,
            text=(
                "<b>🔒 Sensitive command</b>\n\n"
                f"<code>{command}</code> ka result yahan (DM mein) deliver kiya jayega.\n"
                "Agar DM band hai to pehle bot ko /start karo."
            ),
            parse_mode="HTML",
        )
    except Exception:
        # Cannot DM — user hasn't started bot. Reply in group briefly.
        await message.answer(
            f"<b>🔒 {command}</b> — Pehle bot ko DM mein /start karo, tab ye command chalao.",
            parse_mode="HTML",
        )
```

---

### Step 2 — Sab owner command handlers update karo

**REPLACE `/gstatus` handler:**
```python
@router.message(Command("gstatus"))
async def cmd_gstatus(message: Message, db_user=None) -> None:
    group = await _resolve_group_from_chat(message)
    if not group:
        return  # error already sent inside _resolve_group_from_chat

    try:
        from modules.codex_groups import get_group_stats
        stats = await get_group_stats(group.id)
    except Exception as e:
        await message.answer(
            f"<b>⚠ Stats load nahi hui:</b> <code>{e}</code>",
            parse_mode="HTML",
        )
        return

    expires = stats.get("plan_expires_at")
    exp_str = expires.strftime("%d %b %Y") if expires else "N/A"

    await message.answer(
        f"◈ <b>Group Status</b>\n"
        f"━━━━━━━━━━━━━━━\n"
        f"Group:   <b>{stats['group_name']}</b>\n"
        f"Status:  <b>{stats['status'].upper()}</b>\n"
        f"Plan:    <b>{stats['plan_name']}</b>\n"
        f"Members: <code>{stats['member_count']}</code>\n"
        f"Expires: <code>{exp_str}</code>\n"
        f"━━━━━━━━━━━━━━━",
        parse_mode="HTML",
    )
```

**REPLACE `/gusage` handler:**
```python
@router.message(Command("gusage"))
async def cmd_gusage(message: Message, db_user=None) -> None:
    group = await _resolve_group_from_chat(message)
    if not group:
        return

    try:
        from modules.codex_groups import get_group_stats
        stats = await get_group_stats(group.id)
    except Exception as e:
        await message.answer(
            f"<b>⚠ Usage load nahi hua:</b> <code>{e}</code>",
            parse_mode="HTML",
        )
        return

    await message.answer(
        f"📊 <b>Group Usage</b>\n"
        f"━━━━━━━━━━━━━━━\n"
        f"Requests today: <code>{stats['requests_today']}</code>\n"
        f"Members:        <code>{stats['member_count']}</code>\n"
        f"━━━━━━━━━━━━━━━",
        parse_mode="HTML",
    )
```

**REPLACE `/gplan` handler:**
```python
@router.message(Command("gplan"))
async def cmd_gplan(message: Message, db_user=None) -> None:
    group = await _resolve_group_from_chat(message)
    if not group:
        return

    try:
        from modules.codex_groups import get_group_stats
        stats = await get_group_stats(group.id)
    except Exception as e:
        await message.answer(
            f"<b>⚠ Plan info load nahi hui:</b> <code>{e}</code>",
            parse_mode="HTML",
        )
        return

    kb = InlineKeyboardMarkup(inline_keyboard=[[
        InlineKeyboardButton(
            text="🚀 Upgrade plan",
            callback_data=f"group:upgrade:{group.id}",
        ),
    ]])
    await message.answer(
        f"💎 <b>Group Plan</b>\n"
        f"━━━━━━━━━━━━━━━\n"
        f"Plan:   <b>{stats['plan_name']}</b>\n"
        f"Status: <b>{stats['status']}</b>\n"
        f"━━━━━━━━━━━━━━━",
        parse_mode="HTML",
        reply_markup=kb,
    )
```

**REPLACE `/ghelp` handler:**
```python
@router.message(Command("ghelp"))
async def cmd_ghelp(message: Message, db_user=None) -> None:
    # ghelp works anywhere — no group resolve needed
    await message.answer(
        "<b>👑 Group Owner Commands</b>\n"
        "━━━━━━━━━━━━━━━\n"
        "<b>Group mein chalao:</b>\n"
        "/gstatus — Group plan + status\n"
        "/gusage  — Usage stats\n"
        "/gplan   — Current plan + upgrade\n"
        "/ghelp   — Ye list\n"
        "/gban {user_id}       — Member ban\n"
        "/gunban {user_id}     — Member unban\n"
        "/glimit {user_id} {N} — Custom limit\n"
        "/gmembers             — Member list\n"
        "━━━━━━━━━━━━━━━\n"
        "<i>Sab commands group context mein chalate hain.</i>",
        parse_mode="HTML",
    )
```

**REPLACE `/gban` handler:**
```python
@router.message(Command("gban"))
async def cmd_gban(message: Message, db_user=None) -> None:
    group = await _resolve_group_from_chat(message)
    if not group:
        return

    args = (message.text or "").split()[1:]
    if not args:
        await message.answer(
            "<b>Usage:</b> /gban {user_id}",
            parse_mode="HTML",
        )
        return

    try:
        target_tg_id = int(args[0].lstrip("@"))
    except ValueError:
        await message.answer(
            "<b>⊘ Invalid user ID.</b>\n"
            "<b>Usage:</b> /gban {user_id}",
            parse_mode="HTML",
        )
        return

    # Prevent owner banning themselves
    if target_tg_id == message.from_user.id:
        await message.answer(
            "<b>⊘ Aap khud ko ban nahi kar sakte.</b>",
            parse_mode="HTML",
        )
        return

    try:
        from modules.codex_groups import ban_member
        await ban_member(group.id, target_tg_id, message.from_user.id)
        await message.answer(
            f"✔ <b>User <code>{target_tg_id}</code> banned</b> from group bot.",
            parse_mode="HTML",
        )
    except PermissionError:
        await message.answer(
            "<b>⊘ Aap is group ke owner nahi hain.</b>",
            parse_mode="HTML",
        )
    except Exception as e:
        await message.answer(
            f"<b>⚠ Ban failed:</b> <code>{e}</code>",
            parse_mode="HTML",
        )
```

**REPLACE `/gunban` handler:**
```python
@router.message(Command("gunban"))
async def cmd_gunban(message: Message, db_user=None) -> None:
    group = await _resolve_group_from_chat(message)
    if not group:
        return

    args = (message.text or "").split()[1:]
    if not args:
        await message.answer(
            "<b>Usage:</b> /gunban {user_id}",
            parse_mode="HTML",
        )
        return

    try:
        target_tg_id = int(args[0].lstrip("@"))
    except ValueError:
        await message.answer(
            "<b>⊘ Invalid user ID.</b>\n"
            "<b>Usage:</b> /gunban {user_id}",
            parse_mode="HTML",
        )
        return

    try:
        from modules.codex_groups import unban_member
        await unban_member(group.id, target_tg_id, message.from_user.id)
        await message.answer(
            f"✔ <b>User <code>{target_tg_id}</code> unbanned.</b>",
            parse_mode="HTML",
        )
    except PermissionError:
        await message.answer(
            "<b>⊘ Aap is group ke owner nahi hain.</b>",
            parse_mode="HTML",
        )
    except Exception as e:
        await message.answer(
            f"<b>⚠ Unban failed:</b> <code>{e}</code>",
            parse_mode="HTML",
        )
```

**REPLACE `/glimit` handler:**
```python
@router.message(Command("glimit"))
async def cmd_glimit(message: Message, db_user=None) -> None:
    group = await _resolve_group_from_chat(message)
    if not group:
        return

    args = (message.text or "").split()[1:]
    if len(args) < 2:
        await message.answer(
            "<b>Usage:</b> /glimit {user_id} {limit}",
            parse_mode="HTML",
        )
        return

    try:
        target_tg_id = int(args[0])
        limit        = int(args[1])
    except ValueError:
        await message.answer(
            "<b>⊘ Invalid format.</b>\n"
            "<b>Usage:</b> /glimit {user_id} {limit}",
            parse_mode="HTML",
        )
        return

    if limit < 0:
        await message.answer(
            "<b>⊘ Limit 0 ya usse zyada honi chahiye.</b>",
            parse_mode="HTML",
        )
        return

    try:
        from modules.codex_groups import set_member_limit
        await set_member_limit(group.id, target_tg_id, message.from_user.id, limit)
        await message.answer(
            f"✔ <b>Limit set</b>\n"
            f"User: <code>{target_tg_id}</code>\n"
            f"Limit: <code>{limit}</code> requests/window",
            parse_mode="HTML",
        )
    except PermissionError:
        await message.answer(
            "<b>⊘ Aap is group ke owner nahi hain.</b>",
            parse_mode="HTML",
        )
    except Exception as e:
        await message.answer(
            f"<b>⚠ Limit set failed:</b> <code>{e}</code>",
            parse_mode="HTML",
        )
```

**REPLACE `/gmembers` handler:**
```python
@router.message(Command("gmembers"))
async def cmd_gmembers(message: Message, db_user=None) -> None:
    group = await _resolve_group_from_chat(message)
    if not group:
        return

    try:
        from modules.codex_config import get_session
        from models import GroupMember
        from sqlalchemy import select

        async with get_session() as session:
            res = await session.execute(
                select(GroupMember)
                .where(GroupMember.group_id == group.id)
                .order_by(GroupMember.joined_at.desc())
                .limit(20)
            )
            members = res.scalars().all()
    except Exception as e:
        await message.answer(
            f"<b>⚠ Members load nahi hue:</b> <code>{e}</code>",
            parse_mode="HTML",
        )
        return

    if not members:
        await message.answer(
            "<b>📋 Abhi koi member nahi hai.</b>\n"
            "<i>Members tab dikhenge jab group mein bot use karein.</i>",
            parse_mode="HTML",
        )
        return

    lines = [f"<b>👥 Members ({len(members)})</b>\n━━━━━━━━━━━━━━━"]
    for m in members:
        status = " ⛔" if m.is_banned else ""
        limit  = f" (limit:{m.custom_limit})" if m.custom_limit else ""
        lines.append(f"<code>{m.user_tg_id}</code>{status}{limit}")
    lines.append("━━━━━━━━━━━━━━━")

    await message.answer("\n".join(lines), parse_mode="HTML")
```

---

### Step 3 — `/ghelp` bot commands update karo

`main.py` mein `/ghelp` description update karo:
```python
BotCommand(command="ghelp", description="Group owner commands (group mein chalao)"),
```

---

**Safe? YES — Complete analysis:**

| Case | Kya hoga |
|------|----------|
| DM se command | "Group mein jao" message — koi crash nahi |
| Group mein galat owner | "Sirf owner use kar sakta" — koi crash nahi |
| Group not in DB | "Registered nahi" — koi crash nahi |
| Group expired/suspended | Status-specific message — koi crash nahi |
| DB error | Generic error message — koi crash nahi |
| Self-ban attempt | "Khud ko ban nahi kar sakte" — koi crash nahi |
| Invalid user_id | "Invalid format" — koi crash nahi |
| Negative limit | "0 ya zyada honi chahiye" — koi crash nahi |

---

## GAP 3 — Redis Package
**Status: NO ISSUE — No fix needed.**

`redis==5.0.4` mein `redis.asyncio` built-in submodule hota hai.
Ye `aioredis` ka official successor hai (2022 mein merge hua).
Code mein `import redis.asyncio as aioredis` correctly use ho raha hai.
`requirements.txt` change karne ki zaroorat nahi.

---

*Phase 2.5 complete: 4 bugs + 2 PRD gaps — all fixed*
