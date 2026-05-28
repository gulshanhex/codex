# CODEX — AI Engineer Prompts
## 3-Phase Build Prompts · PRD v1.8

---

# ═══════════════════════════════
# PHASE 1 PROMPT — MVP LAUNCH
# ═══════════════════════════════

```
You are a senior Python engineer building CODEX — a
production-grade Telegram AI bot platform.

Read every word carefully. Follow every rule exactly.
No assumptions. No shortcuts. No placeholders.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
TECH STACK (LOCKED — DO NOT DEVIATE)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Language:        Python 3.11+
Bot framework:   Aiogram 3.x (NOT python-telegram-bot)
API server:      FastAPI
Task queue:      Celery + Redis broker
Database:        PostgreSQL (asyncpg + SQLAlchemy async)
ORM:             SQLAlchemy 2.0 async + Alembic migrations
Cache:           Redis (aioredis)
File storage:    Cloudflare R2 (boto3 S3-compatible)
AI — Chat:       Nvidia NIM API
AI — Agent:      FreeModel API (Phase 2)
Workflow:        LangGraph (Phase 2)
Deployment:      Single VPS, systemd or docker-compose
Logging:         Structured JSON (see log schema below)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
MONOREPO STRUCTURE (LOCKED)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Build as a modular monorepo. Every feature lives in its
own module under /modules/. The bot/ directory ONLY
assembles modules — zero business logic in bot/.

/codex
  /modules
    /codex_config       ← system_config + Redis cache
    /codex_unicode      ← animations, symbols, progress bars
    /codex_wizard       ← conversational step wizard engine
    /codex_ratelimit    ← 6hr rolling window rate limiter
    /codex_credits      ← append-only credits ledger
    /codex_coupon       ← JHATKA coupon system
    /codex_pay          ← USDT + SellAuth + UPI redirect
    /codex_referral     ← referral engine + fraud controls
    /codex_files        ← file ops + R2 + Celery jobs
    /codex_notify       ← notification system
    /codex_router       ← 4-layer AI routing
    /codex_groups       ← group bot system
    /codex_admin        ← web dashboard (FastAPI sub-app)
  /bot
    /handlers           ← Aiogram handlers (thin — call modules)
    /middlewares        ← alias, auth, rate limit check
    router.py           ← main dispatcher
  /migrations           ← Alembic
  /tests
  main.py               ← FastAPI app + bot startup

Each module has:
  __init__.py           ← public interface only
  core.py               ← business logic
  models.py             ← SQLAlchemy models (if needed)
  config_keys.py        ← system_config keys this module reads
  exceptions.py         ← module-specific exceptions
  README.md             ← standalone usage docs
  example_usage.py      ← copy-paste example for GitHub

Dependency rule:
  modules → may import codex_config only
  modules → NEVER import from bot/ or other modules
  bot/    → imports from modules freely

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
ABSOLUTE RULES (NEVER BREAK)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
1. NO hardcoded prices, limits, messages, or settings
   anywhere. Everything reads from codex_config
   (system_config DB table, Redis 60s cache).

2. Only these values live in ENV:
   BOT_TOKEN, WEBHOOK_URL, WEBHOOK_SECRET, BOT_MODE,
   POSTGRES_URL, REDIS_URL,
   R2_ACCESS_KEY, R2_SECRET_KEY, R2_BUCKET, R2_ENDPOINT,
   NVIDIA_API_KEY, FREEMODEL_API_KEY,
   SELLAUTH_SECRET, TRONSCAN_API_KEY, ETHERSCAN_API_KEY,
   BOOTSTRAP_ADMIN_TG_ID, ADMIN_DASHBOARD_SECRET

3. File operations NEVER run in webhook loop.
   Always via Celery worker.

4. HTML parse mode EVERYWHERE. Never Markdown.

5. Every action logged with uuid4 request_id.
   Log format: {"ts","level","service","event",
   "request_id","user_id","data"} — JSON Lines.

6. Admin checks always fail-closed (deny by default).

7. No hard deletes ever — status updates only.

8. Dot alias normalized before routing:
   if text.startswith('.'): text = '/' + text[1:]

9. All bot messages use templates from system_config.
   Zero hardcoded user-facing strings in handlers.

10. Each module must work standalone — test it in
    isolation before wiring into bot/.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
BUILD ORDER — PHASE 1
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

STEP 1 — codex_config module
  system_config DB table (id, category, key, value,
  value_type, label, description, updated_by, updated_at)
  get(key, default=None) → typed value
  set(key, value, admin_id) → None
  get_category(category) → dict
  Redis cache: config:{key} TTL 60s
  Admin save → DEL cache key → fresh read
  Bootstrap seeder: insert all default values on first boot

STEP 2 — codex_unicode module
  Spinner frames: ⠋⠙⠹⠸⠼⠴⠦⠧⠇⠏
  Progress bar: █████░░░░░ {pct}%
  Stage renderer: ◈ / ◈◈ / ◈◈◈ etc.
  Tier badge: ⟨ FREE ⟩ / ⟨ PRO ⟩ / ⟨ ELITE ⟩
  Separator: ━━━━━━━━━━━━━━━
  animate(bot, msg, frames, interval=1.0) async helper
  All renderers pure functions — no Aiogram dependency
  except animate()

STEP 3 — codex_wizard module
  Redis: session:{user_id} → {wizard, step, data} TTL 300s
  start_wizard(user_id, name, steps) → WizardState
  get_step(user_id) → WizardStep
  advance(user_id, data) → WizardStep | WizardComplete
  cancel(user_id) → None
  back(user_id) → WizardStep
  Power shorthand bypass: if params present → skip wizard

STEP 4 — Database + Alembic
  Create all tables in one initial migration:
  users, plans, subscriptions, credit_ledger,
  coupons, transactions, wallets, referrals,
  usage_events, jobs, files, notifications,
  broadcasts, admin_actions, abuse_flags,
  user_settings, model_route_logs, groups,
  group_plans, group_members, group_usage_events,
  group_settings, system_config, pricing_config
  
  Separate DB user for app: SELECT/INSERT/UPDATE/DELETE
  Migration user for schema changes only.

STEP 5 — codex_ratelimit module
  Rolling window per user_id + tier.
  Redis keys:
    ratelimit:{user_id}:window_start  TTL = window_hrs * 3600
    ratelimit:{user_id}:count
  check(user_id, tier) → {allowed, remaining, reset_in_sec}
  increment(user_id) → None
  reset(user_id) → None
  Tier configs read from codex_config:
    limit.free.window_hrs, limit.free.requests
    limit.sub.window_hrs, limit.sub.requests
    limit.topup.window_hrs, limit.topup.requests
    limit.both.window_hrs, limit.both.requests

STEP 6 — codex_credits module
  Append-only ledger. Never update/delete rows.
  earn(user_id, amount, type, ref_id, note) → LedgerEntry
  spend(user_id, amount, type, ref_id) → LedgerEntry
    → raises InsufficientCreditsError if balance < amount
  get_balance(user_id) → int
  get_history(user_id, page=1, per_page=10) → List
  refund(user_id, original_ref_id) → LedgerEntry
  users.credits_balance updated on every write (cached field)
  If cached balance ≠ ledger sum → ledger wins

STEP 7 — codex_coupon module
  Format: JHATKA-[A-Z0-9]{16}-CODEX
  Generator: secrets.token_urlsafe → uppercase → slice 16
  validate(code) → CouponResult (exists, status, plan_id)
  redeem(code, user_id) → RedeemResult
    → status: available → assigned → redeemed
    → inline confirm screen before final redeem
  list_coupons(status, source, page) → paginated
  disable(code) → None
  expire(code) → None
  No hard deletes — status field only

STEP 8 — codex_pay module
  USDT TRC20:
    verify_trc20(tx_hash, expected_amount, wallet)
    GET https://apilist.tronscan.org/api/transaction-info?hash={tx_hash}
    Check: amount exact, wallet match, confirmations >= 1
  USDT ERC20:
    verify_erc20(tx_hash, expected_amount, wallet)
    Etherscan free API — isolated adapter
  Both adapters in separate files — easy to swap
  Wallet rotation: round-robin from wallets table
    (5 TRC20 + 5 ERC20, managed from admin panel)
  Invoice lifecycle:
    create_invoice(user_id, amount, chain) → Invoice
    Redis: invoice:{user_id} TTL 1800s
    expire_invoice(user_id) → None
  Tx hash dedup: DB unique constraint on transactions.tx_hash
  UPI: show UPI ID + amount + "Open Support DM" deep link
    No bot-side UPI verification. Manual admin flow only.

STEP 9 — codex_referral module
  generate_code(user_id) → str (unique, stored in users.referral_code)
  attribute(new_user_id, ref_code) → AttributeResult
    → self-referral block (new_user_id == referrer_id → reject)
    → already attributed → reject
  trigger_reward(referrer_id, event):
    events: signup_bonus | activation_bonus | retention_bonus
    amounts: from codex_config referral.* keys
    check: monthly cap, cooldown hours
    fraud: burst detection → abuse_flag
  get_stats(user_id) → {tier, referrals, rewards, pending}
  Tiers: Bronze/Silver/Gold thresholds from codex_config

STEP 10 — Bot skeleton + middlewares
  Aiogram app:
    BOT_MODE=webhook → FastAPI /webhook endpoint
    BOT_MODE=polling → Aiogram long poll
  Middlewares (in order):
    1. alias.py: dot → slash normalization
    2. auth.py: load user from DB, check is_banned
    3. ratelimit.py: call codex_ratelimit.check()
  Admin command scope: separate BotCommandScope
    → admin commands NEVER appear in user menu

STEP 11 — Core handlers (thin — call modules only)
  /start → parse ref_ param → store in Redis → welcome template
  /register → deprecated (access on /start now)
  /help, /menu, /status → static templates from codex_config
  /credits → codex_credits.get_balance()
  /referral → codex_referral.get_stats()
  /timezone, /language → codex_wizard flow
  /redeem → codex_coupon.redeem()
  /buy → codex_wizard flow → codex_pay
  /gen → admin only → codex_wizard → codex_coupon.generate()

STEP 12 — Chat Mode (Nvidia NIM)
  bot.send_chat_action("typing")
  codex_ratelimit.check() → blocked? → rate limit screen
  Send editable message → codex_unicode.animate()
  POST https://integrate.api.nvidia.com/v1/chat/completions
    model: codex_config.get("ai.chat.model")
    max_tokens: codex_config.get("ai.chat.max_tokens")
  Edit message through stages → final response
  Log usage_event (model, tokens, latency, cost_est)
  Provider HTTP 429 → log provider_rate_limit →
    retry once after 2s → fallback if ai.fallback_enabled

STEP 13 — R2 file lifecycle (Celery beat)
  File delivered → store r2_key + expires_at in files table
  Every 30 min:
    files where warned_at IS NULL AND
      expires_at - now() < 24h
      → DM warning → set warned_at
    files where deleted_at IS NULL AND
      expires_at < now()
      → boto3 delete → set deleted_at
  Temp cleanup every 1hr:
    /tmp/codex-jobs folders older than 2hr → force delete

STEP 14 — Admin bot commands
  /ban, /unban, /grant, /revoke → inline confirm
  /gen → codex_wizard → codex_coupon.generate()
  /import → paste SellAuth codes → bulk insert
  /list {status} → paginated coupon list
  /stats → platform snapshot
  /broadcast → wizard → preview → Celery send task
  All actions logged to admin_actions table

STEP 15 — codex_admin module (Phase 1 scope)
  FastAPI sub-app mounted at /admin
  Auth: ADMIN_DASHBOARD_SECRET → session cookie
    HttpOnly, Secure, SameSite=Strict, 7-day expiry
    Login rate limit: 5 attempts per IP per 15min
  Phase 1 sections:
    Overview: users, errors, queue, cost
    Users: search, view, grant, ban
    Coupons: list, gen, disable, expire
    Settings: all system_config keys by category tab
      Each row: label, value, input, Save
      Save → admin_actions log + Redis DEL
    System health: DB / Redis / Worker / Webhook

  GET /health → {"status","db","redis","worker","webhook"}
  Degraded → HTTP 503

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
LOG SCHEMA (ALL SERVICES)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
{"ts":"ISO8601","level":"INFO","service":"bot",
 "event":"event_name","request_id":"uuid4",
 "user_id":123,"data":{}}

Log rotation: 100MB max, 7-day retention, gzip.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
DO NOT BUILD IN PHASE 1
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
LangGraph / Agent Mode
Group bot system
File merge/extract (zip/unzip only)
Referral reward tiers (store only, no rewards yet)
i18n multi-language
Abuse auto-scoring
SellAuth webhook
codex_notify (Phase 1: only inline rate-limit messages)
```

---

# ═══════════════════════════════
# PHASE 2 PROMPT — MONETIZATION + GROUPS
# ═══════════════════════════════

```
Phase 1 is complete and live.
All Phase 1 modules working. Do NOT modify them
unless fixing a confirmed bug.

Now build Phase 2. Read every rule. No shortcuts.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
INSTALL BEFORE STARTING
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
pip install langgraph langchain langchain-core
pip install celery[redis] boto3

Phase 1 packages already installed.
Do NOT reinstall or upgrade them.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
BUILD ORDER — PHASE 2
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

STEP 1 — codex_router module (Agent Mode)
  4-layer routing:
    Layer 0: local deterministic → never calls AI
    Layer 1: Nvidia NIM → normal chat (codex_config: ai.chat.model)
    Layer 2: FreeModel → coding + agentic
    Layer 3: LangGraph → deep agent (premium only)

  LangGraph StateGraph nodes:
    intent_router → task_planner → tool_selector
    → executor → validator → responder

  Gate: user must be in agent mode AND have
    credits >= codex_config.get("ai.agent.min_credits")

  Credits deducted BEFORE task starts via codex_credits.spend()
  Refund via codex_credits.refund() on failure.

  Allowed agent tasks from codex_config (ai.agent.allowed_tasks JSON):
    zip, extract, merge, code_run, file_convert,
    summarize, web_search, code_review, data_parse,
    regex_gen, json_format, text_transform

  Animation stages (editable message via codex_unicode):
    ◈ Analyzing intent...
    ◈◈ Building plan...
    ◈◈◈ Selecting tools...
    ◈◈◈◈ Executing steps...
    ◈◈◈◈◈ Validating output...
    ✔ Complete

  Fallback chain: FreeModel fail → Nvidia NIM → error msg
  Provider rate limit: see PRD Section 48

STEP 2 — codex_files module (all 4 ops)
  Per-user concurrency cap:
    codex_config.get("file.max_concurrent_jobs") → default 2
  
  Wizard flow for each op (via codex_wizard):
    User /zip → "Send files, then /done" → confirm → queue
  
  Worker job flow:
    Create job record → enqueue Celery task
    Worker: /tmp/codex-jobs/{job_id}/input/ work/ output/
    Progress bar via codex_unicode.render_progress(pct)
    Upload to R2 → bot.send_chat_action("upload_document")
    Deliver → cleanup temp folder (always, even on fail)
  
  Queue display: ◎ Queued · Position: #N

STEP 3 — codex_notify module
  Celery beat tasks (see PRD Section 46 for full schedule):
    check_file_expiry_warn    → every 30 min
    delete_expired_files      → every 30 min
    check_subscription_expiry → every 1 hour
    check_plan_expiry_notify  → every 6 hours
    check_group_trial_expiry  → every 6 hours
    cleanup_expired_invoices  → every 15 min
    cleanup_temp_jobs         → every 1 hour
    check_credits_low         → every 6 hours
    check_referral_retention  → every 24 hours
    flush_usage_cache         → every 1 hour
    health_report             → every 24 hours (08:00 UTC)
    abuse_scan                → every 1 hour

  All notification templates from codex_config.
  Max 3 DMs/day per user (anti-spam guard in codex_notify).
  Idempotency: use warned_at / notified_at flags — never
  trigger same notification twice.

STEP 4 — Subscription + plan system (full)
  Plan activation on payment:
    Insert subscriptions row, update user cache
  
  Expiry (codex_notify Celery beat):
    sub.end_at < now() → status=expired
    users.plan_id → free_plan_id
    credits unchanged
    Redis: invalidate cache:user:{id}
  
  Grace period: codex_config.get("subscription.grace_period_hrs")
  Renewal: user re-purchases → new subscriptions row
    (stack days, don't reset)

STEP 5 — codex_referral rewards + tiers
  Trigger signup_bonus on attribution
  Trigger activation_bonus on first payment
  Trigger retention_bonus via Celery beat (30-day check)
  Tiers from codex_config thresholds:
    referral.bronze_threshold / silver / gold
  Anti-fraud: monthly cap + cooldown enforced

STEP 6 — codex_groups module
  new_chat_members event → bot added:
    1. Store group in DB (status=pending)
    2. Send hardcoded in-group welcome + deep link
       "[ 🚀 Activate this group ]"
       → t.me/{botname}?start=group_{group_id}
    3. Notify admin_chat_id (from codex_config)

  Owner DMs bot via deep link:
    /start group_{group_id} param detected
    → show group activation request screen (hardcoded)
    → admin notified with [Approve] [Reject] inline

  Admin approves → assign plan → trial starts
    group.auto_approve from codex_config:
      true → auto-approve with default trial plan
      false → manual approval required

  Two quota modes per group plan:
    Per-member: each member has own limit
    Shared pool: group has total pool, members draw from it

  Personal credits and group quota NEVER mix.
  In group context → group quota applies.
  In DM → personal credits apply.

  Owner commands (unlocked per plan's owner_commands JSON):
    /gstatus /gusage /gplan /ghelp → group + DM
    /glimit /gban /gunban /gmembers /gmode
    /gwelcome /greset /gbroadcast /gsettings /gupgrade → DM only
  
  Commands not in plan's unlocked_commands:
    → "⊘ Ye command is plan mein unlock nahi hai"

  Bot kicked → left_chat_member → status=removed
  Supergroup migration → migrate_to_chat_id → update DB

STEP 7 — codex_admin Phase 2 additions
  New section: Groups
    Table: group name, owner, members, plan, status, expires
    Group detail: info, usage graph, member list, actions
    Approve flow: modal → plan + trial days → confirm → bot activates

  New section: Broadcasts (full)
    Create, preview, send, schedule, delete
    message_ids[] stored per broadcast for delete support
    Delete: bot.delete_message() for each → DB: deleted_at
    After 48hr: delete fails silently → status=delete_failed

  Enhanced: Chat Interface
    GPT-style sidebar: user list
    Main: full bot conversation thread
    Admin can send message to any user as bot
    Material icon toolbar: send, user info, ban, grant

STEP 8 — i18n (EN + HI)
  user_settings.language = 'en' | 'hi'
  Template key format: template.{key}.{lang}
  Fallback: missing hi key → use en
  /language wizard → codex_wizard flow
  All new templates added as .en and .hi variants in
  system_config bootstrap seeder

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
DO NOT BUILD IN PHASE 2
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Team plans
Priority queue
Automated abuse scoring
SellAuth webhook automation
Microservices
```

---

# ═══════════════════════════════
# PHASE 3 PROMPT
# ═══════════════════════════════

```
Phase 2.5 codebase zip: codex_phase2_5_complete.zip (attached)
PRD reference: CODEX_PRD_v19.md (in project docs)

This zip contains complete Phase 1 + 2 + 2.5 codebase.
Do NOT ask for Phase 1 or Phase 2 separately — everything is here.

Do NOT touch working Phase 1/2/2.5 code unless
fixing a confirmed bug.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
WHAT IS ALREADY BUILT (DO NOT REBUILD)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Phase 1:
  Bot core, /start + /register, chat mode (Nvidia NIM),
  6-hr rolling rate limit, coupon system (JHATKA format),
  USDT TRC20/ERC20 + UPI payment, admin bot commands,
  admin web dashboard, codex_config (system_config + Redis cache),
  R2 file lifecycle (48hr warn+delete), Celery beat,
  structured JSON logging, GET /health endpoint

Phase 2:
  LangGraph agent mode (FreeModel), file ops
  (zip/unzip/merge/extract), group bot system,
  referral reward tiers (Bronze/Silver/Gold),
  subscription management + expiry, i18n (EN+HI),
  monitoring + Telegram alerts, daily health report,
  admin dashboard: Groups panel, Group Analytics,
  Chat Interface (admin→user), Broadcast with delete

Phase 2.5 (corrections):
  /register system (access gated), group owner commands
  group-context-only, codex_config cache type fix,
  rate limit decrement on failure, plan slug DB resolution,
  plan_type DB check in files, image generation (/draw):
    free tier → free HTTP API, paid → Google Imagen 3,
    prompt enhancement via Nvidia NIM, NSFW filter,
    daily limits per tier from system_config,
    image_jobs table, codex_imagegen module

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
ABSOLUTE RULES — NEVER BREAK
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
1. Zero hardcoded values. Everything from system_config.
2. No cross-module imports except codex_config.
3. File ops = Celery only. Never webhook loop.
4. HTML parse mode everywhere. Never Markdown.
5. uuid4 request_id on every logged action.
6. Admin checks fail-closed.
7. No hard deletes — status updates only.
8. Every new system_config key:
   → Add to seeder with default
   → Add to module's config_keys.py
   → Add to admin dashboard settings tab

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
BUILD ORDER — PHASE 3
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

STEP 1 — SellAuth webhook automation
  File: modules/codex_pay/webhooks.py (new)
  Route: POST /webhooks/sellauth
  Verify SELLAUTH_SECRET HMAC-SHA256 signature on request body.
  Reject with 403 if signature mismatch.
  Parse coupon code from payload.
  Call coupon_mod.single_create() — same logic as manual /import.
  Status: available, source: sellauth, order_ref from payload.
  Return 200 immediately — process async if needed.
  Log: coupon_imported_webhook event with request_id.
  Add to main.py: app.add_route("/webhooks/sellauth", ...)
  system_config keys:
    sellauth.webhook_enabled   bool  true
    sellauth.default_plan_slug string pro_monthly

STEP 2 — Anti-abuse auto-scoring
  File: modules/codex_admin/abuse.py (new)
  New column: users.abuse_score int default 0
  New Alembic migration: 004_abuse_score.py

  Score increments (all from system_config):
    +1  abuse_flag raised          (abuse.score.flag)
    +2  refund request             (abuse.score.refund)
    +3  referral fraud detected    (abuse.score.referral_fraud)
    +5  USDT chargeback attempt    (abuse.score.chargeback)

  Auto-ban check after every increment:
    threshold = system_config.get("abuse.auto_ban_threshold", 10)
    enabled   = system_config.get("abuse.auto_ban_enabled", False)
    if enabled and score >= threshold:
      → ban user, log admin_action (actor=system)
      → DM user: "Account suspended. Contact support."

  Admin review queue in dashboard:
    New section: "Abuse Queue"
    Table: user, score, flags, last_flag_at, actions
    Inline: [View User] [Ban] [Clear Score] [Dismiss]

  Expose from module:
    increment_score(user_id, reason, amount) → new_score
    get_queue(min_score, limit) → list[UserAbuseRow]
    clear_score(user_id, admin_id) → None

STEP 3 — Priority queue
  Two Celery queues:
    codex-priority   ← premium users (sub or sub+topup)
    codex-standard   ← free + topup users

  system_config keys:
    queue.premium_queue_name   string  codex-priority
    queue.standard_queue_name  string  codex-standard
    queue.max_standard_depth   int     50
      (if standard queue depth > this, free users get wait message)

  In codex_files/core.py — before enqueue:
    tier = get_user_tier(db_user)
    if tier in ("sub", "both"):
        queue_name = await get("queue.premium_queue_name")
    else:
        # Check standard queue depth
        depth = get_standard_queue_depth()
        if depth > await get("queue.max_standard_depth"):
            raise QueueFullError("Queue full. Premium users bypass.")
        queue_name = await get("queue.standard_queue_name")
    task.apply_async(queue=queue_name, ...)

  In celery_app.py — declare both queues:
    app.conf.task_queues = [
        Queue("codex-priority", routing_key="priority"),
        Queue("codex-standard", routing_key="standard"),
    ]

  In worker startup: run two worker processes:
    celery -A worker.celery_app worker -Q codex-priority --concurrency=4
    celery -A worker.celery_app worker -Q codex-standard --concurrency=2

STEP 4 — Worker separation (deployment only — zero code change)
  Move Celery worker processes to separate VPS or container.
  Workers connect via same REDIS_URL + POSTGRES_URL env vars.
  Bot webhook process stays on main VPS.
  Redis + PostgreSQL stay on main VPS.

  docker-compose.worker.yml (new file):
    services:
      worker-priority:
        build: .
        command: celery -A worker.celery_app worker
                   -Q codex-priority --concurrency=4
        env_file: .env
      worker-standard:
        build: .
        command: celery -A worker.celery_app worker
                   -Q codex-standard --concurrency=2
        env_file: .env
      beat:
        build: .
        command: celery -A worker.celery_app beat
        env_file: .env

  No code changes needed. Worker reads from Redis queue,
  writes results to PostgreSQL. Both services share same DBs.

STEP 5 — Advanced analytics dashboard
  File: modules/codex_admin/analytics.py (new)
  New dashboard section: "Analytics"

  Charts (all use Chart.js — already in admin static):

  A) Revenue by plan type
     Source: transactions table, group by plan_id
     Display: bar chart — Free / Pro Monthly / Pro Yearly / Top-up
     Date range filter: 7d / 30d / 90d / all

  B) AI cost breakdown by model
     Source: usage_events, group by model_name
     Display: pie chart + table — model, requests, total_cost_usd
     Date range filter

  C) Retention curves (D1, D7, D30)
     Source: users + usage_events
     D1  = users who used bot day after signup
     D7  = users who used bot 7 days after signup
     D30 = users who used bot 30 days after signup
     Display: line chart over time

  D) Referral conversion funnel
     Source: referrals table
     Stages: Signup → Activated → Retention bonus
     Display: funnel chart with counts + %

  E) Group conversion ranking
     Source: users (source_type='group') + transactions
     Display: table — Group Name | Registered | Paid | %
     Sort by paid conversion %

  Export all as CSV:
    GET /admin/api/analytics/export?type=revenue&range=30d
    Returns: CSV file download

STEP 6 — Team plans
  New tables (Alembic migration 005_teams.py):
    teams:
      id, name, owner_user_id, credits_pool,
      max_members, created_at, status

    team_members:
      id, team_id, user_id, role (owner/member),
      per_member_limit, usage_today,
      joined_at, removed_at

  system_config keys:
    team.default_max_members    int    5
    team.credits_pool_default   int    500

  New commands:
    /team         → Team dashboard
    /team invite  → Generate invite link
    /team kick    → Remove member
    /team usage   → Member usage breakdown

  Plan type: 'team' (new value in plans.type)
  Credits drawn from teams.credits_pool, not user balance.
  Usage tracked per member in team_members.usage_today.
  Celery beat: reset team_members.usage_today daily.

  Admin dashboard:
    New section "Teams": list, create, manage, assign credits

STEP 7 — Advanced group features
  A) Per-member usage analytics (owner view in group)
     /gstats → table of members: requests today, total, last_seen
     Source: group_usage_events per group_id

  B) Group leaderboard (most active members)
     /gleaderboard → top 10 members by request count this week
     Updates weekly via Celery beat

  C) Win-back campaign
     Celery beat — daily check:
       groups where status=expired AND plan_expires_at < now() - 3 days
       AND win_back_sent IS NULL
     → DM to owner:
       "Your group trial ended. Renew with 20% discount: [code]"
     → Generate 20% discount coupon (system_config: group.winback_discount_pct)
     → Set win_back_sent = now() on group record

  New column: groups.win_back_sent timestamp nullable

STEP 8 — pCloud activation (optional — config switch only)
  pCloud adapter already stubbed in codex_files.
  system_config key already exists: storage.primary = 'r2' | 'pcloud'
  
  Verify stub is complete:
    codex_files/pcloud_adapter.py must have:
      upload(file_path, dest_key) → str (url)
      download(key) → bytes
      delete(key) → None

  In codex_files/core.py — delivery path:
    primary = await get("storage.primary", "r2")
    if primary == "pcloud":
        url = await pcloud_adapter.upload(output_path, r2_key)
    else:
        url = await r2_adapter.upload(output_path, r2_key)

  No deploy needed — Redis cache invalidation triggers switch.
  Test: set storage.primary = 'pcloud' in admin panel → verify.

STEP 9 — Multi-language expansion
  Current: EN + HI hardcoded in system_config seeder.
  Phase 3: Admin can add any new language code from dashboard.

  system_config key pattern:
    template.{lang_code}.{template_key}

  Admin dashboard — Languages section:
    List all detected language codes
    [ + Add language ] → enter lang code (e.g. "bn", "ta", "ur")
    System auto-creates blank keys for all existing template keys
    Admin fills translations for each key → save
    User sets /language → gets that language's templates

  In codex_notify/core.py and all template renderers:
    lang = db_user.language or "en"
    key = f"template.{lang}.{template_key}"
    text = await get(key) or await get(f"template.en.{template_key}")
    (Fallback to English if translation missing)

  No code change needed per new language — admin dashboard only.

STEP 10 — AI smart queue + provider health
  File: modules/codex_router/health.py (new)

  Per-provider health score in Redis:
    provider:nvidia:health_score    → float 0.0-1.0 (TTL: 5 min)
    provider:freemodel:health_score → float 0.0-1.0 (TTL: 5 min)
    provider:nvidia:rate_limit_hits → int (TTL: 10 min)
    provider:freemodel:rate_limit_hits → int (TTL: 10 min)

  Health score update after every provider call:
    Success: score = min(1.0, score + 0.1)
    429 error: score = max(0.0, score - 0.3)
               increment rate_limit_hits
    Timeout:  score = max(0.0, score - 0.2)

  Routing prefers healthier provider:
    if nvidia.health < 0.3 and freemodel.health > 0.5:
        use freemodel as Layer 1 fallback
    Log: provider_health_routing event

  Exponential backoff on 429:
    retry_after = resp.headers.get("Retry-After", 5)
    wait = min(retry_after * (2 ** retry_count), 60)
    Log: provider_rate_limit event per hit

  Alert trigger (existing monitoring):
    if rate_limit_hits > system_config.get("alert.rate_limit_hits_threshold"):
        → send Telegram alert to admin_chat_id

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
MODULE PUBLISHING (AFTER PHASE 3)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
After Phase 3 is complete, each module gets published
as a standalone pip package.

For each module in /modules/:

1. Final README.md:
   - What it does (2 sentences)
   - pip install codex-{name}
   - Quick start (10-line example)
   - Config keys it uses
   - Full API reference (all public functions)

2. pyproject.toml:
   name = "codex-{name}"
   version = "1.0.0"
   install_requires = [minimal deps only — not entire Codex stack]

3. example_usage.py:
   Complete runnable example.
   No Codex-specific imports — standalone.
   Works with just: pip install codex-{name}

4. GitHub: separate repo per module under codex-bot org
   Plus umbrella: codex-bot/codex (this monorepo)

Module list (13 total):
  codex-config    codex-unicode    codex-wizard
  codex-ratelimit codex-credits    codex-coupon
  codex-pay       codex-referral   codex-files
  codex-imagegen  codex-router     codex-groups
  codex-admin
```

---

# ═══════════════════════════════
# CONTEXT MANAGEMENT RULES
# (Read before every Phase session)
# ═══════════════════════════════

```
1. Each step = can be its own conversation if complex.
   Never mix unrelated steps in one session.

2. When continuing existing code:
   - Show only the module/file being edited
   - Do NOT repaste entire files
   - State: "Editing modules/codex_pay/webhooks.py — adding X"

3. When adding a feature:
   - Confirm which module it belongs to
   - Check: does this module need a new system_config key?
     → Add to seeder with sensible default
     → Add to module's config_keys.py
     → Add to admin dashboard settings tab (correct category)
   - Check: new DB columns need Alembic migration
     → New file: migrations/versions/00N_description.py
     → Never edit existing migrations

4. Module independence check (before every new file):
   → Does this module import from another module? STOP.
   → Only codex_config import allowed between modules.
   → bot/ handlers can import from any module — that's fine.

5. Token saving rules:
   - Never repaste DB schema (already built — reference by table name)
   - Never repaste full existing files
   - Skip boilerplate — write only new/changed logic
   - For Alembic: write upgrade() + downgrade() only

6. Value rule:
   → Every configurable value: system_config.get(key, default)
   → If key doesn't exist: add to seeder + config_keys.py
   → Never hardcode plan IDs, limits, amounts, messages

7. File operation rule:
   → Any heavy processing? → Celery worker
   → Webhook handler? → queue the task, return immediately

8. Alembic migration naming:
   004_abuse_score.py
   005_teams.py
   006_group_winback.py
   (sequential — never skip, never reuse)

9. Admin dashboard additions:
   → New section? → add to nav sidebar in codex_admin/app.py
   → New settings tab? → add category to settings panel
   → New chart? → use Chart.js (already loaded in admin static)

10. google_imagen.py note:
    model.generate_images() is sync — runs in async context.
    Phase 3 fix if image gen gets heavy:
      loop = asyncio.get_event_loop()
      result = await loop.run_in_executor(None, sync_call)

11. After every step — verify:
    → No new hardcoded values introduced
    → No cross-module imports added
    → New system_config keys in seeder + config_keys.py
    → Alembic migration created for any schema change
    → Admin dashboard updated if user-visible feature added
```
