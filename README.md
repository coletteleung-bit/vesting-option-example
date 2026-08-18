# Vesting option explorer

An interactive, front-end app for leadership to explore three LTI transition designs —
**Option 1 (100/15/5/5)**, **Option 2 (100/50 & 50/50)**, and **Option 3 (100/75/50/25)**
— on a single person's situation. Enter someone's scheduled legacy vesting and see what
each option grants them and what they vest, year by year.

Open **[`index.html`](index.html)** in any browser — no server or build step. It also
serves directly from GitHub Pages.

**Logic reference:** [`LTI_transition_engine_v2.md`](LTI_transition_engine_v2.md)
documents exactly how Year 1 grant values are computed, the rating grid, promotion
handling, and per-schedule behavior — read that before changing the engine or
interpreting an unexpected result.

## What it does

- **Side-by-side comparison** of all three options (any can be hidden via checkbox):
  steady-state settle point, total face granted, highest/lowest vest year.
- **Total annual vest chart** as **stacked bars** — each year shows legacy vest (light)
  stacked under new-grant vest (dark) per option, with a market-anchor (M) reference line.
  Toggles for **Notional vs Current** legacy basis, and **Stacked bars vs Lines**.
- **Year-by-year table** with legacy (notional + current), new vest, total, and % of M
  per option.
- **"How Year 1 value is set"** table — shows the A (notional), B (current), and floor
  calculations per year with the winner highlighted, so it's visible *why* a grant came
  out the size it did.
- **"What gets granted"** grant ladder per option, including legacy vesting (notional and
  current rows), a total annual vest sum row, and % of M.

## Inputs (left panel)

- **Presets** — the eight modelled personas (A–H) plus Blank, each with an individual
  NHG value set (see [`LTI_transition_engine_v2.md`](LTI_transition_engine_v2.md) §9).
- **Shared scenarios** — baked into `index.html`, so anyone opening the deployed link
  sees the same set. Anonymized by policy (role/hire-year labels or a bare initial —
  never a real name paired with individual comp figures). Add one via "Copy current as
  JSON," paste into the `SHARED_SCENARIOS` constant, commit, push.
- **Your saved scenarios** — personal, stored in your browser's `localStorage` only.
- **Market anchor (M)** and **individual NHG value** — `H = max(NHG, M)` is the anchor
  used in the Year 1 formula.
- **Transition year**, **promotion year**, and **legacy vesting** per year — now split
  into **notional (N)** and **current (C)** values, since the model takes the higher of
  a notional-based and a current-based calculation.
- **Rules** — independent pre-transition tail toggles for Option 1 and Option 3, new-hire
  treatment, and an annual-vs-transition-year-only toggle for the Year 1 sizing logic.
- **Assumptions** — promotion multiple, Option 2's 1.15× premium, and Option 2's refresh
  factor, all editable live.
- **Rating grid** — six performance ratings (1–6), each with six editable multiples (A-leg
  for existing employees, A-leg for new hires under two different columns, B-leg, floor,
  and promotion scaling). Nothing is hardcoded — change any cell and every output re-runs.

## Model behavior of note

- **Year 1 value = higher of A (notional), B (current), or the floor.** This replaced the
  earlier gap-sizing engine entirely. See
  [`LTI_transition_engine_v2.md`](LTI_transition_engine_v2.md) for the exact formulas.
- **Promotion scales with performance** via the rating grid's "Promo" column — one lever
  governs the promotion-year ceiling for Options 1/3 and Option 2's permanent promoted
  level.
- **The floor is unconditional** and, by default, recomputed every year — it pays even
  when legacy vesting already exceeds every ceiling. This is the model's largest cost
  lever; see §10 of the engine doc.

## Files

- `index.html` — the app.
- `LTI_transition_engine_v2.md` — **current** engine logic: Year 1 sizing, rating grid,
  promotion, per-schedule behavior, verified worked checks, and known open items.
- `LTI_transition_options_reference.md` — the original Option 1 vs Option 2 gap-sizing
  ruleset. **Superseded** — kept for historical context only, does not describe current
  app behavior.
- `LTI_transition_model_v10.xlsx` — the source workbook for the superseded v1 ruleset.
- `LTI_transition_explorer.html` — the original single-view explorer this app builds on
  (also reflects the superseded v1 ruleset).
