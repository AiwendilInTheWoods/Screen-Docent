# Next session — finish the S-curve

**Read this first, then `PANEL_FINDINGS_2026-08-29-pm.md`, then `STATE.md`.**
Jupyter is being wired up; the numeric work below is what it is for.

## Where the curve stands

`tools/eink_scurve.py` fits a three-parameter tone curve (**pivot / toe / shoulder**) with independent
ends, scored by DIRECT SIMULATION of the production quantiser — no fitted model, so nothing depends on
the measured transfer function's unresolved +26/255 row term.

It works mechanically. **Its objective does not.** Three failures, in order of discovery:

1. **Boundary result.** The first optimum sat exactly on the grid maximum. A boundary result is a
   request to widen, never an answer (ADR-084; `eink_wpfit` already guards this). An automatic
   widening guard is now in and it fires.
2. **The objective was gameable, and got gamed.** Collapse metrics count pixels reaching the extreme
   inks, so a curve that crushes everything into mid-grey drives both to zero and "wins". The
   unconstrained optimum ran to toe = shoulder = 11.66.
3. **Adding fidelity terms did not fix it.** With tone-error and contrast weights the winner still maps
   input 5→62, 13→67, 26→70, 51→74, 77→76, 102→77, 128→77, 179→77 — **the entire range from 5 to 179
   into a 15-level band.** It saves ~43 on shadow collapse and ~84 on highlight collapse while losing
   only ~16 to the contrast penalty, so the trade still favours a grey rectangle.

📏 **The lesson is not "tune the weights again."** Hand-tuning an objective whose correct form is
unknown is exactly what ADR-092's post-mortem warned about — five successive invented metrics each
survived only until the next label arrived. Three rounds of this happened here in twenty minutes.

## What to do next, in this order

### 1. Make the degenerate curve INEXPRESSIBLE, not merely expensive
Constrain the family: require slope `dy/dx` within roughly **[0.4, 2.5] everywhere**, and reject any
(pivot, toe, shoulder) that violates it before scoring. A penalty is a number that can be out-traded;
a constraint cannot. This alone should stop the grey-rectangle solutions.

### 2. VALIDATE THE OBJECTIVE AGAINST THE 23 HUMAN JUDGEMENTS — this is the real gate
`bench-eink/wp3_labels.jsonl` holds 23 three-level white-point calls (12 from today, blinded, on oil
paintings; 11 from 2026-08-28). Each says which of wp 0.64 / 0.76 / 0.88 Josh preferred on a known
work. **Any objective worth using must rank those the way he did.**

Procedure: for each labelled work, score the three white-point renders with the candidate cost
function and check whether the argmin matches the human pick. Report accuracy against the 61% base
rate of always picking the modal answer. If it cannot reproduce the 23 calls, it has no business
choosing a curve for 2,857 paintings. This costs no panel time.

⚠️ Do this BEFORE fitting any curve. An objective that fails here invalidates everything fitted with it.

### 3. Only then fit the curve, and check it against the incumbents
Baselines already measured (6 works, fidelity-weighted cost): production `_adaptive_gamma` 190.80 ·
`wp0.75 g1.0` 181.54 · no correction 220.60. A curve that does not clearly beat 181.54 is not worth
shipping.

### 4. Validate on the panel — ADR-084 governs
Needs the PANEL, not the rig: no camera, no flat field, no calibration ritual. `tools/eink_show.py`
renders any library image at any recipe and writes a matching unquantised reference for the browser
harness. Use the new methodology Josh set out (see PANEL_FINDINGS §7): show the reference and the
derived ideal, take his reaction as an ERROR SIGNAL, and **register the proposed correction BEFORE
looking for support in the data**, or "finding support" becomes motivated search.

⚠️ **The one thing no measurement can settle:** the best curves lift the black point so there is no
true black anywhere in the frame. Whether that is acceptable on a wall is a judgement.

## What would force a re-shoot (nothing here is blocking yet)

| gap | why it needs glass |
|---|---|
| γ < 1 anywhere | never photographed; the lever that fixes black crush |
| the dark end, v < 40 | `--v-lo/--v-hi` were inert; captures re-shot the main grid |
| `inkmix` right-hand columns | ~5 of 15 ink pairs lost to right-edge drift; left columns read fine |
| `tonefine` steps 11,12,24,25 | right-edge drift; B1's remedy (drop them) gives 21/21 monotone |
| ADR-088's floor at `floor_max` > 0.55 | 0.5 is vacuous by construction: max(s^k, s·floor) can never pick the floor at s=0.55, k=2.0 |
| panel-vs-camera below digital 40 | ⛔ needs a REFLECTANCE STANDARD we do not own — re-shooting without a ColorChecker reproduces the same ambiguity |

**The decision rule: we need another shoot if and only if the S-curve cannot be designed and validated
from what we already have.** B1's finding is that the arithmetic needs no rig.

## State of the agent pipeline

| phase | status |
|---|---|
| A integrity | ✅ error bars in `error_bars.json`; use `A1_rederived.jsonl` (or `A2_rederived_fixed.jsonl`) |
| B1 tone-structure | ✅ `B1_tone_structure.md`, transfer function in `B1_transfer_function.json` |
| B2 colour-gamut | ✅ `B2_colour_gamut.md`; chroma survival **S(L) = 1.30/(1+exp((L−152)/15))**, R² 0.845 |
| B3 export | ✅ 13 files in `export/`, `SCHEMA.md` |
| C skeptic · D findings · E skills · F review | ⏸ briefs on disk, **not yet dispatched** |

**Re-dispatch was considered and rejected on evidence:** the five in-place fixes moved tonefine by 2.28
mean (bars: 26 single step / 11 for a mean) and huevalue grid-mean chroma by 0.31 (bar 3.3), with
**0 of 54 conditions crossing a bar**. B1 and B2 stand as written.

## Do not re-derive these; they are settled and recorded
Eleven instrument defects, all sharing one signature — **a check that could only pass**. See
`docs/eink-panel-characterisation.md` and the commit log. Two of them were mine, found while auditing
the others: a collision test built on an incomplete `_reference`, and a variance probe that was
under-determined on flat cells. Both nearly reached Josh as findings.
