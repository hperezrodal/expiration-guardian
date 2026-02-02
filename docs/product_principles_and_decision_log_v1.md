# Product Principles & Decision Log  
**Expiration Guardian v1**

---

## Purpose of This Document

This document defines the **non-negotiable principles, explicit trade-offs, and frozen decisions** that guide Expiration Guardian v1.

Its goal is to:
- Prevent product drift
- Reduce decision fatigue
- Align product, tech, and business choices
- Serve as a long-term reference for future iterations

This is an **internal-facing document**, but safe to share with trusted collaborators.

---

## Core Product Philosophy

**"Never miss what matters."**

_(Extended positioning: "An intelligent inbox that takes care of your expirations.")_

Expiration Guardian exists to remove cognitive load related to dates, expirations, and renewals — without becoming noisy, complex, or risky.

The product succeeds when users **stop thinking about it**.

---

## Product Principles (Non-Negotiable)

### 1. Silence Beats Features
If a feature increases noise, friction, or attention cost, it does not ship.

We optimize for:
- Fewer interactions
- Fewer notifications
- Fewer decisions

Not for:
- Power-user workflows
- Dashboards-first experiences

---

### 2. Confirmation Over Automation
We prefer explicit confirmation to silent assumptions.

The system may:
- Suggest
- Infer
- Recommend

But it must **never create expirations without user confirmation**.

False positives are worse than missed suggestions.

---

### 3. Dates, Not Money
Expiration Guardian is **not a billing product**.

We do not handle:
- Amounts
- Payments
- Invoices
- Financial calculations

Only:
- Dates
- Context
- Alerts

---

### 4. Individuals First, Teams Later
The product is designed for:
- Individuals
- Freelancers
- Very small businesses

Any future team feature must:
- Work perfectly for a single user
- Degrade gracefully without collaboration

No feature should *require* a team to make sense.

---

### 5. Trust Is the Product
Users trust Expiration Guardian with things they don’t want to forget.

Therefore:
- Conservative behavior is preferred
- Transparency beats magic
- Errors must be acknowledged clearly

We never claim legal, financial, or compliance guarantees.

---

## Explicit Non-Goals (v1)

The following are **intentionally out of scope** for v1:

- Tracking user's external payments, bills, or invoices (we track **dates**, not money)
- Bank or financial provider integrations
- Real-time push notifications
- Mobile apps
- Legal or compliance guarantees
- Deep document storage or scanning
- AI auto-actions without confirmation
- Multi-language support

Note: Stripe integration for the product's **own** subscription billing is in scope. The "Dates, Not Money" principle refers to not building features that track users' financial obligations — not to avoiding payment collection for the product itself.

If a feature proposal touches one of these non-goals, it is rejected by default.

---

## Key Product Decisions (Frozen for v1)

### Experience
- Assistant-first, dashboard-secondary
- "Set & forget" by default
- Alerts are the primary interaction

### Input Methods
- Manual creation (baseline)
- Forwarded email → suggestion → confirmation

### Notifications
- Email only
- User-configurable alert timing
- Weekly and/or monthly digest

### Recurring Items
- Supported in basic form
- Monthly and annual only
- Manual setup only
- No billing logic

### Authentication
- Passwordless (magic link only)

### Pricing
- $6.99 / month
- Optional annual plan with discount
- Free trial included

---

## Decision Log (Why We Chose This)

### Why Email-Only Notifications?
- Lowest friction
- Universal
- Matches "inbox assistant" mental model
- Avoids notification fatigue

---

### Why Passwordless Auth?
- Faster onboarding
- Fewer credentials to manage
- Matches low-touch philosophy
- Reduces support burden

---

### Why No Billing Logic?
- High complexity
- High liability
- Distracts from core value (dates)
- Increases error cost dramatically

---

## Metrics That Matter (v1)

We measure **behavior, not vanity metrics**.

Primary metrics:
- % of suggested expirations confirmed (forward-email acceptance rate)
- Alert delivery rate (sent + delivered via SendGrid webhooks)
- Time-to-first-alert (days from signup to first alert sent)
- Churn after first alert cycle

Secondary metrics:
- Documents per user
- Recurring items per user
- Digest delivery rate

Note: Email **open tracking is not used in v1** due to privacy concerns (Apple Mail Privacy Protection, etc.) and unreliability. We track delivery, not opens.

Metrics we intentionally ignore:
- Daily active usage
- Time spent in app
- Feature click-throughs
- Email open rates

---

## Experimentation Guardrails

We allow experiments only if they:
- Do not increase notification volume by default
- Do not auto-create data without confirmation
- Are reversible within one release cycle

---

## Kill / Pivot Criteria

We explicitly define failure conditions.

After 60–90 days:

- If <20% of email-based suggestions are confirmed  
  → Forward-email flow is deprioritized

- If >40% churn happens before first alert  
  → Onboarding is flawed

- If users request billing features repeatedly  
  → Re-evaluate positioning, not features

---

## Ethical Boundaries

- If the user stops paying, data becomes read-only
- Users can export their data at any time
- No dark patterns in alerts or renewals
- No fear-based messaging

Peace of mind is not compatible with pressure.

---

## Final Note

When in doubt, ask:

> *Does this help the user forget about expirations — or does it give them something new to worry about?*

If it’s the latter, it doesn’t ship.

---

**Expiration Guardian**  
Product Principles & Decision Log — v1
