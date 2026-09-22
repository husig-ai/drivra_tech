# Drivra Tech

Product and engineering strategy for the Drivra Mobility platform: Drivra Upahar (driver rewards),
Drivra Express (B2B delivery), the GPS fleet dashboard, and a new rent-to-own driver financing product.

This started as a design exercise for the financing product specifically, then widened into a full
platform audit once it became clear the financing product's credit signal depends on data quality and
isolation the current stack doesn't fully have yet. Both are covered here.

**Read it as a site:** published via GitHub Pages at the repo's Pages URL. Until then, open `index.html`
directly, or browse the pages below. Built on [PaperCSS](https://www.getpapercss.com/) plus
[Mermaid](https://mermaid.js.org/) diagrams in its hand-drawn look, with a color code used throughout:
green for built and fine, amber for built but needing rethinking, gray for not built yet.

## Pages

| Page | Covers |
|---|---|
| [index.html](index.html) | The whole document on one page: portfolio, headline finding, system landscape, debt inventory, vision, financing summary, roadmap and cost. Stands alone if that's all you need |
| [product.html](product.html) | Product portfolio and driver journey across all four products |
| [architecture.html](architecture.html) | Engineering audit: current state, debt inventory, low-hanging fruit vs. real investment, the "one source of truth" vision |
| [financing.html](financing.html) | Full rent-to-own design: competitive landscape, Nepal regulation, system architecture, credit scoring, payments |
| [roadmap.html](roadmap.html) | Phased plan tying platform hardening and the financing build together, plus a first-6-months cost estimate |

## Process

[`PLAN.md`](PLAN.md) is the working plan this was built from. [`TODO.md`](TODO.md) is the live checklist,
commits track progress against it, so `git log` is the actual build history, not just the end state.

## Method

Everything here traces back to a real file, a real commit, or a cited external source. The product-portfolio
and architecture sections come from directly reading the five relevant repositories: their models,
migrations, deployment configs, and their own READMEs, not from documentation alone. The rent-to-own design
draws on that same codebase reading plus external research: competitor products, Nepal's lending/KYC
regulation, Nepal's payment rail landscape, and cost/infra benchmarking. Where a number is a genuine estimate
rather than a confirmed fact, it's labeled as one.

## Status

Draft. See `TODO.md` for what's still open: in particular, an independent re-verification pass on the
load-bearing Nepal regulatory figures is in progress and not yet folded in everywhere.
