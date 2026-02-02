# Technical Specification – Expiration Guardian v1

## 1. Overview

This document describes the technical architecture, data models, background jobs, and system flows for **Expiration Guardian v1**.

The system is designed to be:
- Simple and reliable
- Email-first
- Easy to operate with minimal supervision
- Cheap to run and scale gradually

---

## 2. High-Level Architecture

### Core Components

- **Backend API**: NestJS (Node.js / TypeScript)
- **Frontend**: React SPA (Vite + TypeScript), communicates with API via REST
- **Database**: PostgreSQL
- **ORM**: TypeORM (NestJS integration, migration-based schema management)
- **Email Service**: SendGrid (outbound API + inbound parse webhook)
- **Payments**: Stripe Checkout + Stripe Billing (subscription management)
- **Scheduler / Workers**: Cron-based background jobs (NestJS `@nestjs/schedule`)
- **Infra**: Docker Compose (1–2 machines)
- **Language**: English only in v1 (UI, emails, date parsing)

### Logical Diagram (conceptual)

```
[ React SPA (Vite) ]            [ User Email Client ]
        |                              |
        v                              | (forward email)
[ API (NestJS) ] <---------- [ SendGrid Inbound Parse Webhook ]
        |         \
        v          \-------> [ Stripe Checkout / Billing ]
[ PostgreSQL ]                    |
        |                         v
        v                   [ Stripe Webhooks → API ]
[ Cron / Background Jobs ] ---> [ SendGrid Outbound API ]
                                       |
                                       v
                                [ Owner + Observers Inbox ]
```

The system is **API + SPA + background jobs + inbound email webhook + Stripe webhooks**. There is no real-time or streaming requirement in v1.

---

## 3. Authentication & Identity

### Auth Model

- Passwordless authentication (magic links)
- Email address is the primary identifier
- JWT-based stateless sessions

### Flow (unified signup + login)
1. User enters email on login page (`POST /auth/magic-link`)
2. API checks if email exists in `user` table:
   - **Existing user**: generates magic link token
   - **New user**: creates Account (plan: trial, trial_ends_at: now+14d) + User, generates magic link token, queues welcome email
3. System sends magic link email via SendGrid
4. User clicks link (`GET /auth/verify?token=xxx`); API validates token
5. API returns:
   - **Access token** (JWT, 15-minute TTL, contains user_id + account_id)
   - **Refresh token** (opaque, 7-day TTL, stored in DB, set as httpOnly cookie)
6. Client (React SPA) stores access token in memory (not localStorage)
7. For new users: SPA detects first login, auto-detects browser timezone, saves to account via `PATCH /account/settings`
8. When access token expires, client calls `POST /auth/refresh` with refresh token cookie
9. Refresh token is single-use (rotated on each refresh)

### Token-based Actions (no login required)
- Document confirmation/rejection from forward-email flow: signed URL token (72h TTL)
- Observer unsubscribe: signed URL token (no expiration)
- These tokens are single-purpose and cannot be used for dashboard access

### Security
- Magic link tokens are single-use (invalidated after first use)
- Refresh tokens are revocable (stored in DB with `revoked_at` field)
- All tokens are cryptographically signed (HMAC-SHA256 or RS256)
- Rate limit on magic link requests: 5 per email per hour

No passwords are stored.

---

## 4. Multi-Account / Tenant Model

### Concept

- One **Account** represents one individual or small entity
- One **User** (owner) per account
- Multiple **Observers** (emails only)

There is no concept of multiple active users per account in v1.

---

## 5. Core Data Models (Schemas)

> Field names are illustrative; exact naming may vary.

### Account

```sql
account
- id (uuid, pk)
- plan (enum: trial, active, canceled)
- trial_ends_at (timestamp, nullable)
- stripe_customer_id (string, nullable, unique)  -- set when user subscribes
- stripe_subscription_id (string, nullable)       -- active Stripe subscription
- timezone (string, default 'UTC')  -- IANA timezone (e.g. 'America/New_York')
- digest_weekly (boolean, default true)
- digest_monthly (boolean, default true)
- default_alert_days (int[], default '{30,7,0}')  -- default alert rules for new documents
- created_at
- updated_at
```

---

### User

```sql
user
- id (uuid, pk)
- account_id (fk, unique)  -- one user per account in v1
- email (string, unique)
- created_at
- updated_at
```

One User per Account in v1. Separate entity to support future multi-user expansion.

---

### RefreshToken

```sql
refresh_token
- id (uuid, pk)
- user_id (fk)
- token_hash (string, unique)  -- hashed opaque token
- expires_at (timestamp)
- revoked_at (timestamp, nullable)
- created_at
```

---

### Observer

```sql
observer
- id (uuid, pk)
- account_id (fk)
- email (string)
- is_active (boolean, default true)  -- false when unsubscribed
- created_at
```

Unique constraint on (account_id, email).

---

### Document

```sql
document
- id (uuid, pk)
- account_id (fk)
- title (string)
- type (enum: passport, visa, driver_license, insurance, contract, subscription, warranty, permit, service, other)
- type_custom (string, nullable)  -- only when type = 'other'
- description (text, nullable)
- expiration_date (date)
- is_recurring (boolean, default false)
- recurrence_rule_id (fk, nullable)
- recurring_parent_id (fk → document.id, nullable)  -- links to the original document in a recurring chain
- status (enum: draft, active, expired, archived)
- source (enum: manual, forwarded_email)
- confirmation_token (string, nullable)  -- for draft confirmation flow
- confirmation_expires_at (timestamp, nullable)
- created_at
- updated_at
```

`recurring_parent_id` enables tracing the full history of a recurring chain. The first document in a chain has `recurring_parent_id = null`.

---

### RecurrenceRule

```sql
recurrence_rule
- id (uuid, pk)
- document_id (fk, unique)  -- 1:1 relationship with document
- frequency (enum: monthly, annual)
- horizon_months (int)  -- max months to generate ahead
- created_at
```

Each recurring document has its own recurrence rule (1:1, not shared).

---

### AlertRule

```sql
alert_rule
- id (uuid, pk)
- document_id (fk)
- days_before (int)  -- 0 = on expiration day
```

Default rules (30, 7, 0) are created automatically per document. Users can override per document.

---

### SentAlert (Idempotency)

```sql
sent_alert
- id (uuid, pk)
- document_id (fk)
- alert_rule_id (fk)
- expiration_date (date)  -- the specific expiration date this alert is for
- sent_at (timestamp)
```

Unique constraint on (document_id, alert_rule_id, expiration_date). Prevents duplicate alerts across cron restarts or retries.

---

### EmailEvent (Audit / Metrics)

```sql
email_event
- id (uuid, pk)
- account_id (fk)
- document_id (fk, nullable)
- type (enum: alert, digest, confirmation, magic_link, welcome, observer_welcome)
- status (enum: sent, delivered, bounced)
- sendgrid_message_id (string, nullable)  -- for delivery tracking
- created_at
```

Note: Email open tracking is **not implemented in v1** due to privacy concerns and unreliability (Apple Mail Privacy Protection, etc.). Delivery status is tracked via SendGrid webhooks (sent, delivered, bounced).

---

## 6. Forward Email Pipeline

### Email Ingestion

- Dedicated inbound email address (e.g. `docs@expirationguardian.com`)
- SendGrid Inbound Parse Webhook (POST to `/webhooks/inbound-email`)

### User Identification

1. Extract `from` address from the forwarded email
2. Look up `from` address in `user.email`
3. If match: proceed with extraction
4. If no match: send rejection email to sender with instructions to use their registered email
5. Rate limit: max 20 inbound emails per account per day

### Extraction (v1 approach)

Strategy: **Rule-based parsing with date extraction library** (e.g. `chrono-node`).

1. Extract plain text body (strip HTML if present)
2. Parse dates using date extraction library
3. If multiple dates found: select the latest future date as primary suggestion, include alternatives in confirmation email
4. If one date found: use as expiration date suggestion
5. If no date found: notify user that no date was detected, suggest manual creation via dashboard
6. Suggest document type based on keyword matching (e.g. "passport", "insurance", "renewal" → mapped to enum)
7. Use email subject as suggested document title

LLM-based extraction is a future enhancement (not v1).

### Draft Creation

- Create Document with `status: draft`, `source: forwarded_email`
- Generate signed confirmation token (72h TTL)
- Store token and expiry in document record

### Confirmation Flow

Confirmation email contains three action links:
- **Confirm**: `GET /documents/:id/confirm?token=xxx` → sets status to `active`, creates default alert rules
- **Edit**: `GET /documents/:id/edit?token=xxx` → redirects to pre-filled form in dashboard (triggers magic link if no active session)
- **Reject**: `GET /documents/:id/reject?token=xxx` → sets status to `archived` (soft delete)

No login required for Confirm and Reject. Edit requires dashboard access.

### Draft Cleanup
- Cron job runs daily
- Deletes documents with `status: draft` older than 7 days
- Logs cleanup for audit

---

## 7. Document Lifecycle

### States

- **Draft**: pending user confirmation (from forward-email flow only)
- **Active**: confirmed and being tracked; alerts will fire
- **Expired**: past expiration date; alerts no longer sent
- **Archived**: soft-deleted by user or system; hidden from default view

### All Valid Transitions

| From | To | Trigger |
|------|----|---------|
| Draft | Active | User confirms (click or dashboard) |
| Draft | Archived | User rejects, or draft expires (7-day cleanup cron) |
| Active | Expired | Cron job (`expired_status_updater`) when `expiration_date < local_date` |
| Active | Archived | User manually archives from dashboard |
| Expired | Archived | User manually archives from dashboard |

No transition back from Archived. No hard delete in v1.

### Editing an Active Document
- User can edit: title, type, description, expiration_date, alert rules
- If `expiration_date` is changed: all `sent_alert` records for that document are deleted, and the alert cycle restarts from scratch
- If `alert_rules` are changed: same behavior (clear sent_alerts, restart cycle)
- Editing does not change document status

---

## 8. Recurring Engine (v1)

### Cron Behavior (Daily 02:00 UTC)

For each account with `plan != canceled`:
1. Query documents where `is_recurring = true` AND `status = expired`
2. For each expired recurring document:
   a. Check if a newer active document already exists in the same chain (`recurring_parent_id` or same `recurrence_rule_id`) — if yes, skip
   b. Calculate next expiration date: `expiration_date + frequency` (1 month or 1 year)
   c. Check if next date is within the account's plan horizon (trial: 3 months from now, paid: 36 months)
   d. Check if account has not exceeded recurring item limit (trial: 1, paid: 50)
   e. If all checks pass: create a **new Document** row with:
      - Same: title, type, type_custom, description, account_id, is_recurring=true, source=manual
      - New: expiration_date (calculated), status=active, recurring_parent_id = original document id
      - Create a **new RecurrenceRule** (same frequency, same horizon_months) linked to the new document
      - Create **default AlertRules** copied from the original document's alert rules
   f. If checks fail (horizon exceeded or limit reached): do nothing, log skip reason

### Rules

- Only **one active instance** per recurring chain at any time
- No infinite generation — bounded by plan horizon
- Respects account plan limits
- If the user archives a recurring document, the chain stops (no new instances generated)
- If the user edits the new instance (e.g. changes expiration_date), the edited value is used for subsequent renewals

---

## 9. Alerting System

### Alert Scheduler

- Daily cron job (runs at 06:00 UTC)
- For each active account with `plan != canceled`:
  1. Calculate user's current local date from `account.timezone`
  2. Query active documents with matching alert rules (where `expiration_date - days_before == local_date`)
  3. Check `sent_alert` table for existing entries
  4. If not already sent: send alert email to owner + all active observers
  5. Record in `sent_alert` table

### Alert Recipients

- Account owner (always)
- All observers with `is_active = true`
- Each recipient receives their own email (not CC/BCC)

### Idempotency

- `sent_alert` table has unique constraint on `(document_id, alert_rule_id, expiration_date)`
- Before sending, check if row exists; if yes, skip
- This prevents duplicates from cron restarts, retries, or overlapping job executions
- For recurring documents that regenerate, each new `expiration_date` gets its own alert cycle

---

## 10. Digest Generator

### Weekly Digest (Monday 08:00 UTC)
For each account where `digest_weekly = true` AND `plan != canceled`:
- **Upcoming**: active documents with `expiration_date` within next 30 days (from user's local date)
- **Recently expired**: documents that moved to `expired` in the last 7 days
- **Renewed**: recurring documents auto-renewed in the last 7 days
- If all sections are empty: **digest is not sent** (no empty emails)

### Monthly Digest (1st of month 08:00 UTC)
For each account where `digest_monthly = true` AND `plan != canceled`:
- **Upcoming**: active documents with `expiration_date` within next 60 days
- **Recently expired**: documents that moved to `expired` in the last 30 days
- **Renewed**: recurring documents auto-renewed in the last 30 days
- **PDF attachment**: monthly PDF summary (generated earlier same day at 04:00 UTC, or regenerated on demand)
- If all sections are empty: **digest is not sent**

### Recipients
- Account owner + all active observers
- Each recipient receives their own email (not CC/BCC)
- Digest content is identical for owner and observers

---

## 11. Monthly PDF Generator

### Job (1st of month 04:00 UTC)
For each account with `plan = active`:
1. Generate PDF containing:
   - Header: account owner email, generation date, period covered
   - Table: all **active** documents (title, type, expiration_date, days remaining)
   - Table: **expired** documents from last 30 days (title, type, expiration_date)
   - Table: **recurring** documents (title, type, next expiration, frequency)
   - Footer: total counts per status
2. Store PDF temporarily on filesystem (or S3 in future)
3. PDF is attached to the monthly digest email

### Technology
- PDF generation via `pdfkit` (lightweight, no browser dependency)
- Simple tabular layout, no heavy design in v1

### Access
- Attached to monthly digest email (if enabled)
- Downloadable on-demand from dashboard (`GET /account/pdf-summary?month=YYYY-MM`)
- On-demand requests regenerate the PDF (no long-term storage in v1)

### Retention
- Generated PDFs are stored for 30 days, then deleted by cleanup job
- Users can regenerate any past month's PDF on demand (within data availability)

No legal or compliance guarantees.

---

## 12. Cron Jobs Summary

| Job | Schedule (UTC) | Purpose |
|----|---------------|--------|
| Expired status updater | Daily 01:00 | Move active documents past expiration_date to expired status |
| Recurring engine | Daily 02:00 | Generate next occurrences for recurring documents |
| Draft cleanup | Daily 03:00 | Delete unconfirmed drafts older than 7 days |
| Alert scheduler | Daily 06:00 | Evaluate and send expiration alerts |
| Trial expiring notifier | Daily 07:00 | Send reminder to accounts where trial expires in 2 days |
| Digest weekly | Monday 08:00 | Weekly summary email |
| PDF generator | 1st of month 04:00 | Generate monthly PDF snapshot (covers previous month) |
| Digest monthly | 1st of month 08:00 | Monthly summary email (attaches PDF generated at 04:00) |
| PDF cleanup | 1st of month 05:00 | Delete PDFs older than 30 days |

Note: Jobs are ordered by execution time. PDF generation runs **before** the monthly digest on the same day so the PDF can be attached.

---

## 13. Observability & Metrics

Tracked metrics:
- Documents created (manual vs forward)
- Confirmation acceptance rate
- Alert delivery rate (sent / delivered / bounced via SendGrid webhooks)
- Digest delivery rate
- Trial → paid conversion rate
- Monthly churn rate

Note: No email open tracking in v1 (privacy decision). Delivery status is the proxy.

Logs are structured (JSON) and stored centrally. Recommended: stdout logging with a log aggregator (e.g. Docker log driver → file or future ELK/Loki).

---

## 14. Security Considerations

### Authentication & Tokens
- Magic link tokens: single-use, 15-minute TTL, cryptographically signed
- Access tokens (JWT): 15-minute TTL, signed with server secret
- Refresh tokens: 7-day TTL, single-use (rotated), stored hashed in DB, revocable
- Confirmation tokens (forward-email): 72h TTL, single-purpose

### Rate Limiting
- Magic link requests: 5 per email per hour
- API general: 100 requests per minute per IP
- Inbound email: 20 per account per day
- Login attempts: 10 per IP per hour

### API Security
- CORS: restricted to application domain only
- HTTPS only (HTTP redirected)
- Helmet.js for HTTP security headers
- Input validation on all endpoints (class-validator in NestJS)

### Data
- No sensitive documents stored (only metadata: titles, dates, types)
- Minimal PII (emails only)
- No file uploads in v1
- Database connections via SSL

---

## 15. Scalability Notes

- Stateless API (JWT, no server-side sessions for API)
- Background jobs can be isolated into separate process/container
- Postgres indexes on: `document.expiration_date`, `document.status`, `document.account_id`, `user.email`, `sent_alert` unique constraint
- Horizontal scaling possible without redesign
- SendGrid handles email delivery scaling

---

## 16. Stripe Integration

### Subscription Flow
1. User clicks "Subscribe" on dashboard
2. API creates Stripe Checkout Session (`POST /billing/checkout`) with plan (monthly or annual)
3. User is redirected to Stripe Checkout (hosted by Stripe — no custom payment form)
4. On successful payment, Stripe sends webhook → API updates `account.plan = active`, stores `stripe_customer_id` and `stripe_subscription_id`
5. On failed payment or cancellation, Stripe sends webhook → API updates `account.plan = canceled`

### Stripe Webhooks (POST `/webhooks/stripe`)
Handled events:
- `checkout.session.completed` → activate account
- `invoice.paid` → confirm renewal
- `invoice.payment_failed` → flag for grace period (3 days), then cancel
- `customer.subscription.deleted` → set plan to canceled

### Webhook Security
- Verify Stripe webhook signature on every request
- Idempotent processing (use Stripe event ID as dedup key)

### Customer Portal
- Stripe Customer Portal used for: update payment method, view invoices, cancel subscription
- Accessed via `GET /billing/portal` → redirect to Stripe hosted portal

---

## 17. Email Templates

All emails use a consistent, minimal design aligned with the brand manual (calm, professional, no emojis, no exclamation marks).

### Template Inventory

| Email Type | Trigger | Key Content |
|-----------|---------|-------------|
| Magic Link | User requests login | Link (valid 15min), brief explanation |
| Welcome | New account created | Product overview, how to add docs, forwarding address |
| Observer Welcome | Owner adds observer | Explanation of what they'll receive, unsubscribe link |
| Alert | Cron matches alert rule | Document title, type, expiration date, days remaining, "what to do" hint |
| Confirmation | Forward-email creates draft | Extracted data, Confirm/Edit/Reject links |
| No Date Found | Forward-email extraction fails | Explanation, link to create document manually |
| Rejection (no account) | Inbound email from unknown sender | Explanation, link to sign up |
| Weekly Digest | Monday cron | Upcoming (30d), expired (7d), renewed |
| Monthly Digest | 1st of month cron | Upcoming (60d), expired (30d), renewed, PDF attachment |
| Trial Expiring | 2 days before trial ends | Reminder, subscribe CTA |
| Limit Reached | User hits plan limit | Which limit, subscribe CTA |

### Template Technology
- HTML email templates with inline CSS (email client compatibility)
- Templating engine: Handlebars (or NestJS built-in `@nestjs-modules/mailer`)
- Plain text fallback for every email

---

## 18. Environment & Configuration

### Required Environment Variables

| Variable | Description | Example |
|----------|-------------|---------|
| `DATABASE_URL` | PostgreSQL connection string | `postgresql://user:pass@localhost:5432/eg` |
| `JWT_SECRET` | Secret for signing JWT access tokens | (random 64-char string) |
| `JWT_EXPIRATION` | Access token TTL | `15m` |
| `REFRESH_TOKEN_EXPIRATION` | Refresh token TTL | `7d` |
| `SENDGRID_API_KEY` | SendGrid API key for outbound email | `SG.xxx` |
| `SENDGRID_INBOUND_WEBHOOK_SECRET` | Secret for validating inbound parse webhooks | (from SendGrid) |
| `STRIPE_SECRET_KEY` | Stripe API secret key | `sk_live_xxx` |
| `STRIPE_WEBHOOK_SECRET` | Stripe webhook signing secret | `whsec_xxx` |
| `STRIPE_PRICE_MONTHLY` | Stripe Price ID for monthly plan | `price_xxx` |
| `STRIPE_PRICE_ANNUAL` | Stripe Price ID for annual plan | `price_xxx` |
| `APP_URL` | Application base URL | `https://app.expirationguardian.com` |
| `INBOUND_EMAIL_ADDRESS` | Forwarding email address | `docs@expirationguardian.com` |
| `NODE_ENV` | Environment | `development`, `staging`, `production` |

### Configuration Notes
- All secrets are loaded from environment variables (never hardcoded)
- `.env.example` file in repository with all required variables (no values)
- Docker Compose uses `.env` file for local development
- Production secrets managed via hosting provider's secret management

---

## 19. Edge Cases & Defined Behaviors

| # | Scenario | Defined Behavior |
|---|----------|-----------------|
| 1 | **SendGrid outage** | Emails are queued in-memory. Retry with exponential backoff: 3 attempts at 5min, 15min, 60min intervals. If all fail, log error. Next cron cycle will attempt to send missed alerts (idempotency via `sent_alert` table prevents duplicates for already-sent ones). |
| 2 | **Email bounce (owner)** | Tracked via SendGrid delivery webhook (status: bounced in `email_event`). No automatic account action in v1. Bounced accounts are flagged for manual review via admin query. |
| 3 | **Duplicate observer across accounts** | Allowed. The same email can be an observer on multiple accounts. Each account's alerts are independent. |
| 4 | **Owner adds self as observer** | Prevented. API validates that observer email != owner email. Returns validation error. Owner already receives all alerts. |
| 5 | **Document with past expiration date** | Allowed at creation. Document is immediately set to `status: expired`. No alerts sent. Useful for historical record-keeping. |
| 6 | **Document created on expiration day** | Alert for `days_before = 0` (expiration day) is evaluated in the next alert scheduler run. If created before the daily cron (06:00 UTC), the day-0 alert fires that same day. If created after, it fires the next day (cron checks `local_date >= expiration_date - days_before`). `sent_alert` dedup prevents double-sending. |
| 7 | **User changes timezone** | New timezone takes effect from the next cron cycle. No retroactive recalculation. Already-sent alerts are not re-sent. Pending alert evaluations use the new timezone. |
| 8 | **Account at document limit + forward email** | Draft is created (drafts don't count against limit). Limit is checked at confirmation time. If limit reached, confirmation returns error with upgrade prompt. Draft remains until 7-day expiry. |
| 9 | **Recurring document archived** | Chain stops. No new instances are generated. If the user wants to resume, they create a new recurring document manually. |
| 10 | **Stripe webhook arrives before checkout redirect** | Webhook processing is idempotent (uses Stripe event ID). Account plan is updated regardless of order. SPA polls or checks account status on load. |

---

## 20. Out of Scope (v1)

- Real-time processing (WebSockets, SSE)
- Tracking user's external bills or payments (product tracks **dates**, not money)
- Mobile native apps
- External provider APIs (bank, government, etc.)
- Advanced role management (multi-user, permissions)
- Multi-language support
- LLM-based email parsing
- File uploads or document storage

Note: Stripe integration for the product's own subscription billing **is** in scope. What is out of scope is features that track user financial obligations.

---

**Expiration Guardian v1** prioritizes reliability, clarity, and operational simplicity over feature density.

