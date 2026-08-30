# S5 — the shipping pre-transform, and the price of the PIL-only constraint

**Run:** 2026-08-29. `tools/eink_optimise.py`, verified by `tests/test_eink_optimise.py` (7 tests).
No camera, no rig, no labels, no panel time.

## The ladder, six works, worst case over 0.5–3.0 m (mean ΔE00)

| work | **floor** | production | LUT: norm+clip | **LUT: +precomp** | linear-light FS |
|---|---|---|---|---|---|
| The Night Watch | 1.35 | 16.76 | 14.23 | 14.40 | 10.72 |
| Olympia | 0.00 | 14.03 | 9.94 | 9.58 | 4.65 |
| Sunflowers | 4.22 | 16.68 | 14.62 | 12.91 | 10.73 |
| Flaming June | 7.11 | 19.21 | 17.65 | 15.68 | 11.50 |
| Café Terrace | 5.81 | 15.59 | 13.57 | 11.81 | 8.77 |
| The Kiss | 0.64 | 16.60 | 14.05 | 11.90 | 8.86 |
| **MEAN** | **3.19** | **16.48** | **14.01** | **12.71** | **9.20** |

**What can ship today: 16.48 → 12.71, a 23% reduction, entirely inside the PIL-only constraint.**

## The registered prediction was refuted

> *Predicted before running: pre-compensation derived from flat-patch response recovers **40–70%** of
> the 4.91 ΔE00 gap between Pillow's FS and a linear-light one.*

**Measured: 27%.** It is recorded as refuted rather than restated, and the next section is why —
which is a better answer than the prediction would have been.

## ⛔ The finding: the quantiser's response is NOT ONTO

Pre-compensation can only work if for every colour we want, *some* input makes the quantiser land
there. Measured over the target grid, after inverting the measured response:

```
57.1% of desired colours are unreachable by more than 0.5 dE00
36.5%                                              1.0
27.3%                                              2.0
14.6%                                              5.0
worst 25.8 dE00 · mean 2.22
```

The fixed-point inversion **converges by 6 iterations — to a residual of 0.085 linear, not to zero.**
That is the same fact from the other side.

🔑 **So the ceiling belongs to the quantiser, not to the LUT or the search.** Three independent checks
confirm nothing else is limiting: LUT resolution is irrelevant (17³ 11.905, 33³ 11.900, 65³ 11.915
ΔE00); Pillow's own `Color3DLUT` matches exact trilinear application to 1 level / 0.016 ΔE00; and the
iteration is converged.

**You cannot pre-compensate your way out of diffusing error in the wrong space.**

## ⚠️ The decision this hands back — ADR-102's Decision 2, now with a number

```
floor (gamut only, irreducible)                     3.19
best that can ship under PIL-only                  12.71
linear-light dither                                 9.20
                                          --------------
THE PRICE OF THE PIL-ONLY CONSTRAINT               3.51 dE00
```

For scale, 1 ΔE00 is about a just-noticeable difference, and 3.5 is roughly the difference between
mid-grey and grey-119 — obvious side by side.

S0.5 chose to keep Pillow's Floyd–Steinberg because `epaper.py` runs on the Pi and must stay PIL-only.
That reasoning is still sound; what has changed is that **the cost is now measured rather than assumed
to be small.** The options, in ascending order of disruption:

1. **Ship the 12.71 LUT and stop.** A 23% improvement on production, zero new runtime dependencies,
   zero risk. The remaining 3.51 ΔE00 stays on the table.
2. **Port the linear-light dither to pure PIL.** The wavefront implementation is numpy; a PIL/C
   equivalent is real work and would run far slower than 0.020 s on a Pi — but the panel refresh is
   ~9 s, so a 1–2 s dither is not obviously disqualifying. **Not investigated; this is a guess and is
   flagged as one.**
3. **Allow numpy in the render path on the Pi.** Smallest code change, largest policy change.

⚠️ **This is Josh's call, not the plan's.** All three are defensible; they differ in what they cost
elsewhere, and that is not a colour-science question.

## What ships, concretely

A **33³ `Color3DLUT`**, computed offline, applied in **0.034 s** in pure PIL, feeding Pillow's existing
Floyd–Steinberg. Three derived stages, none fitted:

1. **media-relative normalisation** — the derived white point (S1), worth 2.48 ΔE00;
2. **gamut clip** — colorimetric intent (ADR-103), worth ~0, and that is the finding;
3. **quantiser pre-compensation** — the numerical inverse of a *measured* response, worth 1.30 ΔE00.

⚠️ Stage 3 is not uniformly helpful: on **The Night Watch it is slightly worse** (14.23 → 14.40). The
correction is derived from flat patches, and that work is mostly structure in exactly the tonal region
where the correction is largest. A per-work decision on stage 3 would be a per-work fit, which is what
ADR-093 withdrew — so it stays global, and the regression is recorded rather than tuned away.

## STATUS: COMPLETE
