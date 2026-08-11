# Vesting option explorer

An interactive, front-end app for leadership to explore the two LTI transition designs
(**Option 1 — 100/15/5/5** and **Option 2 — 100/50 & 50/50**) on a single person's
situation. Enter someone's scheduled legacy vesting, and see what each option grants
them and what they vest, year by year.

Open **[`index.html`](index.html)** in any browser — no server or build step. It also
serves directly from GitHub Pages.

## What it does

- **Side-by-side comparison** of Option 1 vs Option 2: steady-state settle point, total
  face granted, highest/lowest vest year.
- **Total annual vest chart** as **stacked bars** — each year shows legacy vest (light)
  stacked under new-grant vest (dark) for both options, with a steady-state ceiling tick.
  A **Lines** toggle switches back to the original line view.
- **Year-by-year table** with legacy, new vest, total, % of ceiling, and the option delta.
- **"What gets granted"** grant ladder per option, now including a **Legacy vesting**
  row and a **Total annual vest** sum row (plus % of ceiling).

## Inputs (left panel)

- **Presets** — the eight modelled personas (A–H) plus Blank.
- **Saved scenarios** — save the current setup (real examples: LTI target, legacy vest
  per year, promotion, ratings, assumptions) under a name. Stored in your browser
  (`localStorage`); load or delete with one click.
- **Annual LTI target**, **transition year**, **promotion year**, and **legacy vest +
  performance rating per year**.
- **Rules** — pre-transition tail toggle (Option 1) and new-hire treatment.
- **Assumptions** — the model's policy dials, editable live: base ceiling (1.25×),
  high-perf ceiling (1.5×), Option 1 promo multiple (2.0×), Option 2 new-hire level
  (1.15×), Option 2 refresh factor (0.5×), and the gap-grant floor (0.25×). Nothing is
  hardcoded — change a dial and every output re-runs.

## Model behavior of note

- **Post-promotion ceiling** rises to **2× the current-level target** from the promotion
  year onward — 200 (= 2×100) under Option 1, 230 (= 2×1.15×100) under Option 2 — and the
  chart, % ceiling, and overshoot flags all reflect it. The grant-sizing math itself is
  unchanged and still reconciles to the validated engine.
- Gap-sizing (`grant = ceiling − scheduled`) stays scoped to the transition year and
  promotion years; everywhere else grants are full size. See
  [`LTI_transition_options_reference.md`](LTI_transition_options_reference.md) for the
  full ruleset.

## Files

- `index.html` — the app.
- `LTI_transition_options_reference.md` — the validated Option 1 vs Option 2 ruleset.
- `LTI_transition_model_v10.xlsx` — the source workbook.
- `LTI_transition_explorer.html` — the original single-view explorer this app builds on.
