# LTI Grant Sizing Logic — Portable Specification

This is a standalone description of the grant-sizing model, written to be
implemented anywhere — it does not assume any particular codebase, file
structure, or UI. It describes **what to compute**, not how this repo's app
happens to compute it. (For traceability back to a working reference
implementation, see `LTI_transition_engine_v2.md` in this same folder,
which documents the same logic as expressed in `index.html`.)

---

## 1. What this model does

For a given person, in a given year, under a given vesting schedule option,
compute two things:

1. **How much new grant to issue** (if any), and in what vesting shape.
2. **What actually vests that year** — the sum of legacy vesting still
   running off, plus whatever prior new grants deliver that year, plus any
   new grant's own Year 1 delivery.

There are three schedule options in the current model (§6), but the sizing
logic in §2–§5 is shared across all of them — only the vesting *shape* and
a handful of per-option flags differ.

---

## 2. Inputs, per person

| Input | Symbol | Notes |
|---|---|---|
| Market anchor | `M` | P75 of market, or band midpoint, for the person's current level. Changes on promotion (see §5). |
| Individual new-hire-grant value | `NHG` | The anchor the person was actually hired against. Fixed for their tenure — does not move with market unless re-set explicitly. |
| Transition year | `T` | The year this person moves onto the new program. |
| Promotion year | `P` (optional) | May be null (never promoted), before `T`, at `T`, or after `T`. |
| New hire flag | — | True if the person has no legacy vesting and is starting fresh on the new program. |
| Legacy vesting, notional | `N(year)` | Per year, what the person's pre-existing (legacy) grants are scheduled to vest, valued at **grant-time / notional** terms. |
| Legacy vesting, current | `C(year)` | Per year, the same legacy vesting, valued at **today's / current** terms (e.g. after a token price move). `C` and `N` are equal unless something has caused the value to diverge. |
| Performance rating, per year | `rating(year)` | One of 1–6 (see §4). |

**Derived per year:**

```
H(year) = max(NHG, M(year))
```

`H` ("higher of") is the anchor actually used in most of the sizing
formulas below. It lets a rich individual hiring anchor persist, but also
lets a big-enough market/promotion move override it.

---

## 3. The Year 1 value formula

This is the core of the model. For a person, in a year where a new grant
is being sized, compute three candidate values and take the highest:

```
A = ceiling(year, rating) − N(year)      // "notional" leg
B = B_mult(rating) × M(year) − C(year)   // "current" leg
Floor = Floor_mult(rating) × M(year)     // unconditional minimum

Year1Value = max(A, B, Floor)
```

- **A** asks: "if legacy vesting is valued at its notional/grant-time
  amount, how much more does this person need to reach their ceiling?"
- **B** asks the identical question, but with legacy valued at its
  *current*, price-adjusted amount. If the underlying asset dropped in
  value since the legacy grant was made, `C < N`, so `B > A` — this leg is
  what protects someone against downside price movement.
- **Floor** is a guaranteed minimum that does not depend on legacy at all.
  It is what pays when someone's legacy vesting already exceeds their
  ceiling under both A and B (i.e., both come out negative or very low).

**Exception — rating 1 issues no grant.** Regardless of what A, B, and
Floor compute to, a rating of 1 results in **zero new grant** for that
year. This should be enforced as a hard override, not just by setting
rating 1's multiples to zero in the grid (so that editing the grid can't
accidentally reintroduce a grant at rating 1).

`ceiling(year, rating)` — the value used in the A-leg — depends on
schedule, promotion status, and whether the person is a new hire; see §5
for the promotion case and §4 for the multiples table.

### Where this formula applies vs. a flat grant

Two configuration switches govern whether the A/B/floor formula runs at
all, or whether a person instead gets a flat, full-size grant with no
netting against legacy:

- **Per schedule, per employee type** (existing vs. new hire): each
  schedule option declares whether A/B/floor applies to existing employees,
  and separately whether it applies to new hires. (In the current model,
  one option's new hires are excluded and get a flat grant instead — see
  §6.)
- **Annual vs. transition-year-only** (global toggle): when "on," the
  formula reruns every year, so every refresh is a fresh higher-of
  calculation. When "off," it only runs in the transition year `T`; every
  other year the person gets a flat, full-size grant instead (§3.1).

### 3.1 The flat, full-size grant (when A/B/floor does not apply)

When A/B/floor is switched off for a given person/year (either because
their employee type is excluded for this schedule, or because the annual
toggle is off and this isn't the transition year), the new grant is simply:

```
FlatGrant = ceiling(year, rating)
```

with **no subtraction of legacy** — it is a full, un-netted grant. This
matters: confining the netting logic to specific years/populations (rather
than running it everywhere) is what preserves performance leverage. If
every grant were netted against legacy, a high performer's extra vesting
this year would just shrink next year's grant by the same amount, and the
performance signal would wash out. Keep the netted (A/B/floor) and
un-netted (flat) cases clearly separated in any reimplementation.

---

## 4. The rating grid

Every multiple used above comes from a rating grid — a lookup table keyed
by performance rating, with (at minimum) these columns:

| Column | Used for |
|---|---|
| A-leg multiple, existing employee | `ceiling()`'s multiple on `H`, for a transitioning/existing employee |
| A-leg multiple, new hire (schedule group 1) | `ceiling()`'s multiple on `H`, for a new hire under schedules that use the "1.00×"-style new-hire treatment |
| A-leg multiple, new hire (schedule group 2) | Same, for schedules that carry a premium on new-hire grants (e.g. a 1.15× anchor instead of 1.00×) |
| B-leg multiple | Multiple on `M` in the B-leg formula |
| Floor multiple | Multiple on `M` for the unconditional floor |
| Promotion-scaling multiple | See §5 — how the promotion-year/promoted-level ceiling scales with this rating |

**This grid should be fully editable/configurable, not hardcoded.**
Every multiple is a policy input, and the model should be re-runnable under
different values without a code change. A representative set of defaults
(not prescriptive — these are this program's current settings):

| Rating | A, existing | A, new hire (grp 1) | A, new hire (grp 2) | B | Floor | Promo |
|---|---|---|---|---|---|---|
| 1 | 0 | 0 | 0 | 0 | 0 | 0 |
| 2 | 0.90 | 0.80 | 0.80 | 0.80 | 0.20 | 0.90 |
| 3 (Meet) | 1.25 | 1.00 | 1.15 | 1.00 | 0.25 | 0.90 |
| 4 | 1.38 | 1.25 | 1.25 | 1.25 | 0.30 | 1.00 |
| 5 (Exceptional) | 1.50 | 1.50 | 1.50 | 1.50 | 0.40 | 1.10 |
| 6 | 2.00 | 2.00 | 2.00 | 2.00 | 0.50 | 1.20 |

Note that at rating 3 ("Meet"), the existing-employee multiple (1.25) is
intentionally higher than either new-hire column (1.00 / 1.15) — an
existing/transitioning employee's ceiling sits above a new hire's Year-1
level by design, reflecting that they're already partway through a career
arc rather than starting one.

---

## 5. Promotion

Promotion is modeled as a single event: a year, or none. From that year
forward, the market anchor doubles (or scales by whatever multiple is
configured — a "promotion multiple," default 2×) and **never reverts**:

```
M(year) = PromotionMultiple × M_original   for all year ≥ P
M(year) = M_original                       for year < P
```

Because `H = max(NHG, M)`, a large enough promotion move will overtake
even a rich individual hiring anchor and become the new binding value.

### 5.1 The promotion year itself is a special case

Depending on the schedule (see §6), the promotion-year grant is handled
one of two ways:

**Pattern A — gap to the new level, unscaled by the normal A-leg
multiple.** The promotion-year ceiling is:

```
ceiling(P, rating) = PromotionMultiple × M_original × PromoMult(rating)
```

Critically, this is **not** further multiplied by the rating grid's
"A-leg, existing" multiple (e.g. 1.25×) — that multiple only governs
*refreshes after* the promotion year. This distinction matters: if you
apply both the promotion multiple and the steady-state multiple in the
promotion year itself, you overshoot the intended step (e.g., landing at
2.5× the new base in the promotion year, when the intended shape is to
land at 2× in the promotion year and build to 2.5× only after several
years of refreshes).

**Pattern B — no distinct promotion year; a permanent flat level.** Some
schedules don't have a separate transition-year treatment for promotion at
all — the person simply moves onto a new, permanently flat ceiling from
the promotion year forward:

```
ceiling(year, rating) = SchedulePremium × PromotionMultiple × M_original × PromoMult(rating)
   for all year ≥ P
```

where `SchedulePremium` is a schedule-specific constant (e.g. a 1.15×
premium some schedules carry at their new-hire and promoted levels).

### 5.2 One lever for promotion-by-performance, across every schedule

The "Promo" column in the rating grid (`PromoMult(rating)`) should be the
**single control** for how a promotion's value responds to the person's
performance rating, and it should apply identically whether the schedule
uses Pattern A or Pattern B above. At the "Meet" rating, `PromoMult` should
be defined as exactly `1.00`, so that whatever the documented/validated
promotion-year numbers are at Meet performance remain unchanged — this
value is the calibration anchor for the whole column.

Do not let each schedule invent its own independent promotion-scaling
logic; route all of them through this one column so a single edit changes
promotion-by-performance consistently everywhere.

### 5.3 After the promotion year

Once past the promotion year (Pattern A schedules), the person returns to
the normal §3 A/B/floor logic, just with the now-larger `M` (and therefore
`H`) feeding every formula. There is no further special-casing.

---

## 6. Vesting schedule shapes

The Year 1 value from §3 is a single number — how much the new grant
delivers in its first year. Each schedule then defines a **vesting
vector**, i.e., what fraction of the total grant face vests in years
1/2/3/4. This program currently models three:

| Schedule | Vesting vector (Y1/Y2/Y3/Y4) | Applies A/B/floor to new hires? | New-hire A-leg column | Has a "room test" (§6.1)? |
|---|---|---|---|---|
| Option 1 | 100% / 15% / 5% / 5% | No — flat grant instead | Group 1 (1.00× at Meet) | No |
| Option 2 | 100% / 50% (grant); 50% / 50% (refresh) | Yes | Group 2 (1.15× at Meet) | Yes |
| Option 3 | 100% / 75% / 50% / 25% | Yes | Group 1 (1.00× at Meet), same as Option 1 | No |

A new grant's total face value is `Year1Value / vestingVector[0]` — i.e.,
sized so that the *first-year delivery* equals the Year 1 value computed
in §3, and the vesting vector determines everything after.

**Pre-transition years (`year < T`) default to a one-year grant** (100% in
year 1, 0% after) for every schedule — a grant with no retention tail.
Each schedule that wants the option to carry its full multi-year shape
into pre-transition years should have its **own, independently
configurable toggle** for this — do not share a single toggle across
schedules, since different schedules may want different defaults (in the
current program, one schedule defaults this to on, another to off, and one
schedule has no such toggle at all because its pre-transition grants are
always one-year by design).

Turning a pre-transition tail toggle on causes multiple years' tails to
stack on top of each other (since each year's grant is still computed
independently via §3), which will visibly push totals above the
steady-state ceiling in the years leading up to the transition. This is
expected, not a bug — it is the direct consequence of carrying a
multi-year shape into years that are otherwise sized as one-off top-ups.

### 6.1 The "room test" (schedules with a two-part transition grant)

A schedule whose steady-state shape is a **two-year-equal grant** (e.g.
100%/50%, repeating) needs special handling in the transition year,
because the transition year has to reconcile three things at once: the
Year 1 value from §3, the existing legacy vesting, and the fact that a
"half now, half next year" grant doesn't cleanly net against a single
year's gap.

```
level = ceiling(T, rating)          // same ceiling function as §3/§5
half  = RefreshFactor × level       // RefreshFactor is a policy input, e.g. 0.5
room  = level − legacyN(T) − half
```

- **If `room ≥ Floor(rating) × M(T)`:** issue two grants in the transition
  year — a one-year Grant 1 sized to `room` (or the Year 1 value, whichever
  is the actual binding constraint — implementers should size Grant 1 to
  guarantee the person actually receives at least `Year1Value` in year T),
  and a two-year-equal Grant 2 of `half` in T and `half` in T+1.
  - **Special case:** if legacy vesting in T is (near) zero, Grant 1 and
    Grant 2's first-year amount will be equal, and these should collapse
    into a **single** grant rather than being displayed as two overlapping
    ones.
- **If `room < Floor(rating) × M(T)`:** there isn't enough room to run both
  halves of Grant 2 in the transition year without going below the floor.
  Instead: Grant 2 is **deferred** to start in T+1 (full `level` in T+1,
  `half` in T+2), and Grant 1 in T is sized to cover the entire gap to
  `level` on its own (still subject to the floor).

---

## 7. Legacy vesting compounds forward

Once a new grant is issued, its future vesting years are **not** treated
as "new" legacy — they get added into the running `N` and `C` totals for
every subsequent year's §3 calculation:

```
N(year) = originalLegacyN(year) + Σ (vesting from all prior new grants, this year)
C(year) = originalLegacyC(year) + Σ (vesting from all prior new grants, this year)
```

A new grant's own future vesting contributes identically to both `N` and
`C` — only the *original*, pre-existing legacy vesting the user enters can
have `N ≠ C`. This is what "legacy carries forward" means in practice: by
the time you're several years past a transition, most of what's in `N`/`C`
is prior new-program grants, not the original legacy at all, and every one
of them still counts.

---

## 8. Worked examples (use these to validate a reimplementation)

Assume `M = H = 100,000` (no individual NHG premium) unless stated.
Rating 3 (Meet) uses the default grid from §4 unless stated.

1. **Steady state, no legacy, Meet rating, annual mode.** Every schedule
   converges to the same *total vested* value each year — but the size of
   the new grant issued to get there differs by schedule, because a
   schedule with a longer tail carries more prior-vesting residual into
   `N`, leaving a smaller gap for the new grant to fill. Approximate
   steady-state new-grant sizes at `M = H = 100,000`, Meet rating: Option 1
   (short tail) ≈ 100,000; Option 2 (two-year-equal) ≈ 62,500; Option 3
   (longest tail) ≈ 45,000. All three should nonetheless show the same
   *total annual vest* (legacy/prior-grant residual + new grant) once
   converged — treat any reimplementation that shows different totals
   across schedules at steady state as a bug, even though the individual
   grant sizes are expected to differ.

2. **Token price halves.** Legacy `N(T) = 100,000`, `C(T) = 50,000`,
   `M = H = 100,000`. A leg = `125,000 − 100,000 = 25,000`. B leg =
   `100,000 − 50,000 = 50,000`. Floor = `25,000`. B wins at 50,000 — and
   note that legacy current (50,000) + new grant (50,000) = 100,000 =
   exactly `M`. This is the intended behavior: the current-value leg
   protects the person up to the market band when price has dropped.

3. **Stacked legacy** (legacy already above the ceiling). `N(T) = C(T) =
   200,000`, `M = H = 100,000`. A leg = `125,000 − 200,000 = −75,000`. B
   leg = `100,000 − 200,000 = −100,000`. Floor = `25,000`. The floor wins
   — the person receives a new grant of 25,000 even though their legacy
   already vests well above the ceiling. This is intentional: the floor is
   unconditional. It is also the model's most expensive edge case, worth
   stress-testing against any population that stays "stacked" for multiple
   years running.

4. **Rating 1.** Regardless of A/B/floor inputs, the new grant is zero.

5. **Promotion, Meet rating, `M_original = 100,000`, `PromotionMultiple =
   2`.** For a Pattern-A schedule (§5.1): promotion-year ceiling =
   `2 × 100,000 × 1.00 (PromoMult at Meet) = 200,000`. The year after,
   normal refresh logic resumes with `M = 200,000`, so A leg =
   `1.25 × 200,000 = 250,000` — the schedule "builds" from 200,000 in the
   promotion year to a 250,000 steady state over subsequent years. For a
   Pattern-B schedule with a 1.15× premium: permanent flat level =
   `1.15 × 2 × 100,000 × 1.00 = 230,000` from the promotion year onward,
   with no further build.

6. **Promotion-by-rating.** Holding `M_original = 100,000`,
   `PromotionMultiple = 2` fixed and varying only the rating (using the
   default grid's Promo column: 0.90 / 0.90 / 1.00 / 1.10 / 1.20 for
   ratings 2–6), a Pattern-A schedule's promotion-year ceiling reads:
   rating 2 → 180,000; rating 3 → 180,000; rating 4 → 200,000; rating 5 →
   220,000; rating 6 → 240,000. Ratings 2 and 3 produce the *same*
   promotion ceiling (both 0.90) — a policy choice. A Pattern-B schedule
   with a 1.15× premium reads: 207,000 / 207,000 / 230,000 / 253,000 /
   276,000 over the same ratings.

---

## 9. Summary of configuration surface

Everything below should be an editable input, not a hardcoded constant, in
any reimplementation:

- Promotion multiple (default 2×)
- Any schedule-specific premium constant (e.g. a 1.15× new-hire/promoted
  premium on one schedule)
- Refresh factor for two-year-equal schedules (default 0.5×)
- The full rating grid: every column, every row, independently editable
- Per-schedule tail toggles for pre-transition years (independent per
  schedule that has one)
- The annual-vs-transition-year-only toggle for whether §3's formula reruns
  every year or only at the transition year

---

## 10. Open design questions to carry forward

1. **Cross-schedule promotion multiples aren't reconciled.** If different
   schedules use different base multiples at their promoted level (e.g.
   1.25× vs. 1.15×-of-2×), the same promotion event will pay differently
   depending on which schedule the person is on, even after routing both
   through the same Promo-column lever. Decide whether this is intentional
   policy (different schedules trade off differently) or should be
   unified.
2. **The floor's annual cost is likely the single biggest cost lever** in
   the whole model, because it is unconditional and (in "annual" mode)
   recomputed every year regardless of how far above the ceiling legacy
   vesting already sits. Any reimplementation should make it easy to
   report "how much floor-only spend is happening, and for how many
   consecutive years," since that number is easy to lose sight of in a
   per-person, per-year view.
3. **Total lifetime cost is sensitive to modeling horizon.** A schedule
   with a long vesting tail (e.g. four years instead of two) will show a
   *lower* Year 1 grant for the same steady-state annual cost, but its
   total face value within any fixed modeling window will look
   disproportionately high near the edges of that window, since some of
   its precommitment simply falls outside the modeled years. Any total-cost
   comparison across schedules should either extend the window well past
   the last transition/promotion event, or explicitly note this skew.
4. **No performance scaling is currently modeled for "an Exceptional
   rating in the exact year of a promotion"** beyond the shared Promo
   column — i.e., there's no separate, larger bump for the coincidence of
   a promotion and a top rating landing in the same year. Confirm whether
   that coincidence should be treated as a distinct case.
