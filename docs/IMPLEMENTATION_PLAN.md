# Expiration Guardian v1 - Implementation Plan

**Document:** Multi-Session Implementation Roadmap
**Version:** 1.0.0
**Created:** 2026-02-02
**Spec References:**
- `docs/functional_specification_expiration_guardian_v_1.md`
- `docs/technical_specification_expiration_guardian_v_1.md`
- `docs/product_principles_and_decision_log_v1.md`
- `docs/brand_manual_v1.md`
- `docs/business_go_to_market_strategy_v1.md`

---

## Progress Tracker

> Update this section as each phase completes. Check the box and add the date.

- [ ] **Phase 0** - Foundations (project scaffold, DB, Docker, CI)
- [ ] **Phase 1** - Core Entities & Auth (Account, User, Document, Observer, magic link, dashboard)
- [ ] **Phase 2** - Forward Email & Alerts (inbound parse, date extraction, confirmation, alert scheduler, digests, PDF)
- [ ] **Phase 3** - Billing & Hardening (Stripe, plan limits, paywall, rate limiting, security)
- [ ] **Phase 4** - Launch Readiness (email templates, testing, landing page, monitoring, export)

---

## Environment Context

```
Stack:          NestJS (API) + React/Vite (SPA) + PostgreSQL + TypeORM
Email:          SendGrid (outbound + inbound parse)
Payments:       Stripe Checkout + Stripe Billing
Infra:          Docker Compose
Language:       English only (v1)
Testing:        Jest + supertest (unit + integration)
Admin:          SQL queries (no admin panel in v1)
```

**Repository Structure (monorepo):**

```
expiration-guardian/
├── api/                          # NestJS backend
│   ├── src/
│   │   ├── modules/
│   │   │   ├── auth/             # Magic link, JWT, refresh tokens
│   │   │   ├── account/          # Account entity + settings
│   │   │   ├── user/             # User entity
│   │   │   ├── document/         # Document CRUD + lifecycle
│   │   │   ├── observer/         # Observer management
│   │   │   ├── alert/            # Alert rules, sent alerts, scheduler
│   │   │   ├── digest/           # Digest generation
│   │   │   ├── pdf/              # PDF generation
│   │   │   ├── forward-email/    # Inbound email processing
│   │   │   ├── billing/          # Stripe integration
│   │   │   └── email/            # SendGrid service (shared)
│   │   ├── common/               # Guards, decorators, pipes, filters
│   │   ├── config/               # Configuration module
│   │   └── migrations/           # TypeORM migrations
│   ├── test/                     # Integration tests
│   ├── Dockerfile
│   └── package.json
├── web/                          # React SPA (Vite)
│   ├── src/
│   │   ├── pages/                # Login, Documents, Settings, Confirm
│   │   ├── components/           # Shared UI components
│   │   ├── hooks/                # Custom hooks (auth, api)
│   │   ├── services/             # API client
│   │   └── types/                # TypeScript types
│   ├── Dockerfile
│   └── package.json
├── docs/                         # Specifications (existing)
├── docker-compose.yml            # Local dev: API + DB + (optional) web
├── docker-compose.prod.yml       # Production: API + DB + web + nginx
├── .env.example                  # All required env vars
├── .gitignore
└── README.md
```

**Key Environment Variables (from technical spec section 18):**

| Variable | Purpose |
|----------|---------|
| `DATABASE_URL` | PostgreSQL connection |
| `JWT_SECRET` | JWT signing secret |
| `SENDGRID_API_KEY` | Outbound email |
| `STRIPE_SECRET_KEY` | Stripe API |
| `STRIPE_WEBHOOK_SECRET` | Stripe webhook signature |
| `APP_URL` | Application base URL |
| `INBOUND_EMAIL_ADDRESS` | Forwarding address |

Full list in technical spec section 18.

---

## Feature Coverage Matrix

| ID | Feature | Phase | Spec Reference |
|----|---------|-------|----------------|
| F01 | Project scaffold (NestJS + React + Docker) | Phase 0 | Tech spec sec. 2 |
| F02 | PostgreSQL + TypeORM migrations | Phase 0 | Tech spec sec. 5 |
| F03 | Account + User entities | Phase 1 | Tech spec sec. 5 |
| F04 | Magic link auth (signup + login) | Phase 1 | Tech spec sec. 3 |
| F05 | JWT access + refresh tokens | Phase 1 | Tech spec sec. 3 |
| F06 | Document CRUD + lifecycle | Phase 1 | Tech spec sec. 5, 7 |
| F07 | Observer management | Phase 1 | Func spec sec. 5 |
| F08 | Dashboard SPA (login, docs, settings) | Phase 1 | Func spec sec. 10 |
| F09 | SendGrid outbound email service | Phase 1 | Tech spec sec. 2 |
| F10 | Forward email pipeline (inbound parse) | Phase 2 | Tech spec sec. 6 |
| F11 | Date extraction (chrono-node) | Phase 2 | Tech spec sec. 6 |
| F12 | Draft confirmation flow | Phase 2 | Tech spec sec. 6 |
| F13 | Alert scheduler cron | Phase 2 | Tech spec sec. 9 |
| F14 | Sent alert idempotency | Phase 2 | Tech spec sec. 9 |
| F15 | Recurring engine cron | Phase 2 | Tech spec sec. 8 |
| F16 | Expired status updater cron | Phase 2 | Tech spec sec. 7, 12 |
| F17 | Weekly + monthly digest | Phase 2 | Tech spec sec. 10 |
| F18 | Monthly PDF generator | Phase 2 | Tech spec sec. 11 |
| F19 | Stripe Checkout + Billing | Phase 3 | Tech spec sec. 16 |
| F20 | Stripe webhooks | Phase 3 | Tech spec sec. 16 |
| F21 | Plan limits enforcement | Phase 3 | Func spec sec. 15 |
| F22 | Paywall mechanics (trial + limits) | Phase 3 | Func spec sec. 15 |
| F23 | Rate limiting | Phase 3 | Tech spec sec. 14 |
| F24 | Security hardening (CORS, Helmet, validation) | Phase 3 | Tech spec sec. 14 |
| F25 | Email templates (all 11 types) | Phase 4 | Tech spec sec. 17 |
| F26 | Unit + integration tests | Phase 4 | - |
| F27 | Data export (JSON + PDF) | Phase 4 | Func spec sec. 15 |
| F28 | Monitoring + structured logging | Phase 4 | Tech spec sec. 13 |

---

## Phase 0: Foundations

### Goal

Scaffold the monorepo, configure Docker Compose for local development, create the PostgreSQL database schema via TypeORM migrations, and validate the full stack boots correctly.

### Dependencies

- None (starting point)

### Exit Criteria

- `docker compose up` starts API + DB + web
- API responds to `GET /health`
- All TypeORM migrations run successfully
- React SPA loads at `http://localhost:5173`
- Jest runs with a passing placeholder test

### Files to Create

```
expiration-guardian/
├── api/
│   ├── src/
│   │   ├── main.ts
│   │   ├── app.module.ts
│   │   ├── config/
│   │   │   └── configuration.ts          # Env var loading + validation
│   │   ├── common/
│   │   │   └── health.controller.ts      # GET /health
│   │   └── migrations/
│   │       └── 001-initial-schema.ts     # All v1 tables
│   ├── test/
│   │   └── app.e2e-spec.ts              # Health check test
│   ├── tsconfig.json
│   ├── tsconfig.build.json
│   ├── nest-cli.json
│   ├── package.json
│   ├── Dockerfile
│   └── .eslintrc.js
├── web/
│   ├── src/
│   │   ├── main.tsx
│   │   ├── App.tsx
│   │   └── vite-env.d.ts
│   ├── index.html
│   ├── tsconfig.json
│   ├── vite.config.ts
│   ├── package.json
│   └── Dockerfile
├── docker-compose.yml
├── .env.example
├── .gitignore
└── package.json                          # Root workspace (npm workspaces)
```

### Step-by-Step Instructions

**0.1 Initialize root workspace**

Create `package.json` at root with npm workspaces:

```json
{
  "name": "expiration-guardian",
  "private": true,
  "workspaces": ["api", "web"]
}
```

Create `.gitignore`:
```
node_modules/
dist/
.env
*.log
```

Create `.env.example` with all variables from tech spec section 18 (no values, only keys).

**0.2 Scaffold NestJS API**

Inside `api/`:

```bash
npm init -y
npm install @nestjs/common @nestjs/core @nestjs/platform-express @nestjs/config @nestjs/typeorm @nestjs/schedule typeorm pg reflect-metadata rxjs class-validator class-transformer
npm install -D @nestjs/cli @nestjs/testing typescript @types/node @types/express jest ts-jest @types/jest supertest @types/supertest eslint @typescript-eslint/eslint-plugin @typescript-eslint/parser
```

Create `src/main.ts`:
- Bootstrap NestJS app on port 3000
- Enable CORS (allow `APP_URL` origin)
- Enable validation pipe globally (`class-validator`)
- Use `ConfigModule.forRoot()` for env loading

Create `src/app.module.ts`:
- Import `ConfigModule` (global, `.env` loading)
- Import `TypeOrmModule.forRootAsync()` (from `DATABASE_URL`)
- Import `ScheduleModule.forRoot()`
- Import `HealthModule`

Create `src/common/health.controller.ts`:
- `GET /health` → returns `{ status: 'ok', timestamp: Date.now() }`

**0.3 Create initial database migration**

File: `api/src/migrations/001-initial-schema.ts`

This single migration creates ALL v1 tables. Follow schemas exactly from tech spec section 5.

Tables to create (in order, respecting FK constraints):

1. **account** — id (uuid PK default gen_random_uuid()), plan (enum trial/active/canceled default 'trial'), trial_ends_at (timestamp nullable), stripe_customer_id (varchar unique nullable), stripe_subscription_id (varchar nullable), timezone (varchar default 'UTC'), digest_weekly (boolean default true), digest_monthly (boolean default true), default_alert_days (int[] default '{30,7,0}'), created_at (timestamp default now()), updated_at (timestamp default now())

2. **user** — id (uuid PK), account_id (uuid FK→account unique), email (varchar unique), created_at, updated_at

3. **refresh_token** — id (uuid PK), user_id (uuid FK→user), token_hash (varchar unique), expires_at (timestamp), revoked_at (timestamp nullable), created_at

4. **observer** — id (uuid PK), account_id (uuid FK→account), email (varchar), is_active (boolean default true), created_at. Unique constraint on (account_id, email).

5. **document** — id (uuid PK), account_id (uuid FK→account), title (varchar), type (enum: passport, visa, driver_license, insurance, contract, subscription, warranty, permit, service, other), type_custom (varchar nullable), description (text nullable), expiration_date (date), is_recurring (boolean default false), recurrence_rule_id (uuid FK→recurrence_rule nullable), recurring_parent_id (uuid FK→document nullable), status (enum: draft, active, expired, archived), source (enum: manual, forwarded_email), confirmation_token (varchar nullable), confirmation_expires_at (timestamp nullable), created_at, updated_at

6. **recurrence_rule** — id (uuid PK), document_id (uuid FK→document unique), frequency (enum: monthly, annual), horizon_months (int), created_at

7. **alert_rule** — id (uuid PK), document_id (uuid FK→document), days_before (int)

8. **sent_alert** — id (uuid PK), document_id (uuid FK→document), alert_rule_id (uuid FK→alert_rule), expiration_date (date), sent_at (timestamp). Unique constraint on (document_id, alert_rule_id, expiration_date).

9. **email_event** — id (uuid PK), account_id (uuid FK→account), document_id (uuid FK→document nullable), type (enum: alert, digest, confirmation, magic_link, welcome, observer_welcome), status (enum: sent, delivered, bounced), sendgrid_message_id (varchar nullable), created_at

Create indexes:
- `document.expiration_date`
- `document.status`
- `document.account_id`
- `user.email`

**0.4 Scaffold React SPA**

Inside `web/`:

```bash
npm create vite@latest . -- --template react-ts
npm install react-router-dom
```

Create minimal `App.tsx` with React Router and a placeholder home page that says "Expiration Guardian" with a login link.

**0.5 Create Docker Compose**

File: `docker-compose.yml`

```yaml
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: expiration_guardian
      POSTGRES_USER: eg_user
      POSTGRES_PASSWORD: eg_password
    ports:
      - "5432:5432"
    volumes:
      - pg_data:/var/lib/postgresql/data

  api:
    build: ./api
    ports:
      - "3000:3000"
    environment:
      DATABASE_URL: postgresql://eg_user:eg_password@db:5432/expiration_guardian
      JWT_SECRET: dev-secret-change-in-production
      JWT_EXPIRATION: 15m
      REFRESH_TOKEN_EXPIRATION: 7d
      APP_URL: http://localhost:5173
      NODE_ENV: development
    depends_on:
      - db

  web:
    build: ./web
    ports:
      - "5173:5173"

volumes:
  pg_data:
```

API Dockerfile:
```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build
CMD ["node", "dist/main.js"]
```

Web Dockerfile (dev mode):
```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
CMD ["npm", "run", "dev", "--", "--host"]
```

**0.6 Validate everything boots**

```bash
docker compose up --build
# Verify:
# - http://localhost:3000/health returns { status: 'ok' }
# - http://localhost:5173 loads React SPA
# - API logs show TypeORM migrations ran successfully
```

### Verification Checkpoint

- [ ] `docker compose up` starts all 3 services without errors
- [ ] `GET http://localhost:3000/health` returns 200
- [ ] PostgreSQL has all 9 tables with correct columns
- [ ] React SPA loads at `http://localhost:5173`
- [ ] `npm test` in `api/` runs Jest with at least 1 passing test
- [ ] `.env.example` contains all 13 env vars from tech spec
- [ ] Enum types match spec exactly (document_type, document_status, plan_type, etc.)

---

## Phase 1: Core Entities & Auth

### Goal

Implement Account, User, Document, Observer entities with full CRUD. Implement magic link authentication (signup + login). Build the minimal dashboard SPA with login, document list, create/edit, and settings screens.

### Dependencies

- Phase 0 complete (schema exists, API boots)

### Exit Criteria

- New user can enter email → receive magic link → click → arrive at dashboard
- User can create, edit, archive documents from dashboard
- User can add/remove observers from settings
- Documents with past expiration dates are created as `expired`
- JWT access + refresh token flow works end-to-end

### Files to Create

```
api/src/modules/
├── auth/
│   ├── auth.module.ts
│   ├── auth.controller.ts            # POST /auth/magic-link, GET /auth/verify, POST /auth/refresh
│   ├── auth.service.ts               # Magic link generation, JWT signing, token validation
│   ├── jwt.strategy.ts               # Passport JWT strategy
│   ├── jwt-auth.guard.ts             # Route guard
│   └── token.service.ts              # Refresh token CRUD (hash, rotate, revoke)
├── account/
│   ├── account.module.ts
│   ├── account.controller.ts         # GET /account, PATCH /account/settings
│   ├── account.service.ts
│   └── account.entity.ts
├── user/
│   ├── user.module.ts
│   ├── user.service.ts
│   └── user.entity.ts
├── document/
│   ├── document.module.ts
│   ├── document.controller.ts        # CRUD: GET /documents, POST, PATCH, POST /:id/archive
│   ├── document.service.ts           # Business logic (create, edit, archive, lifecycle rules)
│   ├── document.entity.ts
│   ├── alert-rule.entity.ts
│   └── recurrence-rule.entity.ts
├── observer/
│   ├── observer.module.ts
│   ├── observer.controller.ts        # GET /observers, POST, DELETE /:id
│   ├── observer.service.ts
│   └── observer.entity.ts
├── email/
│   ├── email.module.ts
│   └── email.service.ts              # SendGrid wrapper (send magic link, welcome, etc.)

web/src/
├── pages/
│   ├── LoginPage.tsx                  # Email input → request magic link
│   ├── AuthCallbackPage.tsx           # Receives token from magic link redirect
│   ├── DocumentListPage.tsx           # List all documents with filters
│   ├── DocumentFormPage.tsx           # Create + Edit form (shared)
│   ├── DocumentDetailPage.tsx         # View detail + alert history
│   └── SettingsPage.tsx               # Observers, digest, timezone, alerts
├── components/
│   ├── Layout.tsx                     # Nav + content wrapper
│   ├── ProtectedRoute.tsx             # Redirect to login if no token
│   └── DocumentStatusBadge.tsx        # Color-coded status
├── hooks/
│   ├── useAuth.ts                     # Auth state, login, logout, refresh
│   └── useApi.ts                      # Fetch wrapper with JWT injection
├── services/
│   └── api.ts                         # API client (base URL, interceptors)
└── types/
    └── index.ts                       # Shared TypeScript types
```

### Step-by-Step Instructions

**1.1 Create TypeORM entities**

Create entity files for all tables. Each entity maps 1:1 to the migration schema from Phase 0. Use TypeORM decorators (`@Entity`, `@Column`, `@PrimaryGeneratedColumn('uuid')`, `@ManyToOne`, `@OneToOne`, `@OneToMany`).

Key relationships:
- Account `1:1` User (via `user.account_id`)
- Account `1:N` Observer
- Account `1:N` Document
- Document `1:N` AlertRule
- Document `1:1` RecurrenceRule (optional)
- Document `N:1` Document (self-ref via `recurring_parent_id`)
- User `1:N` RefreshToken

**1.2 Implement auth module**

`auth.service.ts`:
- `requestMagicLink(email: string)`:
  1. Check if user exists in `user` table
  2. If not: create Account (plan: trial, trial_ends_at: now+14d) + User. Queue welcome email.
  3. Generate signed token (HMAC-SHA256 with `JWT_SECRET`, payload: `{ userId, type: 'magic_link' }`, 15min TTL)
  4. Send magic link email via `email.service` with link: `{APP_URL}/auth/callback?token=xxx`
  5. Rate limit check: max 5 per email per hour (query `email_event` for recent magic_link entries)

- `verifyMagicLink(token: string)`:
  1. Verify and decode token
  2. Check token hasn't been used (single-use: store used token hash in `refresh_token` table or separate table)
  3. Generate JWT access token (15min TTL, payload: `{ userId, accountId }`)
  4. Generate opaque refresh token, hash it, store in `refresh_token` table (7d TTL)
  5. Return `{ accessToken, refreshToken }`

- `refreshToken(tokenValue: string)`:
  1. Hash the incoming token
  2. Look up in `refresh_token` table
  3. Check not expired, not revoked
  4. Revoke current token (`revoked_at = now()`)
  5. Generate new refresh token (rotation)
  6. Generate new access token
  7. Return both

`auth.controller.ts`:
- `POST /auth/magic-link` — body: `{ email }` → calls `requestMagicLink`
- `GET /auth/verify?token=xxx` — validates token, sets refresh cookie, redirects to `{APP_URL}/documents`
- `POST /auth/refresh` — reads refresh token from httpOnly cookie, returns new access token

**1.3 Implement document module**

`document.service.ts`:
- `create(accountId, dto)`:
  1. Validate plan limits (count active + draft docs for account, compare to plan limit)
  2. If `expiration_date` < today (user's timezone): set status to `expired`
  3. Else: set status to `active`, source to `manual`
  4. Create default alert rules (from `account.default_alert_days`)
  5. If `is_recurring`: create RecurrenceRule with frequency and horizon_months
  6. Return created document

- `update(documentId, accountId, dto)`:
  1. Load document, verify ownership
  2. If `expiration_date` changed or alert rules changed: delete all `sent_alert` records for this document
  3. Update fields
  4. Return updated document

- `archive(documentId, accountId)`:
  1. Load document, verify ownership
  2. Set status to `archived`
  3. Return updated document

- `findAll(accountId, filters)`:
  1. Query documents for account
  2. Apply filters: status (default: exclude archived), type, search
  3. Include alert rules, recurrence rule
  4. Sort by expiration_date ASC
  5. Return list

**1.4 Implement observer module**

`observer.service.ts`:
- `create(accountId, email)`:
  1. Validate email != owner email (edge case C.4)
  2. Validate plan limits (count active observers)
  3. Check unique constraint (account_id, email)
  4. Create observer (is_active: true)
  5. Send observer welcome email
  6. Return observer

- `remove(observerId, accountId)`: delete observer
- `unsubscribe(observerId)`: set `is_active = false` (called via signed token link)
- `findAll(accountId)`: return all observers for account

**1.5 Implement email service**

`email.service.ts`:
- Wraps `@sendgrid/mail`
- Methods: `sendMagicLink(to, link)`, `sendWelcome(to, data)`, `sendObserverWelcome(to, data)`
- All sends create an `email_event` record
- In development: log email to console instead of sending (configurable via `NODE_ENV`)

**1.6 Build React SPA**

`useAuth.ts` hook:
- State: `accessToken` (in memory), `isAuthenticated`, `user`
- `login(email)`: POST `/auth/magic-link`
- `handleCallback(token)`: GET `/auth/verify?token=xxx`, store access token
- `refresh()`: POST `/auth/refresh` (cookie sent automatically)
- `logout()`: clear token, POST `/auth/logout` to revoke refresh token
- Interceptor: attach `Authorization: Bearer {token}` to all API requests. On 401: try refresh, if fails → redirect to login.

Pages:
- `LoginPage`: email input → "Send magic link" button. Show "Check your email" message after submit.
- `AuthCallbackPage`: extracts token from URL, calls verify, redirects to `/documents`
- `DocumentListPage`: table with columns (title, type, expiration, status, days remaining). Filter tabs: Active, Expired, Drafts, Archived. "Add document" button.
- `DocumentFormPage`: form with fields: title, type (dropdown), expiration_date (date picker), description (textarea), is_recurring (toggle → frequency dropdown + horizon). Alert rules section: list of days_before with add/remove. Submit creates or updates.
- `SettingsPage`: timezone dropdown (IANA), digest toggles (weekly, monthly), default alert days, observers list with add/remove.

### Verification Checkpoint

- [ ] `POST /auth/magic-link` with new email creates Account + User + sends email
- [ ] `POST /auth/magic-link` with existing email sends magic link only
- [ ] `GET /auth/verify` returns JWT + sets refresh cookie
- [ ] `POST /auth/refresh` rotates refresh token and returns new access token
- [ ] `POST /documents` creates document with default alert rules
- [ ] `PATCH /documents/:id` updates document and clears sent_alerts if date changed
- [ ] `POST /documents/:id/archive` sets status to archived
- [ ] Document with past date is created as `expired`
- [ ] Observer creation validates email != owner email
- [ ] Observer creation respects plan limits
- [ ] React SPA: full login → dashboard → create document flow works
- [ ] Rate limiting on magic link requests works (5/email/hour)

---

## Phase 2: Forward Email & Alerts

### Goal

Implement the forward email pipeline (SendGrid inbound parse → date extraction → confirmation flow), all cron jobs (alert scheduler, recurring engine, expired updater, draft cleanup, digests), and the monthly PDF generator.

### Dependencies

- Phase 1 complete (documents + auth work end-to-end)
- SendGrid account configured with Inbound Parse webhook

### Exit Criteria

- User can forward email → receive confirmation → confirm/reject via link
- Alerts fire correctly based on alert rules and timezone
- Recurring documents auto-renew when expired
- Weekly and monthly digests are sent with correct content windows
- Monthly PDF is generated and attached to digest

### Files to Create

```
api/src/modules/
├── forward-email/
│   ├── forward-email.module.ts
│   ├── forward-email.controller.ts    # POST /webhooks/inbound-email
│   ├── forward-email.service.ts       # Parse, extract, create draft
│   ├── date-extractor.service.ts      # chrono-node wrapper
│   └── type-suggester.service.ts      # Keyword → document type mapping
├── alert/
│   ├── alert.module.ts
│   ├── alert-scheduler.service.ts     # Daily cron: evaluate + send alerts
│   ├── sent-alert.entity.ts
│   └── alert-rule.entity.ts           # (already created in Phase 1)
├── recurring/
│   ├── recurring.module.ts
│   └── recurring-engine.service.ts    # Daily cron: generate next instances
├── digest/
│   ├── digest.module.ts
│   └── digest.service.ts             # Weekly + monthly digest generation
├── pdf/
│   ├── pdf.module.ts
│   └── pdf.service.ts                # Monthly PDF generation (pdfkit)
├── cron/
│   ├── cron.module.ts
│   └── expired-updater.service.ts    # Daily cron: active → expired
│   └── draft-cleanup.service.ts      # Daily cron: delete old drafts
├── document/
│   └── document.controller.ts        # ADD: GET /:id/confirm, GET /:id/reject
```

### Step-by-Step Instructions

**2.1 Implement forward email pipeline**

Install: `npm install chrono-node`

`forward-email.controller.ts`:
- `POST /webhooks/inbound-email`:
  1. Parse SendGrid Inbound Parse webhook payload (multipart form: `from`, `to`, `subject`, `text`, `html`)
  2. Call `forward-email.service.processInbound(payload)`

`forward-email.service.ts`:
- `processInbound(payload)`:
  1. Extract sender email from `from` field
  2. Look up user by email
  3. If no match: send rejection email (tech spec sec. 6, edge case C.8 N/A since no user)
  4. Check rate limit: count inbound emails for account today (max 20)
  5. Extract dates via `date-extractor.service`
  6. Suggest type via `type-suggester.service`
  7. Create Document with status: `draft`, source: `forwarded_email`
  8. Generate signed confirmation token (HMAC, 72h TTL), store on document
  9. Send confirmation email with Confirm/Edit/Reject links

`date-extractor.service.ts`:
- Uses `chrono-node` (English locale)
- `extract(text: string): { dates: Date[], primary: Date | null }`
  1. Parse text with `chrono.parse(text)`
  2. Filter to future dates only
  3. If multiple: primary = latest future date
  4. If one: primary = that date
  5. If none: primary = null

`type-suggester.service.ts`:
- Keyword map: `{ passport: 'passport', visa: 'visa', insurance: 'insurance', license: 'driver_license', contract: 'contract', subscription: 'subscription', warranty: 'warranty', permit: 'permit', renewal: 'service' }`
- `suggest(text: string): DocumentType`:
  1. Lowercase text
  2. Check each keyword
  3. Return first match, or `other`

**2.2 Implement confirmation endpoints**

Add to `document.controller.ts`:

- `GET /documents/:id/confirm?token=xxx`:
  1. Load document by ID
  2. Verify token matches `confirmation_token` and not expired
  3. Check plan limits at this point (edge case C.8)
  4. Set status to `active`
  5. Create default alert rules
  6. Clear confirmation_token
  7. Redirect to `{APP_URL}/documents/{id}` (or show success page)

- `GET /documents/:id/reject?token=xxx`:
  1. Load document, verify token
  2. Set status to `archived`
  3. Show "Document rejected" confirmation page

**2.3 Implement alert scheduler**

`alert-scheduler.service.ts`:
- Decorated with `@Cron('0 6 * * *')` (daily 06:00 UTC)
- Logic follows tech spec section 9 exactly:
  1. Query all accounts where `plan != 'canceled'`
  2. For each account: calculate local date from `account.timezone`
  3. Query active documents with alert rules where `expiration_date - days_before == local_date`
  4. For each match: check `sent_alert` table
  5. If not already sent: send alert email to owner + active observers
  6. Insert `sent_alert` row
  7. Create `email_event` records

**2.4 Implement recurring engine**

`recurring-engine.service.ts`:
- Decorated with `@Cron('0 2 * * *')` (daily 02:00 UTC)
- Logic follows tech spec section 8 exactly:
  1. Query documents: `is_recurring = true AND status = 'expired'`
  2. For each: check no active sibling in chain
  3. Calculate next date
  4. Check within plan horizon
  5. Check recurring limit not exceeded
  6. Create new document + recurrence rule + alert rules

**2.5 Implement expired status updater**

`expired-updater.service.ts`:
- Decorated with `@Cron('0 1 * * *')` (daily 01:00 UTC)
- For each account: calculate local date
- Update documents: `status = 'active' AND expiration_date < local_date` → set `status = 'expired'`

**2.6 Implement draft cleanup**

`draft-cleanup.service.ts`:
- Decorated with `@Cron('0 3 * * *')` (daily 03:00 UTC)
- Delete documents: `status = 'draft' AND created_at < now() - 7 days`
- Log count deleted

**2.7 Implement digest service**

`digest.service.ts`:
- `generateWeeklyDigest()`: `@Cron('0 8 * * 1')` (Monday 08:00 UTC)
  1. For each account where `digest_weekly = true AND plan != 'canceled'`
  2. Calculate local date
  3. Query: upcoming 30 days, expired last 7 days, renewed last 7 days
  4. If all empty: skip (no empty noise)
  5. Send digest email to owner + active observers

- `generateMonthlyDigest()`: `@Cron('0 8 1 * *')` (1st of month 08:00 UTC)
  1. Same pattern but: upcoming 60 days, expired last 30 days, renewed last 30 days
  2. Attach PDF (generated at 04:00 same day)

**2.8 Implement PDF generator**

Install: `npm install pdfkit`

`pdf.service.ts`:
- `generateMonthlyPdf()`: `@Cron('0 4 1 * *')` (1st of month 04:00 UTC)
  1. For each account with `plan = 'active'`
  2. Build PDF with pdfkit: header, active docs table, expired table, recurring table, footer
  3. Store to `./storage/pdfs/{accountId}/{YYYY-MM}.pdf`
  4. Clean up PDFs older than 30 days

- `generateOnDemand(accountId, month)`:
  1. Same generation logic but for specific month
  2. Return PDF buffer

### Verification Checkpoint

- [ ] Forward email: inbound webhook creates draft document with extracted date
- [ ] Forward email: unrecognized sender receives rejection email
- [ ] Forward email: confirmation link sets draft → active
- [ ] Forward email: rejection link sets draft → archived
- [ ] Forward email: expired token returns error
- [ ] Alert scheduler: sends alerts for documents matching rules + timezone
- [ ] Alert scheduler: does NOT send duplicate alerts (sent_alert dedup)
- [ ] Recurring engine: creates new document when recurring doc expires
- [ ] Recurring engine: respects horizon and plan limits
- [ ] Recurring engine: archived recurring doc stops chain
- [ ] Expired updater: moves past-date active docs to expired
- [ ] Draft cleanup: deletes 7-day-old drafts
- [ ] Weekly digest: correct 30d/7d windows, skips empty
- [ ] Monthly digest: correct 60d/30d windows, includes PDF
- [ ] PDF: generates correct tabular content
- [ ] All cron jobs log execution results

---

## Phase 3: Billing & Hardening

### Goal

Integrate Stripe for subscription billing. Enforce plan limits and paywall mechanics. Add rate limiting, security headers, and input validation hardening.

### Dependencies

- Phase 2 complete (all core features work)
- Stripe account configured with products and prices

### Exit Criteria

- User can subscribe via Stripe Checkout (monthly or annual)
- Stripe webhooks update account plan status
- Trial expiration blocks new document creation
- Plan limits are enforced at every creation point
- Rate limiting is active on all endpoints
- Security headers are configured

### Files to Create

```
api/src/modules/
├── billing/
│   ├── billing.module.ts
│   ├── billing.controller.ts          # POST /billing/checkout, GET /billing/portal, POST /webhooks/stripe
│   ├── billing.service.ts             # Stripe session creation, webhook processing
│   └── plan-guard.service.ts          # Plan limit enforcement (shared)
├── common/
│   ├── guards/
│   │   └── plan-limit.guard.ts        # Decorator-based limit checks
│   ├── middleware/
│   │   └── rate-limit.middleware.ts    # Rate limiting
│   └── filters/
│       └── http-exception.filter.ts   # Consistent error responses
```

### Step-by-Step Instructions

**3.1 Implement Stripe integration**

Install: `npm install stripe`

`billing.service.ts`:
- `createCheckoutSession(accountId, plan: 'monthly' | 'annual')`:
  1. Get or create Stripe Customer (use `stripe_customer_id` on account, or create new)
  2. Create Checkout Session with `STRIPE_PRICE_MONTHLY` or `STRIPE_PRICE_ANNUAL`
  3. Set success_url: `{APP_URL}/billing/success`
  4. Set cancel_url: `{APP_URL}/billing/cancel`
  5. Return session URL

- `createPortalSession(accountId)`:
  1. Get Stripe Customer
  2. Create Customer Portal Session
  3. Return portal URL

- `handleWebhook(payload, signature)`:
  1. Verify Stripe signature with `STRIPE_WEBHOOK_SECRET`
  2. Extract event type + data
  3. Idempotent processing (log event ID, skip if already processed)
  4. Handle events per tech spec section 16:
     - `checkout.session.completed` → `account.plan = 'active'`
     - `invoice.paid` → confirm renewal
     - `invoice.payment_failed` → grace period logic
     - `customer.subscription.deleted` → `account.plan = 'canceled'`

`billing.controller.ts`:
- `POST /billing/checkout` (auth required) → redirect URL
- `GET /billing/portal` (auth required) → redirect URL
- `POST /webhooks/stripe` (no auth, signature verified) → webhook handler

**3.2 Implement plan limit enforcement**

`plan-guard.service.ts`:
- `checkDocumentLimit(accountId)`: count active+draft docs, compare to plan limit (trial: 3, paid: unlimited)
- `checkRecurringLimit(accountId)`: count active recurring docs (trial: 1, paid: 50)
- `checkObserverLimit(accountId)`: count active observers (trial: 1, paid: 10)
- `checkHorizon(accountId, date)`: check if date is within plan horizon (trial: 3mo, paid: 36mo)
- `isTrialExpired(account)`: check `trial_ends_at < now()` for trial accounts
- `isAccountActive(account)`: check `plan != 'canceled'` AND not expired trial

Apply guard checks to:
- `POST /documents` (doc limit + horizon)
- `POST /documents/:id/confirm` (doc limit)
- `POST /observers` (observer limit)
- All write endpoints (account active check)

**3.3 Implement trial expiration cron**

Add to cron module:
- `@Cron('0 7 * * *')` (daily 07:00 UTC)
- Query accounts: `plan = 'trial' AND trial_ends_at < now() - 2 days`
- Send "trial expiring" email (tech spec section 17)
- When `trial_ends_at < now()`: enforce read-only (plan-guard blocks writes)

**3.4 Implement rate limiting**

Install: `npm install @nestjs/throttler`

Configure in `app.module.ts`:
- Global: 100 requests/minute/IP
- Magic link: 5/email/hour (custom decorator)
- Login attempts: 10/IP/hour

**3.5 Security hardening**

Install: `npm install helmet`

In `main.ts`:
- `app.use(helmet())` — security headers
- CORS: allow only `APP_URL` origin
- `app.useGlobalPipes(new ValidationPipe({ whitelist: true, forbidNonWhitelisted: true }))` — strict validation
- All DTOs use `class-validator` decorators

### Verification Checkpoint

- [ ] `POST /billing/checkout` creates Stripe Checkout Session and returns URL
- [ ] Stripe webhook `checkout.session.completed` updates plan to active
- [ ] Stripe webhook `customer.subscription.deleted` updates plan to canceled
- [ ] Canceled account cannot create documents (returns 403)
- [ ] Trial account at 3 docs cannot create more (returns 403 with upgrade prompt)
- [ ] Trial account at 14 days is read-only
- [ ] Rate limiting returns 429 on excessive requests
- [ ] Security headers present in all responses
- [ ] Invalid request bodies are rejected with 400
- [ ] Stripe Customer Portal accessible for active accounts

---

## Phase 4: Launch Readiness

### Goal

Create all email templates, write unit + integration tests for critical paths, implement data export, set up structured logging, and prepare for deployment.

### Dependencies

- Phase 3 complete (billing + security work)

### Exit Criteria

- All 11 email types have HTML + plain text templates
- Critical business logic has unit tests (alert scheduling, recurring engine, date extraction, plan limits)
- API endpoints have integration tests (auth flow, document CRUD, billing webhooks)
- Data export (JSON + PDF) works from dashboard
- Structured JSON logging is configured
- Production Docker Compose is ready

### Files to Create

```
api/src/modules/email/
├── templates/                       # Email templates
│   ├── magic-link.hbs
│   ├── welcome.hbs
│   ├── observer-welcome.hbs
│   ├── alert.hbs
│   ├── confirmation.hbs
│   ├── no-date-found.hbs
│   ├── rejection-no-account.hbs
│   ├── weekly-digest.hbs
│   ├── monthly-digest.hbs
│   ├── trial-expiring.hbs
│   └── limit-reached.hbs

api/src/modules/account/
└── export.service.ts                # JSON + PDF data export

api/test/
├── auth.e2e-spec.ts                 # Auth flow integration tests
├── document.e2e-spec.ts             # Document CRUD integration tests
├── alert-scheduler.spec.ts          # Alert scheduling unit tests
├── recurring-engine.spec.ts         # Recurring logic unit tests
├── forward-email.spec.ts            # Email parsing unit tests
├── plan-limits.spec.ts              # Plan limit enforcement tests
└── billing-webhook.e2e-spec.ts      # Stripe webhook integration tests

docker-compose.prod.yml              # Production config
```

### Step-by-Step Instructions

**4.1 Create email templates**

Install: `npm install @nestjs-modules/mailer nodemailer handlebars`

Each template is an `.hbs` file following the brand manual:
- No emojis, no exclamation marks
- Professional, neutral, calm tone
- Subject lines: informational, date-driven
- Body: what is happening, when, what to do (nothing else)
- Inline CSS for email client compatibility
- Plain text fallback auto-generated from HTML

Template content follows tech spec section 17 inventory.

**4.2 Write unit tests**

Key test files:

`alert-scheduler.spec.ts`:
- Test: alert fires at correct day (30d, 7d, 0d)
- Test: timezone calculation produces correct local date
- Test: idempotency — duplicate cron run doesn't send twice
- Test: canceled account gets no alerts
- Test: document created on expiration day gets day-0 alert

`recurring-engine.spec.ts`:
- Test: expired recurring doc creates new instance
- Test: horizon limit prevents out-of-range creation
- Test: plan limit prevents excess recurring items
- Test: archived recurring doc stops chain
- Test: only one active per chain

`forward-email.spec.ts`:
- Test: chrono-node extracts date from realistic email text
- Test: multiple dates → latest future date selected
- Test: no date found → null returned
- Test: keyword matching suggests correct type

`plan-limits.spec.ts`:
- Test: trial at 3 docs blocks creation
- Test: paid unlimited allows creation
- Test: trial expired blocks writes
- Test: observer limit enforced

**4.3 Write integration tests**

`auth.e2e-spec.ts`:
- Test: POST /auth/magic-link creates account for new email
- Test: POST /auth/magic-link returns 200 for existing email
- Test: GET /auth/verify with valid token returns access token
- Test: GET /auth/verify with expired token returns 401
- Test: POST /auth/refresh rotates token
- Test: Rate limiting blocks 6th request per hour

`document.e2e-spec.ts`:
- Test: Full CRUD cycle (create → read → update → archive)
- Test: Past expiration date creates expired doc
- Test: Update expiration_date clears sent_alerts
- Test: Plan limit enforcement returns 403

**4.4 Implement data export**

`export.service.ts`:
- `exportJSON(accountId)`:
  1. Query all documents, observers, alert rules for account
  2. Return structured JSON

- `exportPDF(accountId)`:
  1. Use same logic as monthly PDF
  2. Return PDF buffer

Add to `account.controller.ts`:
- `GET /account/export?format=json` (auth required)
- `GET /account/export?format=pdf` (auth required)

Add to SPA settings page: "Export my data" with format selector.

**4.5 Set up structured logging**

Install: `npm install nestjs-pino pino pino-pretty`

Configure Pino logger in NestJS:
- JSON format in production
- Pretty format in development
- Log: request method, path, status code, duration
- Log: cron job execution (start, count, errors)
- Log: email sends (type, recipient, success/failure)

**4.6 Production Docker Compose**

`docker-compose.prod.yml`:
- API: built production image, `NODE_ENV=production`
- Web: nginx serving built React SPA, proxying `/api` to API
- DB: PostgreSQL with persistent volume
- No port exposure except 80/443 via nginx

### Verification Checkpoint

- [ ] All 11 email templates render correctly with sample data
- [ ] Email templates have plain text fallback
- [ ] Unit tests pass: alert scheduler, recurring engine, date extraction, plan limits
- [ ] Integration tests pass: auth flow, document CRUD, billing webhooks
- [ ] `GET /account/export?format=json` returns complete account data
- [ ] `GET /account/export?format=pdf` returns downloadable PDF
- [ ] Structured JSON logs appear in production mode
- [ ] `docker-compose -f docker-compose.prod.yml up` serves the app correctly
- [ ] All env vars from `.env.example` are documented and required

---

## Architectural Decisions Log

### ADR-01: Monorepo with npm workspaces

**Decision**: Single repository with `api/` and `web/` folders, linked via npm workspaces.
**Rationale**: Single developer, shared TypeScript types possible, one CI pipeline, simpler deployment coordination. Matches team size.

### ADR-02: TypeORM over Prisma

**Decision**: TypeORM for ORM and migrations.
**Rationale**: First-class NestJS integration (`@nestjs/typeorm`), decorator-based entity definitions match NestJS patterns, migration CLI built-in. Prisma's schema-first approach is cleaner but adds build step complexity.

### ADR-03: React SPA over SSR

**Decision**: Separate React SPA (Vite) over Next.js or NestJS server-rendered templates.
**Rationale**: Dashboard is minimal (7 screens). SPA is simpler to reason about, no SSR hydration complexity, clear API boundary. SEO is not needed for the dashboard (only for future landing page).

### ADR-04: pdfkit over Puppeteer for PDF

**Decision**: `pdfkit` for PDF generation.
**Rationale**: Lightweight, no headless browser dependency. PDFs are simple tables. Puppeteer would add Docker complexity and memory overhead for no benefit.

### ADR-05: Refresh token in httpOnly cookie

**Decision**: Refresh token stored as httpOnly secure cookie, access token in memory.
**Rationale**: httpOnly cookie prevents XSS theft of refresh token. In-memory access token prevents localStorage-based attacks. Standard pattern for SPA + API auth.

### ADR-06: All cron jobs in single NestJS process

**Decision**: Background jobs run in the same NestJS process using `@nestjs/schedule`.
**Rationale**: Simplicity. At v1 scale (hundreds of users), cron jobs complete in seconds. No need for separate worker process or job queue (BullMQ). If scale demands it later, extracting to a worker is straightforward.

### ADR-07: No admin panel in v1

**Decision**: Administration via SQL queries, no admin UI.
**Rationale**: First users (alpha/beta) are manageable with direct DB access. Admin panel is development time better spent on core features. Add when user count makes SQL impractical.

### ADR-08: SendGrid for both inbound and outbound

**Decision**: SendGrid handles both outbound email and inbound parse.
**Rationale**: Single vendor, single account, simpler configuration. Inbound Parse is included in SendGrid plans. Avoids managing separate MX records for a different inbound service.

---

## Continuation Prompt Template

Use this prompt to resume implementation in a new Claude session:

```
Lee los siguientes archivos para contexto:

1. docs/IMPLEMENTATION_PLAN.md (plan de implementacion — contiene progress tracker, estructura del repo, instrucciones paso a paso por fase)
2. docs/functional_specification_expiration_guardian_v_1.md (especificacion funcional)
3. docs/technical_specification_expiration_guardian_v_1.md (especificacion tecnica — schemas, cron jobs, security, edge cases, env vars, email templates)

Revisa la seccion "Progress Tracker" del plan para ver que fases estan completadas.
Continua con la siguiente fase pendiente.

Si la fase actual esta parcialmente completada, revisa los archivos mencionados en "Files to Create" para ver cuales ya existen.

Contexto del proyecto:
- Directorio: [DIRECTORIO_DEL_PROYECTO]
- Monorepo: api/ (NestJS) + web/ (React Vite)
- Base de datos: PostgreSQL via TypeORM
- Docker Compose para desarrollo local

Stack:
- Backend: NestJS + TypeORM + @nestjs/schedule + class-validator
- Frontend: React + Vite + TypeScript + react-router-dom
- Email: SendGrid (@sendgrid/mail) + Inbound Parse webhook
- Payments: Stripe (stripe npm package)
- Auth: JWT (access token 15min) + refresh token (httpOnly cookie, 7 days)
- PDF: pdfkit
- Date parsing: chrono-node
- Testing: Jest + supertest

Reglas:
- Seguir las especificaciones de los docs (son autoritativas, no redisenar)
- Seguir las instrucciones paso a paso del plan para la fase actual
- No agregar features fuera del scope de v1
- Respetar las decisiones arquitectonicas (ADR) del plan
- Al completar una fase, actualizar el Progress Tracker
- Si algo es ambiguo, preguntar UNA vez. Si no, asumir y documentar la asuncion.
- English only para codigo y UI. Comentarios pueden ser en espanol si es mas claro.
```

---

## Quick Reference: API Endpoints

| Method | Path | Auth | Phase | Purpose |
|--------|------|------|-------|---------|
| GET | `/health` | No | 0 | Health check |
| POST | `/auth/magic-link` | No | 1 | Request magic link |
| GET | `/auth/verify` | No | 1 | Verify magic link token |
| POST | `/auth/refresh` | Cookie | 1 | Refresh access token |
| POST | `/auth/logout` | JWT | 1 | Revoke refresh token |
| GET | `/account` | JWT | 1 | Get account info |
| PATCH | `/account/settings` | JWT | 1 | Update settings |
| GET | `/account/export` | JWT | 4 | Export data (JSON/PDF) |
| GET | `/documents` | JWT | 1 | List documents |
| POST | `/documents` | JWT | 1 | Create document |
| GET | `/documents/:id` | JWT | 1 | Get document detail |
| PATCH | `/documents/:id` | JWT | 1 | Update document |
| POST | `/documents/:id/archive` | JWT | 1 | Archive document |
| GET | `/documents/:id/confirm` | Token | 2 | Confirm draft (forward email) |
| GET | `/documents/:id/reject` | Token | 2 | Reject draft |
| GET | `/documents/:id/edit` | Token | 2 | Redirect to edit form |
| GET | `/observers` | JWT | 1 | List observers |
| POST | `/observers` | JWT | 1 | Add observer |
| DELETE | `/observers/:id` | JWT | 1 | Remove observer |
| GET | `/observers/:id/unsubscribe` | Token | 1 | Observer self-unsubscribe |
| POST | `/billing/checkout` | JWT | 3 | Create Stripe Checkout |
| GET | `/billing/portal` | JWT | 3 | Stripe Customer Portal |
| POST | `/webhooks/stripe` | Signature | 3 | Stripe webhook |
| POST | `/webhooks/inbound-email` | N/A | 2 | SendGrid Inbound Parse |
| GET | `/account/pdf-summary` | JWT | 2 | On-demand PDF download |

---

## Quick Reference: Database Tables

| Table | Key Fields | Phase |
|-------|-----------|-------|
| account | id, plan, trial_ends_at, stripe_customer_id, timezone, digest prefs | 0 |
| user | id, account_id (unique), email (unique) | 0 |
| refresh_token | id, user_id, token_hash, expires_at, revoked_at | 0 |
| observer | id, account_id, email, is_active | 0 |
| document | id, account_id, title, type, expiration_date, status, source, confirmation_token | 0 |
| recurrence_rule | id, document_id (unique), frequency, horizon_months | 0 |
| alert_rule | id, document_id, days_before | 0 |
| sent_alert | id, document_id, alert_rule_id, expiration_date (unique triple) | 0 |
| email_event | id, account_id, document_id, type, status, sendgrid_message_id | 0 |

---

> **End of Implementation Plan**
>
> Total scope: 5 phases covering the complete Expiration Guardian v1.
> Phase 0: Project foundations and database.
> Phase 1: Core entities, auth, and dashboard.
> Phase 2: Email pipeline, cron jobs, and notifications.
> Phase 3: Billing and security.
> Phase 4: Polish, testing, and launch prep.
