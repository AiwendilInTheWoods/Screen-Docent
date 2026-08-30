# S4 — S-CIELAB: an objective that knows about viewing distance

**Run:** 2026-08-29. `tools/eink_scielab.py`, verified by `tests/test_eink_scielab.py` (9 tests).
No camera, no rig, no labels, no panel time.

## What it is

Decompose into opponent channels, filter each by that channel's contrast sensitivity at the real
viewing geometry, then take CIEDE2000. Standard (Zhang & Wandell 1996), not ours.

🔑 **The grain term stops being a term.** Every previous objective bolted "grain" on as a separate
hand-weighted quantity. It was never a separate quantity — it was the missing spatial model showing
through as a residual. Filter properly and grain is a *consequence*.

**Geometry, from the panel and not from the vendor.** Active area 270.4 × 202.8 mm at 1600 × 1200 →
pitch **0.1690 mm, 150.3 ppi**. ⚠️ The parts list says "200 ppi", which contradicts its own
active-area figure; a 33% pitch error is a 33% error in every angular quantity.

```
distance    px/deg    panel subtends       luminance sigma (px)
   0.5 m      51.6    31.0 x 23.2 deg      1.46 /  6.87 /  223.9
   1.0 m     103.3    15.5 x 11.6 deg      2.92 / 13.74 /  447.8
   1.5 m     154.9    10.3 x  7.7 deg      4.38 / 20.60 /  671.7
   2.0 m     206.5     7.7 x  5.8 deg      5.85 / 27.47 /  895.6
   3.0 m     309.8     5.2 x  3.9 deg      8.77 / 41.21 / 1343.4
```

Filtered in the Fourier domain with the analytic Gaussian transfer function — a correctness choice,
not a speed one: exact (no kernel sampling, which breaks at σ = 1.46 px), DC gain exactly 1 by
construction, and spatial convolution is infeasible anyway (the widest lobe needs a 4030 px half-width
at 3 m, wider than the image). The published weights sum to 0.918/0.861/0.859 and **must** be
renormalised or every flat region acquires a difference that is not there.

## It passes the check the withdrawn objective failed

`test_degenerate_renders_lose_to_real_ones` — flat mid-grey, pure black and pure white must all score
**worse** than an honest render, at every distance, on real art. ADR-097's objective preferred the grey
rectangle. This one does not. **It costs seconds and it is the check that would have caught two months
of work in twenty minutes.**

Also verified: reduces exactly to plain CIELAB on a flat patch (one test, four bugs — DC gain, weight
renormalisation, opponent round-trip, filter sign); the mean survives filtering; the wide
boundary-dominated luminance lobe does not change any *ranking* (`w3_zero` re-run); and stride-4
subsampling of the difference field is unbiased to <0.05 ΔE00.

## The ladder, on six works (worst case over 0.5–3.0 m)

| work | production | + derived white point | + gamut clip | + gamut map | + linear-light FS |
|---|---|---|---|---|---|
| The Night Watch | 16.76 | 14.22 | 14.22 | 14.66 | **10.80** |
| Olympia | 14.03 | 10.17 | 9.79 | 9.85 | **4.60** |
| Sunflowers | 16.68 | 15.52 | 14.62 | 14.53 | **10.60** |
| Flaming June | 19.21 | 17.16 | 17.58 | 17.95 | **11.69** |
| Café Terrace | 15.59 | 12.50 | 13.49 | 13.84 | **8.99** |
| The Kiss | 16.60 | 14.41 | 14.01 | 14.04 | **8.77** |
| **MEAN** | **16.48** | **14.00** | 13.95 | 14.15 | **9.24** |

**16.48 → 9.24, a 44% reduction.** For scale, 1 ΔE00 is roughly a just-noticeable difference.

## Two findings that need a decision, not just recording

### 1. The objective implements COLORIMETRIC intent, and Josh chose PERCEPTUAL

Gamut compression is *very slightly worse* than plain clipping under this objective (14.15 vs 13.95),
consistently. That is not a bug in either — it is the classic tension stated in numbers. A **pointwise**
ΔE00 to the source rewards accuracy; perceptual intent deliberately sacrifices accuracy to preserve
*relationships*, and no pointwise metric can see a relationship. **The objective cannot express the
intent that was chosen.**

Three ways out, and only the first two are honest: accept colorimetric intent because it is what the
objective can defend; or use the 23 human labels as the **tiebreak on this one question**, which is
legitimate *review* use and not gating. Inventing a relationship term is the third, and it is exactly
the move that produced five successive metrics in ADR-092. **The effect is small (0.2 ΔE00) — this is a
question worth deciding cheaply, not worth another programme.**

### 2. The biggest win by far is the thing the plan classified as "not a production change"

Decomposing the 7.24 ΔE00 improvement:

```
derived white point   2.48  (34%)   ships as a 3-D LUT — cheap, planned
gamut mapping         ~0    ( 0%)   see finding 1
linear-light dither   4.91  (68%)   S0.5 ruled this OUT of production
```

**S0.5 chose to keep Pillow's Floyd–Steinberg and pre-compensate with a 3-D LUT, on the reasoning that
`epaper.py` must stay PIL-only. On this measurement that vehicle captures about a third of the
available gain.** Whether a pre-transform can recover the rest is precisely what S5 optimises, and it
is now the most important question in the programme rather than a footnote. If it cannot, the PIL-only
constraint has a price and it is ~5 ΔE00 — which is a decision for Josh, not for the plan.

## STATUS: COMPLETE
