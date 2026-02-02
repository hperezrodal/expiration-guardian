# Functional Specification – Expiration Guardian v1

## 1. Product Overview

**Product name:** Expiration Guardian
**Tagline:** *Never miss what matters.*
**Extended positioning:** An intelligent inbox that takes care of your expirations.

Expiration Guardian is a lightweight SaaS that helps individuals and small businesses never miss important expirations. It works as a **silent assistant**: users set it up once, and the system monitors expiration dates, sends timely alerts, and provides peace of mind with minimal ongoing interaction.

The product prioritizes automation, low cognitive load, and trust over complex dashboards or heavy workflows.

---

## 2. Product Philosophy

**Guiding principle:** *Never miss what matters.*

Design pillars:
- Set & forget
- Email-first experience
- Conservative automation (always confirm before creating data)
- Low friction, low maintenance
- Calm, human, trustworthy tone

The product should feel more like a caring assistant than a traditional SaaS application.

---

## 3. Target Users

### Primary Persona
- Individuals (freelancers, immigrants, solo founders, families)
- Need reminders for documents, services, subscriptions, and renewals
- Not interested in managing tools daily

### Secondary Persona
- Small businesses (1–10 people)
- Use the product informally for operational hygiene
- May share alerts with partners or accountants

The UX is optimized for **single-user usage**, while still allowing passive sharing.

---

## 4. Core Use Cases

- Get alerted before an important document expires
- Forward an email with an expiration notice and confirm it in one click
- Track recurring services (monthly or annual)
- Receive a weekly or monthly overview of upcoming expirations
- Download a monthly PDF summary for personal records

---

## 5. User Model

### Account
- One account represents one individual or small entity
- Contains plan, billing state, and preferences
- Has one **owner** (User entity)
- Has zero or more **observers**

### User (Owner)
- One user per account (v1)
- Authenticated via magic link
- Has email, timezone, and preferences
- User and Account are separate entities to support future multi-user expansion

### Observers
- Email addresses associated to an account
- Receive alerts and digest emails for all documents in the account
- Cannot log in, cannot manage documents

Observer lifecycle:
- **Added** by the account owner (from dashboard)
- **Notified** via welcome email explaining what they will receive
- **Can unsubscribe** via link in any alert/digest email (self-service)
- **Removed** by the account owner at any time
- If an observer wants to become a full user, they must create their own separate account

---

## 6. Authentication & Onboarding

- Passwordless authentication (magic link via email)
- No passwords stored
- Magic link generates JWT access token (15-minute TTL) + refresh token (7-day TTL, stored in DB)
- Access token is stateless (JWT); refresh token is server-side revocable

### Unified Signup / Login Flow
There is no separate signup page. The flow is identical for new and existing users:

1. User enters email on the login screen
2. System checks if user exists:
   - **Existing user**: sends magic link email
   - **New user**: creates Account (plan: trial, trial_ends_at: now + 14 days) + User, sends magic link + welcome email
3. User clicks magic link → arrives at dashboard
4. For new users: timezone is auto-detected from browser on first dashboard load and saved to account. User can change it in settings.

### Welcome Email (new users only)
- Sent immediately after account creation
- Content: brief explanation of the product, how to add a document, how to forward emails, and the dedicated forwarding address
- Tone: calm, welcoming, informative (per brand manual)

### Interactions that require login (JWT)
- Dashboard access (view/create/edit/delete documents)
- Account settings (observers, digest preferences, alert defaults)
- Data export

### Interactions that do NOT require login
- Email-based document confirmation (signed token in URL)
- Email-based document rejection (signed token in URL)
- Observer unsubscribe (signed token in URL)
- Magic link request (public endpoint)

---

## 7. Document Model

A **document** represents anything with an expiration or renewal date.

### Supported Types (v1)
- Passport
- Visa
- Driver License
- Insurance
- Contract
- Subscription
- Warranty
- Permit
- Service
- Other (custom text)

### Document Fields
- Title / name
- Type
- Expiration date
- Optional description
- Alert rules (default or overridden)
- Observers (inherited from account)

Documents are treated uniformly regardless of origin.

### Document Status
- **Draft**: created from forward-email, pending user confirmation
- **Active**: confirmed and being tracked; alerts will be sent
- **Expired**: past expiration date; no alerts sent
- **Archived**: manually archived by user or rejected draft; hidden from default view

### Document Operations
- **Create**: from dashboard (manual) or forward-email (draft → confirm)
- **Edit**: user can edit all fields (title, type, date, description, alert rules) of an active or draft document. If `expiration_date` is changed on an active document, previously sent alerts for that document are cleared and the alert cycle restarts.
- **Archive**: user can archive a document from any status (active, expired, draft). Archived documents are hidden from the default dashboard view but accessible via filter. Archiving is a soft operation (no data is deleted).
- **Delete**: not available in v1. Users archive instead. Permanent deletion only via account deletion request.
- **Creating a document with a past expiration date**: allowed. The document is immediately set to `expired` status. No alerts are sent. Useful for record-keeping.

---

## 8. Recurring Services (v1)

Recurring services are supported in a **basic, non-billing** form.

### Characteristics
- No payments, amounts, or integrations
- Date-based only
- Treated as documents with recurrence rules

### Supported Recurrence
- Monthly
- Annual

### Behavior
When a recurring document expires:
1. The system automatically creates a **new document** with the next expiration date (current date + frequency)
2. The new document inherits: title, type, description, alert rules, and recurrence rule from the original
3. The original document remains in `expired` status (visible in history)
4. Only **one future instance** exists per recurring chain at any time
5. The new document is only created if the next date falls within the account's plan horizon

From the user's perspective:
- Recurring items "renew themselves" — the user sees a new active document appear automatically
- The expired original is visible in the document list (filterable by status)
- The user does not need to take any action for the renewal to happen

### Creation
- Manual creation only (v1)

### Limits (Plan-based)

| Limit | Trial | Paid |
|-------|-------|------|
| Recurring items | 1 | 50 |
| Future horizon | 3 months | 36 months |

---

## 9. Forward Email Feature

### Overview
Users can forward emails containing expiration information to a dedicated system email address (e.g. `docs@expirationguardian.com`).

### User Identification
- System matches the `from` address of the forwarded email against registered user emails
- If match found: proceeds with extraction and confirmation flow
- If no match found: sends a rejection email to the sender explaining they must forward from their registered email address
- No account creation from inbound email (security measure)

### Behavior
1. User forwards an email from their registered email address
2. System analyzes content and extracts:
   - Possible expiration dates
   - Document type suggestion
   - Source context (original sender / subject)
3. System sends a **confirmation email** to the user with extracted data
4. User can:
   - **Confirm** with a single click (signed token, no login required) — creates the document
   - **Edit** by clicking link that redirects to a pre-filled form in the dashboard (requires login via magic link if no active session)
   - **Reject** with a single click (signed token, no login required) — discards the draft
5. Confirmation tokens expire after 72 hours; unconfirmed drafts are auto-deleted after 7 days

### Plan Limits and Drafts
- Drafts (pending confirmation) do **not** count against the document limit
- The document limit is evaluated at the moment of confirmation
- If the user has reached their plan limit when confirming, the system informs them and suggests subscribing (or upgrading)
- The draft remains available until it expires (7 days) in case the user upgrades

### Principles
- Never auto-create documents
- Always require explicit confirmation
- Conservative by default
- If no date can be extracted, system notifies user and suggests manual creation

User confirmation serves as both trust mechanism and feedback signal.

---

## 10. Web Interface (Dashboard)

The product is email-first, but a minimal web dashboard is required for management tasks that cannot be done via email.

### Screens (v1)

| Screen | Purpose | Auth required |
|--------|---------|---------------|
| Login | Enter email to receive magic link | No |
| Document list | View all documents (active, expired, drafts) with status and next alert | Yes |
| Create document | Form: title, type, expiration date, description, recurrence, custom alert rules | Yes |
| Edit document | Same form, pre-filled. Allows editing all fields and archiving | Yes |
| Document detail | View document info, alert history, observers receiving alerts | Yes |
| Account settings | Manage observers, digest preferences, timezone, default alert rules | Yes |
| Confirm draft | Pre-filled form from forward-email flow. Accessible via signed token | Token or session |

### Design principles
- Minimal screens, minimal elements per screen
- No onboarding wizard; first action is "Add a document" or "Forward an email"
- Dashboard is secondary to email — users should rarely need to visit it
- Responsive web (no native mobile app in v1)

---

## 11. Timezone Handling

### User timezone
- Each account stores a `timezone` field (IANA timezone, e.g. `America/New_York`)
- Set during onboarding (auto-detected from browser, editable in settings)
- All user-facing dates are displayed in the user's timezone

### Internal storage
- `expiration_date` is stored as a **date** (no time component, no timezone)
- "30 days before" means 30 calendar days before the expiration date in the user's local timezone

### Cron job behavior
- Background jobs run on a fixed UTC schedule
- When evaluating alerts, the system calculates the user's current local date based on their timezone
- Alerts are sent based on the user's local date, not UTC date

---

## 12. Alerts & Notifications

### Channels
- Email only (v1)

### Alert Types
- Pre-expiration alerts
- Expiration day alert
- Digest summaries

### Default Alert Rules
- 30 days before
- 7 days before
- On expiration day

Users can override alert rules per document.

---

## 13. Digest Emails

Users can opt into:
- Weekly digest
- Monthly digest
- Or both

Both are enabled by default on account creation.

### Weekly Digest Content
- **Upcoming expirations**: documents expiring in the next 30 days
- **Recently expired**: documents that expired in the last 7 days
- **Recurring renewals**: recurring documents that were auto-renewed in the last 7 days

### Monthly Digest Content
- **Upcoming expirations**: documents expiring in the next 60 days
- **Recently expired**: documents that expired in the last 30 days
- **Recurring renewals**: recurring documents that were auto-renewed in the last 30 days
- **Monthly PDF summary**: attached to the monthly digest email (see section 14)

### Recipients
- Account owner (always, if digest is enabled)
- All active observers receive the same digest content as the owner

Digest emails are calm, concise, and informative. If there is nothing to report (no upcoming, no expired, no renewals), the digest is **not sent** (avoid empty noise).

---

## 14. Monthly PDF Summary

### Purpose
- Personal organization
- Sharing with third parties (accountant, lawyer, partner)
- Light archival record

### Content
- Report title with account owner name and generation date
- Table of all **active** documents: title, type, expiration date, days remaining
- Table of **expired** documents (last 30 days): title, type, expiration date
- Table of **recurring** documents: title, type, next expiration, frequency
- Total counts per status

### Delivery
- **Attached to the monthly digest email** (if monthly digest is enabled)
- **Downloadable from the dashboard** at any time (on-demand regeneration)
- Observers also receive the PDF if they receive the monthly digest

### Characteristics
- Generated monthly by cron job (1st of each month, before digest)
- Simple tabular layout, no heavy branding in v1
- Informational only — no legal or compliance guarantees
- Not generated for canceled accounts

---

## 15. Subscription & Billing

### Pricing (v1)
- Monthly: $6.99/month
- Annual: $66.99/year (~20% discount)
- 14-day free trial (no credit card required)

### Payment Processing
- Subscription billing handled via **Stripe Checkout + Stripe Billing**
- User is redirected to Stripe Checkout when subscribing (no custom payment forms)
- Stripe manages: card collection, recurring charges, invoices, and cancellation
- Webhooks from Stripe update account plan status (trial → active, active → canceled)
- Note: This refers to the product's own subscription billing. Expiration Guardian does **not** track or manage the user's external bills or payments (see Product Principles: "Dates, Not Money").

### Plan Limits

| Limit | Trial | Paid |
|-------|-------|------|
| Documents | 3 | Unlimited |
| Recurring items | 1 | 50 |
| Tracking horizon | 3 months | 36 months |
| Observers | 1 | 10 |

### Paywall Mechanics
- Trial expires at **14 days OR when any limit is reached** (whichever comes first)
- When trial expires by time: account becomes read-only, prompts to subscribe
- When trial hits a limit: user is informed of the limit and prompted to subscribe
- After trial expiration (either trigger): no new documents, no alerts sent, dashboard is read-only
- Existing data remains visible and exportable

### Cancellation
- Account becomes read-only (can view but not create/edit)
- Alerts and digests stop immediately
- Historical data remains accessible indefinitely
- Monthly PDF generation stops
- Observers stop receiving alerts
- User can reactivate at any time; all data is restored to active state
- Data is never deleted unless the user explicitly requests account deletion

### Data Export
- Users can export their data at any time (active or canceled)
- Export format: JSON (full data) and PDF (summary)
- Accessible from account settings

---

## 16. Out of Scope (v1)

Explicitly excluded from v1:
- Legal or compliance guarantees
- Tracking user's external bills, payments, or invoices (Expiration Guardian tracks **dates**, not money)
- Bank or provider integrations
- Mobile applications
- SMS / WhatsApp notifications
- Team workflows with multiple active users

Note: Stripe integration for the product's own subscription billing **is** in scope. What is out of scope is building billing/payment features for users to track their own financial obligations.

---

## 17. Success Metrics

Key metrics tracked from day one:
- Documents created per user
- Forward-email vs manual creation ratio
- Confirmation acceptance rate
- Alert delivery rate (via SendGrid webhooks; no open tracking in v1)
- Digest delivery rate
- Churn after trial

These metrics guide future iterations and feature prioritization.

---

## 18. Future Considerations (Non-binding)

- Mobile push notifications
- Advanced recurring rules
- Team accounts
- Compliance-oriented plans
- Smart defaults per document type

---

**Expiration Guardian v1** is intentionally minimal: trusted reminders, zero noise, and quiet reliability.

