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
- [x] Independent re-verification pass on the load-bearing regulatory figures, against primary/authoritative
      sources where findable (BAFIA §31 and §57 text, NRB Sixth Amendment to hire-purchase rules,
      eSewa/Khalti developer docs directly). One real correction found and folded in: the "35-day notice"
      comes from the Debt Recovery Act 2058, not BAFIA §57 itself. One important open question surfaced and
      flagged prominently rather than guessed at: whether a hire-purchase company specifically has standing
      under BAFIA §57 — needs a lawyer, not more research.
- [x] Architecture-pattern research (identity layer, warehouse, RDS migration, connection pooling) — folded
      into `architecture.html` and `roadmap.html`. Confirmed the right-sized approach is simpler than generic
      enterprise patterns: one small identity service + plain API, batch ETL into one small Postgres reporting
      DB, no Kafka/CDC/PgBouncer needed at this scale.
- [x] Second commit/push with all corrections folded in

## Next
- [ ] Enable GitHub Pages on this repo — attempted via `gh api`, blocked by the local session's permission
      classifier (treats it as a settings change needing explicit approval). Needs either Ayush's approval in
      this session or manual toggle in Settings → Pages (Deploy from branch: `main`, path `/`).
- [ ] Review pass: re-read all five pages together for consistency (cross-links, repeated numbers matching
      across pages, no stale claims from earlier drafts) now that the correction pass has landed
- [ ] Consider whether the Fonepay recurring-payment gap (couldn't find their dev docs at all) is worth a
      direct outreach rather than leaving as "unresolved"
- [ ] Optional: a short primary-source follow-up on the NRB Hire Purchase PDF itself
      (nrb.org.np/contents/uploads/2026/08/Notice-1_Hire_Purchase.pdf) — WebFetch couldn't extract its text
      (legacy Nepali font encoding); someone opening it directly or OCR'ing it would upgrade several Sixth
      Amendment claims from secondary-source to primary-source confidence

## Explicitly out of scope for this pass
- GPS ETL Lambda (`trakzee-supabase-etl`) — confirmed working, not being changed
- Any actual code changes to yapigo, drivra_express_backend, or gps_dashboard — this is a planning
  document; the hardening/consolidation items it lists are proposals, not yet executed
