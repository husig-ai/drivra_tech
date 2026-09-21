# Rent-to-Own Driver Financing — Full Product & System Design

## Context

yapigo ("Drivra Upahar") runs a rewards/loyalty layer on top of Yango's Fleet API for motorbike
drivers in Nepal — tiers, points, badges, quests, a leaderboard. The business wants to extend into
letting drivers rent-to-own their motorcycle: an entire adjacent fintech capability — KYC/identity,
a driver "credit score," underwriting, financing contracts, an installment ledger, collections, and
default/repossession handling, most likely as a separate portal.

Decisions locked in with the user before designing: **standalone service** (own repo/DB, not a
yapigo module), **independent payment collection** (decoupled from Yango), **internal-only credit
score v1** (no external bureau dependency to start).

The user then asked for this to go much deeper before anything gets built: full architecture (auth,
FastAPI, database, storage/AWS vs Supabase, the complete integration map across yapigo/Yango/
payment rails), the actual UI/UX journey for every persona, competitive analysis, positioning, a
phased roadmap, and a rough cost-to-build estimate — thought through product, engineering,
operations, and driver-journey angles together, and packaged as a shareable HTML document.

**This plan file is that full design.** Once approved, the next step is publishing it as a polished
HTML artifact (the requested deliverable) — nothing else gets built this session.

---

## 1. What yapigo is today (why this is new territory, not an extension)

Confirmed by direct codebase exploration:
- yapigo is a **read-mostly rewards layer** on Yango's Fleet API, not the ride-hailing system of
  record. It never writes to a driver's Yango balance; it owns no dispatch data beyond a synced
  copy of *completed* orders.
- **No ratings, no cancellation data** anywhere — Yango's API doesn't expose it to this
  integration. "Driver performance" today means trip volume, gross fare, tenure, activity streaks.
- **No KYC/document upload capability at all** — no license/ID/photo capture, no object storage
  integration. Yango's API carries no license/KYC fields either.
- **No real money ledger** — only a points ledger (`PointTransaction`: append-only, signed delta,
  `idempotency_key`, cached balance). Yango's real driver-balance API exists in the client code but
  is deliberately unwired.
- **Reusable patterns**: the append-only ledger shape above; the `login_status` three-axis
  lifecycle (external status, admin-owned gate, `updated_by_admin_id` audit trail); the "every
  capability is a page + an endpoint over one service layer" admin console convention; the
  SparrowSMS client abstraction; APScheduler for background jobs; and the hard rule that every new
  table's migration must revoke Supabase's auto-granted `anon`/`authenticated` PostgREST access.

Isolating financing in its own service — sharing only a driver's phone/identity and a read-only
pull of yapigo's performance data — keeps regulated financial/KYC data out of the rewards app.

---

## 2. Competitive landscape & positioning

### Direct local competitor: RYD Nepal (rydnepal.com)

Already running this exact model in Kathmandu. Founded 2025, 120+ Hero Super Splendor 125cc bikes,
claims 500+ active riders, self-funded/no disclosed investors, single-SKU fleet.

- **Pricing** (three tiers, same bike): Weekly Rs 5,600/wk; **Pro Monthly Rs 7,000/wk — the only
  tier with ownership after 1.5 years**; Prepay Rs 21,000/mo (cheapest per-day rate, no ownership).
  Most of their "own it in 18 months" marketing describes only one of three plans.
- **KYC**: driving license + citizenship copy only, in person or online, "under 30 minutes." No
  income proof, no credit check, no guarantor.
- **Onboarding**: no app — web form + WhatsApp + in-person at their Kapan workshop. Rs 1,500 fuel
  coupon + a "Sagoon kit" (helmet, phone mount, raincoat) on pickup.
- **Servicing**: zero down payment, free service every 1,500km, 24/7 breakdown line, replacement
  bike within 30–40 minutes if repair runs long. Missed-payment mechanics are **not disclosed
  anywhere public** — marketed vaguely as "no penalty fees, no credit-bureau drama."
- **Positioning angle worth noting**: one rented bike is marketed as usable across
  Pathao/InDrive/Yango/Uber Bike/Tootle simultaneously — "one fixed daily cost, five earning
  streams." Their sharpest attack copy targets *other unnamed rental competitors* ("bikes
  deteriorate by month two," "you fix it, I'll think about it"), not predatory lenders or banks.
- **Trust gap**: only three self-hosted testimonials, zero independent reviews found (no
  Trustpilot/Google reviews/Reddit) — their credibility is entirely self-curated.

### Global comparables (behavior/UX patterns worth borrowing or avoiding)

| Company | Model | Notable pattern |
|---|---|---|
| **M-KOPA** | PAYG motorbike financing, Kenya | Missed payment → remote lock, term just extends, **explicitly "no penalty," no debt spiral**; can return the bike and walk away clean with deposit refunded. Strongest trust-building policy found in this research. |
| **Tugende** | Lease-to-own boda financing, Uganda/Kenya | Requires a family guarantor + 3-week training before financing — a soft underwriting layer, not just a data score. Builds its own proprietary repayment dataset via its own payments platform rather than external bureau data. |
| **Watu Credit** | Lease-to-own, East Africa | GPS remote-immobilize on default; real reputational problem — Facebook impersonation scams using their brand, and consumer-advocacy complaints about collection conduct. |
| **Moove** | Uber's financing partner, Africa/UK/India | Auto-deducts from ride earnings; Nigeria driver protests over insufficient take-home pay and aggressive in-person debt collection when marketed flexibility didn't match practice — a clear "don't let policy and practice diverge" cautionary case. |

**Category-wide finding**: none of these companies — RYD included — have any real, independently
documented UX (no screenshots, no UX pattern-library coverage anywhere). A well-designed,
transparent, actually-screenshotted driver experience would be a genuine differentiator in this
category on its own.

### Nepal digital-lending UX (for the in-app financing pattern specifically)

- **eSewa easyloan / Khalti+NIC Asia "Super Chamatkarik Loan" / Foneloan** (white-labeled across
  11+ Nepali banks' apps) all follow the same shape: **wallet/account transaction history
  substitutes for a credit bureau score**, application happens inside an already-trusted app, and
  underwriting is handed off to a licensed bank/PSP partner behind the scenes. Foneloan's
  buy-now-pay-later product embedded inside bank apps is the closest Nepali analogue to what this
  product should feel like — a financing flow embedded inside an app drivers already trust and use
  daily, rather than a separate destination like RYD Nepal's web/WhatsApp flow.

### Recommended positioning

- **Lead differentiator**: this is the only rent-to-own offer built *inside* an app the driver
  already uses for their daily rewards/earnings tracking — meaning real usage history (tenure,
  trip volume, consistency) can translate directly into faster approval and better terms for
  loyal drivers, something neither RYD Nepal (web/WhatsApp-only, license+citizenship-only KYC) nor
  a bank (needs a traditional file) can offer.
- **Trust policy, not just messaging**: adopt an M-KOPA-style explicit no-debt-spiral policy
  (missed payment extends term / bike locks rather than compounding penalty fees) and make sure
  product mechanics match the marketing — Moove's Nigeria experience shows what happens when they
  don't. **Correction for Nepal**: NRB's hire-purchase rules already **cap** penalty interest at
  2pp/year and explicitly ban penalty-on-penalty compounding (see §3) — so "no debt spiral" is
  achievable as a *compliant*, not just aspirational, policy if Path A is chosen.
- **Transparency as a feature**: a visible in-app ownership-progress tracker (percentage-owned,
  remaining installments, clear payoff date) — something no competitor researched currently shows.
- **Tie into the existing loyalty system**: better rent-to-own terms (lower down payment or
  priority allocation) for higher-tier yapigo drivers — a natural synergy this product has and RYD
  Nepal structurally can't replicate.
- Naming/branding (e.g. extending "Upahar" with an ownership-themed Nepali name) is an open
  question for the team's Nepali-speaking marketing input — not resolved here.

---

## 3. Regulatory & legal structuring (gates almost everything else)

Nepal Rastra Bank (NRB) regulation, not engineering choice, determines what's actually buildable:

- Under BAFIA, **only a licensed BFI (Class A/B/C/D) may originate/hold consumer loans** — a pure
  tech company cannot lend directly.
- Vehicle financing specifically sits under a **Hire Purchase Company** license: **NPR 300M**
  minimum paid-up capital, **60% LTV cap (≥40% down payment required)**, service fee capped at 1%,
  overdue-penalty interest capped at **2pp/year with no compounding**, single-borrower
  concentration limits, NRB "fit and proper" test for founders/directors.
- **CIB Nepal** (credit bureau) is only accessible to NRB-registered BFIs.
- A **licensed BFI** gets BAFIA §57 self-help repossession (35-day notice, no court order needed,
  three public-auction attempts before Debt Recovery Tribunal for any residual); an unlicensed
  lessor most likely only has ordinary contract/property remedies.
- **AML/KYC obligations apply regardless of structure**: e-KYC with biometric verification under
  the Asset (Money) Laundering Prevention Act 2008; suspicious-transaction reports filed via goAML
  within **3 days**; records retained a minimum of **5 years**.

| Path | Shape | Trade-off |
|---|---|---|
| **A. Partner with a licensed lender** (e.g. approach an existing Class C hire-purchase company such as Hulas Finserv, Nepal's largest two-wheeler financier) — this service is the distribution + underwriting-data + servicing tech layer, the partner is lender of record | Mirrors Grab's GrabScheme in Indonesia (platform = channel/data, bank-backed multifinance company = balance sheet). Avoids the NPR 300M capital bar; gets §57 repossession rights and CIB access through the partner. | Needs a real partnership deal; ≥40% down payment required by regulation; product terms constrained by partner's risk appetite. |
| **B. Lease-with-purchase-option** (RYD Nepal's structure) | Lower capital bar, can offer low/zero down payment, faster to launch. | Legally less tested; likely loses §57 fast-repossession privilege; the lease/hire-purchase line isn't bright and could draw regulatory attention as the space grows. |
| **C. Become a licensed Hire Purchase Company** | Full control, best long-term economics if this becomes a core line. | NPR 300M capital + full NRB compliance burden — a multi-quarter/legal undertaking, not a v1 move. |

**Recommendation for design purposes: architect for Path A as the default assumption** — closest
fit to the decisions already locked in, cleanest regulatory footing without a NPR 300M capital
commitment. **This is a legal decision for Nepali financial-regulatory counsel, not resolved here**
— the data model below treats "lender of record" as a field, so it doesn't block starting the
KYC/underwriting/servicing build, but it must be resolved before signing real contracts with
drivers.

---

## 4. Driver journey (end to end)

1. **Discovery** — an "Own Your Bike" entry point inside the existing Upahar app, surfaced based on
   tenure/tier; a lightweight pre-check against existing yapigo data ("you may be eligible") before
   any paperwork.
2. **Application** — desired plan/vehicle, down payment amount, basic details.
3. **KYC** — citizenship certificate + National ID + driving license photo capture, selfie for
   liveness/face-match, optional guarantor capture for thin-file applicants (mirrors Watu/Tugende's
   cold-start pattern).
4. **Underwriting decision** — rules-based scorecard (§7) → approve / decline / refer-to-manual,
   always admin-overridable with an audit-logged reason.
5. **Contract & terms** — vehicle allocation, down payment, installment schedule, agreement type
   per the resolved legal path (§3); e-signature/acceptance captured.
6. **Disbursement/handover** — lien recorded (bluebook annotation + STRO filing where required),
   physical handover logged, first schedule activated.
7. **Servicing** — recurring installments, reminders, payment intake, ledger posting, an always-
   visible ownership-progress tracker.
8. **Ongoing monitoring** — periodic re-pull of yapigo performance data catches early drift (falling
   trip volume = early risk signal) before a payment is even missed.
9. **Completion** (final installment → ownership transfer, lien released) **or** **delinquency →
   dunning → default/repossession** per the resolved legal path.

---

## 5. System architecture

**Standalone service**: own repo, own Postgres database (Supabase-hosted, reusing existing team
know-how), reusing yapigo's stack — FastAPI + SQLAlchemy 2.0 + Alembic + Jinja2/HTMX admin console
— but its own deployment and security perimeter, since it holds KYC documents and financial data.

**Integration seam with yapigo**: a narrow read-only internal API yapigo exposes (e.g.
`GET /internal/drivers/{phone}/profile` → tenure, `lifetime_gross`, `lifetime_trips`, streaks,
`login_status`), authenticated with a service-to-service token — not direct DB access. Keeps both
systems' schemas independently evolvable, matching the "standalone service" decision.

**Auth**: reuse yapigo's proven shape, but decoupled. Driver auth = phone + SparrowSMS OTP
(reuse the client), issuing a JWT scoped to a distinct `financing_driver` audience — deliberately
**not** SSO-trusting yapigo's session, to keep the services' security boundaries independent for
v1 (a "Continue with Upahar" bridge is a reasonable later enhancement, not a v1 requirement). Admin
auth mirrors yapigo's bcrypt+JWT pattern, but with more granular roles than yapigo's owner/staff
split: **KYC reviewer**, **underwriter**, **collections/ops**, **owner/compliance**.

**Storage decision — Supabase Storage over raw AWS S3 for v1**: at this scale (a few thousand
drivers × ~5 documents), storage/egress cost is trivial on either (low single-digit dollars/month).
The deciding factor is **integration effort**: Supabase Storage is wired directly into Postgres
Row-Level Security, so "only the assigned KYC reviewer or the document's owner can read this file"
can be written as a SQL policy against the same schema already modeling drivers/loans/officers —
versus hand-rolling IAM/Cognito-equivalent logic for raw S3. Supabase Storage is itself
S3-compatible under the hood, so this isn't a lock-in decision; raw S3 becomes worth revisiting
mainly for KMS-based per-key audit trails if a regulator/auditor later requires that level of
detail.

**eKYC/document verification — DIY AWS Textract + Rekognition for v1, not a vendor**: per-driver
verification cost is roughly **$0.07** (Textract `AnalyzeID` for the 2 ID documents + Rekognition
liveness check + face-compare) versus **$2–7 per verification** for global vendors like
Jumio/Onfido. Two Nepal-aware vendors exist (Shufti Pro, FACEKI) but neither publishes pricing —
worth getting quotes, but the DIY path is roughly an order of magnitude cheaper and keeps v1 lean;
the trade-off is owning fraud-detection tuning and liability in-house rather than transferring it
to a vendor, which is acceptable for a manually-reviewed v1 (§10 Phase 1) where a human still signs
off on every KYC approval regardless.

**Data model (key tables, not full DDL)**:

| Table | Purpose | Key fields |
|---|---|---|
| `drivers` | Identity, linked to yapigo via phone | phone, name, dob, address, kyc_status |
| `kyc_documents` | Uploaded documents | driver_id, doc_type (citizenship/NID/license/PAN/selfie), file_path, status, reviewed_by, reviewed_at |
| `guarantors` | Cold-start co-signers | driver_id, name, phone, id_document_path, relationship |
| `vehicles` | Asset registry — deliberately separate from yapigo's `drivers.vehicle_*` columns, which are wholesale-overwritten by Yango sync and reflect only the driver's *current* Yango-assigned vehicle | chassis_number, engine_number, plate_number, brand_model, cost, bluebook_ref, lien_status, stro_reference |
| `loan_products` | Configurable plan terms | tenure_months, down_payment_pct, interest_rate, penalty_rate (mirrors yapigo's `RedemptionRule` singleton-config pattern) |
| `applications` | Origination workflow state | driver_id, product_id, vehicle_id, status, decision_reason, decided_by, decided_at |
| `loan_accounts` | The active agreement | application_id, driver_id, vehicle_id, principal, down_payment, tenure_months, status, **lender_of_record** (yapigo-financing vs named BFI partner — the field that absorbs the §3 legal-path decision without blocking the build) |
| `repayment_schedule` | Amortization | loan_account_id, installment_no, due_date, amount_due, status |
| `ledger_transactions` | Append-only ledger, mirrors `PointTransaction`'s shape | loan_account_id, kind (disbursement/installment_paid/penalty/waiver/writeoff), signed amount, channel, idempotency_key |
| `delinquency_events` | Collections state | loan_account_id, bucket (current/watch/substandard/loss), entered_at |
| `audit_log` | Every state change/admin action | actor_admin_id, action, entity_type, entity_id, before/after, created_at (mirrors `login_status`'s `updated_by_admin_id` pattern throughout, not just for one field) |

Every new table's migration revokes Supabase's default `anon`/`authenticated` PostgREST grants, per
yapigo's existing hard rule.

**Full integration map**:

```
Upahar (app_rider, Flutter)  ──▶  yapigo backend  ──(read-only sync)──▶  Yango Fleet API
        │                              │
        │  "Own Your Bike" entry      │ GET /internal/drivers/{phone}/profile
        ▼                              ▼
  Financing driver portal  ◀──────  Financing backend (standalone FastAPI service)
        │                              │
        │                              ├──▶ Supabase Storage (KYC docs, RLS-scoped)
        │                              ├──▶ AWS Textract + Rekognition (eKYC v1)
        │                              ├──▶ SparrowSMS (reused client — OTP + payment reminders)
        │                              ├──▶ Payment rails: ConnectIPS (mandate/pull, flat Rs 2–8) +
        │                              │     Fonepay QR (free merchant-side) preferred;
        │                              │     eSewa/Khalti supported as manual-push fallback
        │                              ├──▶ Partner BFI systems (if Path A) — underwriting handoff,
        │                              │     CIB Nepal check, regulatory reporting
        │                              └──▶ DoTM bluebook + Secured Transactions Registry Office —
        │                                    offline/manual workflow, no public API exists
        ▼
  Admin console (Jinja+HTMX, same convention as yapigo)
```

---

## 6. UI/UX — three surfaces

**Driver-facing** (start as a lightweight web portal, not a second Flutter app — revisit once the
flow is proven; entry point linked from the existing Upahar app):
1. Eligibility/offer screen ("you may be eligible" pre-check)
2. Application form (plan selection, down payment)
3. KYC capture flow (camera-first document capture + selfie, clear per-document status)
4. Guarantor entry (conditional, for thin-file applicants)
5. Application status tracker
6. Contract review + e-signature
7. **Ownership dashboard** (the differentiating screen): running balance, percentage-owned,
   next payment due, full payment history, one-tap "pay now" linking to the preferred rail
8. Reminder/notification surface (SMS today; in-app/push later)

**Admin console** (new tabs, following yapigo's existing "page + endpoint" convention):
Underwriting queue → KYC review → Applications/Contracts → Ledger/Collections dashboard →
Delinquency queue → Vehicle/Asset registry → Reports (NRB-facing: LTV, concentration, rate-cap
compliance).

**Field agent tool** (simple, mobile-friendly web page, not a full app): record a cash payment
collected in person — matches how Nepali MFIs and Hulas Finserv actually collect from
lower-smartphone-penetration drivers, per the payments research (§8).

---

## 7. Credit scoring v1 (internal-only, as decided)

A **rules-based scorecard**, not an ML model — every thin-file lender researched (Watu, Tugende)
starts here, and there's no proprietary repayment history yet to train anything on.

**Inputs available day one, all from yapigo**: tenure (`activated_at`), `lifetime_trips` /
`lifetime_gross` and their consistency (`daily_stats`/`weekly_stats`), `current_streak` /
`longest_streak`, `login_status == active`, tier/XP.

**Cold-start handling**: guarantor requirement and/or higher down payment for drivers with little
yapigo history — mirrors Watu's guarantor model and Tugende's reference-plus-training approach,
rather than an outright decline.

**Decision flow**: hard gates (KYC complete, active status, minimum tenure) → weighted score →
approve / decline / refer-to-manual-review, every decision and override audit-logged. Evolves into
a behavioral model once the service has its own repayment-history dataset (mirrors Tugende's
approach of building proprietary data via its own servicing platform).

Explicitly **not** in v1: telematics/GPS-based scoring or remote asset-disable enforcement — the
Moove/M-KOPA/Watu pattern is powerful but adds IoT hardware dependency and real reputational risk
(documented tracker-collusion allegations against comparable East African lenders); revisit only
once the core product is proven.

---

## 8. Payments & collections

Confirmed: Nepali wallets (eSewa, Khalti, Fonepay) **do not support auto-debit/recurring pull** —
collection is a manual push each cycle. **ConnectIPS/NCHL supports mandate-based direct debit**,
already used for hire-purchase EMI collection — the best path to real automation for drivers with a
bank account.

**Fee comparison** (shapes which rail to steer drivers toward): ConnectIPS is flat and cheap
(Rs 2–8/transaction regardless of amount, NRB bars passing this to customers); Fonepay QR is
**free for merchants**; eSewa/Khalti card-based loading runs 1.75–5%. **Product decision: default
drivers toward ConnectIPS or Fonepay QR, support eSewa/Khalti as fallback, not the primary path.**

v1 architecture, matching how the market leader (Hulas Finserv) already operates:
- SMS reminder (reused SparrowSMS client) ahead of each due date → driver pays via
  ConnectIPS/Fonepay QR (or eSewa/Khalti as fallback) → reconciliation marks the ledger entry paid.
- ConnectIPS mandate as an opt-in automated path once the manual flow is proven.
- Cash/agent collection fallback via the field agent tool (§6).
- Delinquency bucketing (current → watch → substandard → loss, standard BFI classification) drives
  dunning and, eventually, the repossession trigger.

---

## 9. Compliance & risk notes

- Every new table follows yapigo's Supabase PostgREST-grant-revoke migration convention.
- KYC documents are the most sensitive data this system holds: Supabase Storage + RLS (§5),
  encrypted at rest, **5-year minimum retention** per AML record-keeping rules, suspicious-activity
  reports filed via goAML within **3 days** of detection.
- Vehicle lien registered both on the bluebook (everyday check) and at the Secured Transactions
  Registry Office under the Secured Transactions Act 2063 (formal legal priority) — confirm the
  interplay of the two with counsel.
- If Path A (partner lender): CIB Nepal checks and §57 repossession privileges flow through the
  partner — the partnership contract needs to define who owns which compliance obligation.
- Any future IoT/GPS/remote-disable capability needs a tamper-evident, auditable data trail from
  day one, given documented disputes in comparable markets over tracker reliability.

---

## 10. Cost to build (rough planning estimate — not a quote)

**Team & timeline for Phase 1 MVP**: 3–5 people — one senior backend engineer owning the
lending/fintech domain, one additional backend engineer for KYC/payment integrations (reusing
yapigo's Supabase/FastAPI patterns directly cuts this down from generic industry estimates), one
mobile/frontend engineer (lighter load if reusing the Upahar app shell for the driver entry point),
part-time product/compliance-ops (the NRB KYC/AML workstream is real work, not overhead), part-time
QA/DevOps. **Roughly 4–7 months** to a v1 covering origination + DIY-verified KYC + basic servicing/
collections + SMS + payment-gateway integration — narrower than generic 8–10 month industry
estimates because of pattern reuse and a deliberately narrow v1 scope (single asset class, one
country, manual underwriting acceptable at first, same path Tugende/M-KOPA reportedly took before
automating).

**Rough engineering cost**: no Nepal-specific benchmark exists publicly. Using Asia-blended dev
rates against a lending-app-sized effort estimate (~2,000–2,500 hours) gives roughly **$40K–$125K**
in engineering time; adding the commonly-cited 30–40% compliance overhead for a regulated product
puts an all-in planning range around **$60K–$170K** — likely lower with in-house Nepal-based
salaries rather than agency billing rates. Treat this as a bounding estimate for the planning
conversation, not a budget commitment.

**Rough monthly infra cost at 1,000–5,000 driver scale**:

| Item | Monthly range | Basis |
|---|---|---|
| Supabase (Postgres + Storage + Auth) | $25–$600 | Pro plan up, scales with usage |
| eKYC (DIY Textract/Rekognition) | $15–$75 | ~$0.07/verification × est. monthly new-driver volume |
| SMS (SparrowSMS, 5–10/driver/mo) | $40–$375 | ~NPR 0.85–1.4/SMS |
| Payment/collection fees | Near-zero if ConnectIPS/Fonepay-led; higher if card/wallet-load-heavy | §8 |
| App/backend hosting | $20–$100 | Small container/VM |
| **Rough total** | **~$150–$1,000+/month** | Most sensitive to eKYC vendor choice (vs. DIY) and payment-rail mix |

**Key unresolved cost inputs**: actual Shufti Pro/FACEKI per-verification quotes (neither publishes
pricing), actual negotiated eSewa/Khalti merchant fees, and — the single biggest unknown — whatever
timeline/cost the §3 regulatory path adds (BFI partnership negotiation vs. own-license pursuit).

---

## 11. Roadmap

**Phase 0 — Not engineering.** Resolve the §3 legal structuring path with Nepali
financial-regulatory counsel; direct competitive diligence on RYD Nepal; start conversations with a
potential licensed-lender partner if pursuing Path A.

**Phase 1 — MVP, human-in-the-loop (~4–7 months, per §10).** Driver/KYC data model, document
upload with DIY Textract/Rekognition-assisted (but human-approved) verification, manual
underwriting review (no automated scorecard yet), basic servicing ledger, admin portal, SMS
reminders, ConnectIPS/Fonepay-first manual payment reconciliation, field-agent cash collection
tool. Goal: a small pilot cohort (10–20 drivers) fully human-verified before automating anything.

**Phase 2 — Automate what Phase 1 proved.** Rules-based scorecard (§7) wired to yapigo's read-only
performance API, guarantor workflow, ConnectIPS mandate automation, delinquency dashboard, formal
lien-registration workflow tracking.

**Phase 3 — Scale.** Behavioral scoring from the service's own accumulated repayment data, driver
self-service native app (folded into Upahar or standalone), evaluate telematics/asset-monitoring
only if the reputational/hardware trade-off is judged worth it.

---

## 12. Open questions for the business

1. Legal structuring path (§3) — gates down-payment minimums, repossession rights, and go-to-market
   timeline.
2. Capital source / lending partner — already in view, or does this design need to inform partner
   outreach (e.g. to Hulas Finserv or a comparable Class C/D institution)?
3. Vehicle sourcing — bulk dealer/brand relationship, or drivers bring their own vehicle for
   refinancing?
4. Target down payment/tenure economics — Path A likely forces ≥40% down under NRB's LTV cap, Path
   B could go much lower (RYD Nepal advertises zero down).
5. Positioning vs. RYD Nepal specifically — displace them, differentiate on trust/UX/loyalty-tier
   terms, or is there room for both given the size of the driver population.
6. Branding/naming for the offering (see §2) — a marketing decision, not resolved here.

---

## 13. Next step

This plan file is the full design. Once approved, the immediate next action is publishing it as a
polished HTML artifact (product overview, architecture diagrams, UI journey, competitive/
positioning summary, roadmap, cost estimate) for the user to review and share — no code gets
written this session. After that, the way to validate the design itself is pressure-testing it
against 2–3 concrete driver scenarios (a first-time applicant with 2 months of yapigo tenure, a
top-tier driver with a year of history, a driver who goes delinquent halfway through) once the §3
legal path has a real answer, since that answer changes concrete details throughout (down payment
minimum, whether a partner lender's own KYC/underwriting requirements stack on top of the internal
scorecard, who holds repossession authority).
