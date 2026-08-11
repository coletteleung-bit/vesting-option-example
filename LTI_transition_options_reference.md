# LTI transition model — Option 1 vs Option 2

**Status:** rules as validated through the Aug 2026 build sessions, engine v8/v9 (workbook), v3 (JS explorer). Both implementations reconcile to the cent across 522 test cells.
**Index convention:** all figures are % of the person's annual LTI target unless noted (target = 100).

---

## 1. The two designs

| | Option 1 | Option 2 |
|---|---|---|
| Grant / refresh shape | 100/15/5/5 over 4 years | New hire 100/50 over 2 years; refreshes 50/50 over 2 years |
| Steady-state ceiling | 125% of target | 125% of target (existing employees); 115% (new hire, until promoted); 230 flat (promoted) |
| Face value per $1 of Y1 delivery | 1.25x | 2.0x (refresh), 1.5x (100/50 grant) |
| Retention tail before earning again | Yes — 15/5/5 decaying | No, until the next grant — flat halves |

Both are "annual-equivalent" designs: the point of either is that Year 1 of a new grant delivers the full annual target, so cash-flow competitiveness in Year 1 no longer requires a token-price bet. They diverge on how much of the award is precommitted as an unvested tail versus reset every one to two years.

---

## 2. The ceiling framework

Every "gap-sized" grant (see §3) is sized as:

```
grant = ceiling(y) − scheduled_vest(y)
```

`ceiling(y)` depends on the person's status that year:

| Status | Ceiling | Notes |
|---|---|---|
| Base | 1.25 × target | Steady state for both options once through transition |
| High performance (outsized) | 1.50 × target | Input on Assumptions (`perfM`). Was 1.40 in earlier drafts, revised to 1.50. |
| Promoted, Option 1 | 2.00 × current-level target | Input (`promo1`). Permanent — every later grant make-wholes to it. |
| Promoted, Option 2 | 230 (= 2 × 1.15 × target) | Fixed number, not a multiple of the person's own target — see §4. |

**Critical rule — gap-sizing is scoped, not universal.** Gap-sizing (grant = ceiling − scheduled) applies **only** in:
- the transition year (T), and
- a promotion year (and, for Option 1, any grant year at or before T if the person is promoted before T).

**Everywhere else, every grant is full size** — fixed, or scaled only by the performance multiplier, never netted against what's already vesting. This is the single most important correction made mid-build: an earlier draft gap-sized every year, which reinstated the legacy make-whole mechanic and eliminated performance leverage. Confining it to T and promo years is what lets the vesting shape and the performance multiplier do real work everywhere else.

**Floor.** Any gap grant issued in the transition year or a pre-transition year is floored at `0.25 × target`, with the performance multiplier applying on top (so 25 normal, 37.5 in a high-performance year). Input: `floorF`. This only binds when legacy vesting is already at or above the ceiling (see Persona H, §7).

---

## 3. Option 1 — 100/15/5/5

### 3.1 Normal year (not T, not a promo year)
One grant, full size:
- New hire (first cycle): `target × 1.0` → delivers 100/15/5/5.
- Refresh, average: `target × 1.0` → 100/15/5/5.
- Refresh, high performance: `target × perfM` (150) → 150/22.5/7.5/7.5. The tail scales proportionally with the Y1 delivery.

### 3.2 Pre-transition years (y < T)
One grant, gap-sized to the base ceiling, **floored at 25** (37.5 if high-perf):
```
grant = MAX(floor, 125 − legacy_scheduled)
```
Shape is controlled by the **tail toggle** (Assumptions input, on/off):
- **Toggle ON** (default): shaped 100/15/5/5. Because the grant is gap-sized each year and gap-sized grants are *not netted against each other's tails*, the tails stack and pre-transition years drift **above** 125 — e.g. Persona C reads 125, 128.75, 130, 131.25 through years 1–4 before the transition year corrects it back down.
- **Toggle OFF**: one-year grant only (no tail). Pre-transition years stay exactly at the floor/gap amount, no drift, but no unvested retention hook either.

This toggle also governs the **promotion grant** whenever the promotion falls at or before the transition year (§3.4).

### 3.3 The transition year (y = T), no promotion
**Two grants**, both floored:
- **Grant 1** — one-year, gap-sized: `MAX(floor, ceiling − legacy_scheduled)`. Note: the gap is measured against **legacy only**, not against legacy + any pre-transition grant tails still running. This was a specific correction ("the transition year grant that is a gap should only count what's remaining from the NHG").
- **Grant 2** — the standardised tail: fixed at `target × [0, 0.25, 0.10, 0.05]` (face 0.40× target). **Never performance-scaled**, even in an outsized year — this holds regardless of the person's rating that year, by explicit instruction.

Together these two grants are what let someone entering with zero unvested tail catch up to the full 125/15/5/5-equivalent build over the following three years without a permanent overpayment.

### 3.4 Promotion (Option 1)
- **In the promotion year itself:** `MAX(floor if y≤T else 0, 200 − legacy_or_scheduled)`, one grant.
  - If the promotion year is **at or before T**: shape follows the **tail toggle** (100/15/5/5 if on, one-year if off). No separate transition tail grant is issued — the promotion grant replaces the whole transition package.
  - If the promotion year is **after T**: shape is always 100/15/5/5 (the toggle only governs pre-T behaviour).
- **Every year after promotion:** fixed `200/30/10/10` (i.e., ceiling 200 × the 100/15/5/5-equivalent shape scaled 2x), building to a **250 steady state** over 4 years.
- **Promoted before T, transition year still arrives:** treated as a promotion-year grant at T (gap to 200, toggle-governed shape) — not a second transition package. This is the "neither" resolution: no double transition tail, no separate promo tail; one instrument, sized to the higher target.

### 3.5 Consequence: two different "generosity" knobs
With the tail toggle **on**, a promotion reaches its 250 steady state about a year faster, because the promotion grant itself throws its own tail. Toggle **off** produces a visible flat spot: the year immediately after a promotion vests the same as the promotion year itself (e.g., 200, 200), because a one-year grant leaves nothing vesting behind it. This flat spot is the practical argument for defaulting the toggle **on**.

---

## 4. Option 2 — 100/50 and 50/50

### 4.1 The three target levels
Option 2 is **fully target-driven**. A person's "target" for refresh-sizing purposes is one of three fixed states:

| State | Target used | Refresh half (`0.5 × target`) | Applies to |
|---|---|---|---|
| New hire, unpromoted | 115 (`1.15 × 100`) | 57.5 | Persona A only, until promoted |
| Existing / grandfathered | 125 (base ceiling) | 62.5 | Everyone hired under the legacy program |
| Promoted | 230 | 115 | Anyone, once promoted, forever |

The **refresh factor (0.5x) is an input**, not hardcoded, so the halves above (57.5/62.5/115) all derive from one assumption cell.

**Resolved inconsistency:** an earlier draft kept the ongoing refresh at 62.5 regardless of the person's own target, which meant a 1.15x new hire never actually reached a 143.75 steady state — they just converged to the same 125 as everyone else, making the 1.15x premium pointless beyond Year 1. The fix ties the refresh half directly to whichever target level the person is currently on, so a new hire genuinely settles lower (115) until promoted, and a promoted person genuinely settles higher (230) — permanently.

### 4.2 Normal year
- New hire, Year 1: `1.15 × target` → 115, delivered as 100/50 (115 then 57.5 the following year).
- Refresh, average: `0.5 × current-level-target` (57.5 / 62.5 / 115 depending on state above), delivered 50/50 (i.e. that amount now, same amount again next year).
- Refresh, high performance: gap-sized up to `1.50 × target` (150 ceiling), not fixed — this is the one place Option 2 still gap-sizes outside T/promo, because there's no separate "outsized" grant type to define its own fixed size.

### 4.3 Pre-transition years (y < T)
One-year grant, gap-sized to the base ceiling, floored at 25 (or the perf-scaled floor):
```
grant = MAX(floor, 125 − legacy)
```
No tail toggle applies to Option 2 — pre-transition grants are always one-year here.

### 4.4 The transition year (y = T) — "room test"
This is the most structurally distinct piece of Option 2. Grant 2 is meant to be a **two-year equal grant** at half the applicable level (57.5/62.5/115), starting in T. Whether that's actually possible depends on how much room legacy leaves:

```
level  = 230 if already promoted at/before T, else 125
half   = 0.5 × level
room   = level − legacy(T) − half
```

- **If `room ≥ floor` (25):** two grants issued in T.
  - Grant 1: one-year, `= room` (i.e., fills exactly what's left after legacy and Grant 2's first half).
  - Grant 2: two-year equal, `half` in T and `half` in T+1.
  - **Special case — zero legacy:** if legacy(T) = 0, Grant 1's amount equals Grant 2's first-year amount exactly, so the two collapse into **one single grant** of `level` (100/50 shape) rather than two overlapping ones. (Persona B, whose legacy is fully run off by T, is the example: single grant of 230 then 115, not two separate grants that happen to sum to the same thing.)
- **If `room < floor`** (legacy already leaves no room for a full second grant): Grant 2 is **pushed to T+1** and becomes the 100/50 grant in full (`level` in T+1, `0.5×level` in T+2). Grant 1 in T is sized to fill the whole gap to `level` on its own (floored). No refresh is separately issued in T+1 — Grant 2's own first-year delivery covers it. (Personas E, G, H and any case where legacy is already near or above the ceiling hit this branch.)

**Promotion interaction in T.** If the person is promoted at or before T, `level` becomes 230 and `half` becomes 115 in the formulas above — it is literally the same mechanism, just at the higher target. This is why a promotion landing in the transition year doesn't need a separate rule: it's the room test evaluated at level = 230 instead of 125.

### 4.5 Promotion (Option 2), outside the transition year
- **In the promotion year, before or after T:** one-year grant, gap to 230 (`MAX(floor, 230 − legacy)` if before T, `MAX(0, 230 − scheduled)` if after — no floor needed once fully on the new system).
- **The year after promotion:** fixed 100/50 grant at the new level: `230` then `115` the year after that.
- **Every year after that:** fixed 50/50 refresh at `115` per half, forever (steady state 230).

### 4.6 Worked examples that anchor the rules above
These four were given as ground truth and every rule above was derived to reproduce them exactly:
- **Persona C, promoted before T:** Grant 1 = 90 (fills the gap to 230 net of legacy and the coming half), Grant 2 = 115 + 115 (two-year equal). Total 230 flat from the promotion year on.
- **Persona C, promoted in T:** identical structure and numbers to the above — confirms the room test is level-invariant to *when* the promotion happened, only *whether* it happened by T.
- **Persona H, promoted in T:** one-year grant of 105 in T (legacy 125 leaves no room for a second grant), then 230/115 the following two years (Grant 2 pushed to T+1).
- **Persona B, promoted in T:** collapses to a single grant, 230 then 115 (zero legacy case, §4.4).

---

## 5. Formatting and presentation conventions (workbook)

- **Grants in rows, years in columns.** (Reverted from a grants-in-columns layout that was found confusing.)
- **Option 1 on the left, Option 2 on the right** of the same block, sharing identical year headers so totals line up row for row.
- **Transition-year column:** shaded light yellow, across both options, in every block.
- **Promotion-year column:** bold, bright blue text, across both options.
- **% of steady-state ceiling row** under every total-vest row, so overshoot years (drift above 125/230, or the floor pushing above legacy) are visible rather than buried in the raw total.
- **Summary tab** opens with a **cost comparison** block (total grant face issued, Option 1 vs Option 2, delta, and % delta) before the year-by-year vest comparison, per scenario, so "which option costs more" has a direct answer rather than requiring the reader to sum face values themselves.
- All multipliers, the refresh factor, the floor, and the tail toggle are **input cells on an Assumptions tab** — nothing is hardcoded into formulas, so the model can be re-run under different policy settings without a rebuild.

---

## 6. The eight personas (current roster)

| ID | Description | Transition year (T) | Legacy vest by year |
|---|---|---|---|
| A | New hire 2027, no legacy grant | 2027 | — |
| B | Hired 2026, 0% of target left at transition | 2030 | 100/100/100/0 (2027–2030) |
| C | Hired 2026, 25% left at transition | 2030 | 100/100/100/25 |
| D | Hired 2026, 50% left at transition | 2030 | 100/100/100/50 |
| E | Hired 2026, 75% left at transition | 2030 | 100/100/100/75 |
| F | Hired 2023, 50% left at transition | 2027 | 50 (2027 only) |
| G | Hired 2023, 75% left at transition | 2027 | 75 (2027 only) |
| H | Hired 2024, 125% left at transition (stacked legacy) | 2028 | 150 (2027), 125 (2028) |

Four scenarios are run against each: **Base** (no promotion, average performance), **Promo in T**, **Promo before T** (not applicable for A, since T is A's first year), and **Outsized 1.5x in 2030**.

Persona H is the only one that currently tests the 0.25x floor — its legacy is at or above the ceiling in both its active years, so the floor (not the gap) is what pays, and it overshoots the ceiling in every scenario by construction. It's flagged as a persona that should probably be joined by a second stacked-legacy case, since H only tests the floor for two years.

---

## 7. Open questions not yet resolved

1. **The 250 vs 230 permanent promotion gap.** The same promotion event pays a 250 steady state under Option 1 and 230 under Option 2 — a permanent 20-point difference for the same career event, driven by Option 1's promotion ceiling being defined as `2.0 × target` with a 125%-style build, while Option 2's is a flat `2 × 1.15 × target`. Not yet reconciled.
2. **The tail toggle decides which option costs more.** With the toggle on, Option 1 is the more expensive design in most scenarios; with it off, Option 2 is. Cost comparisons should probably be shown for both toggle states rather than just one.
3. **Face value vs LINK units.** All cost comparisons to date are in face value (index points), which is a reasonable proxy for near-term dollar cost but does not capture the actual point of Option 2 (shorter precommitment horizon under LINK price volatility). A token-unit comparison under Bull/Bear/Flat scenarios has been requested but not yet built for this ruleset.
4. **Stacked-legacy population.** Only Persona H tests legacy-above-ceiling, and only for two years. The floor's real cost hasn't been stress-tested against a population that stays stacked for longer.

---

## 8. Companion artifacts

- `LTI_transition_model_v9.xlsx` — full workbook, 8 personas × 4 scenarios × 2 options, live formulas, Assumptions-driven.
- `LTI_transition_explorer.html` — single-person interactive calculator (dollar-denominated), same engine ported to JavaScript, validated to the cent against the workbook's Python engine across all 522 test cells.
- `LTI_Vesting_Model_PRD_v11.md` — the pre-existing, broader PRD (bridging grants, plan registry, token-issuance findings) that this transition-specific ruleset sits alongside; the two are not yet merged into one document.
