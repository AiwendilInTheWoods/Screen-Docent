# S3 — perceptual-intent gamut mapping, and the tone curve that isn't there

**Run:** 2026-08-29. `tools/eink_gamut.py`, verified by `tests/test_eink_gamut.py` (16 invariant tests).
No camera, no rig, no labels, no panel time.

## 🔴 The headline is a correction to the plan

**The plan predicted the S-curve would fall out of perceptual intent's lightness stage — "derived, not
fitted". On this palette it does not, and the reason is exact rather than approximate.**

Relative-colorimetric reproduction scales *radiance* by `Y_white`. Expressed in **media-relative L\***
that is the **identity** — verified to 0.000000 across the range. The panel's black ink is `(0,0,0)`
and its white ink *is* the media white, so the destination lightness range is **[0, 100]: precisely the
source's.** Ranges that already match need no compression, and nothing S-shaped emerges from
compressing nothing.

📏 **So the tone problem was never a range problem, and two months of curve-fitting were aimed at a
misdiagnosis.** What is actually wrong with the shadows is two other things, both now measured:

| | finding | size |
|---|---|---|
| **S2** | the quantiser conserves the wrong quantity | +13 to +21 L\* too light |
| **S1** | one ink below blue — a LEVEL DENSITY problem | 38.3% of the L\* range |

**A tone curve can fix neither.** It redistributes content inside a range that already matches; it
cannot conserve radiance and it cannot create levels. The grey-rectangle degeneracy of ADR-097 was the
optimiser correctly reporting that the objective it was given had no solution in that family.

⚠️ **The one thing that would put a toe back is a measurement we do not have.** The palette gives black
as a *perfect* `(0,0,0)`. Real e-ink black reflects something; if `L*_black > 0` the ranges stop
matching, black-point compensation becomes necessary, **and it produces a toe.** `black_L` is that
parameter — 0.0 today **by assumption, not by measurement.** This is the strongest argument in the
whole programme for buying a ColorChecker.

## What is left is chroma, and it is almost everything

The achievable gamut is **1.11% of the sRGB cube** (S1), so this is a chroma problem end to end. The
method is cusp-knee compression of the SGCK family: work inside a constant-hue leaf (hue-preserving by
construction), aim at an anchor on the lightness axis at the **destination cusp's** lightness for that
hue, and compress with a knee so the inner core is untouched.

| work | % out of gamut | source C\* | clipped | mapped (knee 0.9) | chroma kept |
|---|---|---|---|---|---|
| Olympia | 0.0% | 9.4 | 9.4 | 9.3 | 99.9% |
| The Night Watch | 12.1% | 11.3 | 11.0 | 10.8 | 95.2% |
| Sunflowers | 52.6% | 45.4 | 41.6 | 40.7 | 89.7% |
| Flaming June | **61.8%** | 35.8 | 28.5 | 27.5 | 76.8% |

## Three bugs found by the invariant tests, all real

1. **The anchor was outside the gamut.** 3 of 72 hue bins have a cusp above L\* 100 (up to 113, the
   yellows). That is a real point on the gamut *boundary*, but the anchor must be a **neutral**, and a
   neutral brighter than the white ink does not exist. Unclamped, the boundary bisection returned zero
   and those hues collapsed onto the anchor — **48 chromatic points with hue errors up to 98°**.
2. **The media-relative normalisation was missing entirely.** Converting sRGB straight to
   media-relative Lab leaves the source white at **L\* 145.7** against a destination of 100, so the
   chroma knee squashed the grey axis — doing badly and implicitly what S1's exact derived transform
   does explicitly. White rendered at L\* 96.1. And it needed to be a **chromatic adaptation**, not just
   a luminance scale: the ink white is a\* −0.9 / b\* −0.9 against D65, so scaling magnitude alone leaves
   "neutral" meaning two different axes at once and shaves ~0.07 L\* off greys near white.
3. **`in_source` tested the un-normalised cube** — `[0,1]³` where the normalised source is `[0, Y_white]³`
   — the same omission a second time, with the same symptom.

📏 All three were found by *invariants*, not by looking at pictures. A gamut map has no reference
answer to check against, so the tests assert what must hold by construction: output always in gamut,
hue preserved exactly, ordering preserved, continuity, the knee leaving the core bit-identical, each
ink mapping to itself, and `knee=1.0` degenerating to pure clipping — the null hypothesis this family
contains.

## The anchor margin, measured rather than chosen

The bipyramid degenerates to a point at its apexes, so an anchor there makes every ray through it
ill-conditioned. A **dense** sweep of 2736 (hue, L\*) rays — a coarse one reported clean and missed it
entirely — found chroma-ordering **inversions**: 7 rays with a backward step, worst **−5.85 C\***, all
at L\* 88–98 and hue 75–85, exactly where the clamped yellow cusp put the anchor on top of the content.

```
anchor margin    worst backward step    rays worse than -0.5 C*    mean output C*
      0                -5.85                     7                    21.55
      5                -0.89                     3                    21.54
     12                -0.33                     0                    21.50
```

**12 L\* buys strict ordering for 0.05 C\* of mean chroma — 0.2%**, and the residual −0.33 C\* is a third
of the ~1 C\* discrimination threshold. Ordering is what perceptual intent promises. The residual is
stated, not rounded to zero.

## Cost and the shipping vehicle

Boundary distances are found by **vectorised bisection**, not algebra: the hull is a polytope in linear
RGB but Lab is nonlinear in XYZ, so a straight ray in Lab is a curve in RGB. That costs **~14 s for a
full 1600×1200 frame** — fine for a maintainer tool, impossible on a Pi. S5 bakes the finished map into
a **33³ `Color3DLUT`**, applied in **0.034 s** in pure PIL. The bisection is the authoring cost, paid once.

## STATUS: COMPLETE
