# 🔐 Found Core — Paid Telegram Community Platform

[![Banner](assets/banner.png)](https://github.com/Maksim-Volosh/found-core-app)

![Python](https://img.shields.io/badge/Python-3.12-blue?logo=python&logoColor=white)
![FastAPI](https://img.shields.io/badge/FastAPI-0.137-009688?logo=fastapi&logoColor=white)
![Aiogram](https://img.shields.io/badge/Aiogram-3.29-2CA5E0?logo=telegram&logoColor=white)
![Pydantic](https://img.shields.io/badge/Pydantic-2.13-E92063?logo=pydantic&logoColor=white)
![SQLAlchemy](https://img.shields.io/badge/SQLAlchemy-2.0-D71F00)
![Alembic](https://img.shields.io/badge/Alembic-1.18-6BA539)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-15-4169E1?logo=postgresql&logoColor=white)
![Stripe](https://img.shields.io/badge/Stripe-checkout-635BFF?logo=stripe&logoColor=white)
![CryptoBot](https://img.shields.io/badge/CryptoBot-invoices-F7931A?logo=telegram&logoColor=white)
![APScheduler](https://img.shields.io/badge/APScheduler-3.11-lightgrey)
![Docker](https://img.shields.io/badge/Docker-ready-2496ED?logo=docker&logoColor=white)
![Architecture](https://img.shields.io/badge/Architecture-Clean--Architecture-yellowgreen)

<br>

## 📚 Table of Contents

- [🎓 About](#-about)
- [📌 Features](#-features)
- [🧱 Tech Stack](#-tech-stack)
- [📁 Project Structure](#-project-structure)
- [🏗️ Architecture Layers](#️-architecture-layers)
- [🔁 Data Flows](#-data-flows)
- [💾 Database Schema](#-database-schema)
- [🚀 Getting Started](#-getting-started)
- [🔌 Telegram and Webhook Setup](#-telegram-and-webhook-setup)
- [⚙️ Environment Variables Reference](#️-environment-variables-reference)
- [📚 API Reference](#-api-reference)
- [🤖 Bot Reference](#-bot-reference)
- [🧭 Roadmap](#-roadmap)

<br>

## 🎓 About

**Found Core** is the backend + Telegram bot behind a paid community: people pay a recurring subscription to unlock a main Telegram chat and a set of themed sub-chats called **directions** (e.g. topic-specific rooms), some of which additionally require manual **screening** (curator approval) before access is granted.

The system is split into two cooperating services:

- **`app/`** — a **FastAPI** backend, structured as a clean/hexagonal architecture, that owns PostgreSQL, all business rules (pricing, subscriptions, access), and two payment integrations (**Stripe** checkout and **CryptoBot** crypto invoices).
- **`bot/`** — an **aiogram 3** Telegram bot that is the *only* client of that backend. It never touches the database directly — every action a user takes in Telegram is translated into an HTTP call to `app/`.

A background **APScheduler** job inside the backend runs every 3 hours to expire lapsed subscriptions (kicking users from every chat they had access to) and to send 7-day / 3-day renewal reminders.

<br>

## 📌 Features

- **Clean Architecture layering** — `domain` (pure entities/interfaces) → `application` (use cases) → `infrastructure` (SQLAlchemy, payment providers, Telegram calls) → `api` (thin FastAPI routers), wired per-request by a plain `Container` composition root
- **Level-based pricing** — each user has a `level` (1–10) that maps to a price via a configurable price matrix; levels 8–10 are free
- **Dual payment providers** — Stripe Checkout Sessions and CryptoBot invoices, selected per request, with a 20% discount for 3+ month purchases and automatic reuse of a still-fresh pending payment
- **Signature-verified webhooks** — Stripe (`stripe-signature` + `stripe.Webhook.construct_event`) and CryptoBot (`crypto-pay-api-signature` + HMAC-SHA256) both verified before a payment is ever marked paid
- **Subscription lifecycle automation** — an APScheduler job (every 3 hours) revokes access for expired subscriptions and sends one-shot 7-day/3-day renewal reminders
- **Screening gate** — a global screening flag (checked before a user can even start a purchase) plus a per-direction screening flag for sub-chats that need curator approval
- **In-chat direction registration** — admins turn any Telegram group into a paid "direction" with a `/register` command run *inside* that group, protected by a shared secret key
- **Self-expiring, single-use invite links** — access to the main chat and to directions is granted via one-time Telegram invite links (5–30 minutes), re-validated the moment the user actually joins
- **Full admin panel inside the bot** — user browsing/search, ban/unban, level changes, screening toggles, comped subscription grants, per-direction access control, and direction editing, gated by an aiogram middleware
- **Superadmin tier** — a narrower role on top of admin, currently scoped to promoting/demoting other admins

<br>

## 🧱 Tech Stack

| Technology | Description |
|---|---|
| 🐍 Python 3.12 | Runtime for both the backend and the bot (single shared `requirements.txt`) |
| ⚡ FastAPI 0.137 | HTTP API framework (routers, dependency injection, OpenAPI/Swagger) |
| 🧩 Pydantic v2 / pydantic-settings | DTOs/schemas, nested `.env`-driven settings |
| 🗄️ SQLAlchemy 2.0 (async) | Async ORM over PostgreSQL via `asyncpg` |
| 🧬 Alembic | Versioned database migrations |
| 🐘 PostgreSQL 15 | Primary relational database |
| 🤖 Aiogram 3.29 | Telegram bot framework (long polling, FSM, middlewares) |
| 💳 Stripe SDK | Hosted Checkout Sessions for card payments |
| 💠 CryptoBot (Crypto Pay API) | Crypto invoices via raw `aiohttp` calls to `pay.crypt.bot` |
| ⏱️ APScheduler | In-process cron-style job for subscription expiry/reminders |
| 🌐 aiohttp | Async HTTP client used by the bot to call the backend, and by the CryptoBot provider |
| 🐳 Docker / Docker Compose | Containerized backend + PostgreSQL + Redis stack |
| 🧱 Clean Architecture | Layering: domain → application → infrastructure → api |

<br>

## 📁 Project Structure

```
.
├── alembic
│   ├── env.py
│   ├── README
│   ├── script.py.mako
│   └── versions
│       ├── 2026_07_03_1210-b1c2ae265884_create_database.py
│       └── 2026_07_09_1521-23e5d13ed5c0_add_screening_status_to_user.py
├── alembic.ini
├── app
│   ├── api
│   │   └── v1
│   │       ├── __init__.py
│   │       ├── mappers
│   │       │   ├── direction.py
│   │       │   └── user.py
│   │       ├── routers
│   │       │   ├── access.py
│   │       │   ├── admin.py
│   │       │   ├── direction.py
│   │       │   ├── payment.py
│   │       │   ├── user.py
│   │       │   └── webhook.py
│   │       └── schemas
│   │           ├── access.py
│   │           ├── auth.py
│   │           ├── direction.py
│   │           ├── __init__.py
│   │           ├── payment.py
│   │           └── user.py
│   ├── application
│   │   ├── services
│   │   │   └── payment.py
│   │   └── use_cases
│   │       ├── access.py
│   │       ├── admin.py
│   │       ├── clear_expired.py
│   │       ├── direction.py
│   │       ├── __init__.py
│   │       ├── payment.py
│   │       ├── send_reminder.py
│   │       ├── user.py
│   │       └── webhook.py
│   ├── core
│   │   ├── composition
│   │   │   ├── container.py
│   │   │   └── di.py
│   │   └── config.py
│   ├── domain
│   │   ├── entities
│   │   │   ├── direction.py
│   │   │   ├── __init__.py
│   │   │   ├── payment.py
│   │   │   ├── subscription.py
│   │   │   ├── user.py
│   │   │   └── user_subscription.py
│   │   ├── exceptions
│   │   │   ├── direction.py
│   │   │   ├── __init__.py
│   │   │   ├── payment.py
│   │   │   └── user.py
│   │   ├── interfaces
│   │   │   ├── bot.py
│   │   │   ├── direction.py
│   │   │   ├── __init__.py
│   │   │   ├── payment_provider.py
│   │   │   ├── payment.py
│   │   │   ├── subscription.py
│   │   │   └── user.py
│   │   └── mappers
│   │       └── user.py
│   ├── infrastructure
│   │   ├── helpers
│   │   │   └── db
│   │   │       ├── db_helper.py
│   │   │       └── __init__.py
│   │   ├── mappers
│   │   │   ├── direction.py
│   │   │   ├── payment.py
│   │   │   ├── subscription.py
│   │   │   └── user_mapper.py
│   │   ├── models
│   │   │   ├── base.py
│   │   │   ├── direction.py
│   │   │   ├── __init__.py
│   │   │   ├── payment.py
│   │   │   ├── subscription.py
│   │   │   └── user.py
│   │   ├── payment_providers
│   │   │   ├── crypto_bot_provider.py
│   │   │   └── stripe_provider.py
│   │   ├── repositories
│   │   │   ├── direction.py
│   │   │   ├── __init__.py
│   │   │   ├── payment.py
│   │   │   ├── subscription.py
│   │   │   ├── telegram_bot.py
│   │   │   └── user.py
│   │   └── schedulers
│   │       └── subscription.py
│   └── main.py
├── app.log
├── app_scheduler.log
├── assets
│   └── banner.png
├── bot
│   ├── config.py
│   ├── run.py
│   └── src
│       ├── api
│       │   ├── api_client.py
│       │   └── services
│       │       ├── access.py
│       │       ├── admin.py
│       │       ├── auth.py
│       │       ├── direction.py
│       │       ├── __init__.py
│       │       ├── payment.py
│       │       └── user.py
│       ├── container.py
│       ├── handlers
│       │   ├── access.py
│       │   ├── admin_create_direction.py
│       │   ├── admin.py
│       │   ├── back.py
│       │   ├── __init__.py
│       │   ├── payment.py
│       │   ├── profile.py
│       │   ├── start.py
│       │   └── support.py
│       ├── http
│       │   ├── errors.py
│       │   └── http_client.py
│       ├── keyboards
│       │   └── keyboards.py
│       ├── middlewares
│       │   ├── admin.py
│       │   └── auth.py
│       └── states
│           └── admin.py
├── CLAUDE.md
├── docker-compose.yaml
├── Dockerfile
├── dump.py
└── requirements.txt

37 directories, 108 files
```

> `app.log` / `app_scheduler.log` are runtime log files (not committed intentionally, just present in this checkout), and `dump.py` is an empty scratch file — neither is part of the application.

> If you're reviewing this project: start from `app/application/use_cases/` and follow the dependencies outward into `domain/` (what it needs) and `infrastructure/` (how it's actually fetched/stored).

<br>

## 🏗️ Architecture Layers

The backend follows a Clean Architecture-inspired layering where dependencies only ever point **inward**, toward the domain:

```
┌──────────────────────────────────────────────────────────────────┐
│  api/                                                            │
│  FastAPI routers · request/response schemas · entity↔schema      │
│  mappers — no business logic, just translate HTTP ↔ use cases    │
└───────────────────────────────┬──────────────────────────────────┘
                                │ calls
┌───────────────────────────────▼──────────────────────────────────┐
│  application/                                                    │
│  use_cases (business workflows) · services (shared workflow      │
│  logic, e.g. "mark payment paid & extend subscription")          │
└───────────────────────────────┬──────────────────────────────────┘
                                │ depends only on interfaces (ports)
┌───────────────────────────────▼──────────────────────────────────┐
│  domain/                                                         │
│  entities (dataclasses) · interfaces (IUserRepository,           │
│  IPaymentProvider, IBotService, …) · exceptions — zero I/O       │
└───────────────────────────────▲──────────────────────────────────┘
                                │ implements the ports
┌───────────────────────────────┴──────────────────────────────────┐
│  infrastructure/                                                 │
│  SQLAlchemy models & repositories · Stripe/CryptoBot providers   │
│  the scheduler · the backend's own Telegram bot client           │
└──────────────────────────────────────────────────────────────────┘
                                ▲
                                │ wires a concrete graph per request
                    core/composition/Container  (app/core/composition)
```

- **`core/composition/container.py`** defines `Container` — a plain class (not a DI framework) instantiated once per request with an `AsyncSession`. Its `get_*_use_case()` / `*_repo()` methods assemble concrete infrastructure objects behind domain interfaces.
- **`core/composition/di.py`** exposes `get_container` as a FastAPI dependency, so every router just does `container: Container = Depends(get_container)`, calls one use case, and maps the result to a response schema.
- **`domain/` never imports `infrastructure/`.** Every cross-layer call goes through an interface in `app/domain/interfaces/` (`IUserRepository`, `ISubscriptionRepository`, `IPaymentRepository`, `IDirectionRepository`, `IPaymentProvider`, `IBotService`).
- The **payment provider** is chosen at runtime inside `Container._get_payment_provider`, based on `PaymentProviderType.STRIPE`/`CRYPTO` — adding a new provider means implementing `IPaymentProvider` and registering it there.

The `bot/` side mirrors this in miniature: `bot/src/container.py` is its own composition root wiring one `*Service` per backend resource, each calling out through a single shared `HTTPClient`.

<br>

## 🔁 Data Flows

#### 1) Every Telegram update → authenticate against the backend first

```
Message / CallbackQuery
  → AuthMiddleware (bot/src/middlewares/auth.py)
    → POST /user/auth  { telegram_id, username, first_name, last_name }
      → UserAuthUseCase.auth
        → user not found?  → create it            (implicit registration)
        → user.is_banned?  → raise UserIsBanned    → 403
        → name/username changed? → sync them
        → return user + latest subscription
  ← injects backend_user_id / is_user_admin / is_user_superadmin
  → handler runs with that context
```

**Result:** there is no explicit "sign up" step — the first message a Telegram user ever sends creates their backend record, and every single update is re-authenticated and re-authorized before any handler code runs.

---

#### 2) Buy or renew a subscription

```
🥉 1 month / 🥈 2 months / 🥇 3 months (-20%)
  → 💳 Stripe
    → POST /payment/create  { user_id, months, provider_type: STRIPE }
      → CreatePaymentUseCase
        → price = price_matrix[user.level] × months
        → months >= 3 ?  price -= 20%
        → a fresh PENDING payment for this (provider, months, amount)?
             CRYPTO: reused if < 14 min old · STRIPE: reused if < 12 h old
        → StripePaymentProvider.create_checkout_session(...)
        → persist PENDING payment, return checkout_url
  → user pays on Stripe's hosted page
  → POST /webhook/stripe   (header: stripe-signature)
    → stripe.Webhook.construct_event(...)  — bad signature → 400
    → event == "checkout.session.completed"
      → ProcessSuccessfulPaymentService
        → mark payment PAID (idempotent if already PAID)
        → active subscription exists? expires_at += 30 days
        → else create a new subscription, now → now + 30 days
```

**Note:** fulfilment always grants **30 days per successful payment**, regardless of how many months were purchased — see [Roadmap](#-roadmap).

---

#### 3) Get access to a chat (main community or a direction)

```
🔄 Проверить оплату  /  🚀 Основное сообщество  /  a direction button
  → GET /access/main?user_id=…              (or a direction's screening check)
      → CheckMainAccessUseCase:
          admin?               → allowed
          banned?              → denied
          screening pending?   → denied
          price(level) == 0?   → allowed  (levels 8–10 are free)
          subscription ACTIVE? → allowed
  → allowed:
      bot.unban_chat_member(chat_id, user_id)
      bot.create_chat_invite_link(chat_id, member_limit=1, expire_date=+5–30 min)
      → send the single-use link to the user
  → user taps the link and joins
      → chat_member JOIN_TRANSITION handler re-checks access
          still allowed? → welcome message (auto-deleted after 5s)
          not allowed?   → bot.ban_chat_member(...) + DM explaining why
```

**Result:** a leaked invite link is not enough by itself — every join is independently re-validated against the backend at the moment it happens.

---

#### 4) Subscription expiry & reminders (every 3 hours, in-process scheduler)

```
APScheduler "check_subscriptions_expired"  (interval: 3h)
  → ClearExpiredSubscriptionsUseCase
      for each subscription where status=ACTIVE and expires_at < now:
        bot.ban_user(telegram_id, main_chat_id)
        bot.ban_user(telegram_id, chat_id)   for every registered direction
        all bans succeeded?
          yes → subscription.status = EXPIRED, DM the user
          no  → leave ACTIVE, retried next run
  → SendSubscriptionRemindersUseCase
      expires_at <= now+7d and !reminded_7_days → DM "7 days left", set flag
      expires_at <= now+3d and !reminded_3_days → DM "3 days left",  set flag
```

**Note:** both reminder flags are reset whenever a payment successfully extends an active subscription.

<br>

## 💾 Database Schema

PostgreSQL via async SQLAlchemy 2.0, migrated with Alembic (2 migrations, head `23e5d13ed5c0`). Constraint names follow a shared naming convention defined on `Base.metadata` (`app/infrastructure/models/base.py`) so Alembic autogenerate produces consistent `ix_/uq_/fk_/pk_` names.

```
user  (PK user_id, unique telegram_id)
 │
 ├──1:N── subscription        (PK subscription_id, FK user_id ON DELETE CASCADE)
 ├──1:N── payment              (PK payment_id, FK user_id ON DELETE CASCADE,
 │                               unique provider_payment_id)
 └──1:N── user_direction_access (PK user_direction_access_id,
                                  FK user_id ON DELETE CASCADE)
                                  │
                                  └──N:1── direction (PK telegram_chat_id)
```

| Table | Key columns | Notes |
|---|---|---|
| **`user`** | `user_id` PK · `telegram_id` unique+indexed · `username` · `first_name`/`last_name` · `level` (default `1`) · `is_banned`/`is_admin`/`is_superadmin` · `screening_status` (`NOT_STARTED`\|`APPROVED`) · `created_at`/`updated_at` | The single identity record; created on first bot interaction |
| **`subscription`** | `subscription_id` PK · `user_id` FK · `started_at` · `expires_at` · `status` (`ACTIVE`\|`EXPIRED`\|`CANCELLED`) · `reminded_7_days`/`reminded_3_days` | No unique constraint — a user can have multiple rows; "current" = latest by `expires_at` |
| **`payment`** | `payment_id` PK · `user_id` FK · `amount` (cents) · `months` · `currency` · `status` (`PENDING`\|`PAID`\|`FAILED`\|`CANCELLED`, indexed) · `provider` (`STRIPE`\|`CRYPTO`) · `provider_payment_id` unique+indexed · `provider_checkout_url` | `provider_payment_id` is the idempotency key webhooks look up by |
| **`direction`** | `telegram_chat_id` PK (the Telegram chat ID itself) · `name` · `owner_username` · `requires_screening` | Registered in-chat via `/register`, see [Telegram and Webhook Setup](#-telegram-and-webhook-setup) |
| **`user_direction_access`** | `user_direction_access_id` PK · `user_id` FK · `telegram_chat_id` FK (indexed) · `screening_status` | Per-user, per-direction screening state; no unique constraint on the pair, deduplication is enforced in application code |

<br>

## 🚀 Getting Started

#### ✅ Clone the repository
---

```bash
git clone https://github.com/Maksim-Volosh/found-core-paid-bot.git
cd found-core-paid-bot
```

#### ⚙️ Configure the backend environment
---

Settings load from `.env.template` first, then `.env` (later file wins), using nested keys with prefix `APP_CONFIG__` and `__` as the delimiter (`SettingsConfigDict(env_nested_delimiter="__", env_prefix="APP_CONFIG__")` in `app/core/config.py`).

`.env.template` only covers the core/DB/cache settings — **8 required settings are not in it** (Stripe, Telegram and CryptoBot config have no defaults and the app will fail to start without them). Create `.env` with everything filled in:

```env
# Core
APP_CONFIG__DETAILS__TITLE=Found Core API
APP_CONFIG__DETAILS__DESCRIPTION=API for the payment system
APP_CONFIG__SECURITY__BOT_API_KEY=change-me
APP_CONFIG__API__PREFIX=/api
APP_CONFIG__RUN__HOST=0.0.0.0
APP_CONFIG__RUN__PORT=8000
APP_CONFIG__RUN__RELOAD=True

# Database (PostgreSQL)
APP_CONFIG__DB__URL=postgresql+asyncpg://postgres_user:postgres_password@db:5432/found_core_db
APP_CONFIG__DB__ECHO=0
APP_CONFIG__DB__ECHO_POOL=False
APP_CONFIG__DB__POOL_SIZE=5
APP_CONFIG__DB__MAX_OVERFLOW=10

# Cache (Redis) — required to boot, see Roadmap
APP_CONFIG__CACHE__URL=redis://redis:6379/0

# Stripe
APP_CONFIG__STRIPE__API_KEY=sk_test_...
APP_CONFIG__STRIPE__WEBHOOK_SECRET=whsec_...
APP_CONFIG__STRIPE__SUCCESS_URL=https://t.me/your_bot
APP_CONFIG__STRIPE__CANCEL_URL=https://t.me/your_bot

# CryptoBot (Crypto Pay API)
APP_CONFIG__CRYPTO_BOT__API_KEY=...
APP_CONFIG__CRYPTO_BOT__IS_TESTNET=True

# Telegram (used by the backend's own Bot client — bans, DMs)
APP_CONFIG__TELEGRAM__BOT_TOKEN=123456:...
APP_CONFIG__TELEGRAM__MAIN_CHAT_ID=-100...
```

Also give the `db` service in `docker-compose.yaml` its Postgres credentials — it reads `env_file: .env`, so add:

```env
POSTGRES_USER=postgres_user
POSTGRES_PASSWORD=postgres_password
POSTGRES_DB=found_core_db
```

#### 🚀 Build and start the backend
---

```bash
docker compose up --build
```

Then, in a separate terminal, apply migrations:

```bash
docker compose exec found_core_app alembic upgrade head
```

- API docs: `http://localhost:8000/docs`
- OpenAPI JSON: `http://localhost:8000/openapi.json`

Running without Docker works the same way, against a Postgres instance you provide:

```bash
pip install -r requirements.txt
alembic upgrade head
python -m app.main
```

#### ⚙️ Configure and run the bot
---

The bot is not in `docker-compose.yaml` — run it as a separate process. It reads flat (non-nested) env vars from `bot/.env` via `python-dotenv`:

```env
BOT_TOKEN=123456:your-botfather-token
API_URL=http://localhost:8000/api/v1
API_KEY=change-me
MAIN_CHAT_ID=-100...
SECRET_REGISTER_KEY=some-shared-secret
```

```bash
cd bot
python run.py
```

> The bot's modules import with `from config import ...` / `from src... import ...` (no package prefix), so it **must** be launched with `bot/` as the working directory.

<br>

## 🔌 Telegram and Webhook Setup

- **Add the bot as an administrator** to the main community chat and to every chat you intend to register as a direction — it needs rights to ban/unban members and to create invite links.
- **`MAIN_CHAT_ID`** is the numeric ID of the main chat (negative for supergroups); forward a message from that chat to `@userinfobot` or check the bot's own logs on first contact to find it.
- **Register a direction**: run `/register <SECRET_REGISTER_KEY> <owner_username> <true|false> <Direction Name>` **inside** the group chat you want to register — `message.chat.id` becomes the direction's `telegram_chat_id`. The command message is deleted either way (to avoid leaking the key), and a confirmation is posted and auto-deleted after a few seconds.
- **Stripe webhook**: point a Stripe webhook endpoint at `POST https://your-domain/api/v1/webhook/stripe`, subscribed at least to `checkout.session.completed`, using the signing secret as `APP_CONFIG__STRIPE__WEBHOOK_SECRET`.
- **CryptoBot webhook**: configure your CryptoBot app's webhook to `POST https://your-domain/api/v1/webhook/cryptobot`; the signature is HMAC-SHA256 of the payload keyed by `sha256(APP_CONFIG__CRYPTO_BOT__API_KEY)`, sent as `crypto-pay-api-signature`.

<br>

## ⚙️ Environment Variables Reference

### 🧩 Core

- `APP_CONFIG__DETAILS__TITLE` / `APP_CONFIG__DETAILS__DESCRIPTION` — Swagger title/description
- `APP_CONFIG__API__PREFIX` — global API prefix (default `/api`; combined with the router's own `/v1` this makes `/api/v1/...`)
- `APP_CONFIG__RUN__HOST` / `APP_CONFIG__RUN__PORT` / `APP_CONFIG__RUN__RELOAD` — uvicorn bind settings

### 🔐 Security

- `APP_CONFIG__SECURITY__BOT_API_KEY` — the internal API key shared between the bot and the backend, sent as the `x-api-key` header on every request from `bot/`.

### 🐘 Database (PostgreSQL)

- `APP_CONFIG__DB__URL` — async DSN, `postgresql+asyncpg://user:password@host:port/dbname`
- `APP_CONFIG__DB__ECHO` / `APP_CONFIG__DB__ECHO_POOL` — SQL/pool logging
- `APP_CONFIG__DB__POOL_SIZE` / `APP_CONFIG__DB__MAX_OVERFLOW` — connection pool sizing (code default for `POOL_SIZE` is `50`; `.env.template` ships `5` — pick deliberately)

### 🔴 Cache (Redis)

- `APP_CONFIG__CACHE__URL` — **required for the app to boot** (a `docker-compose.yaml` Redis service exists), but no Redis client is installed and nothing in the codebase reads from it today. See [Roadmap](#-roadmap).

### 💳 Stripe

- `APP_CONFIG__STRIPE__API_KEY` — secret key used to create Checkout Sessions
- `APP_CONFIG__STRIPE__WEBHOOK_SECRET` — used to verify `stripe-signature`
- `APP_CONFIG__STRIPE__SUCCESS_URL` / `APP_CONFIG__STRIPE__CANCEL_URL` — where Stripe redirects after checkout

### 💠 CryptoBot

- `APP_CONFIG__CRYPTO_BOT__API_KEY` — Crypto Pay API token; also the HMAC key (sha256'd) for webhook verification
- `APP_CONFIG__CRYPTO_BOT__IS_TESTNET` — switches between `pay.crypt.bot` and `testnet-pay.crypt.bot`

### 📨 Telegram (backend-side bot client)

- `APP_CONFIG__TELEGRAM__BOT_TOKEN` — used by the backend's own `aiogram.Bot` instance (same token as the polling bot) to send DMs and ban expired users
- `APP_CONFIG__TELEGRAM__MAIN_CHAT_ID` — the main community chat, used by the expiry job and the access check

### 💰 Payment

- `APP_CONFIG__PAYMENT__DEFAULT_CURRENCY` — default `EUR`
- `APP_CONFIG__PAYMENT__PRICE_MATRIX` — **keyed by user `level` (1–10), not by months purchased**; price is per month, in cents:

  | Level | Price / month |
  |---|---|
  | 1 – 3 | €5.00 |
  | 4 – 7 | €3.00 |
  | 8 – 10 | Free |

  A purchase of 3+ months additionally gets a 20% discount on the total.

### 🤖 Bot (`bot/.env`, flat keys, no prefix)

- `BOT_TOKEN` — from @BotFather
- `API_URL` — backend base URL including `/api/v1` (default `http://localhost:8000/api/v1`)
- `API_KEY` — sent as `x-api-key` on every request to the backend
- `MAIN_CHAT_ID` — numeric ID of the main chat
- `SECRET_REGISTER_KEY` — shared secret required by the in-chat `/register` command

<br>

## 📚 API Reference

> All routes are mounted at **`/api/v1`** (`APP_CONFIG__API__PREFIX` + the router's own `/v1` prefix). Interactive docs live at `/docs`.

#### 🔓 Authentication
---

Requests from `bot/` to the backend carry an internal `x-api-key` header (see [Environment Variables Reference](#️-environment-variables-reference)). The API is not intended for direct/public use — it's a private contract between the bot and its backend, not a public integration surface.

#### 👤 User — `/user`
---

| Method | Path | Description | Errors |
|---|---|---|---|
| `POST` | `/user/auth` | Upsert/authenticate a user by Telegram ID; called on every bot update | `403` banned |
| `GET` | `/user/{user_id}` | Fetch a user + current subscription by internal ID | `403` banned · `404` not found |

#### 💳 Payment — `/payment`
---

| Method | Path | Description | Errors |
|---|---|---|---|
| `POST` | `/payment/create` | Create (or reuse) a checkout session for N months | `400` invalid months (0 &lt; months ≤ 12) · `404` user not found · `403` screening not passed · **`202`** no payment required (free level or admin) |

#### 🛡️ Admin — `/admin`
---

Internal endpoints backing the bot's admin panel (user management, subscription grants, direction management — see [Bot Reference](#-bot-reference) for the actual capabilities). Not documented in detail here since this is an internal management surface, not a public integration point.

#### 🧭 Direction — `/direction`
---

| Method | Path | Description | Errors |
|---|---|---|---|
| `GET` | `/direction/` | List all directions *(note the trailing slash — `/direction` without it 307-redirects)* | `404` none found |
| `GET` | `/direction/{telegram_chat_id}` | Fetch one direction | `404` not found |
| `GET` | `/direction/{user_id}/access?telegram_chat_id=` | Read a user's access record for one direction | `404` not found |
| `POST` | `/direction/access` | Create a user↔direction access record | `404` user or direction not found · `409` already exists |

#### 🚦 Access — `/access`
---

| Method | Path | Description |
|---|---|---|
| `GET` | `/access/main?user_id=` | Boolean gate for the main chat — never raises, always returns `{"allowed": bool}` |

#### 🔔 Webhook — `/webhook`
---

| Method | Path | Header | Description | Errors |
|---|---|---|---|---|
| `POST` | `/webhook/stripe` | `stripe-signature` | Verifies signature, acts only on `checkout.session.completed` | `400` missing header / bad signature / processing failed |
| `POST` | `/webhook/cryptobot` | `crypto-pay-api-signature` | HMAC-verifies, acts only on `update_type=invoice_paid` | `400` verification or processing failed |

<br>

## 🤖 Bot Reference

The bot renders one of four menus depending on role, decided by `AuthMiddleware` on every update:

| Menu | Shown to | Key actions |
|---|---|---|
| **Guest** | no active access | 🚀 Join the community · 💼 My profile · ❓ Support/FAQ |
| **Resident** | active subscriber | 💼 My profile · 💸 Renew subscription · ✨ Directions · ❓ Support/FAQ |
| **Admin** | `is_admin` | everything Resident has, plus 🛡️ the admin panel |
| **Superadmin** | `is_superadmin` | everything Admin has, plus 👑 promote/demote admins |

#### Admin panel capabilities
---

| Capability | Admin | Superadmin |
|---|---|---|
| Browse / search / view any regular user's card | ✅ | ✅ |
| Ban / unban, change level (1–10), toggle global screening | ✅ | ✅ |
| Grant a free subscription (1/2/3 months) | ✅ | ✅ |
| Toggle a user's per-direction access | ✅ | ✅ |
| Rename a direction, change its owner, toggle its screening requirement | ✅ | ✅ |
| Register a new direction (`/register` in-chat) | ✅ | ✅ |
| View/manage another **admin's or superadmin's** card | ❌ | ✅ |
| Promote/demote admins | ❌ | ✅ |

Payments are purchased in three steps — **months → provider → checkout** — with only Stripe currently wired up end-to-end in the bot UI (CryptoBot's handlers exist in the backend but are disabled on the bot side, see [Roadmap](#-roadmap)); a manual bank-transfer contact is offered as a fallback.

<br>

## 🧭 Roadmap

Known gaps, tracked here rather than glossed over:

- **Decide on Redis.** `APP_CONFIG__CACHE__URL` is a required setting and a Redis container is part of the compose stack, but no Redis client is installed and nothing reads the URL — either wire it in (e.g. FSM storage, caching) or drop it from the required config.
- **Re-enable CryptoBot in the bot UI.** The backend's `CryptoBotPaymentProvider` is fully implemented and webhook-verified; the bot's `crypto_payment_*` handlers are commented out, leaving Stripe as the only reachable provider.
- **Honor `months` on fulfilment.** `ProcessSuccessfulPaymentService` always extends a subscription by a flat 30 days regardless of how many months were paid for (the admin "grant subscription" path does multiply correctly).
- **Complete `.env.template`.** Stripe, Telegram and CryptoBot settings have no defaults and are missing from the template entirely, so it cannot boot the app as-is.
- **Testing, linting, type-checking, CI.** No `tests/` directory, no pytest, no linter/type-checker configuration beyond `black` being installed, and no CI workflow.
- **`LICENSE`.** Not yet added.
- **Containerize the bot.** `Dockerfile`/`docker-compose.yaml` currently cover only the FastAPI backend.
