# LTI transition model v2 — higher-of-A/B/floor engine (Options 1, 2, 3)

**Status:** current logic, as implemented in `index.html`. Supersedes
[`LTI_transition_options_reference.md`](LTI_transition_options_reference.md),
which describes the earlier gap-sizing ruleset and is kept for historical
context only.
**Source of truth:** this document is a description of the code, not the
other way around — if the two disagree, `index.html`'s `simulate()` function
and the `SCHEDULES` / `GRID_DEFAULTS` constants are what actually runs.

---

## 1. What changed from v1

The v1 model sized each grant as `ceiling − scheduled_vest`, scoped to the
transition year and promotion years only. v2 replaces that with a single
rule that runs (by default) every year: **the new grant's Year 1 value is
the higher of three independent calculations.** Legacy vesting is no longer
a single number — it now has a **notional** value and a **current** value,
which can diverge (e.g. token price moved since grant), and the model takes
whichever framing is more generous to the person that year.

A third grant shape, **Option 3 (100/75/50/25)**, was added on top of this
engine — same Y1 sizing rule, different vesting vector.

---

## 2. Anchors

| Symbol | Meaning | Where it lives |
|---|---|---|
| `M` | Market anchor — P75 or band midpoint for the person's level. Doubles on promotion. | Input, per person |
| `NHG` | Individual new-hire-grant value — the anchor they were actually hired against. | Input, per person, fixed for their tenure |
| `H` | `max(NHG, M)` for the given year. | Derived |
| `N` | Legacy vesting, **notional** value, for a given year. | Input, per year |
| `C` | Legacy vesting, **current** value, for a given year (can differ from N if token price moved). | Input, per year |

`M` is promotion-aware: from the promotion year onward, `M_after = promoMult × M_before`
(`promoMult` is an editable assumption, default 2). `H` is recomputed every
year from the *current* `M`, so a large promotion can outrun an individual's
NHG premium and become the binding anchor.

---

## 3. Year 1 value: higher of A, B, floor

For a person/year/schedule, three numbers are computed:

```
A = ceilVal(year, rating) − N       // "notional" leg
B = ratingGrid[rating].b × M − C    // "current" leg
Floor = ratingGrid[rating].fl × M   // unconditional minimum
Y1 = max(A, B, Floor)
```

- **A** asks: if this person's legacy vesting had held its *notional* value,
  how much more do they need to reach their ceiling?
- **B** asks the same question in *current* (token-price-adjusted) terms.
- **Floor** is an unconditional minimum, independent of legacy — it pays even
  when legacy already exceeds the ceiling under both A and B.

Rating 1 is a hard exception: it issues **no grant at all**, regardless of
what A/B/floor would compute (see §5, rating 1 row is all zeros).

`ceilVal(year, rating)` — the ceiling used in the A-leg — is schedule- and
context-dependent; see §6.

**Where this logic applies.** Controlled per-schedule by `abcForNewHire` /
`abcForExisting` in `SCHEDULES`, and by the "Recompute A/B/C every year"
toggle (`annual`, default **on**):
- **On** for existing (transitioning) employees under all three options.
- **Off** for new hires under **Option 1** — they get a flat `1.00 × H`
  grant with no netting against N/C, since a new hire has no legacy to net
  against. New hires under **Options 2 and 3** *do* run A/B/floor.
- If `annual` is unchecked, the A/B/floor computation still runs but only
  fires in the transition year (`y === T`); every other year gets a
  full-size grant with no netting (see §7, "Full-size grant" branch).

---

## 4. Legacy vesting carries forward

Every new grant issued adds to *both* N and C at full value going forward —
a new grant's notional and current value start out identical; only
pre-existing legacy vesting (entered by the user) can diverge between the
two. This means:

```
N(year) = legacyN(year) + sum of prior grants vesting that year
C(year) = legacyC(year) + sum of prior grants vesting that year
```

All prior grants under a schedule — pre-transition top-ups, the transition
grant, refreshes — count toward N and C in later years' A/B calculations.
Nothing is netted out or ignored once issued.

---

## 5. The rating grid

Six ratings, fully editable in the UI (`RATINGS = [1..6]`, no hardcoded
floor on what a rating can be typed as). Each rating has six columns:

| Column | Meaning |
|---|---|
| `aEx` | A-leg multiple on H, **existing** (transitioning) employee |
| `aNH1` | A-leg multiple on H, **new hire**, Options 1 and 3 |
| `aNH2` | A-leg multiple on H, **new hire**, Option 2 |
| `b` | B-leg multiple on M |
| `fl` | Floor, as a multiple of M |
| `pr` | Promotion scaling — see §6 |

Current defaults:

| Rating | aEx | aNH1/3 | aNH2 | b | fl | pr |
|---|---|---|---|---|---|---|
| 1 | 0 | 0 | 0 | 0 | 0 | 0 |
| 2 | 0.80 | 0.75 | 0.75 | 0.75 | 0.15 | 1.00 |
| 3 (Meet) | 1.15 | 1.00 | 1.10 | 1.00 | 0.20 | 1.00 |
| 4 | 1.20 | 1.07 | 1.15 | 1.07 | 0.25 | 1.10 |
| 5 (Exceptional) | 1.25 | 1.15 | 1.20 | 1.15 | 0.30 | 1.15 |
| 6 | 1.30 | 1.30 | 1.30 | 1.30 | 0.40 | 1.30 |

Rating 1 issuing no grant is enforced in code (`NO_GRANT_RATING = 1`), not
just by zeroing the grid — even if someone edits row 1's multiples to be
nonzero, rating 1 will still produce zero new grant.

---

## 6. Promotion

Promotion is a single input: a year, or none. From that year on,
`M` doubles (`promoMult`, editable, default 2) and stays doubled — there is
no reversion.

**The promotion *year itself* is a special case**, controlled by
`promoYearFlat` (true for Options 1 and 3, false for Option 2):

- **Options 1 and 3:** the promotion-year grant gaps straight to the new
  level — `ceilVal = promoMult × M_before × ratingGrid[rating].pr` — with
  **no netting against legacy** beyond what A/B/floor already does, and
  critically, the **1.25× (or whatever `aEx` is) does not apply in the
  promotion year itself.** It only governs *refreshes after* the promotion
  year. At Meet (`pr = 1.00`), this reproduces the documented behavior:
  promotion year lands on `2 × M`, subsequent years build to `aEx × 2 × M`
  steady state (e.g. Meet: 200 in the promo year, 250 steady state).
- **Option 2:** there is no separate promotion-year mechanic. From the
  promotion year onward, Option 2 sits **flat** at
  `o2nh × promoMult × M × ratingGrid[rating].pr` forever (`o2nh` = 1.15 by
  default). At Meet this is 1.15 × 2 × M = 230 — matching Option 1/3's 250
  only insofar as both use the same `pr` lever; the underlying multiple
  (1.15 vs 1.25) is intentionally different and not reconciled (see v1 doc
  §7, item 1, for the historical discussion of this gap).

**The `pr` column is the single lever for how promotion responds to
performance**, across all three schedules — editing it changes both the
Option 1/3 promotion-year ceiling and Option 2's permanent promoted level
in the same direction, so there is one dial rather than three independent
ones.

After the promotion year, refreshes for Options 1 and 3 return to the
normal A/B/floor logic (§3), just with the now-doubled `M` feeding `H`.

---

## 7. Per-schedule behavior

### Option 1 — 100/15/5/5
- New hires: flat `1.00 × H`, no A/B/floor.
- Transitioning employees: A/B/floor every year (or transition-year-only if
  the annual toggle is off).
- Pre-transition years (`y < T`): one-year grants by default. A separate
  **"Tail on pre-transition grants, Option 1"** toggle (`tails`) makes them
  carry the full 100/15/5/5 shape instead, which causes tails to stack and
  pre-T totals to drift above the ceiling — this is intentional and
  documented behavior from v1, preserved here.
- Transition year and refreshes: full 100/15/5/5 shape.

### Option 2 — 100/50 and 50/50
- A/B/floor applies to **everyone**, new hires included.
- Pre-transition years: always one-year grants — no tail toggle exists for
  Option 2 (`tailToggle: null`).
- **Room test in the transition year** (unchanged mechanic from v1, now
  fed by the new `ceilVal`): computes `lvl = ceilVal(T, rating)`,
  `half = o2ref × lvl`, `room = lvl − legacyN(T) − half`.
  - If `room ≥ floor`: two grants — a one-year Grant 1 sized to fill the
    gap, and a two-year-equal Grant 2. If legacy is ~zero, these collapse
    into a single `100/50` grant.
  - If `room < floor`: Grant 2 is pushed to `T+1` as a full `100/50`;
    Grant 1 in T is sized to cover the whole gap on its own.
- Refreshes: `50/50` steady state, sized by A/B/floor.

### Option 3 — 100/75/50/25
- New hires: A/B/floor at the Option-1-style `aNH1` column (1.00× at Meet) —
  same treatment as Option 1's new-hire column, distinct from Option 2's
  1.15×-anchored `aNH2`.
- Transitioning employees: A/B/floor every year, same as Options 1 and 2.
- No room test — the transition-year grant is a single grant sized to
  `Y1`, shaped `100/75/50/25`.
- Pre-transition years: one-year grants by **default**. A separate
  **"Tail on pre-transition grants, Option 3"** toggle (`tails3`), off by
  default, makes them carry the full `100/75/50/25` shape instead — same
  mechanic as Option 1's toggle, independent switch, independent default.
  (This default was deliberately flipped after an earlier build mistakenly
  had Option 3 always carrying its tail pre-transition; see commit
  history — "the tail for Option 3 should only kick off from the
  transition year.")
- Promotion: same flat-to-`2M` promotion-year treatment as Option 1
  (`promoYearFlat: true`, `premiumOnPromo: false`).

At steady state (no legacy, Meet rating), all three schedules converge to
the **same annual face value**, `aEx × H` (1.25 × H by default) — they
differ only in how that value is split across the vesting vector, which
changes how much stays unvested at any point in time:

| Schedule | Y1 grant size at steady state | Unvested overhang |
|---|---|---|
| Option 1 (100/15/5/5) | 1.00 × H | 0.40 × H |
| Option 2 (100/50, half of 1.25H) | 0.625 × H | 0.625 × H |
| Option 3 (100/75/50/25) | 0.50 × H | 1.25 × H |

Option 3 is the longest precommitment of the three despite issuing the
smallest Y1 grant — a longer tail necessarily means a smaller Year 1
delivery for the same annual face cost.

---

## 8. Worked checks (verified against the app)

These were confirmed by direct simulation during the build and are safe to
treat as regression checks:

- **Steady state, no legacy, Meet rating:** all three schedules converge to
  `1.25 × H` (125k at H=100k), with per-schedule Y1 grants of 100k / 62.5k /
  ~51.4k for Options 1/2/3 respectively.
- **Token price halves** (N=100k, C=50k, M=H=100k): A=25k, B=50k → B wins;
  realized value lands exactly on M.
- **Stacked legacy** (N=C=200k, M=H=100k): A=−75k, B=−100k → floor pays
  25k, on top of the already-over-ceiling legacy.
- **Rating 1:** zero new grant, regardless of A/B/floor inputs.
- **Promotion at Meet, M0=100k, promoMult=2:** promotion-year ceiling reads
  200k (Options 1/3) and 230k (Option 2); steady state after settles at
  250k (Options 1/3) and 230k flat (Option 2).
- **Promotion ceiling scaling by rating** (M0=100k): reading the `pr`
  column (1.00/1.00/1.10/1.15/1.30 for ratings 2–6), Options 1/3 read
  200k/200k/220k/230k/260k across those ratings; Option 2 reads
  230k/230k/253k/264.5k/299.5k over the same range.
- **Option 3 tail toggle:** off holds pre-transition years flat at the
  ceiling with no drift; on reproduces the same stacking drift documented
  for Option 1 in the v1 ruleset (e.g. 125/144/156 on a 25%-left persona).

---

## 9. Scenarios: presets, saved, shared

Three tiers of starting points, all consuming the same config shape
(`M`, `nhg`, `T`, `promo`, `newhire`, `tails`, `tails3`, `annual`, the
`a` block of promotion/Option-2 assumptions, `legN`/`legC` per year,
`rating` per year, and the full `grid`):

- **Presets** (`PRESETS` constant) — the eight legacy personas (A–H) plus
  Blank, expressed as % of target for quick exploration. Personas A–G
  carry an individual NHG equal to M (100000); persona H carries a richer
  NHG (125000) to exercise the "higher of NHG or M" anchor.
- **Shared scenarios** (`SHARED_SCENARIOS` constant) — baked into
  `index.html`, visible to everyone who opens the deployed link. Added by
  using the in-app "Copy current as JSON" button and pasting the result
  into this constant, then committing. Anonymized by policy — no real
  names paired with individual comp figures go into this file (see
  commit history for the reasoning); current entries use role/hire-year
  labels or a bare initial.
- **Saved scenarios** — personal, stored in the browser's `localStorage`
  under key `lti_scenarios_v2`, never leave the user's machine.

---

## 10. Known open items

Carried over from v1 or introduced by this rebuild, not yet resolved:

1. **Option 1 vs Option 2 promotion multiple is still not reconciled** —
   1.25× vs 1.15× on the same `2M` base is a deliberate, documented choice,
   not an oversight, but it means the same promotion event pays differently
   depending on schedule even with the `pr` lever held equal.
2. **The floor is unconditional and now runs every year by default** — a
   person whose legacy already exceeds every ceiling still collects
   `fl × M` on top of it, every single year, under the `annual` default.
   This is the single largest cost lever in the model and is worth
   stress-testing against a population that stays stacked for several
   years (the v1 doc's "Persona H" concern, now sharper since annual
   recomputation is the default rather than transition-year-only).
3. **Option 3's total lifetime face cost tends to run highest** of the
   three schedules in multi-year exploration, because its longer tail
   means grants extend further past the model's 2035 horizon — some of
   its precommitment is simply not visible within the modeled window.
4. **No performance scaling in the promotion year for Options 1/3** beyond
   the `pr` column — an Exceptional rating in the exact year someone is
   promoted is governed by the same lever as an Exceptional rating in any
   other promoted year, not a separate, larger bump.
