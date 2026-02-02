# Functional Specification – Expiration Guardian v1

## 1. Product Overview

**Product name:** Expiration Guardian  
**Tagline:** *An intelligent inbox that takes care of your expirations*

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
- One **owner** (authenticated user)
- Zero or more **observers** (email addresses only)

Observers:
- Receive alerts and digests
- Cannot log in
- Cannot manage documents

---

## 6. Authentication

- Passwordless authentication (magic link via email)
- No passwords stored
- Login is optional for most interactions
- Email confirmations do not require login

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
- A recurring item can:
  - Regenerate a new expiration date automatically
  - Act as a recurring reminder
- Implementation details are abstracted from the user

### Creation
- Manual creation only (v1)

### Limits (Plan-based)
- Maximum number of recurring items
- Maximum future horizon (e.g. 12 months)

---

## 9. Forward Email Feature

### Overview
Users can forward emails containing expiration information to a dedicated system email address.

### Behavior
1. User forwards an email
2. System analyzes content and extracts:
   - Possible expiration dates
   - Document type
   - Source (sender / subject)
3. System sends a **confirmation email** to the user
4. User confirms (or edits) with a single click

### Principles
- Never auto-create documents
- Always require explicit confirmation
- Conservative by default

User confirmation serves as both trust mechanism and feedback signal.

---

## 10. Alerts & Notifications

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

## 11. Digest Emails

Users can opt into:
- Weekly digest
- Monthly digest
- Or both

Digest content:
- Upcoming expirations
- Recently expired items
- Recurring reminders

Digest emails are calm, concise, and informative.

---

## 12. Monthly PDF Summary

### Purpose
- Personal organization
- Sharing with third parties
- Light archival record

### Characteristics
- Generated monthly
- Informational only
- No legal or compliance guarantees

---

## 13. Subscription & Billing

### Pricing (v1)
- $6.99 / month
- Annual plan with discount
- 14-day free trial (no credit card required)

### Limits
- Number of documents
- Number of recurring services
- Maximum tracking horizon

### Cancellation
- Account becomes read-only
- Alerts stop
- Historical data remains accessible

---

## 14. Out of Scope (v1)

Explicitly excluded from v1:
- Legal or compliance guarantees
- Billing or payment tracking
- Bank or provider integrations
- Mobile applications
- SMS / WhatsApp notifications
- Team workflows with multiple active users

---

## 15. Success Metrics

Key metrics tracked from day one:
- Documents created per user
- Forward-email vs manual creation ratio
- Confirmation acceptance rate
- Alerts opened
- Digest engagement
- Churn after trial

These metrics guide future iterations and feature prioritization.

---

## 16. Future Considerations (Non-binding)

- Mobile push notifications
- Advanced recurring rules
- Team accounts
- Compliance-oriented plans
- Smart defaults per document type

---

**Expiration Guardian v1** is intentionally minimal: trusted reminders, zero noise, and quiet reliability.

