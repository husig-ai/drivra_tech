# TODO — Drivra Tech

Living checklist for this document while it's being built. Updated as work lands; each checkpoint is a
commit, so `git log` is the actual progress history.

## Done
- [x] Read yapigo (Drivra Upahar) end to end: models, auth, admin console, sync pipeline, deploy
- [x] Read drivra_express_backend, gps_dashboard, drivra, drivra-web — confirmed what's actually shared
      infra (Upahar ↔ GPS Dashboard's Supabase project) vs. already isolated (Express's own Postgres)
- [x] Competitive research: RYD Nepal (direct local competitor), M-KOPA, Tugende, Watu, Moove, Grab,
      Uber Vehicle Solutions, Ola, Bangladesh bank financing, Bajaj/Hero FinCorp
- [x] Nepal regulatory research: NRB/BAFIA lending rules, hire-purchase licensing, KYC/AML, vehicle
      lien/repossession, payment rail capabilities (first pass)
- [x] Cost/infra benchmarking: eKYC vendor pricing, S3 vs Supabase Storage, SMS/payment rail fees,
      rough team/timeline/cost estimate
- [x] `index.html`, `product.html`, `architecture.html`, `financing.html`, `roadmap.html` — first
      complete draft of all five pages
- [x] Repo created, first push

## In progress
- [ ] Independent re-verification pass on the load-bearing regulatory figures (NPR 300M capital, 60%
      LTV cap, BAFIA §57 scope) against more authoritative sources — agent running, will fold in when back
- [ ] Research on identity-layer architecture patterns (event-driven vs. CDC vs. batch ETL), AWS RDS
      migration practicalities, and connection-pooling thresholds — same agent, feeds into `architecture.html`

## Next
- [ ] Fold verification-pass findings into `financing.html` (regulatory section) and `architecture.html`
      (vision/identity-layer section) — correct anything that doesn't hold up, cite more precisely what does
- [ ] Expand `architecture.html` with the concrete identity-layer design once the pattern research lands
      (which of: dedicated identity service / event bus / CDC / batch ETL fits a team this size)
- [ ] Add a proper `README.md` for the repo root
- [ ] Enable GitHub Pages on this repo
- [ ] Review pass: re-read all five pages together for consistency (cross-links, repeated numbers matching
      across pages, no stale claims from earlier drafts)
- [ ] Confirm with Ayush whether the specific Supabase project ref / literal config values should stay out
      of the public site (currently kept narrative-only, no literal keys or project refs published)

## Explicitly out of scope for this pass
- GPS ETL Lambda (`trakzee-supabase-etl`) — confirmed working, not being changed
- Any actual code changes to yapigo, drivra_express_backend, or gps_dashboard — this is a planning
  document; the hardening/consolidation items it lists are proposals, not yet executed
