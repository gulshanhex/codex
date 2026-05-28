# CODEX — Product Requirements Document
## Version 1.9 · Phase 2.5 + Image Generation

---

## 1. Product Overview

**Codex** is a production-ready Telegram-based AI automation platform offering:

- Freemium AI chat and agentic workflows
- Subscription + top-up billing (SellAuth / USDT / Manual UPI via DM)
- Coupon-based access control (JHATKA format)
- File processing (zip, unzip, merge, extract) with 48hr R2 storage lifecycle
- **Image generation — `/draw` command (free API for free tier, Google Imagen 3 for paid)**
- Referral growth engine
- Admin panel with full visibility — both Telegram + dedicated web dashboard
- Conversational wizard UX for all multi-step commands
- 6-hour rolling rate limit window (not midnight reset)

**Target:** Public users on Telegram  
**Initial scale target:** 7,000–10,000 total users, ~1,000 paid users  
**Deployment:** Single VPS, Python monolith, scale-out later

---

## 2. Objective

Launch a monetizable, abuse-resistant Telegram AI bot that:
1. Keeps infrastructure cost near zero at early scale
2. Converts free users to paid via clear premium value
3. Handles crypto payments natively without custom exchange logic
4. Routes AI intelligently to protect per-request cost
5. Is maintainable by one developer solo

---

## 3. Users & Personas

| Persona | Description | Primary Goal |
|---|---|---|
| Free User | Telegram user, light usage | Quick AI help, try features |
| Paid Subscriber | Power user, monthly/yearly plan | Higher limits, agent mode |
| Top-up User | Occasional heavy user | Pay per need, no commitment |
| Crypto User | USDT payer | Fast activation, no friction |
| Admin | Developer / founder | Visibility, control, support |

---

## 4. Core Problems Being Solved

1. AI API cost uncontrolled without routing + gating
2. Crypto users have no direct payment path
3. Coupon sales (SellAuth) need bot-side redemption
4. Manual support is slow without structured admin tools
5. File operations (zip/unzip) should not call AI
6. Referral abuse inflates cost without fraud controls

---

## 5. Success Criteria

- Phase 1 live within 4 weeks of dev start
- <$50/month infra cost at 1,000 DAU
- AI cost per active free user < $0.01/day
- Zero crashes from provider failure (graceful fallback)
- Paid conversion rate ≥ 5% of active free users

---

## 6. Must-Have Requirements (LOCKED)

### 6.1 Bot Core
- [ ] /start onboarding with main menu
- [ ] HTML parse mode for all messages
- [ ] Unicode symbols + animations for premium UI feel
- [ ] Reply buttons for navigation, inline buttons for actions (see Section 30)
- [ ] Dot (.) aliases for all commands — `.gen` = `/gen` etc. (see Section 29)
- [ ] Conversational wizard mode — command → step-by-step bot prompts (see Section 35)
- [ ] Power-user shortcut — command with inline params also supported
- [ ] Chat Mode (Nvidia NIM) + Agent Mode (FreeModel via LangGraph)
- [ ] Manual mode switch by user (not auto-routed)
- [ ] 6-hour rolling rate limit window per plan tier (see Section 36)
- [ ] Typing indicator before AI response, upload_document before file delivery

### 6.2 AI Routing
- [ ] Layer 0: Local deterministic handlers (time, credits, referral, file ops)
- [ ] Layer 1: Nvidia NIM — normal chat
- [ ] Layer 2: FreeModel — coding + agentic tasks
- [ ] Layer 3: LangGraph deep agent (premium only)
- [ ] Graceful fallback if provider fails

### 6.3 Payment System
- [ ] SellAuth integration — BTC/LTC coupon sales
- [ ] USDT direct activation — TRC20 (Tronscan) + ERC20 (Etherscan free tier)
- [ ] UPI — support DM redirect only (no bot-side UPI flow, see Section 37)
- [ ] 10 USDT wallet rotation (round-robin: 5 TRC20 + 5 ERC20)
- [ ] TRC20 tx verify: `GET https://apilist.tronscan.org/api/transaction-info?hash={tx_hash}`
- [ ] ERC20 tx verify: Etherscan free API
- [ ] Both verifiers isolated in separate adapter modules
- [ ] Tx hash verification (1 confirmation minimum, exact amount match)
- [ ] Fixed invoice amounts (no variable USDT)
- [ ] Invoice TTL: 30 min (Redis key)

### 6.4 Coupon System
- [ ] Format: `JHATKA-XXXXXXXXXXXXXXXX-CODEX`
- [ ] Cryptographically secure 16-char alphanumeric middle
- [ ] Sources: sellauth / manual / usdt
- [ ] Status lifecycle: available → assigned → redeemed (+ expired, disabled)
- [ ] No hard delete — status-only updates
- [ ] Admin `/gen` (`.gen`) command for manual generation
- [ ] Admin `/import` for SellAuth codes
- [ ] Coupon confirm screen with inline buttons before activation

### 6.5 Plans & Entitlements
- [ ] Free tier with daily usage cap
- [ ] Pro Monthly subscription
- [ ] Pro Yearly subscription
- [ ] Top-up credits (fixed + custom)
- [ ] Agent Mode gated behind premium or credit threshold

### 6.6 File Operations
- [ ] /zip, /unzip, /merge, /extract — local code only (no AI)
- [ ] Queue-based worker for heavy jobs
- [ ] Isolated temp: `/tmp/codex-jobs/{job_id}/input/` · `/work/` · `/output/`
- [ ] Auto cleanup of temp folder after upload (failure pe bhi)
- [ ] **Storage: Cloudflare R2 only** — pCloud removed from active delivery path
- [ ] pCloud adapter kept as stub module for future optional switch (auth/config ready, not active)
- [ ] File lifecycle: deliver → R2 store → 24hr warning → 48hr delete
- [ ] Scheduled Celery beat job handles 48hr deletion
- [ ] `files` table tracks: `r2_key`, `expires_at`, `warned_at`, `deleted_at`
- [ ] Per-user file size cap configurable from admin panel
- [ ] Worker handles all jobs (never in webhook loop)
- [ ] Per-user concurrency cap: max 2 jobs simultaneously

### 6.7 Referral System
- [ ] Unique referral link per user
- [ ] `/start ref_CODE` param parsing with self-referral block
- [ ] Reward on activation, not just signup
- [ ] Anti-fraud: no self-referral, cooldowns, caps
- [ ] Tiered reward thresholds (Bronze / Silver / Gold)

### 6.8 Admin Panel — Full Control Principle (LOCKED)

**Core rule:** Only infrastructure credentials live in env/config (bot token, DB URL, Redis URL, R2 keys, AI API keys, TG IDs for bootstrap). **Everything else is configurable from the admin web dashboard.** No feature, price, limit, message template, or setting should require a code deploy to change.

- [ ] Telegram: admin bot commands for quick actions (Section 11)
- [ ] **Web dashboard** — full control center, see Section 38 for complete spec
- [ ] Material Icons everywhere in web dashboard (no emojis), mobile + PC responsive
- [ ] All plans: create, edit, delete, price, credits, duration — from dashboard
- [ ] All rate limits and windows — from dashboard
- [ ] All file policies (size caps, TTL, warn timing) — from dashboard
- [ ] All message templates (welcome, error, rate limit, broadcast, etc.) — from dashboard
- [ ] All referral rules (bonus amounts, thresholds, tiers) — from dashboard
- [ ] All abuse thresholds — from dashboard
- [ ] All AI routing config (which model for which task) — from dashboard
- [ ] All payment settings (USDT wallets, amounts, chains) — from dashboard
- [ ] Feature flags (enable/disable any feature per plan tier) — from dashboard
- [ ] Bot Telegram credentials (token, chat IDs, etc.) — from dashboard (after bootstrap)
- [ ] Broadcast: create, preview, send, schedule, **delete** (within 48hr Telegram window)
- [ ] Full user CRUD: view, ban, grant, revoke, change plan, reset limits
- [ ] Coupon management: gen, bulk import, list, filter, disable, expire
- [ ] Admin action log: every change timestamped and attributed
- [ ] Admin Telegram commands scoped — hidden from general user command menu

### 6.9 Logging & Observability
- [ ] Every important action logged with request_id
- [ ] Usage events per request (model, latency, cost estimate)
- [ ] Error logs with reproduction context
- [ ] Admin action audit trail (JSON format)
- [ ] GET /health endpoint: db + redis + worker status

---

## 7. Should-Have Requirements

- Real-time Unicode animation stages during jobs (message edit loop)
- Queue position display for waiting users
- Premium badge in UI (`⟨ PRO ⟩`)
- Referral dashboard in bot
- Usage dashboard per user (/usage command)
- Timezone-aware time display with "estimated" label when unconfirmed
- Language preference setting
- Inline pagination for lists (10 per page)

---

## 8. Nice-to-Have Requirements

- Team plans (Phase 3)
- Priority queue for premium users
- Advanced analytics dashboard
- Automated abuse score system
- SellAuth webhook for instant coupon import (vs manual import)

---

## 9. Constraints

- Single VPS initially — no load balancer
- Python only — no polyglot services
- Must handle graceful restart without data loss
- File ops must not block the Telegram webhook loop
- R2 is the only active file storage layer (no local permanent storage)
- pCloud adapter is stubbed — not in active delivery path, switch via system_config
- No microservices until Phase 3 scale justifies it
- Budget: infra must stay under $50/month at 1K DAU

---

## 10. Non-Goals / Out of Scope

- Mobile app
- Custom exchange / fiat payment processor
- Voice/video message handling
- Multi-bot architecture (Phase 1–2)
- Auto-routing between Chat/Agent mode

---

## 11. Screen & Command Map

### User Commands + Dot Aliases
```
/start    .start    → Bot info + register prompt (no access)
/register .register → Full registration — grants access + credits
/help     .help     → Help screen
/menu     .menu     → Main menu
/usage    .usage    → Usage stats
/referral .ref      → Referral link + stats
/buy      .buy      → Purchase flow
/premium  .pro      → Premium info + upgrade
/support  .support  → Support contact
/settings .set      → Settings menu
/language .lang     → Language preference
/status   .status   → Bot + account status
/mode     .mode     → Switch Chat ↔ Agent
/agent    .agent    → Enter Agent Mode
/chat     .chat     → Enter Chat Mode
/time     .time     → Current time
/timezone .tz       → Set timezone
/credits  .credits  → Credit balance
/files    .files    → File manager
/queue    .q        → Queue status
/zip      .zip      → Zip files
/unzip    .unzip    → Unzip file
/merge    .merge    → Merge files
/extract  .extract  → Extract archive
/draw     .draw     → Generate image
/redeem   .redeem   → Redeem coupon
/topup    .topup    → Top up credits
```

### Group Owner Commands (ALL run in GROUP context — not DM)
```
/gstatus  .gstatus   → Group plan status + usage
/gusage   .gusage    → Group usage stats
/gplan    .gplan     → Current group plan + upgrade
/ghelp    .ghelp     → Group owner command list
/gban     .gban      → Ban member from bot in group
/gunban   .gunban    → Unban member
/glimit   .glimit    → Set per-member limit override
/gmembers .gmembers  → List members + usage
/gcancel  .gcancel   → Cancel active wizard step

RULE: All owner commands require group chat context.
DM attempt → bot replies "Group mein jao" + group list.
Multi-group owners: command runs on the group where it's sent.
Sensitive output (member lists, ban results) — bot replies
in group. No DM-only commands.
```

### Admin Commands + Dot Aliases (scoped — hidden from users)
```
/admin   .admin    → Admin menu
/users   .users    → User list
/broadcast .bc     → Broadcast message
/stats   .stats    → Platform stats
/plans   .plans    → Plan management
/logs    .logs     → Error logs
/ban     .ban      → Ban user
/unban   .unban    → Unban user
/grant   .grant    → Grant credits
/revoke  .revoke   → Revoke credits
/gen     .gen      → Generate coupon
/import  .import   → Import SellAuth codes
/list    .list     → List coupons by status
/disable .disable  → Disable coupon
/expire  .expire   → Force expire coupon
/gapprove .gapprove → Approve group activation
/greject  .greject  → Reject group request
/gsuspend .gsuspend → Suspend a group
/gassign  .gassign  → Assign plan to group
```

### Main Reply Keyboard (always visible after /start)
```
[ 💬 Chat ]     [ 🧠 Agent ]    [ 📁 Files ]
[ 🎁 Referral ] [ 💎 Premium ]  [ 💳 Top Up ]
[ 📊 Usage ]    [ ⚙️ Settings ]  [ 🆘 Help ]
```

Mode state bar (above keyboard):
```
Mode: 💬 Chat  |  Plan: FREE  |  Credits: 47
```

---

## 12. User Flows

### 12.1 Onboarding Flow
```
/start (DM or group):
  → Show bot info + feature overview
  → ref_CODE stored in Redis (TTL 24h) if present
  → NO credits, NO access granted
  → Show: [ 🚀 Register Now ] button

/register (DM only):
  → Already registered? → show menu
  → New user:
    → ref_CODE attributed (self-referral blocked)
    → source_type + source_ref recorded
    → Free tier activated
    → signup_bonus credits granted (from system_config)
    → credit_ledger entry written
    → Main menu shown

Group context:
  → User mentions bot → "DM mein /register karo"
  → After /register in DM → group access unlocked
  → source_type='group', source_ref=group_tg_id
```

### 12.2 Chat Mode Flow
```
User sends message
  → bot.send_chat_action("typing")
  → Mode check: Chat Mode active?
  → Rate limit check
  → Credit/quota check
  → Unicode animation: ⠙ Thinking...
  → Route to Nvidia NIM
  → Edit message: ▸▸▸ Finalizing...
  → Edit message: final response
  → Log usage event
```

### 12.3 Agent Mode Flow
```
User enters Agent Mode
  → Premium/credit eligibility check
  → User sends task
  → bot.send_chat_action("typing")
  → Unicode stages via editable message:
      ◈ Analyzing intent...
      ◈◈ Building plan...
      ◈◈◈ Selecting tools...
      ◈◈◈◈ Executing steps...
      ◈◈◈◈◈ Validating output...
      ✔ Complete
  → LangGraph: intent → complexity gate → planner → tools → exec → validate
  → Final response delivered
  → Log usage + cost
```

### 12.4 File Operation Flow
```
User sends /zip (or file attachment)
  → Create job record
  → Queue job
  → Editable message: ◎ Queued — Position #N
  → Worker picks up
  → Isolated temp: /tmp/codex-jobs/{job_id}/input/ work/ output/
  → Local processing with progress bar updates:
      █████░░░░░ Processing (50%)
  → bot.send_chat_action("upload_document")
  → Upload result to R2 + pCloud link
  → Telegram file delivery
  → Cleanup entire job folder
  → Log job complete
```

### 12.5 SellAuth Purchase Flow
```
User /buy → Select plan → Choose SellAuth
  → Bot sends SellAuth checkout link
  → User pays BTC/LTC on SellAuth
  → Coupon appears in SellAuth dashboard/email
  → Admin /import (.import) codes into DB → status: available
  → User /redeem JHATKA-XXXX-CODEX
  → Bot validates format + DB check
  → Confirm screen with inline [✔ Confirm] [✗ Cancel]
  → Plan activated → Coupon status: redeemed
```

### 12.6 USDT Payment Flow
```
User /buy → Select plan → Choose USDT → Select chain
  → Bot assigns wallet (round-robin from 10)
  → USDT invoice screen shown (see Section 33)
  → User sends USDT → submits tx hash
  → Blockchair API verify: amount + wallet + chain + ≥1 confirmation
  → Direct plan activation (no coupon)
  → Log transaction
```

### 12.7 Manual Coupon Generation (Admin)
```
Admin /gen (.gen)
  → Inline: [Subscription Plan] [Top-Up Fixed] [Top-Up Custom]
  → Select → confirm amount
  → Bot generates: JHATKA-{CRYPTO_RANDOM_16}-CODEX
  → Saved: source=manual, status=available
  → Bot displays code to admin
```

### 12.8 Referral Flow
```
User shares ref link → t.me/{bot}?start=ref_CODE
  → New user clicks → /start in DM
  → ref_CODE stored Redis 24h
  → User sends /register
    → Attribution applied (self-referral blocked)
    → source_type='referral', source_ref=referrer_id
    → Signup bonus to NEW USER (not referrer yet)
  → On first payment → activation bonus to REFERRER
  → 30-day activity → retention bonus to REFERRER
  → Threshold tiers: Bronze / Silver / Gold
  → Fraud checks: monthly cap, burst detection, cooldown
```

---

## 13. Admin Flows

### 13.1 User Investigation
```
/admin → Users → Search by tg_id/username
  → View: plan, credits, usage, referrals, abuse_flags
  → Inline actions: [Grant credits] [Ban] [View logs] [Inspect jobs]
```

### 13.2 Coupon Management
```
.list available   → Unused coupons (paginated 10/page)
.list redeemed    → Used coupons with user + timestamp
.disable CODE     → Block coupon (inline confirm)
.expire CODE      → Force expire (inline confirm)
.gen              → Create new coupon
.import           → Bulk import SellAuth codes
```

### 13.3 Broadcast
```
.bc
  → Admin types message (HTML supported)
  → Bot shows preview
  → Inline: [✔ Send to all] [✗ Cancel]
  → Sent → logged in admin_actions
```
Broadcast format:
```
📢 <b>Announcement from Codex</b>
━━━━━━━━━━━━━━━
{message_body}
━━━━━━━━━━━━━━━
<i>— Codex Team</i>
```

---

## 14. Data Model

### Core Tables
```sql
users             — tg_id, username, status, is_registered,
                    source_type, source_ref,
                    referral_code, referred_by,
                    credits_balance, timezone, timezone_source,
                    timezone_confirmed, language, current_mode,
                    is_banned, created_at, last_seen_at
plans             — name, slug, type, price_inr, price_usdt,
                    duration_days, credits, agent_access, active
subscriptions     — user_id, plan_id, start_at, end_at, status
credit_ledger     — user_id, delta, reason, source, ref_id, created_at
coupons           — code, plan_id, source, status, created_by,
                    assigned_to, redeemed_by, redeemed_at, order_ref
transactions      — user_id, method, tx_hash, amount, wallet_address,
                    chain, confirmations, status, plan_id, created_at
wallets           — address, chain_type, status, assigned_count, last_used_at
referrals         — referrer_id, referred_id, status, rewarded_at
usage_events      — user_id, request_id, model_name, route,
                    prompt_tokens, response_tokens, latency_ms,
                    cost_est_usd, success, created_at
jobs              — job_id, user_id, type, status, input_path,
                    output_path, error, retry_count, created_at, updated_at
files             — job_id, user_id, r2_key, filename, size_mb,
                    expires_at, warned_at, deleted_at, created_at
image_jobs        — user_id, request_id, provider, prompt_original,
                    prompt_enhanced, status, credits_deducted,
                    error, created_at
notifications     — user_tg_id, type, template_key, context_json,
                    status, sent_at, failed_reason, ref_id, created_at
broadcasts        — title, message_html, target_type, target_filter,
                    status, total_targets, sent_count, failed_count,
                    message_ids, scheduled_at, sent_at, deleted_at,
                    created_by, created_at
admin_actions     — admin_id, action, target_ref, detail_json, created_at
abuse_flags       — user_id, signal_type, detail, created_at
system_config     — category, key, value, value_type, label,
                    description, updated_by, updated_at
groups            — tg_group_id, owner_tg_id, name, status, plan_id,
                    trial_used, trial_start_at, plan_expires_at,
                    settings_json, created_at
group_members     — group_id, user_tg_id, is_banned, custom_limit,
                    requests_today, joined_at
group_usage_events — group_id, user_tg_id, request_id, type,
                     model, created_at
```

### Redis Keys (Transient)
```
config:{key}                    → system_config cache {"v","t"} (TTL 60s)
cache:user:{tg_id}              → plan/credit snapshot (TTL 60s)
ratelimit:{user_id}:window_start → 6-hr rolling window timestamp
ratelimit:{user_id}:count       → request count in window
session:{user_id}               → wizard state JSON (TTL 300s)
invoice:{user_id}               → active USDT invoice (TTL 1800s)
ref:{tg_id}                     → pending referral code (TTL 86400s)
imggen:{user_id}:free_api:{YYYY-MM-DD} → daily image count (TTL: midnight UTC)
imggen:{user_id}:google:{YYYY-MM-DD}   → daily image count (TTL: midnight UTC)
admin_import_codes:{admin_id}   → pending import session (TTL 300s)
```

---

## 15. Technical Stack (Final)

| Layer | Technology |
|---|---|
| Language | Python 3.11+ |
| Bot Framework | Aiogram 3.x |
| API Server | FastAPI |
| Agent / Workflow | LangGraph |
| Database | PostgreSQL (asyncpg + SQLAlchemy 2.0 async) |
| ORM + Migrations | SQLAlchemy async + Alembic |
| Cache / Broker | Redis (redis.asyncio) |
| Worker Queue | Celery + Celery Beat |
| AI — Chat | Nvidia NIM |
| AI — Agent | FreeModel |
| AI — Image (paid) | Google AI Studio — Imagen 3 |
| AI — Image (free) | Free HTTP API (prompt → image bytes) |
| File Storage | Cloudflare R2 (primary, boto3 S3-compat) |
| File Storage Stub | pCloud (adapter ready, not active) |
| Admin UI | FastAPI static + vanilla JS (dark, Material Icons) |
| Deployment | Single VPS (systemd or docker-compose) |
| Logging | Structured JSON Lines |

---

## 16. Model Routing Logic

```
Incoming request
  ↓
Is task deterministic?
  → YES → Local handler (no AI call)
  ↓
Is user in Agent Mode?
  → NO → Nvidia NIM (Layer 1)
  ↓
Is task coding/implementation?
  → YES → FreeModel coding route (Layer 2)
  ↓
Is task multi-step / complex?
  → YES + premium → LangGraph deep agent (Layer 3)
  → YES + free → Upgrade prompt
```

### Complexity Gate Rules
- High: build subsystem, design workflow, multi-file, tool orchestration
- Low: status, credits, simple chat, single snippet, file ops

---

## 17. AI Cost Protection Rules

1. Deterministic tasks → NEVER call AI
2. Free users → Nvidia NIM only (cheapest)
3. Deep agent mode → Premium/credits required
4. Per-request cost estimate logged every call
5. Daily soft cap → warning at 80%, block at 100%
6. Fallback: FreeModel fails → Nvidia NIM for non-critical
7. Rate limit: 10 req/min per user, 100 req/day free tier

---

## 18. Coupon System Spec

### Format
```
JHATKA-XXXXXXXXXXXXXXXX-CODEX
       ↑ 16 chars, A-Z0-9, cryptographically random
```

### Validation Regex
```
^JHATKA-[A-Z0-9]{16}-CODEX$
```

### Lifecycle
```
available → assigned → redeemed (terminal)
              ↓
           expired
              ↓
          disabled
```

### Source Types
- `sellauth` — externally generated, imported by admin
- `manual` — generated by admin via /gen (.gen)
- `usdt` — NOT issued (direct activation)

---

## 19. USDT Payment Rules

- Fixed invoice amount (no variable)
- 10 wallet addresses (5 TRC20 + 5 ERC20), round-robin
- Invoice TTL: 30 minutes (Redis key)
- **TRC20:** `GET https://apilist.tronscan.org/api/transaction-info?hash={tx_hash}`
- **ERC20:** Etherscan free API — isolated adapter module
- Both verifiers swappable independently
- Minimum: 1 confirmation, tolerance: ±0 (exact amount match)
- Tx hash dedup — same hash cannot activate twice (DB check)
- Manual fallback: user DMs support with proof → admin resolves
- No coupon issued — direct plan activation on verify

---

## 21. Image Generation System

### Command
```
/draw {prompt}   .draw {prompt}
```
Has its own daily limit system — NOT counted in 6-hr rolling window.

### Provider Routing
```
Free tier  → Free HTTP API (standard quality)
           → 0 credits deducted
           → N images/day (from system_config)

Paid tiers → Google AI Studio Imagen 3 (high quality)
           → credits_per_img deducted
           → N images/day (from system_config)
```

### Prompt Enhancement
Every prompt passes through enhancement before generation:
1. NSFW keyword blocklist check (original prompt)
2. Nvidia NIM prompt engineer call — expands short prompt to detailed image prompt
3. NSFW check again (enhanced prompt)
4. Send to provider

If NSFW detected → block immediately, show clean error, no API call.

### Daily Limits (all from system_config)
```
imagegen.free_api.daily.free    → N (e.g. 1)
imagegen.free_api.daily.topup   → N (e.g. 3)
imagegen.free_api.daily.sub     → N (e.g. 5)
imagegen.free_api.daily.both    → N (e.g. 10)

imagegen.google.daily.free      → 0 (no google for free)
imagegen.google.daily.topup     → N (e.g. 2)
imagegen.google.daily.sub       → N (e.g. 5)
imagegen.google.daily.both      → N (e.g. 10)
```

Daily reset: midnight UTC (Redis key TTL).

### DB Table: `image_jobs`
Logs every generation attempt — provider, original prompt,
enhanced prompt, status, credits deducted, error if failed.

### ENV Var
```
GOOGLE_API_KEY   ← from aistudio.google.com
```

### Module: `codex_imagegen`
```
modules/codex_imagegen/
  __init__.py
  core.py              ← orchestrator
  prompt_enhancer.py   ← Nvidia NIM enhancement + NSFW filter
  exceptions.py        ← ImageGenError, NSFWBlockedError, DailyLimitError
  config_keys.py
  providers/
    free_api.py        ← GET request → image bytes
    google_imagen.py   ← google-generativeai SDK
```

---

## 22. Phase 2.5 Implementation Corrections

These are corrections to implementation that were against PRD intent:

### 22.1 Registration Flow Correction
`/start` does NOT grant access. `/register` (DM only) grants access + credits.
Group context: bot prompts user to DM `/register`.
source_type + source_ref tracked on all users at registration.

### 22.2 Group Owner Commands — Context Fix
All owner commands now require GROUP context (not DM).
`_resolve_group_from_chat()` extracts group from `message.chat.id`.
Multi-group owners: command targets the group where it's sent.
Exception handling: not-owner, not-active, DB error, self-ban, invalid args.

### 22.3 codex_config Cache Format Fix
Cache now stores `{"v": value, "t": value_type}` — type preserved on hit.
Old format (raw value only) caused bool/int type loss silently.
Deploy step: `redis-cli KEYS "config:*" | xargs redis-cli DEL`

### 22.4 Rate Limit Decrement on Provider Failure
`decrement()` added to codex_ratelimit.
Called in all 3 provider-fail except branches in chat handler.
Result: slot refunded if provider errors — user not penalized.

### 22.5 Plan Resolution — No Hardcoded IDs
`plan_map = {"pro_monthly": 2, "pro_yearly": 3}` removed everywhere.
All plan resolution now uses `Plan.slug` DB lookup.
`/import` wizard now shows plan selection from DB.
`/gen` shorthand now resolves via DB.

### 22.6 File Size — plan_type Not plan_id
`plan_id != 1` assumption removed from codex_files and files handler.
Now: `Plan.plan_type == "free"` DB check — safe for any plan structure.

---
  ```
- Upload to R2 only — pCloud not in active delivery path
- pCloud adapter module exists (stubbed) for future easy switch
- File metadata in PostgreSQL `files` table
- Auto-cleanup of temp folder after upload — failure pe bhi cleanup
- **48-hour R2 lifecycle:**
  ```
  File delivered to user
    ↓
  Stored in R2
    ↓
  24hr mark → Celery task → warn user:
    "⚠ <b>File expiry warning</b>
     Aapki file <code>24 hrs</code> mein delete hogi.
     Download karo agar nahi kiya:"
    ↓
  48hr mark → Celery beat job → R2 delete
    ↓
  DB: files.deleted_at updated
  ```
- Per-user file size cap configurable via admin pricing panel
- Worker handles all jobs (never in webhook loop)
- Per-user concurrency cap: max 2 jobs simultaneously

---

## 21. Referral System Rules

- One unique code/link per user
- `/start ref_CODE` param parsing — self-referral block first
- Reward triggers: signup → activation → 30-day retention
- Anti-fraud: no self-referral, IP flag, burst auto-flag, monthly cap
- Tiers: Bronze / Silver / Gold based on referred paid count

---

## 22. Abuse Prevention Controls

| Threat | Control |
|---|---|
| Prompt flooding | Rate limit: 10 req/min, 100 req/day |
| Referral farming | Activity-based rewards, caps, IP check |
| Agent abuse | Premium gate + daily agent call cap |
| File flooding | Size cap + max 2 concurrent jobs per user |
| Queue flooding | Per-user job concurrency cap |
| USDT fraud | Tx hash dedup, exact amount, 1 confirmation |
| Coupon fraud | Unique code, one-time redeem, no delete |
| Fake signups | No reward without activity |

---

## 23. Real-Time Feedback & Unicode Animations

### Implementation rule
Bot sends one editable message. `message.edit_text()` called every 0.8–1.2s via `asyncio.sleep()` loop. Final content replaces animation frame. Never block webhook loop.

### Spinner frames (for normal AI response)
```
⠋  ⠙  ⠹  ⠸  ⠼  ⠴  ⠦  ⠧  ⠇  ⠏   (braille, cycle)
```

### Stage templates

**Normal AI response:**
```
⠙ <b>Thinking...</b>
⠸ <b>Drafting response...</b>
▸▸▸ <b>Finalizing...</b>
✔ <b>Done</b>
```

**File job:**
```
◎ <b>Queued</b>  •  Position: #2
⠹ <b>Initializing...</b>
█████░░░░░  <b>Processing</b>  (50%)
⠼ <b>Uploading...</b>
✔ <b>Ready</b>
```

**Agent mode:**
```
◈ <b>Analyzing intent...</b>
◈◈ <b>Building plan...</b>
◈◈◈ <b>Selecting tools...</b>
◈◈◈◈ <b>Executing steps...</b>
◈◈◈◈◈ <b>Validating output...</b>
✔ <b>Complete</b>
```

**Queue wait (premium badge):**
```
━━━━━━━━━━━━━━━
⟨ PRO ⟩  Priority queue

🕒 <b>Position:</b> #1
⏱ <b>Est. wait:</b> ~10s
◎ <b>Status:</b> Initializing
━━━━━━━━━━━━━━━
```

### Rules
- Update only on meaningful stage change
- Never fake percentage — only real progress
- Premium users: show priority badge
- Clock spinner for queue wait screens

---

## 24. Timezone System

Priority order:
1. User-set timezone (DB)
2. Saved preference
3. Rough locale estimate — clearly labeled as "(estimated)"
4. UTC fallback

Fields: `timezone`, `timezone_source`, `timezone_confirmed`, `timezone_updated_at`

Display rule — unconfirmed:
```html
🕐 <b>Current time</b>
<code>14:32</code>  <i>(estimated: Asia/Kolkata)</i>
Apna timezone set karo: /timezone
```

---

## 25. UI Standards

- HTML parse mode everywhere — no Markdown
- `<b>` titles, `<i>` emphasis, `<code>` values, `<pre>` blocks
- Unicode symbols for branding, separators, status indicators (see Section 28)
- Every screen: Title → Short explanation → Action buttons → Next step
- Buttons: one action per button, primary first, consistent naming
- Tone: smart, clean, direct, premium
- Long AI output (>4096 chars): split into multiple messages or offer file download
- Never show raw tracebacks to users — clean error messages only

---

## 26. Deployment Architecture

### Process Layout
```
Process 1: Aiogram bot (webhook handler)
Process 2: FastAPI (webhooks, health, admin endpoints)
Process 3: Celery/RQ worker (file jobs, uploads, cleanup)
Process 4: PostgreSQL
Process 5: Redis
```

### Webhook vs Polling
- Production: Webhook mode (FastAPI endpoint)
- Local dev: Polling mode (Aiogram long poll)
- Switch via env: `BOT_MODE=webhook|polling`

### Single VPS Layout
```
/app/
  bot/          → Aiogram handlers
  api/          → FastAPI routes
  core/         → Business logic
  integrations/ → Nvidia, FreeModel, SellAuth, Tronscan, Etherscan, R2, pCloud(stub)
  worker/       → Celery tasks
  database/     → SQLAlchemy models, migrations
  config/       → Settings, env vars
  utils/        → Helpers, logging, validation
  docs/
  tests/
```

### Health Check
```
GET /health → {"status":"ok","db":"ok","redis":"ok","worker":"ok"}
```
Degraded: HTTP 503 + failed service name.

### Environment Variables (Bootstrap Only — Infra Credentials)

**Rule:** Only the credentials needed to boot the system live in env. Everything else — plans, limits, templates, wallets, pricing, feature flags, referral rules — lives in the `system_config` DB table and is managed from the admin dashboard.

```
# Telegram (bootstrap only — rest managed from admin panel)
BOT_TOKEN
WEBHOOK_URL
WEBHOOK_SECRET
BOT_MODE            # webhook | polling

# Infrastructure
POSTGRES_URL
REDIS_URL

# Storage
R2_ACCESS_KEY
R2_SECRET_KEY
R2_BUCKET
R2_ENDPOINT

# AI Providers
NVIDIA_API_KEY
FREEMODEL_API_KEY

# Payment Verifiers
SELLAUTH_SECRET
TRONSCAN_API_KEY    # optional — free tier works without key
ETHERSCAN_API_KEY

# Admin bootstrap (first admin only — rest added from dashboard)
BOOTSTRAP_ADMIN_TG_ID

# Admin dashboard auth
ADMIN_DASHBOARD_SECRET   # token for web dashboard login
```

After first boot, admin logs into web dashboard and configures everything else — bot username, support chat ID, USDT wallets, plan prices, limits, templates, feature flags, all Telegram chat IDs — without touching env or restarting the server.

---

## 27. Risks & Tradeoffs

| Risk | Impact | Mitigation |
|---|---|---|
| FreeModel API changes | High | Adapter pattern, fallback route |
| USDT tx verification delay | Medium | Manual fallback, clear UX message |
| SellAuth coupon import lag | Medium | /import command, webhook later |
| Single VPS overload at scale | Medium | Worker separation, queue depth monitoring |
| Referral fraud burst | Medium | Auto-flag + cap system |
| pCloud link expiry | Low | TTL policy + R2 as permanent backup |
| Redis restart losing state | Low | Only transient state in Redis, recover from DB |

---

## 28. Unicode Symbol System

### Usage categories

| Purpose | Examples |
|---|---|
| Branding / headings | `𝗖𝗢𝗗𝗘𝗫` · `ᴄᴏᴅᴇx` |
| Section labels | `◈ Credits` · `◉ Status` · `⬡ Agent` |
| Status indicators | `✦ Active` · `✗ Expired` · `⊘ Banned` |
| Tier badges | `⟨ FREE ⟩` · `⟨ PRO ⟩` · `⟨ ELITE ⟩` |
| Separators | `━━━━━━━━━━━━━━━` |
| Progress bar | `█████░░░░░ 50%` |
| Bullets | `▸` · `▹` · `◦` |
| Checkmarks | `✔` · `✘` · `◎` · `○` |
| Spinner | `⠋⠙⠹⠸⠼⠴⠦⠧⠇⠏` |
| Clock | `🕐🕑🕒🕓🕔🕕` |

### Rules
- Readability first — agar weird lage, plain text prefer karo
- Braille spinner sirf editable message mein — static message mein nahi
- Progress bar sirf real percentage ke saath
- Premium screens mein zyada Unicode polish allowed
- Admin screens mein minimal Unicode — clarity priority

---

## 29. Command Dot-Alias System

### Rule
Every command has a dot alias. `/command` and `.command` both route to the same handler.

### Normalization (parse before routing)
```python
if text.startswith('.'):
    text = '/' + text[1:]   # .gen → /gen
```

### Full alias table — User
| Slash | Dot | Short |
|---|---|---|
| /start | .start | — |
| /help | .help | — |
| /menu | .menu | — |
| /usage | .usage | — |
| /referral | .referral | .ref |
| /buy | .buy | — |
| /premium | .premium | .pro |
| /support | .support | — |
| /settings | .settings | .set |
| /language | .language | .lang |
| /status | .status | — |
| /mode | .mode | — |
| /agent | .agent | — |
| /chat | .chat | — |
| /time | .time | — |
| /timezone | .timezone | .tz |
| /credits | .credits | — |
| /files | .files | — |
| /queue | .queue | .q |
| /zip | .zip | — |
| /unzip | .unzip | — |
| /merge | .merge | — |
| /extract | .extract | — |
| /redeem | .redeem | — |
| /topup | .topup | — |

### Full alias table — Admin
| Slash | Dot | Short |
|---|---|---|
| /admin | .admin | — |
| /users | .users | — |
| /broadcast | .broadcast | .bc |
| /stats | .stats | — |
| /plans | .plans | — |
| /logs | .logs | — |
| /ban | .ban | — |
| /unban | .unban | — |
| /grant | .grant | — |
| /revoke | .revoke | — |
| /gen | .gen | — |
| /import | .import | — |
| /list | .list | — |
| /disable | .disable | — |
| /expire | .expire | — |
| /gapprove | .gapprove | — |
| /greject | .greject | — |
| /gsuspend | .gsuspend | — |
| /gassign | .gassign | — |

### Full alias table — Group Owner
| Slash | Dot | Short | Where |
|---|---|---|---|
| /gstatus | .gstatus | — | Group + DM |
| /gusage | .gusage | — | Group + DM |
| /gplan | .gplan | — | Group + DM |
| /ghelp | .ghelp | — | Group + DM |
| /gcancel | .gcancel | — | Group + DM |
| /glimit | .glimit | — | DM only |
| /gban | .gban | — | DM only |
| /gunban | .gunban | — | DM only |
| /gmembers | .gmembers | — | DM only |
| /gmode | .gmode | — | DM only |
| /gwelcome | .gwelcome | — | DM only |
| /greset | .greset | — | DM only |
| /gbroadcast | .gbroadcast | .gbc | DM only |
| /gsettings | .gsettings | — | DM only |
| /gupgrade | .gupgrade | — | DM only |

---

## 30. Button Type Classification

### Types defined
| Type | When |
|---|---|
| Reply button | Persistent navigation — keyboard hamesha visible |
| Inline button | Contextual action — sirf specific message ke saath |
| Both | Main menu se bhi aur message-level action bhi |

### Feature → Button matrix

| Feature / Command | Reply | Inline | Notes |
|---|:---:|:---:|---|
| /start onboarding | ✔ | ✗ | Main menu reply keyboard |
| /menu | ✔ | ✗ | Reply keyboard refresh |
| /help | ✔ | ✗ | Static info |
| /chat (mode) | ✔ | ✗ | Mode toggle via reply |
| /agent (mode) | ✔ | ✗ | Mode toggle via reply |
| /mode | ✔ | ✗ | Shows mode, switch via reply |
| /time | ✔ | ✗ | Deterministic, no action |
| /status | ✔ | ✗ | Read-only |
| /support | ✔ | ✗ | Contact/link |
| /timezone | ✔ | ✔ | Reply: open · Inline: select/confirm |
| /credits | ✔ | ✔ | Reply: open · Inline: top-up CTA |
| /usage | ✔ | ✔ | Reply: open · Inline: upgrade CTA |
| /referral | ✔ | ✔ | Reply: open · Inline: copy link, share |
| /buy | ✔ | ✔ | Reply: open · Inline: plan + method select |
| /premium | ✔ | ✔ | Reply: open · Inline: upgrade, compare |
| /topup | ✔ | ✔ | Reply: open · Inline: amount + confirm |
| /files | ✔ | ✔ | Reply: open · Inline: file actions |
| /queue | ✔ | ✔ | Reply: open · Inline: cancel job |
| /settings | ✔ | ✔ | Reply: open · Inline: toggle each setting |
| /language | ✔ | ✔ | Reply: open · Inline: language select |
| /zip | ✔ | ✔ | Reply: trigger · Inline: confirm options |
| /unzip | ✔ | ✔ | Reply: trigger · Inline: confirm |
| /merge | ✔ | ✔ | Reply: trigger · Inline: file + confirm |
| /extract | ✔ | ✔ | Reply: trigger · Inline: confirm |
| /redeem | ✔ | ✗ | Text input flow — inline not needed |
| Chat (free — limit hit) | ✗ | ✔ | Upgrade prompt inline |
| Chat (agent — running) | ✗ | ✔ | Cancel/retry inline |
| Admin: /admin | ✗ | ✔ | Admin menu = inline only |
| Admin: /users | ✗ | ✔ | Inline pagination |
| Admin: /broadcast | ✗ | ✔ | Preview + confirm inline |
| Admin: /stats | ✗ | ✔ | Optional inline refresh |
| Admin: /plans | ✗ | ✔ | Inline edit |
| Admin: /logs | ✗ | ✔ | Inline filter/pagination |
| Admin: /ban /unban | ✗ | ✔ | Confirm inline |
| Admin: /grant /revoke | ✗ | ✔ | Confirm inline |
| Admin: /gen | ✗ | ✔ | Plan select + confirm inline |
| Admin: /import | ✗ | ✔ | Confirm bulk inline |
| Admin: /list | ✗ | ✔ | Status filter inline |
| Admin: /disable /expire | ✗ | ✔ | Confirm inline |
| Admin: /gapprove | ✗ | ✔ | Plan select + confirm inline |
| Admin: /greject | ✗ | ✔ | Confirm inline |
| Admin: /gsuspend | ✗ | ✔ | Confirm inline |
| Group owner: /gstatus | ✔ | ✗ | Read-only — reply trigger |
| Group owner: /gusage | ✔ | ✗ | Read-only |
| Group owner: /gplan | ✔ | ✔ | Reply: open · Inline: upgrade CTA |
| Group owner: /ghelp | ✔ | ✗ | Static info |
| Group owner: /gban | ✗ | ✔ | User select + confirm inline |
| Group owner: /glimit | ✗ | ✔ | Amount input + confirm inline |
| Group owner: /gmembers | ✗ | ✔ | Inline pagination + actions |
| Group owner: /gmode | ✗ | ✔ | chat/agent toggle inline |
| Group owner: /gwelcome | ✗ | ✔ | Text input + preview + confirm |
| Group owner: /gsettings | ✗ | ✔ | Settings wizard inline |
| Group owner: /gupgrade | ✗ | ✔ | Plan select + payment method |
| Group owner: /gbroadcast | ✗ | ✔ | Preview + confirm inline |

### Inline callback data format
```
action:feature:param
```
Examples:
```
buy:plan:pro_monthly
buy:method:usdt
coupon:confirm:JHATKA-XXX-CODEX
admin:ban:user_123
file:cancel:job_456
upgrade:trigger:limit_hit
tz:set:Asia/Kolkata
list:page:3:available
group:approve:group_id:plan_id
group:reject:group_id
group:ban_member:group_id:user_id
group:upgrade:group_id:plan_id
group:mode:group_id:agent
```
Max 64 bytes (Telegram limit).

### Inline pagination rule
5+ items → paginate at 10/page.
Buttons: `[ ← Prev ]  Page 2/5  [ Next → ]`

---

## 31. Standard Message Templates

### Rate limit screen
```html
<b>⊘ Limit reached</b>

Aapka aaj ka free usage khatam ho gaya.

━━━━━━━━━━━━━━━
▸ Kal reset hoga: <code>00:00 UTC</code>
▸ Ya abhi upgrade karein 👇
━━━━━━━━━━━━━━━
```
Inline: `[ 💎 Go Premium ]  [ 💳 Top Up ]`

### Error screen (user-facing)
```html
<b>⚠ Something went wrong</b>

<i>Error:</i> <code>provider_timeout</code>

Bot retry kar raha hai. Baar baar ho to /support use karo.
```

### Coupon redeem confirm
```html
◈ <b>Coupon verified</b>

<code>JHATKA-XXXX-CODEX</code>
Plan: <b>Pro Monthly</b>
Valid: <b>30 days</b>

Confirm karna chahte ho?
```
Inline: `[ ✔ Confirm ]  [ ✗ Cancel ]`

### USDT invoice
```html
💳 <b>USDT Payment</b>
━━━━━━━━━━━━━━━
Chain:   <code>TRC20</code>
Amount:  <code>9.90 USDT</code>
Wallet:  <code>TXxxxxxxxxxxxxxxxxxxxxxx</code>
Expires: <code>14:32 UTC</code>
━━━━━━━━━━━━━━━
Payment ke baad tx hash submit karo:
```
Inline: `[ 📋 Copy wallet ]  [ ✅ Submit tx hash ]  [ ❌ Cancel ]`

---

## 32. Admin Action Log Format

Every admin action written to DB:
```json
{
  "admin_id": 123456,
  "action": "grant_credits",
  "target_ref": "user:789",
  "detail": {"amount": 50, "reason": "support compensation"},
  "created_at": "2024-01-15T10:30:00Z"
}
```

---

## 33. Delivery Phases (Final)

### Phase 1 (MVP — Launch)
- Bot skeleton + onboarding (wizard UX)
- HTML + Unicode UI + all standard message templates from system_config
- Dot-alias command parsing (user + admin)
- Reply/inline button system (per Section 30 matrix)
- Conversational wizard for /gen, /buy, /timezone, /language
- Chat Mode (Nvidia NIM) with Unicode animation stages
- Free tier + 6-hour rolling rate limit
- Local handlers (time, credits, referral, help, status)
- R2 integration + 48hr file lifecycle + expiry warnings
- Coupon system (manual /gen + SellAuth /import)
- USDT payment flow (Tronscan TRC20 + Etherscan ERC20)
- UPI DM redirect
- Admin bot commands: /gen, /list, /ban, /grant, /stats, /broadcast
- Admin web dashboard (basic: overview, users, coupons, settings)
- system_config table — all settings from dashboard
- Referral link system + /start ref_ parsing
- Notification system (plan expiry, file expiry, usage warnings)
- Structured logging + health endpoint
- Webhook + polling mode switch via env
- Celery beat scheduler (file cleanup, expiry checks)

### Phase 2 (Monetization + Groups)
- Premium plans + subscriptions + renewal flow
- Top-up credits (fixed + custom) + credits ledger
- SellAuth full flow
- File ops (merge, extract) + progress bar animation
- Queue + worker hardening
- Agent Mode (FreeModel + LangGraph) with full stage animation
- Usage dashboard per user
- Referral threshold rewards + tier system (Bronze/Silver/Gold)
- Inline pagination for all list screens
- Queue position display
- **Group bot system** (Section 42) — full launch
  - Group onboarding + admin approval flow
  - Group plans (2-week free trial auto-approve)
  - Group owner commands
  - Member usage + limits
  - Group upsell to personal plan
- Admin web dashboard Phase 2: Groups panel, chat interface, broadcast management
- i18n language system (Phase 2 languages: EN + HI)

### Phase 3 (Scale & Optimize)
- Advanced agent workflows + tool orchestration
- Better analytics + revenue dashboard
- Automated abuse scoring system
- Worker separation + horizontal scaling
- SellAuth webhook automation
- Deeper retention flows + win-back campaigns
- pCloud activation (optional — switch via config)
- Multi-language expansion
- AI provider rate limit handling + smart queue
- Advanced group features (per-member analytics, group leaderboard)

---

## 34. Open Items

1. SellAuth webhook vs manual import — Phase 1 decision
2. Nvidia NIM model string finalization
3. FreeModel endpoint verify (https://freemodel.dev/dashboard/docs)
4. ERC20 Etherscan API key + rate limit check
5. Agent Mode daily cap per premium user (exact number)
6. Admin web dashboard: self-hosted vs Cloudflare Pages?
7. Unicode font rendering test on iOS/Android Telegram clients
8. File size caps per plan tier — finalize numbers

---

## 35. Conversational Command Wizard

### Problem
User `/gen` bolta hai → bot ek baar mein sab maangta hai → confusion aur friction.

### Solution: Two-mode command handling

**Mode A — Wizard (default)**
User command bhejta hai bina params ke → bot step by step puchta hai via inline buttons aur text replies.

**Mode B — Power shorthand**
User command + inline params deta hai → bot directly process karta hai.

Both modes same handler mein — parser decide karta hai.

---

### Wizard flow — example: `/gen`

```
User: /gen

Bot:
◈ <b>Coupon Generator</b>
━━━━━━━━━━━━━━━
Kaunsa type chahiye?

[ Subscription Plan ]  [ Top-Up Fixed ]  [ Top-Up Custom ]
```

```
Admin selects: Subscription Plan

Bot:
Plan choose karo:

[ Pro Monthly ]  [ Pro Yearly ]
[ ← Back ]
```

```
Admin selects: Pro Monthly

Bot:
◈ <b>Confirm generation</b>

Type:   <b>Subscription</b>
Plan:   <b>Pro Monthly</b>
Valid:  <b>30 days</b>

[ ✔ Generate ]  [ ✗ Cancel ]
```

```
Admin confirms:

✔ <b>Coupon generated</b>

<code>JHATKA-H8V95BD38K94J9F-CODEX</code>

Source: manual  |  Plan: Pro Monthly
Status: available

[ 📋 Copy ]  [ Generate another ]
```

---

### Wizard flow — example: `/buy` (user)

```
User: /buy

Bot:
💎 <b>Choose a plan</b>
━━━━━━━━━━━━━━━

[ Pro Monthly — ₹X ]
[ Pro Yearly — ₹X (save Y%) ]
[ Top-Up Credits ]
[ ← Back ]
```

```
User selects: Pro Monthly

Bot:
💳 <b>Payment method</b>

[ BTC / LTC via SellAuth ]
[ USDT (TRC20 / ERC20) ]
[ UPI — Manual DM ]
[ ← Back ]
```

```
User selects: USDT

Bot:
🔗 <b>Select chain</b>

[ TRC20 (Tron) ]  [ ERC20 (Ethereum) ]
[ ← Back ]
```
→ USDT invoice screen (Section 31)

---

### Wizard flow — example: `/timezone` (user)

```
User: /timezone

Bot:
🌍 <b>Timezone Setup</b>

Current: <code>Asia/Kolkata</code> (confirmed)

[ Change timezone ]  [ ← Back ]
```

```
User: Change timezone

Bot:
Type your city or timezone:
e.g. "Mumbai", "Asia/Dubai", "UTC+5:30"
```

```
User: Dubai

Bot:
◈ <b>Confirm timezone</b>

<code>Asia/Dubai</code>  (UTC+4)

[ ✔ Confirm ]  [ Try again ]  [ ← Back ]
```

---

### Wizard state management

- Active wizard state stored in Redis: `session:{user_id}` → `{wizard: "buy", step: 2, data: {...}}`
- Timeout: wizard expires after 5 minutes of inactivity
- `/cancel` ya `.cancel` — wizard exit anytime
- `← Back` button — previous step
- State cleared after completion or cancel

---

### Power shorthand examples

```
/gen pro_monthly          → skip wizard, generate Pro Monthly coupon directly
/ban 123456789            → ban user ID directly
/grant 123456789 50       → grant 50 credits directly
/timezone Asia/Dubai      → set timezone directly
```

Parser check: agar params present → skip wizard, direct execute.

---

### Which commands get wizard

| Command | Wizard | Shorthand |
|---|:---:|:---:|
| /gen | ✔ | ✔ |
| /buy | ✔ | ✗ |
| /topup | ✔ | ✗ |
| /timezone | ✔ | ✔ |
| /language | ✔ | ✔ |
| /settings | ✔ | ✗ |
| /ban | ✗ | ✔ |
| /grant | ✗ | ✔ |
| /broadcast | ✔ | ✗ |
| /import | ✔ | ✗ |
| /disable | ✗ | ✔ |
| /expire | ✗ | ✔ |
| /redeem | ✗ | ✔ |

---

## 36. Rate Limit — 6-Hour Rolling Window

### Problem with midnight reset
User 11:55 PM ko limit hit karta hai → 5 minute baad reset → but user 11:30 PM ko limit hit kare toh 30 min baad reset hoga midnight se. Inconsistent. Also creates "dump everything before midnight" behavior.

### 6-Hour rolling window
User jis moment limit hit kare, usse 6 hrs baad same amount refill ho. No fixed midnight. Smooth and predictable.

### Per-tier limits

| Tier | Window | AI Requests | Agent Tasks | Notes |
|---|---|---|---|---|
| Free | 6 hrs rolling | 20 | 0 | No agent access |
| Top-up only | 4–5 hrs rolling | 40 | 5 | Credits spent separately |
| Subscription only | 3–4 hrs rolling | 80 | 15 | Plan-based |
| Subscription + Top-up | 3 hrs rolling | 120 | 25 | Best combo — fastest refill |

Top-up credits for agentic tasks/file ops are separate from the rolling AI request counter. Credits are spent regardless of window.

### Redis key structure

```
ratelimit:{user_id}:window_start   → timestamp of window start (TTL = window duration)
ratelimit:{user_id}:count          → request count in current window
```

On limit hit:
```html
<b>⊘ Limit reached</b>

Aapka is window ka usage poora ho gaya.

━━━━━━━━━━━━━━━
🔄 Reset in: <code>2 hrs 14 min</code>
▸ Ya abhi upgrade karein
━━━━━━━━━━━━━━━
```
Inline: `[ 💎 Go Premium ]  [ 💳 Top Up ]`

Countdown calculated from `window_start + window_duration - now`.

---

## 37. UPI Payment — Manual DM Only

### Decision
No bot-side UPI flow. UPI mein instant verification possible nahi hai without paid gateway. Bot-side form = support mess + fraud risk.

### User experience in bot

```
User selects: UPI — Manual DM

Bot:
💳 <b>UPI Payment — Manual</b>
━━━━━━━━━━━━━━━
Amount:   <b>₹X</b>
UPI ID:   <code>codex@upi</code>
━━━━━━━━━━━━━━━
Payment ke baad:
1. Screenshot lo
2. Transaction ID note karo
3. Support ko DM karo

Reply time: 2–5 hours

[ 💬 Open Support DM ]  [ ← Back ]
```

`Open Support DM` button → `tg://user?id={SUPPORT_BOT_ID}` deep link.

### Admin side
Admin support DM mein screenshot dekhe → verify → `/grant USER_ID CREDITS` ya plan activate kare.

No automated UPI verification. Fully manual.

---

## 38. Admin Web Dashboard

### Purpose
Telegram bot commands → good for quick actions.
Web dashboard → better for bulk operations, data visualization, chat management.

### Stack
- Single HTML file per page or lightweight JS (no heavy framework needed at Phase 1)
- Served from FastAPI static files or Cloudflare Pages
- Auth: secret token in header or session cookie — admin only
- Material Icons (Google) for all icons — no emojis
- Mobile + PC responsive (CSS Grid / Flexbox)
- Dark theme preferred (matches bot premium feel)

### Dashboard sections

#### 1. Overview
- Total users · DAU · MAU
- Active subscriptions · Top-up users
- AI cost estimate today / month
- Queue depth
- Recent errors (last 10)
- Revenue estimate

#### 2. User Management
- Search by TG ID / username / plan
- User card: plan, credits, usage history, referrals, abuse flags, last seen, join date
- Actions: Grant credits · Revoke credits · Ban · Unban · Change plan · Reset rate limit · View logs · Inspect jobs
- Pagination (20 per page) · Export CSV

#### 3. Chat Interface (Admin → User)
- GPT/Claude-style chat panel — sidebar: user list · main: message thread
- Admin selects any user → sees full bot conversation history
- Admin can type + send message to user as bot (Telegram bot.send_message)
- Material icon toolbar: send · user info panel · ban shortcut · grant shortcut
- Message input: fixed bottom bar, scrollable history above

#### 4. Coupon Management
- Table: code · plan · source · status · created_at · redeemed_by · redeemed_at
- Filters: status · source · plan · date range
- Actions: Generate single · Bulk generate · Import CSV · Disable · Expire
- Bulk actions via checkboxes: bulk disable, bulk expire

#### 5. Settings & Config (→ Section 40)
All `system_config` keys grouped by category tabs: Bot · Plans · Limits · Files · Payments · AI · Referral · Abuse · Templates · Features. Each row: label · current value · edit field · Save button · last changed metadata.

#### 6. Broadcast (→ Section 41)
Create · Preview · Send · Schedule · Delete broadcasts. Full message_ids tracking for deletion within 48hr Telegram window.

#### 7. Logs & Errors
- Error log stream (last 100, auto-refresh)
- Filter: by type · user · date range
- Failed job inspector: job_id · type · error · retry count · re-queue button
- Admin action log: who changed what and when

#### 8. System Status
- Live health: DB · Redis · Worker · Bot webhook (green/red indicators)
- API cost tracker (today / week / month)
- Queue depth counter
- Active USDT invoices pending

### Responsive layout rules
- Mobile: collapsible sidebar (hamburger), single column content
- PC: fixed sidebar (220px) + main content area
- Material Icons: always icon + text label (not icon alone)
- Chat interface: `height: calc(100vh - 120px)`, scrollable messages, fixed input bar
- Tables: horizontal scroll on mobile

---


---

## 40. Admin Full Control — system_config Table

### Principle
Everything not in env lives in `system_config`. Single source of truth for all runtime settings. Admin dashboard reads and writes this table. Bot and workers read it (via Redis cache). No hardcoded values anywhere in codebase except bootstrap defaults.

### DB Table

```sql
system_config
  id            serial primary key
  category      varchar not null      -- 'bot', 'plans', 'limits', 'files', 'payments', 'referral', 'abuse', 'ai', 'templates', 'features'
  key           varchar not null unique
  value         text not null
  value_type    varchar not null      -- 'int', 'float', 'string', 'bool', 'json'
  label         varchar               -- Human-readable label for admin UI
  description   text                  -- Tooltip/help text in admin UI
  updated_by    int references users(id)
  updated_at    timestamp
```

### Cache
All values cached in Redis: `config:{key}` TTL 60s. Admin save → Redis DEL → next read fetches fresh.

### Admin dashboard: Settings Panel

Single page with tabs by category. Each row: label · current value · input/toggle · Save button · "Last changed by X at Y" tooltip.

Every save → logged to `admin_actions` with before/after values.

---

### Category: Bot Settings
| Key | Label | Type |
|---|---|---|
| `bot.token` | Bot Token | string |
| `bot.username` | Bot Username | string |
| `bot.webhook_url` | Webhook URL | string |
| `bot.webhook_secret` | Webhook Secret | string |
| `bot.support_tg_id` | Support Chat TG ID | string |
| `bot.support_username` | Support Username | string |
| `bot.admin_chat_id` | Admin Log Chat ID | string |
| `bot.admin_tg_ids` | Admin TG IDs (comma) | string |
| `bot.mode` | Bot Mode (webhook/polling) | string |
| `bot.maintenance_mode` | Maintenance Mode | bool |
| `bot.maintenance_message` | Maintenance Message | string |

---

### Category: Plans
Each plan stored as JSON row. Admin can create new plan, edit, or soft-delete.

| Key | Label | Type |
|---|---|---|
| `plan.{id}.name` | Plan Name | string |
| `plan.{id}.type` | Type (subscription/topup) | string |
| `plan.{id}.price_inr` | Price (INR) | float |
| `plan.{id}.price_usdt` | Price (USDT) | float |
| `plan.{id}.duration_days` | Duration (days) | int |
| `plan.{id}.credits` | Credits granted | int |
| `plan.{id}.agent_access` | Agent Mode access | bool |
| `plan.{id}.file_size_mb` | Max file size (MB) | int |
| `plan.{id}.active` | Plan visible to users | bool |
| `topup.credits_per_inr` | Credits per ₹1 | float |
| `topup.min_inr` | Min top-up (INR) | int |
| `topup.max_inr` | Max top-up (INR) | int |
| `topup.min_usdt` | Min top-up (USDT) | float |

---

### Category: Rate Limits
| Key | Label | Type |
|---|---|---|
| `limit.free.window_hrs` | Free window (hrs) | int |
| `limit.free.requests` | Free requests/window | int |
| `limit.free.agent_tasks` | Free agent tasks | int |
| `limit.topup.window_hrs` | Top-up window (hrs) | int |
| `limit.topup.requests` | Top-up requests/window | int |
| `limit.topup.agent_tasks` | Top-up agent tasks | int |
| `limit.sub.window_hrs` | Sub window (hrs) | int |
| `limit.sub.requests` | Sub requests/window | int |
| `limit.sub.agent_tasks` | Sub agent tasks | int |
| `limit.both.window_hrs` | Sub+Topup window (hrs) | int |
| `limit.both.requests` | Sub+Topup requests/window | int |
| `limit.both.agent_tasks` | Sub+Topup agent tasks | int |

---

### Category: File Policies
| Key | Label | Type |
|---|---|---|
| `file.expiry_hours` | File TTL in R2 (hrs) | int |
| `file.warn_at_hours` | Warn user at X hrs | int |
| `file.free.max_mb` | Free user max file MB | int |
| `file.sub.max_mb` | Sub user max file MB | int |
| `file.max_concurrent_jobs` | Max jobs per user | int |

---

### Category: Payments & Wallets
| Key | Label | Type |
|---|---|---|
| `payment.usdt.wallets_trc20` | TRC20 wallets (JSON array) | json |
| `payment.usdt.wallets_erc20` | ERC20 wallets (JSON array) | json |
| `payment.usdt.invoice_ttl_min` | Invoice TTL (minutes) | int |
| `payment.sellauth.enabled` | SellAuth enabled | bool |
| `payment.upi.enabled` | UPI DM enabled | bool |
| `payment.upi.id` | UPI ID shown to users | string |
| `payment.upi.name` | UPI display name | string |
| `payment.usdt.enabled` | USDT payments enabled | bool |

---

### Category: AI Routing
| Key | Label | Type |
|---|---|---|
| `ai.chat.model` | Chat model name (Nvidia) | string |
| `ai.agent.model` | Agent model name (FreeModel) | string |
| `ai.chat.max_tokens` | Chat max tokens | int |
| `ai.agent.max_tokens` | Agent max tokens | int |
| `ai.agent.min_credits` | Min credits for agent | int |
| `ai.agent.cost_per_task` | Credits per agent task | int |
| `ai.fallback_enabled` | Fallback to cheaper route | bool |
| `ai.cost_warn_threshold` | Daily cost warn (USD) | float |

---

### Category: Referral Rules
| Key | Label | Type |
|---|---|---|
| `referral.signup_bonus` | Signup referral credits | int |
| `referral.activation_bonus` | Activation bonus credits | int |
| `referral.retention_bonus` | 30-day retention bonus | int |
| `referral.max_per_month` | Max rewards per referrer/month | int |
| `referral.cooldown_hrs` | Cooldown between rewards (hrs) | int |
| `referral.bronze_threshold` | Bronze tier (paid refs) | int |
| `referral.silver_threshold` | Silver tier (paid refs) | int |
| `referral.gold_threshold` | Gold tier (paid refs) | int |
| `referral.enabled` | Referral system on/off | bool |

---

### Category: Abuse Thresholds
| Key | Label | Type |
|---|---|---|
| `abuse.req_per_min_limit` | Max requests per minute | int |
| `abuse.file_flood_limit` | Max file jobs in 10min | int |
| `abuse.ref_burst_limit` | Referral burst threshold | int |
| `abuse.ref_burst_window_hrs` | Ref burst window (hrs) | int |
| `abuse.auto_ban_enabled` | Auto-ban on threshold | bool |
| `abuse.auto_ban_threshold` | Requests before auto-ban | int |

---

### Category: Message Templates
All user-facing bot messages editable from dashboard. Stored as string/HTML. Bot reads from `system_config`, not hardcoded strings.

| Key | Label |
|---|---|
| `template.welcome` | /start welcome message |
| `template.help` | /help screen content |
| `template.rate_limit` | Rate limit hit message |
| `template.upgrade_prompt` | Upgrade prompt (inline) |
| `template.error_generic` | Generic error message |
| `template.file_expiry_warning` | 24hr file warning message |
| `template.maintenance` | Maintenance mode message |
| `template.referral_welcome` | Message to referred new user |
| `template.referral_reward` | Reward notification to referrer |
| `template.usdt_invoice` | USDT invoice template |
| `template.coupon_confirm` | Coupon redeem confirm |
| `template.broadcast_header` | Broadcast message header |

Templates support variables: `{user_first_name}`, `{credits}`, `{plan_name}`, `{reset_in}`, `{file_name}`, `{expires_at}`, etc.

---

### Category: Feature Flags
Any feature can be toggled without code deploy.

| Key | Label | Type |
|---|---|---|
| `feature.agent_mode` | Agent Mode enabled | bool |
| `feature.file_ops` | File ops enabled | bool |
| `feature.referral` | Referral system enabled | bool |
| `feature.topup` | Top-up credits enabled | bool |
| `feature.sellauth` | SellAuth payments enabled | bool |
| `feature.usdt` | USDT payments enabled | bool |
| `feature.upi` | UPI DM option shown | bool |
| `feature.timezone` | Timezone feature enabled | bool |
| `feature.language` | Language setting enabled | bool |
| `feature.new_registrations` | Allow new signups | bool |

---

## 41. Broadcast Management (Full)

### Broadcast flow

```
Admin → Dashboard → Broadcasts → New Broadcast
  → Rich text editor (HTML supported)
  → Variable picker: {user_first_name}, {plan_name}, etc.
  → Preview rendered (exactly as Telegram will show)
  → Target: All users | Subscription users | Free users | Top-up users | Custom segment
  → Schedule: Send now | Schedule for datetime
  → Send → Celery task → batch send (100 users/sec max to avoid flood)
  → Record saved in `broadcasts` table with message_ids[]
```

### Broadcast delete

Telegram allows bots to delete their own messages within 48 hours.

```
Admin → Broadcasts list → Select broadcast → Delete

Bot calls:
  for message_id in broadcast.message_ids:
    bot.delete_message(chat_id=user_id, message_id=message_id)

DB: broadcast.deleted_at = now, status = 'deleted'
```

After 48 hours: delete API call fails silently (Telegram limit). Record marked `delete_failed` — message stays in user chat but DB records it.

### DB Table: `broadcasts`
```sql
id            serial primary key
title         varchar               -- admin internal label
message_html  text                  -- full HTML message body
target_type   varchar               -- 'all', 'free', 'sub', 'topup', 'custom'
target_filter jsonb                 -- custom filter params
status        varchar               -- 'draft', 'sending', 'sent', 'deleted', 'delete_failed'
total_targets int
sent_count    int
failed_count  int
message_ids   jsonb                 -- [{user_id, message_id}, ...] for delete support
scheduled_at  timestamp
sent_at       timestamp
deleted_at    timestamp
created_by    int references users(id)
created_at    timestamp
```

### Dashboard broadcast list
| Column | Values |
|---|---|
| Title | Admin label |
| Target | All · Free · Sub · Top-up |
| Status | Draft · Sending · Sent · Deleted |
| Reach | X / Y sent |
| Sent at | Datetime |
| Actions | View · Delete · Duplicate |


---

## 42. Group Bot System

### Business rationale
Personal bot growth = slow. Group deployment = viral. One group owner with 500 members = 500 potential users who discover the bot, get hooked on free trial, then convert to personal plans. Groups are the growth engine.

---

### 42.1 Group Onboarding Flow

**Step 1: Bot added to group**
Group owner adds bot as admin (admin required for reading all messages when privacy mode is off — alternatively bot reads only @mentions + commands without admin, but admin preferred for full functionality).

Telegram update received: `new_chat_members` event with `bot` as new member. Adder's `user_id` is available from `from` field — this is the group owner/adder.

**Step 2: Bot sends in-group welcome (hardcoded)**
```html
👋 <b>Codex Bot added!</b>

Iss group mein Codex activate karne ke liye:
▸ Group owner → bot ko DM karo
▸ /start bhejo

<i>Owner ne DM nahi kiya? Neeche click karo:</i>
```
Inline button: `[ 🚀 Activate this group ]` → deep link: `https://t.me/{botname}?start=group_{group_id}`

**Why in-group first?** Bot cannot DM a user who has never started it. So bot sends in-group message with deep link. Owner clicks → DM opens → bot sends full onboarding.

**Step 3: Owner DMs bot (via deep link)**
Deep link param: `start=group_{group_id}`
Bot detects group context → sends owner a special hardcoded group onboarding message:

```html
🏢 <b>Group Activation Request</b>
━━━━━━━━━━━━━━━
Group:  <b>{group_name}</b>
ID:     <code>{group_id}</code>

Bot ko activate karne ke liye admin approval chahiye.

Approval ke baad aapko:
▸ Group plan assign hoga
▸ Members bot use kar payenge
▸ Aapko group owner commands milenge

Request submit ho gayi. Admin se contact:
```
Inline: `[ 📨 Check status ]  [ 💬 Contact support ]`

**Step 4: Admin notified**
Bot sends message to `bot.admin_chat_id` (from `system_config`):

```html
🔔 <b>New group activation request</b>

Group:   <b>{group_name}</b>
Group ID: <code>{group_id}</code>
Owner:   <b>{owner_name}</b> (<code>{owner_id}</code>)
Members: <b>{member_count}</b>
```
Inline: `[ ✅ Approve + Assign Plan ]  [ ❌ Reject ]`

**Step 5: Admin approves from dashboard**
Admin selects group plan → bot activates group → owner gets DM confirmation → bot sends in-group activation message.

**Step 6: Activation confirmation (in-group)**
```html
✅ <b>Codex activated in this group!</b>

Plan:  <b>{plan_name}</b>
Limit: <b>{member_request_limit} requests/member per window</b>

Commands: /help@{botname}

Group Owner: <b>{owner_name}</b>
```

---

### 42.2 Group States

| State | Meaning |
|---|---|
| `pending` | Owner submitted request, awaiting admin approval |
| `active` | Approved, bot working in group |
| `trial` | Free trial period active (admin-set duration) |
| `expired` | Plan expired, bot paused in group |
| `suspended` | Admin manually suspended |
| `removed` | Bot was kicked from group |

---

### 42.3 Group Plans

Separate from personal plans. Defined and priced in admin panel (`system_config` + `group_plans` table).

Each group plan defines:

| Field | Description |
|---|---|
| `name` | Plan name (e.g. "Group Starter", "Group Pro") |
| `price_inr` | Monthly price (INR) |
| `price_usdt` | Monthly price (USDT) |
| `max_members_active` | Max members who can use bot concurrently |
| `member_request_limit` | Requests per member per window |
| `member_window_hrs` | Member rate limit window (hrs) |
| `group_shared_pool` | Shared request pool for group (optional mode) |
| `agent_access` | Agent mode available in group | 
| `file_ops_access` | File operations in group |
| `unlocked_commands` | JSON list of commands available |
| `owner_commands` | JSON list of owner-only commands |
| `trial_days` | Free trial days (0 = no trial) |
| `active` | Plan visible in dashboard |

---

### 42.4 Member Usage in Groups

**Two quota modes** (admin configures per group plan):

**Mode A — Per-member quota**
Each member has their own `member_request_limit` per window. Independent of other members.

**Mode B — Shared pool**
Group gets a total `group_shared_pool` requests per window. All members draw from it. First-come-first-served. Owner can see pool status.

Member usage tracked in `group_usage_events` table.

**Personal plan + group plan:**
If a user has a personal Codex plan, do they get their personal limits in the group too? Decision: **No mixing by default**. In group context, group plan limits apply. Personal plan is separate. Admin can override per group via `system_config`.

**If bot is used in group by member who has never started the bot personally:**
Bot can still respond to them in group. No personal account needed to use basic group commands. Personal account required only for: credits, referral, personal upgrade, settings.

---

### 42.5 Group Owner Commands

Group owner has elevated permissions in their group. These commands work **in the group** or **via DM** (noted per command).

Commands unlocked based on group plan's `owner_commands` JSON field — admin controls which commands are available per plan.

| Command | Alias | Where | Description |
|---|---|---|---|
| `/gstatus` | `.gstatus` | Group + DM | Group plan, usage, expiry |
| `/gusage` | `.gusage` | Group + DM | Usage stats for whole group |
| `/gplan` | `.gplan` | Group + DM | Current plan details + upgrade |
| `/ghelp` | `.ghelp` | Group + DM | Group owner command list |
| `/glimit @user X` | `.glimit` | DM only | Set per-member limit override |
| `/gban @user` | `.gban` | DM only | Ban member from using bot in this group |
| `/gunban @user` | `.gunban` | DM only | Unban member |
| `/gmembers` | `.gmembers` | DM only | List active members + usage |
| `/gmode chat\|agent` | `.gmode` | DM only | Set default mode for group |
| `/gwelcome [text]` | `.gwelcome` | DM only | Set custom welcome message for group |
| `/greset @user` | `.greset` | DM only | Reset a member's usage window |
| `/gbroadcast [text]` | `.gbc` | DM only | Send bot message to all group members |
| `/gsettings` | `.gsettings` | DM only | Open group settings wizard |
| `/gupgrade` | `.gupgrade` | DM only | Upgrade group plan |

**Why DM-only for sensitive commands?** Prevents public exposure of usage data, bans, and settings. Keeps group chat clean.

---

### 42.6 Member Commands in Group Context

Members use standard commands in group (with `@botname` suffix optional if bot is admin):

Available commands per group plan's `unlocked_commands` field. Admin defines this per plan.

**Example: Group Starter plan unlocked commands:**
```json
["chat", "help", "time", "timezone", "status", "mode"]
```

**Example: Group Pro plan unlocked commands:**
```json
["chat", "agent", "help", "time", "timezone", "status", "mode", "zip", "unzip", "credits", "usage"]
```

Commands not in the plan's `unlocked_commands` list → bot replies:
```html
⊘ <b>Not available</b>

Ye command is group ke plan mein unlock nahi hai.

Group owner se contact karo ya /gplan dekho.
```

**Personal-only commands that redirect to DM from group:**
- `/buy`, `/premium`, `/topup`, `/referral`, `/settings`, `/credits` (personal balance)
- Bot replies: "Ye command DM mein use karo 👉" + deep link button

---

### 42.7 Group-Specific Errors & Edge Cases

| Scenario | Bot behavior |
|---|---|
| Bot added but NOT as admin | In-group: "Bot ko admin banao tab activate hoga" |
| Owner never DMed bot before | In-group welcome + deep link to DM |
| Group plan expired | Bot pauses → in-group notice to owner → DM reminder |
| Bot kicked from group | `left_chat_member` event → DB status: `removed` → log |
| Bot added back after removal | Treat as new group request — restart onboarding |
| Owner bans a member in group | That member gets bot-command blocked in that group only |
| Member tries to use agent without access | "Group plan mein Agent Mode nahi hai. /gplan dekho." |
| Group shared pool exhausted | "Group ka limit poora ho gaya. Kal reset hoga." + show reset time |
| Member personal plan > group plan | Group limits apply in group (no mixing) |
| Non-owner tries `/gban` | "Ye command sirf group owner ke liye hai." |
| Supergroup migration | `migrate_to_chat_id` event → update group_id in DB |

---

### 42.8 DB Tables for Groups

```sql
groups
  id                serial primary key
  tg_group_id       bigint unique not null
  group_name        varchar
  owner_tg_id       bigint not null
  plan_id           int references group_plans(id)
  status            varchar   -- pending, active, trial, expired, suspended, removed
  trial_ends_at     timestamp
  plan_expires_at   timestamp
  authorized_by     int       -- admin user id
  authorized_at     timestamp
  bot_is_admin      boolean default false
  created_at        timestamp
  updated_at        timestamp

group_plans
  id                serial primary key
  name              varchar
  price_inr         numeric
  price_usdt        numeric
  max_members_active int
  member_request_limit int
  member_window_hrs int
  group_shared_pool int       -- null = per-member mode
  agent_access      boolean
  file_ops_access   boolean
  unlocked_commands jsonb     -- ["chat","help","zip",...]
  owner_commands    jsonb     -- ["gban","glimit",...]
  trial_days        int default 0
  active            boolean default true
  created_at        timestamp

group_members
  id                serial primary key
  group_id          int references groups(id)
  user_tg_id        bigint
  joined_at         timestamp
  is_banned         boolean default false
  banned_at         timestamp
  custom_limit      int       -- null = use plan default
  last_seen_at      timestamp

group_usage_events
  id                serial primary key
  group_id          int references groups(id)
  user_tg_id        bigint
  request_id        varchar
  model_name        varchar
  route             varchar
  success           boolean
  created_at        timestamp

group_settings
  id                serial primary key
  group_id          int references groups(id)
  key               varchar
  value             text
  updated_at        timestamp
  -- Keys: 'welcome_message', 'default_mode', 'custom_limit_override', etc.
```

---

### 42.9 Admin Dashboard — Group Management

New section in admin web dashboard: **Groups**

| Column | Values |
|---|---|
| Group name | TG group name |
| Owner | username / TG ID |
| Members | active count |
| Plan | plan name |
| Status | pending · active · trial · expired · suspended |
| Expires | date |
| Actions | View · Approve · Assign plan · Suspend · Delete |

**Group detail page:**
- Group info: name, ID, owner, member count, plan, status
- Usage graph (requests/day)
- Member list: username · usage · banned status · custom limit
- Inline actions: ban member · reset limit · change plan · suspend group
- Group settings view/edit
- Plan expiry override

**Approve flow from dashboard:**
Admin clicks Approve → modal: select group plan + set trial days (0 = no trial) → Confirm → bot activates group + notifies owner.

---

### 42.10 Group system_config Keys

Added to admin Settings panel under new "Groups" tab:

| Key | Label | Type |
|---|---|---|
| `group.enabled` | Group bot system enabled | bool |
| `group.auto_approve` | Auto-approve all group requests | bool |
| `group.default_trial_days` | Default free trial days for new groups | int |
| `group.trial_plan_id` | Plan assigned during trial | int |
| `group.notify_admin_on_join` | Notify admin when bot added to group | bool |
| `group.max_members_per_group` | Global max members cap | int |
| `group.member_can_set_timezone` | Members can set their own timezone | bool |
| `group.dm_redirect_commands` | Commands that always redirect to DM | json |
| `group.free_trial_message` | Trial period announcement template | string |
| `group.expiry_warning_days` | Warn owner X days before expiry | int |

---

### 42.11 Launch Strategy — Groups

**Week 1–2 (Free trial period):**
- `group.auto_approve = true`
- `group.default_trial_days = 14`
- Trial plan: generous limits (set from admin panel)
- Every group that joins gets 14-day free, no questions asked

**Conversion (Week 3+):**
- 7 days before trial end → owner gets DM: "Trial ending in 7 days"
- 3 days before → reminder DM + in-group notice
- Trial ends → bot pauses in group → "Plan upgrade karo"
- Early adopter discount: configurable from admin panel (`group.early_adopter_discount_pct`)

**Personal plan upsell from groups:**
Group members who interact with bot → bot occasionally (not spammy — max once) sends DM:
```html
💡 <b>Personal Codex</b>

{group_name} mein bot use kar rahe ho?
Apna personal account banao — higher limits, referral rewards, aur zyada features.
```
Inline: `[ 🚀 Start personal account ]`

Frequency: once per member, after 3+ group interactions. Controlled by `system_config`.


---

## 43. Notification System

### Principle
Bot proactively DMs users for time-sensitive events. All notification messages are templates from `system_config` (Section 40, category: templates). All notification jobs run via Celery beat.

### Notification types

| Type | Trigger | Template key | Channel |
|---|---|---|---|
| Plan expiry warning | 7 days before expiry | `notif.plan_expiry_7d` | DM |
| Plan expiry warning | 3 days before expiry | `notif.plan_expiry_3d` | DM |
| Plan expiry warning | 1 day before expiry | `notif.plan_expiry_1d` | DM |
| Plan expired | On expiry | `notif.plan_expired` | DM |
| File expiry warning | 24hr after delivery | `notif.file_expiry_24h` | DM |
| File deleted | 48hr after delivery | `notif.file_deleted` | DM |
| Usage 80% warning | When 80% of window used | `notif.usage_warn_80` | DM |
| Usage 100% (limit hit) | Real-time on limit | `notif.usage_limit` | In-chat |
| Referral reward earned | On reward trigger | `notif.referral_reward` | DM |
| Coupon redeemed | On successful redeem | `notif.coupon_redeemed` | DM |
| Group trial ending | 7 days before group trial end | `notif.group_trial_7d` | DM to owner |
| Group trial ending | 1 day before group trial end | `notif.group_trial_1d` | DM to owner |
| Group plan expired | On group plan expiry | `notif.group_expired` | DM to owner + in-group |
| Group activated | On admin approval | `notif.group_activated` | DM to owner |
| USDT invoice expiring | 5 min before invoice TTL | `notif.usdt_expiring` | DM |
| Credits low | When credits < threshold | `notif.credits_low` | DM |

### DB Table: `notifications`
```sql
id            serial primary key
user_tg_id    bigint not null
type          varchar not null
template_key  varchar not null
context_json  jsonb            -- variables for template rendering
status        varchar          -- queued, sent, failed, skipped
sent_at       timestamp
failed_reason varchar
ref_id        varchar          -- plan_id, file_id, group_id etc.
created_at    timestamp
```

### Notification rules
- Max 3 DMs per day per user from bot (anti-spam)
- If user has blocked bot → `failed` status, no retry
- Each notification type has a `sent` flag in DB — no duplicates
- Credits-low threshold configurable from admin panel (`notif.credits_low_threshold`)
- All notification on/off toggleable per type from admin panel
- Admin can manually trigger any notification type from dashboard

### Celery beat schedule for notifications
```python
# Run every hour
check_plan_expiry_notifications()    # 7d, 3d, 1d warnings
check_group_trial_expiry()           # group trial warnings
check_file_expiry_warnings()         # 24hr file warning
check_credits_low()                  # credits below threshold
```

---

## 44. Credits Ledger System

### Principle
Credits are the internal currency. Every credit movement — earn, spend, refund, admin grant, expiry — is recorded in `credit_ledger` as an immutable append-only log. Current balance is always computed from ledger sum (or cached in `users.credits_balance`).

### Credit sources (earn)
| Source | Event | Amount | Configurable |
|---|---|---|---|
| `signup_bonus` | New user /start | X credits | Yes (admin panel) |
| `referral_signup` | Referred user starts | X credits to referrer | Yes |
| `referral_activation` | Referred user pays | X credits to referrer | Yes |
| `referral_retention` | Referred user active 30d | X credits to referrer | Yes |
| `admin_grant` | Admin manually grants | Any | — |
| `topup_purchase` | Top-up payment | Per plan | Yes |
| `promotion` | Admin promo | Any | — |

### Credit spend (debit)
| Source | Event | Amount | Configurable |
|---|---|---|---|
| `agent_task` | Agent mode request | X credits/task | Yes (admin panel) |
| `file_op` | Zip/unzip/merge/extract | X credits/op | Yes |
| `extra_request` | Over plan limit | X credits/request | Yes |
| `admin_revoke` | Admin revokes | Any | — |

### DB Table: `credit_ledger`
```sql
id            serial primary key
user_id       int references users(id)
delta         int not null          -- positive=earn, negative=spend
balance_after int not null          -- running balance snapshot
type          varchar not null      -- signup_bonus, topup, agent_task, admin_grant, etc.
ref_id        varchar               -- order_id, job_id, admin_action_id etc.
note          text                  -- human readable reason
created_at    timestamp not null
created_by    int                   -- null=system, user_id=admin
```

### Balance rules
- `users.credits_balance` is a cached field — updated on every ledger write
- If cache and ledger diverge → ledger wins (recalculate from sum)
- Negative balance not allowed — check before debit
- Admin grant: logged with `created_by = admin_id`
- Refund: positive delta with type `refund`, ref_id = original transaction

### Credits display
```html
💳 <b>Credits</b>
━━━━━━━━━━━━━━━
Balance:  <code>247 credits</code>
Plan:     <b>Pro Monthly</b>

▸ Agent task:  <code>10 credits</code>
▸ File op:     <code>5 credits</code>
━━━━━━━━━━━━━━━
```
Inline: `[ 💳 Top Up ]  [ 📊 Usage history ]`

### Credit history endpoint
`/credits` → shows last 10 ledger entries with type + delta + date. Paginated.

---

## 45. Subscription Expiry + Renewal Flow

### Expiry detection
Celery beat runs `check_subscription_expiry()` every hour.

```python
expired_subs = SELECT * FROM subscriptions
               WHERE end_at < now() AND status = 'active'
```

### On expiry — step by step

```
1. subscriptions.status → 'expired'
2. users.plan_id → free_plan_id
3. users.credits_balance → unchanged (credits remain)
4. Redis: invalidate cache:user:{id}
5. DM user: plan expiry notification (template: notif.plan_expired)
6. Log to admin_actions
7. If user had agent_access: agent mode blocked silently
8. If user in active job: job completes, next job blocked
```

### Grace period
Configurable from admin panel: `subscription.grace_period_hrs` (default: 0 — no grace).

If grace > 0:
- Subscription marked `grace` status
- User can still use plan features
- After grace period: full expiry flow above

### Renewal (manual — no auto-charge)
User must manually re-purchase. Bot sends:
```html
◈ <b>Plan expired</b>

Aapka <b>Pro Monthly</b> plan expire ho gaya.

Renew karo aur uninterrupted access pao:
```
Inline: `[ 🔄 Renew now ]  [ 💳 Top Up credits ]`

### Downgrade behavior
| Feature | After expiry |
|---|---|
| Chat (normal) | Still works (free limits apply) |
| Agent mode | Blocked — credits required |
| File size cap | Drops to free tier cap |
| Rate limit window | Resets to free tier (6hr) |
| Credits | Unchanged — remain usable |
| Referral | Still active |

### Subscription DB status values
`active` → `grace` (optional) → `expired` → `renewed` (new row created on repurchase)

---

## 46. Celery Beat Scheduler

### All scheduled tasks

| Task | Schedule | What it does |
|---|---|---|
| `check_file_expiry_warn` | Every 30 min | Find files where `delivered_at + 24h < now` and `warned_at IS NULL` → send warning DM → set `warned_at` |
| `delete_expired_files` | Every 30 min | Find files where `expires_at < now` and `deleted_at IS NULL` → delete from R2 → set `deleted_at` |
| `check_subscription_expiry` | Every 1 hour | Find active subs where `end_at < now` → run expiry flow |
| `check_plan_expiry_notify` | Every 6 hours | Find subs expiring in 7d/3d/1d → send DM if not already sent |
| `check_group_trial_expiry` | Every 6 hours | Find groups with trial ending in 7d/1d → DM owner |
| `check_group_plan_expiry` | Every 1 hour | Find expired group plans → run group expiry flow |
| `cleanup_expired_invoices` | Every 15 min | Find USDT invoices past TTL in Redis → release wallet slot → notify user |
| `cleanup_temp_jobs` | Every 1 hour | Find /tmp/codex-jobs folders older than 2hr → force delete |
| `check_credits_low` | Every 6 hours | Users with credits < threshold → DM warning (once per 24hr) |
| `check_referral_retention` | Every 24 hours | Find referred users active 30d → credit referrer retention bonus |
| `flush_usage_cache` | Every 1 hour | Sync Redis usage counters to PostgreSQL |
| `health_report` | Every 24 hours | Send daily health summary to admin_chat_id |
| `abuse_scan` | Every 1 hour | Check for abuse patterns → auto-flag → notify admin if threshold hit |

### Celery beat config
```python
CELERYBEAT_SCHEDULE = {
    'file-expiry-warn':       {'task': 'worker.check_file_expiry_warn',     'schedule': 1800},
    'delete-expired-files':   {'task': 'worker.delete_expired_files',       'schedule': 1800},
    'sub-expiry':             {'task': 'worker.check_subscription_expiry',  'schedule': 3600},
    'plan-expiry-notify':     {'task': 'worker.check_plan_expiry_notify',   'schedule': 21600},
    'group-trial-notify':     {'task': 'worker.check_group_trial_expiry',   'schedule': 21600},
    'group-plan-expiry':      {'task': 'worker.check_group_plan_expiry',    'schedule': 3600},
    'invoice-cleanup':        {'task': 'worker.cleanup_expired_invoices',   'schedule': 900},
    'temp-job-cleanup':       {'task': 'worker.cleanup_temp_jobs',          'schedule': 3600},
    'credits-low':            {'task': 'worker.check_credits_low',          'schedule': 21600},
    'referral-retention':     {'task': 'worker.check_referral_retention',   'schedule': 86400},
    'flush-usage-cache':      {'task': 'worker.flush_usage_cache',          'schedule': 3600},
    'daily-health-report':    {'task': 'worker.health_report',              'schedule': 86400},
    'abuse-scan':             {'task': 'worker.abuse_scan',                 'schedule': 3600},
}
```

### Idempotency rule
Every scheduled task must be idempotent — running it twice must not double-trigger actions. Use `warned_at`, `notified_at`, `deleted_at` flags as guards.

---

## 47. Language & i18n System

### Phase 1 scope
English only at launch. Hindi added in Phase 2. More languages via admin panel in Phase 3.

### How it works
- User sets language via `/language` wizard
- Saved in `user_settings.language` (e.g. `en`, `hi`)
- Bot fetches message by key: `template.{key}.{lang}` from `system_config`
- Fallback: if `hi` key missing → use `en`

### system_config keys for i18n
Each template has language variants:
```
template.welcome.en  →  "Welcome to Codex..."
template.welcome.hi  →  "Codex mein aapka swagat hai..."
template.rate_limit.en  →  "Limit reached..."
template.rate_limit.hi  →  "Aapka limit poora ho gaya..."
```

Admin can add/edit all translations from Settings → Templates panel (language tab selector).

### Language selection wizard
```
User: /language (.lang)

Bot:
🌐 <b>Language / भाषा</b>

Current: English

[ 🇬🇧 English ]  [ 🇮🇳 हिन्दी ]
```

After selection:
```
✔ Language set to हिन्दी

Ab bot Hindi mein jawab dega.
```

### Adding new languages
Admin dashboard → Settings → Templates → Add language → Enter language code → All template keys appear with blank fields → Admin fills translations → Save.

No code changes needed to add a new language.

---

## 48. AI Provider Rate Limit Handling

### Problem
Nvidia NIM and FreeModel both have rate limits. If bot hits provider rate limit, user gets a broken experience unless handled gracefully.

### Detection
Provider returns HTTP 429 or specific error code. Each adapter module detects this and raises `ProviderRateLimitError`.

### Handling strategy

```
Request → Provider call
  ↓
HTTP 429 / RateLimitError?
  ↓
  YES:
    1. Log: provider, user_id, timestamp, retry_after
    2. Check: can we fallback to cheaper route?
       - FreeModel rate limited → try Nvidia NIM (if task allows)
       - Nvidia NIM rate limited → queue request with delay
    3. If fallback possible → execute fallback silently
    4. If no fallback → queue request:
         Redis: rpush("retry_queue:{provider}", {user_id, prompt, retry_after})
         Notify user: "⠙ Slight delay — processing your request..."
    5. Celery task: retry after retry_after seconds (max 3 retries)
    6. If 3 retries fail → graceful failure message
  ↓
  NO: proceed normally
```

### User-facing message during provider rate limit
```html
⠙ <b>One moment...</b>

High demand — aapka request queue mein hai.
Est. wait: <code>~30 seconds</code>
```
(Edited to final response when ready)

### Failure message (after 3 retries)
```html
⚠ <b>Service temporarily busy</b>

Abhi provider unavailable hai. 2-3 minute baad try karo.

Credits deducted nahi kiye gaye.
```

### Tracking
All rate limit hits logged:
```json
{
  "event": "provider_rate_limit",
  "provider": "nvidia_nim",
  "user_id": 123,
  "retry_after": 30,
  "fallback_used": false,
  "resolved": true,
  "latency_total_ms": 31200
}
```

Admin dashboard → Logs → Filter: `provider_rate_limit` → see frequency + impact.

---

## 49. Structured Log Schema

### Log format
All logs are JSON, one object per line (JSON Lines). Written to stdout (captured by systemd/Docker) + optionally to `logs/app.log` with rotation.

### Base schema (every log entry)
```json
{
  "ts": "2024-01-15T10:30:00.123Z",
  "level": "INFO",
  "service": "bot",
  "event": "event_name",
  "request_id": "req_abc123",
  "user_id": 123456789,
  "chat_id": 123456789,
  "data": {}
}
```

`service` values: `bot` · `api` · `worker` · `scheduler`

### Event types

| Event | Level | When |
|---|---|---|
| `user_start` | INFO | /start called |
| `message_received` | DEBUG | Any message |
| `command_routed` | INFO | Command dispatched |
| `ai_request_start` | INFO | AI call initiated |
| `ai_request_complete` | INFO | AI response received |
| `ai_request_failed` | ERROR | AI call failed |
| `provider_rate_limit` | WARN | Provider 429 |
| `route_decision` | INFO | Routing layer decision |
| `job_created` | INFO | Worker job queued |
| `job_complete` | INFO | Worker job finished |
| `job_failed` | ERROR | Worker job error |
| `file_uploaded_r2` | INFO | File stored in R2 |
| `file_deleted_r2` | INFO | File removed from R2 |
| `file_expiry_warn_sent` | INFO | 24hr warning DM sent |
| `payment_invoice_created` | INFO | USDT invoice created |
| `payment_verified` | INFO | Tx verified |
| `payment_failed` | WARN | Tx verification failed |
| `coupon_redeemed` | INFO | Coupon activated |
| `plan_activated` | INFO | Plan granted to user |
| `plan_expired` | INFO | Plan expired |
| `referral_attributed` | INFO | Referral recorded |
| `referral_reward_granted` | INFO | Reward credited |
| `rate_limit_hit` | WARN | User hit rate limit |
| `abuse_flag_raised` | WARN | Abuse pattern detected |
| `admin_action` | INFO | Admin performed action |
| `notification_sent` | INFO | DM notification sent |
| `notification_failed` | WARN | DM blocked/failed |
| `group_joined` | INFO | Bot added to group |
| `group_activated` | INFO | Group plan assigned |
| `group_expired` | INFO | Group plan expired |
| `health_check` | INFO | /health endpoint called |
| `config_changed` | INFO | system_config updated |
| `scheduler_task_start` | DEBUG | Beat task begins |
| `scheduler_task_complete` | INFO | Beat task done |
| `error_unhandled` | ERROR | Uncaught exception |

### Example log entries
```json
{"ts":"2024-01-15T10:30:00Z","level":"INFO","service":"bot","event":"ai_request_complete","request_id":"req_abc123","user_id":789,"data":{"model":"nvidia-llama-3","route":"layer1","prompt_tokens":120,"response_tokens":340,"latency_ms":1240,"cost_est_usd":0.0003,"success":true}}

{"ts":"2024-01-15T10:30:05Z","level":"WARN","service":"bot","event":"rate_limit_hit","user_id":789,"data":{"tier":"free","window_hrs":6,"count":20,"limit":20,"reset_in_sec":14400}}

{"ts":"2024-01-15T10:31:00Z","level":"ERROR","service":"worker","event":"job_failed","request_id":"job_xyz","user_id":789,"data":{"job_type":"zip","error":"FileNotFoundError","retry_count":2,"will_retry":true}}
```

### Log rotation
- Max file size: 100MB
- Keep: 7 days of daily rotated files
- Compress: gzip older than 1 day
- Configured via `logging.handlers.RotatingFileHandler` or logrotate

---

## 50. Monitoring & Alerting

### Metrics to track (stored in `system_metrics` or external)

| Metric | Type | Alert threshold |
|---|---|---|
| AI cost today (USD) | Gauge | > `ai.cost_warn_threshold` from config |
| Error rate (errors/min) | Rate | > 5 errors/min |
| Queue depth | Gauge | > 50 jobs pending |
| Provider rate limit hits | Counter | > 10 in 10 min |
| Failed payments | Counter | > 3 in 1 hour |
| New abuse flags | Counter | > 5 in 1 hour |
| Bot webhook failures | Counter | Any |
| DB connection failures | Counter | Any |
| Redis connection failures | Counter | Any |
| Active USDT invoices | Gauge | > 20 (wallet pressure) |
| Group requests pending | Gauge | > 10 unreviewed |

### Alert channels
All alerts → admin_chat_id (Telegram message via bot).
Format:
```html
🚨 <b>Alert: {alert_name}</b>

Value:     <code>{current_value}</code>
Threshold: <code>{threshold}</code>
Time:      <code>{timestamp}</code>

Action: {recommended_action}
```

### Alert rules in system_config

| Key | Label |
|---|---|
| `alert.cost_threshold_usd` | Daily cost alert threshold |
| `alert.error_rate_per_min` | Error rate alert |
| `alert.queue_depth_max` | Queue depth alert |
| `alert.enabled` | Alerts on/off globally |
| `alert.cooldown_min` | Min minutes between same alert |

Alert cooldown prevents spam — same alert fires max once per `cooldown_min`.

### Daily health report (Celery beat — daily)
Sent to admin_chat_id every day at 08:00 UTC:
```html
📊 <b>Codex Daily Report</b>
━━━━━━━━━━━━━━━
👥 Users:       <code>1,240 total · 87 new</code>
💎 Paid:        <code>134 active subs · 23 top-ups</code>
🤖 AI requests: <code>4,820</code>
💰 Est. cost:   <code>$3.20</code>
📁 Files:       <code>89 processed</code>
🏢 Groups:      <code>12 active</code>
⚠ Errors:      <code>3</code>
━━━━━━━━━━━━━━━
```

---

## 51. Security Model

### Bot security
- Webhook endpoint validates `X-Telegram-Bot-Api-Secret-Token` header
- Admin commands: handler checks `user_id in ADMIN_TG_IDS` (from system_config, Redis-cached)
- Group owner commands: handler checks `user_id == groups.owner_tg_id`
- All admin checks fail-closed (deny by default)

### Admin dashboard security
- Single secret token auth (`ADMIN_DASHBOARD_SECRET` from env)
- Session cookie: 7-day expiry, `HttpOnly`, `Secure`, `SameSite=Strict`
- All dashboard API routes: `require_admin_session` middleware
- Rate limit dashboard login: 5 attempts per IP per 15 min
- All config changes require re-auth confirmation (password re-entry) for critical fields (bot token, webhook)

### Payment security
- USDT: tx hash dedup (DB unique constraint on `transactions.tx_hash`)
- USDT: amount exact match ±0
- USDT: wallet ownership verified (only our 10 wallets accepted)
- Coupon: format regex validation before DB lookup
- Coupon: one-time redeem enforced at DB level (unique constraint + status check)
- SellAuth: webhook signature verified via `SELLAUTH_SECRET`

### Data security
- No raw API keys stored in DB — only in env
- User TG IDs stored, no passwords (Telegram auth only)
- Admin actions always logged with `admin_id` + before/after values
- No PII beyond what Telegram provides (username, first_name, tg_id)

### DB security
- PostgreSQL: separate user for app (not superuser)
- App user: SELECT/INSERT/UPDATE/DELETE only — no DROP, no schema changes
- Migrations run separately (Alembic) with migration user
- Redis: requirepass set, not exposed publicly

---

## 52. DB Migration Strategy

### Tool: Alembic (SQLAlchemy)

### Rules
- Every schema change = new migration file
- Never edit existing migration — always add new
- Migration files committed to repo
- Run on deploy before starting app processes
- Rollback: `alembic downgrade -1`

### Migration workflow
```bash
# Generate new migration
alembic revision --autogenerate -m "add_group_settings_table"

# Apply
alembic upgrade head

# Rollback one step
alembic downgrade -1

# Check current version
alembic current
```

### Deploy order
```
1. Pull new code
2. alembic upgrade head
3. Restart bot process
4. Restart worker process
5. Reload Celery beat
```

Never reverse steps 2 and 3 — schema must match code before bot starts.

---

*Document version: 1.8 — COMPLETE*
*v1.0: Initial | v1.1: Unicode, aliases, buttons | v1.2: Merge | v1.3: Wizard, rolling limit, Tronscan, UPI DM, Admin dashboard | v1.4: Full admin control, system_config | v1.5: Group bot system | v1.6: Notifications, Credits, Celery, i18n, Logs, Monitoring, Security, Migrations | v1.8: Module architecture §53, group commands §11/29/30, phases updated §33, Notifications §43, Credits §44, Subscriptions §45, Celery §46, i18n §47, Provider limits §48, Logs §49, Monitoring §50, Security §51, Migrations §52*

---

## 53. Module Architecture (Standalone Modules)

### Principle
Codex is built as a **modular monorepo**. Each module is a self-contained Python package that can be extracted and used in any Telegram bot independently. The main `bot/` directory assembles these modules — it does not contain business logic itself.

### Repo structure
```
codex/
  modules/
    codex-pay/
    codex-coupon/
    codex-ratelimit/
    codex-referral/
    codex-credits/
    codex-files/
    codex-notify/
    codex-wizard/
    codex-router/
    codex-groups/
    codex-admin/
    codex-config/
    codex-unicode/
  bot/              ← assembles modules, no business logic
  docs/
  tests/
```

### Dependency rule
```
bot/ → imports from modules/
modules/ → can import codex-config only
modules/ → NEVER import from bot/ or other modules (except codex-config)
```

---

### Module specs

#### codex-pay
**What:** Complete payment adapter for Telegram bots.
**Includes:** USDT TRC20 (Tronscan), USDT ERC20 (Etherscan), SellAuth coupon sales, UPI DM redirect, wallet rotation (round-robin), invoice TTL (Redis), tx hash dedup.
**Exposes:**
```python
create_invoice(user_id, plan_id, chain) → Invoice
verify_tx(tx_hash, chain) → VerifyResult
rotate_wallet(chain) → str
```
**Standalone use:** Any bot needing crypto payment verification.
**Dependencies:** Redis, PostgreSQL, codex-config

---

#### codex-coupon
**What:** Coupon generation, validation, lifecycle management.
**Includes:** JHATKA format generator, regex validator, status lifecycle (available→assigned→redeemed→expired→disabled), source tracking (sellauth/manual), bulk import.
**Exposes:**
```python
generate_coupon(plan_id, source) → str
validate_coupon(code) → CouponResult
redeem_coupon(code, user_id) → RedeemResult
list_coupons(status, page) → List[Coupon]
```
**Standalone use:** Any bot selling digital products via coupon codes.
**Dependencies:** PostgreSQL, codex-config

---

#### codex-ratelimit
**What:** Rolling window rate limiter for Telegram bots.
**Includes:** 6-hour rolling window, per-tier limits, Redis key management, countdown display.
**Exposes:**
```python
check_limit(user_id, tier) → LimitResult  # allowed, remaining, reset_in
increment(user_id) → None
get_reset_time(user_id) → int  # seconds until reset
reset(user_id) → None
```
**Standalone use:** Any bot needing per-user rate limiting with rolling windows.
**Dependencies:** Redis, codex-config

---

#### codex-referral
**What:** Referral engine with fraud prevention and tier rewards.
**Includes:** Unique code generation, /start param attribution, self-referral block, reward triggers (signup/activation/retention), tier system (Bronze/Silver/Gold), burst fraud detection, monthly cap.
**Exposes:**
```python
generate_ref_code(user_id) → str
attribute_referral(new_user_id, ref_code) → AttributeResult
trigger_reward(referrer_id, event_type) → RewardResult
get_referral_stats(user_id) → ReferralStats
```
**Standalone use:** Any bot needing a referral growth engine.
**Dependencies:** PostgreSQL, Redis, codex-config

---

#### codex-credits
**What:** Immutable append-only credits ledger.
**Includes:** Earn/spend/refund tracking, running balance, admin grant/revoke, balance cache, history pagination.
**Exposes:**
```python
earn(user_id, amount, type, ref_id) → LedgerEntry
spend(user_id, amount, type, ref_id) → LedgerEntry
get_balance(user_id) → int
get_history(user_id, page) → List[LedgerEntry]
refund(user_id, ref_id) → LedgerEntry
```
**Standalone use:** Any bot with internal credits/coins/points system.
**Dependencies:** PostgreSQL, codex-config

---

#### codex-files
**What:** File operations worker system with cloud storage.
**Includes:** zip/unzip/merge/extract (local, no AI), Celery job queue, isolated temp folders (`/tmp/codex-jobs/{job_id}/`), R2 upload/delete, 48hr lifecycle, per-user concurrency cap (max 2 jobs).
**Exposes:**
```python
queue_job(user_id, type, files) → Job
get_job_status(job_id) → JobStatus
cancel_job(job_id) → None
upload_to_r2(file_path, job_id) → str  # r2_key
delete_from_r2(r2_key) → None
```
**Standalone use:** Any bot needing file processing with cloud storage.
**Dependencies:** Redis (queue), PostgreSQL, R2, Celery, codex-config

---

#### codex-notify
**What:** Template-based notification system via Celery beat.
**Includes:** Plan expiry warnings (7d/3d/1d), file expiry DMs, usage 80% warning, group trial warnings, credits-low alert, anti-spam (max 3 DMs/day), dedup tracking.
**Exposes:**
```python
send_notification(user_tg_id, type, context) → None
schedule_notification(user_tg_id, type, send_at, context) → None
mark_sent(notif_id) → None
```
**Standalone use:** Any bot needing proactive user DM notifications.
**Dependencies:** PostgreSQL, Redis, codex-config, Celery beat

---

#### codex-wizard
**What:** Conversational step-by-step command wizard engine.
**Includes:** Multi-step wizard state (Redis), Back button, Cancel anytime, timeout (5 min), power-shorthand bypass (if params present → skip wizard), inline button navigation.
**Exposes:**
```python
start_wizard(user_id, wizard_name, steps) → WizardState
get_step(user_id) → WizardStep
advance(user_id, data) → WizardStep | WizardComplete
cancel(user_id) → None
back(user_id) → WizardStep
```
**Standalone use:** Any bot needing multi-step guided command flows.
**Dependencies:** Redis, codex-config

---

#### codex-router
**What:** 4-layer AI routing with provider adapters.
**Includes:** Layer 0 (local), Layer 1 (Nvidia NIM), Layer 2 (FreeModel), Layer 3 (LangGraph), complexity gate, fallback chain, per-request cost logging.
**Exposes:**
```python
route(user_id, message, mode) → RouteDecision
call_provider(route, prompt) → ProviderResponse
get_cost_estimate(model, tokens) → float
```
**Standalone use:** Any bot needing multi-provider AI routing with cost control.
**Dependencies:** Nvidia API, FreeModel API, LangGraph, codex-config

---

#### codex-groups
**What:** Telegram group management system for bots.
**Includes:** Group onboarding flow, admin approval, group plans (per-member/shared pool), owner commands (/gban, /glimit, /gbroadcast etc.), member usage tracking, trial period, expiry flow.
**Exposes:**
```python
register_group(tg_group_id, owner_tg_id) → Group
activate_group(group_id, plan_id) → None
check_member_limit(group_id, user_tg_id) → LimitResult
get_group_stats(group_id) → GroupStats
ban_member(group_id, user_tg_id) → None
```
**Standalone use:** Any bot that needs to work in Telegram groups with per-group plans.
**Dependencies:** PostgreSQL, Redis, codex-config

---

#### codex-admin
**What:** Standalone admin web dashboard.
**Includes:** Dark theme HTML/JS, Material Icons, responsive (mobile + PC), user management, coupon management, broadcast (create/preview/send/delete), settings panel (all system_config keys), chat interface (admin→user GPT-style), logs viewer, system health.
**Exposes:** FastAPI router (`/admin/*` routes), mounts as sub-app.
**Standalone use:** Any FastAPI-based Telegram bot needing an admin panel.
**Dependencies:** FastAPI, PostgreSQL, codex-config

---

#### codex-config
**What:** Dynamic configuration manager — single source of truth.
**Includes:** `system_config` DB table, Redis 60s cache, typed get/set, admin save → cache invalidate, before/after logging.
**Exposes:**
```python
get(key, default=None) → Any        # type-cast by value_type
set(key, value, admin_id) → None    # saves + invalidates cache
get_category(category) → dict       # all keys in a category
```
**Standalone use:** Any bot needing runtime-configurable settings without redeploy.
**Dependencies:** PostgreSQL, Redis

---

#### codex-unicode
**What:** Unicode animation and UI symbol system.
**Includes:** Braille spinner frames, block progress bar, stage templates (AI/file/agent), clock spinner, tier badge renderer, separator generator, message edit loop helper.
**Exposes:**
```python
get_spinner_frame(index) → str
render_progress(percent) → str
render_stage(stage_name, stage_list) → str
render_tier_badge(tier) → str
animate(bot, message, frames, interval) → None  # async edit loop
```
**Standalone use:** Any Telegram bot wanting animated Unicode UI.
**Dependencies:** Aiogram (for animate helper), none for renderers

---

### Module dependency graph
```
codex-unicode    → (none)
codex-config     → PostgreSQL, Redis
codex-ratelimit  → codex-config, Redis
codex-credits    → codex-config, PostgreSQL
codex-coupon     → codex-config, PostgreSQL
codex-referral   → codex-config, PostgreSQL, Redis
codex-wizard     → codex-config, Redis
codex-notify     → codex-config, PostgreSQL, Redis, Celery
codex-pay        → codex-config, PostgreSQL, Redis
codex-files      → codex-config, PostgreSQL, Redis, R2, Celery
codex-imagegen   → codex-config, PostgreSQL, Redis, Nvidia NIM, Google AI
codex-router     → codex-config
codex-groups     → codex-config, PostgreSQL, Redis
codex-admin      → codex-config, PostgreSQL (all tables read)
bot/             → all modules
```

---

### Each module contains
```
codex-{name}/
  README.md         ← what it does, how to install, example usage
  setup.py          ← pip installable
  codex_{name}/
    __init__.py     ← public interface (exposes only what's needed)
    core.py         ← main logic
    models.py       ← DB models (if needed)
    config_keys.py  ← list of system_config keys this module uses
    exceptions.py   ← module-specific exceptions
  tests/
    test_core.py
  example_usage.py  ← standalone example for GitHub readers
```

---

### GitHub portfolio positioning

Each module ships as a **separate GitHub repository** under a `codex-bot` organization (or `your-username/codex-*` repos), plus one umbrella repo `codex` that assembles them.

| Repo | What it shows |
|---|---|
| `codex-pay` | Crypto payment verification, blockchain API integration |
| `codex-coupon` | Digital goods access control system |
| `codex-ratelimit` | Redis-based rate limiting engine |
| `codex-referral` | Growth engine with fraud prevention |
| `codex-credits` | Append-only financial ledger pattern |
| `codex-files` | Async file processing with cloud storage |
| `codex-imagegen` | Multi-provider AI image generation + NSFW filter |
| `codex-router` | Multi-provider AI routing with cost control |
| `codex-groups` | Multi-tenant bot system |
| `codex-admin` | Full-stack admin dashboard |
| `codex-wizard` | Conversational UX engine |
| `codex` (umbrella) | System design, integration, monorepo |

**Recruiter reads:** 11+ production-quality Python packages, each solving a real problem, each usable independently.


---

*Document version: 1.9 — COMPLETE*
*v1.0–1.2: Initial + merge | v1.3: Wizard, rolling limit, Tronscan, UPI, Admin dashboard | v1.4: system_config full control | v1.5: Group bot system | v1.6: Notifications, credits ledger, subscription expiry, Celery beat, i18n, provider rate limits, log schema, monitoring, security, DB migrations | v1.7: /register system, personal vs group credits separated, source tracking, group analytics, pCloud constraint corrected | v1.8: Phase 2 complete — LangGraph agent, file ops, referral rewards, group onboarding, i18n EN+HI, monitoring alerts | v1.9: Image generation (/draw), Phase 2.5 corrections (registration flow, group-context-only owner commands, codex_config cache type fix, rate limit decrement, plan slug DB resolution, file plan_type check)*
