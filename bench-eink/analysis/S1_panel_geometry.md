# S1 — the panel's geometry, computed in a real colour space

**Run:** 2026-08-29, `python tools/eink_panel_model.py`. No camera, no rig, no labels, no panel time.
**Verified by:** `tests/test_eink_panel_model.py` (10 tests) on top of `tests/test_eink_color.py`
(42 tests, incl. the Sharma 34-pair CIEDE2000 reference set).
**Data:** `S1_panel_geometry.json`.

> ⛔ **THIS IS THE REVIEW GATE.** It re-adjudicates ADR-090/091/093/094. If any of it is wrong,
> everything built on top of it is wasted, so it is published before S2 is written.

## 0. What this rests on, and what it does not

Every absolute number inherits `SPECTRA6_DITHER_PALETTE` — **Pimoroni's measurement of a different
physical EL133UF1.** The module reads that constant and never rewrites it; the standing prohibition
holds. `_MEASURED_INK_XYZ` is the single hook where a colorimeter reading would land, so that
measurement is a one-constant change and every number here re-derives.

**Robust to the palette being wrong:** that the gamut is the hull; that the white point is *derived*
rather than free; the *sign* and *shape* of the dither error; every structural test.
**Not robust:** the ink table, hull volume, the cusps, the 0.641, the +13.1 L\*.

## 1. The ink table — and the ordering that changes

| ink | flat mean (`INK_LUM`) | linear Y | L\* abs | **L\* media** | C\* | h° |
|---|---|---|---|---|---|---|
| black | 0.0 | 0.0000 | 0.0 | 0.0 | 0.0 | — |
| blue | 71.3 | 0.0493 | 26.5 | 43.3 | 31.5 | 298.1 |
| green | 73.0 | 0.0882 | 35.6 | 56.0 | 26.3 | 152.0 |
| red | 101.0 | 0.1221 | 41.5 | 64.3 | 54.7 | 23.5 |
| white | 163.3 | 0.3684 | 67.2 | **100.0** | 0.0 | — |
| **yellow** | 156.3 | **0.5069** | 76.5 | **113.0** | 85.2 | 96.0 |

```
flat RGB mean (the code) :  black < blue < green < red < yellow < WHITE
linear Y                 :  black < blue < green < red < white < YELLOW
L* media-relative        :  black < blue < green < red < white < YELLOW
```

🔑 **Yellow is the panel's most luminous ink, 38% above white.** The flat mean inverts it because
yellow's blue channel (71) drags the average down, while blue carries only 7.2% of luminance and green
71.5%. Physically unsurprising: this panel's "white" is a mediocre grey at 64% reflectance, and its
yellow pigment reflects red and green strongly.

## 2. ADR-094 re-adjudicated — one end survives, the other does not

ADR-094 says the panel is *"starved at BOTH ends… exactly ONE ink above luminance 101 (white, 163) and
exactly ONE below 71."* Recomputed:

| end | ADR-094 | in L\* media-relative | verdict |
|---|---|---|---|
| dark | one ink below blue | black→blue is **38.3% of the whole range**, the largest gap | ✅ **SURVIVES, and it is the real one** |
| light | one ink above red | **two** — white (100) and yellow (113), yellow on top | ❌ **does not survive as stated** |

**The correct restatement.** The panel has one genuine starvation — the shadows — plus a large
mid-to-high gap (red 64.3 → white 100, 31.6% of the range). At the top there is no wall: there is a
**trade**, because the only thing above media white is yellow, so exceeding white's luminance costs a
warm cast. A scalar "ceiling" is exactly the arithmetic that hides a trade, and hiding it is why the
highlight story has been treated as a hard limit for two months.

📏 The two-ended framing was not wrong about the *shadows*. It was wrong that the two ends are the
same kind of problem. They are not: one is an absence, the other is a price.

## 3. The gamut is a bipyramid, and it is 1.11% of sRGB

At a viewing distance where the dither fuses, error diffusion realises **area-weighted averages**;
averages of *radiance* are convex combinations; so the reproducible set is the **convex hull of the six
inks in linear light** — a solid, not six points.

```
6 vertices · 12 edges · 8 triangular faces · Euler V−E+F = 2 ✓
volume = 1.11% of the linear-RGB unit cube
faces: every one is an apex (black or white) plus two of {red, yellow, blue, green}
```

Black and white are the apexes; red/yellow/blue/green form the equator. Eight faces makes the
gamut-boundary primitive a handful of dot products — **closed-form ray/hull intersection, no sampled
boundary descriptor, and therefore no interpolation error** in the gamut mapping to come.

⚠️ **Correction to the design, found while building: the CUSP is not closed-form.** The hull is a
polytope in linear RGB, but Lab is a *nonlinear* function of XYZ, so hull edges are **curves** in Lab
and a constant-hue leaf does not cut them algebraically. `cusp_table` is a dense surface sample.
Convergence measured against an independent Monte-Carlo: worst chroma undershoot **0.60 C\* at n=96,
0.20 at n=160, unchanged at n=256** — so 0.20 is the Monte-Carlo's own floor, and n=160 is the default.
Recorded because a "closed form" that is quietly a sample is the failure signature this project keeps
finding.

## 4. The white point is DERIVED, not chosen

A reference is authored against a white of 1.0; the panel's white reflects `Y_white` of that. Relative
colorimetric reproduction therefore scales **radiance** and re-encodes:

```
e(d) = 255 · srgb_encode( srgb_decode(d/255) · Y_white )
```

```
Y_white = 0.3684
asymptotic encoded ratio  srgb_encode(Y_white) = 0.641
the ratio e(d)/d          0.333 (dark end)  ...  0.642 (top)     <- A CURVE, NOT A SCALE
shipped constant                              0.75
human mean (n=23)                             0.727 (r3) / 0.800 (r4)
```

🔑 **`wp` is not a lever. It is what "put this reference on this paper" means**, and the shipped
*linear scale* is the wrong **shape** as well as (probably) the wrong value — the correct transform
compresses the shadows about twice as hard as the highlights. What is left over — 0.641 vs the human
0.73–0.80, in the direction of *more* lightness — is a **one-parameter preference residual** with a
name (elevated lightness on low-luminance reflective media) and a sign. Not a free knob.

⚠️ **This figure was wrong once already, today.** It was first stated as `Y_white**(1/2.4)` = 0.660,
which drops the sRGB affine terms (`encode(y) = 1.055·y^(1/2.4) − 0.055`). Caught by S1's own test. It
lands on the naive palette ratio 163.3/255 = 0.6405 **only because this white ink is near-neutral**, so
its flat channel mean approximates its encoded luminance — a coincidence of neutrality, not an identity.

## 5. The gamma-space dither error, isolated — prediction registered, then run

**Registered before running:** Floyd–Steinberg diffuses error in *gamma-encoded* units while the fused
image is an average of *radiance*. sRGB encoding is concave, so by Jensen the realised radiance must
exceed what the encoded arithmetic asserts — **positive everywhere, largest where the EOTF curvature is
largest (the shadows), vanishing at the endpoints.** A different shape refutes the theory.

| criterion | result |
|---|---|
| positive everywhere below the media ceiling | ✅ |
| zero at black | ✅ |
| peak in the shadows | ✅ **+13.1 L\* at d = 24** |

⚠️ **The first run appeared to REFUTE this** — minimum −42.4 L\*. Every negative value sat above
**d = 163.4**, the level whose radiance equals the paper's, where the reference is simply brighter than
the panel can be. That is **ceiling clipping — a gamut fact** — and the measurement was of two things at
once. Separating them, the prediction holds exactly. Recorded rather than quietly fixed: *"the
prediction failed so I changed the measurement"* is the move this project guards against, and what
actually changed is that the measurement was decomposed.

**So the shadows are lifted +13 L\* above what the pipeline believes it rendered — in the exact region
where §2 says the panel has only one ink.** Defect 1 and the one surviving starvation are the same
region of the tone range.

## 6. What this changes downstream

1. **ADR-094 is amended, not withdrawn.** Shadow starvation stands and is the load-bearing half.
   The highlight half is restated as a chroma trade, not a ceiling.
2. **ADR-090/091 need re-reading in this light** — "highlight collapse" was measured against a ceiling
   at the wrong place, with a luminance that ranks the inks wrongly at that end.
3. **ADR-093's constant is not overturned but it is re-founded**: 0.75 is the *preference*, 0.641 is
   the *physics*, and the difference is now a single named quantity rather than a fitted knob.
4. **S3's gamut mapping gets a real destination** — an 8-face hull with closed-form boundary queries,
   and a supra-white cusp at L\* 113 that the standard algorithm has to be told about.

## STATUS: COMPLETE
