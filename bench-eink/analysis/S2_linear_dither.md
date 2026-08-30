# S2 — error diffusion in linear light, and what the incumbent quantiser costs

**Run:** 2026-08-29. `tools/eink_dither.py`, verified by `tests/test_eink_dither.py` (12 tests).
No camera, no rig, no labels, no panel time.

## What was built

Two defects live in the production quantiser and they are independent:

```
error diffused in gamma-encoded units   ->  the wrong quantity is conserved
nearest ink by unweighted RGB distance  ->  the wrong ink is chosen
```

`mode="legacy"` reproduces both — it *is* the incumbent, re-implemented. `mode="linear"` fixes both:
target and error live in linear RGB, and the ink is chosen by CIEDE2000 in media-relative Lab via a
precomputed 64³ LUT. One code path, one difference, one number.

**The wavefront.** Floyd–Steinberg looks serial and is not. Pixel (y,x) reads error only from
(y,x−1), (y−1,x−1), (y−1,x) and (y−1,x+1) — every one with a strictly smaller `k = 2y + x`. So all
pixels sharing a `k` are independent: **3,998 vectorised steps instead of 1.92 million serial ones.**
Measured **0.56 s** for a full 1600×1200 frame in pure numpy, against the 0.60 s the plan budgeted.

⚠️ Each of the four scatter offsets is injective *individually*, so fancy-index `+=` is safe per
offset. It would **not** be safe if the offsets were merged into one index array — two sources in the
same wavefront can hit the same target, and buffered `+=` would silently drop one. The topological
ordering is asserted directly in the tests rather than trusted.

## R1 — the new code contains the old

Legacy mode reproduces **Pillow's own tone to within 0.67 L\*** across the range, while disagreeing
with it on **up to 70% of individual pixels.** That is the correct and expected result: same
algorithm, different integer arithmetic and tie-breaking, so the dither *pattern* differs and the mean
it integrates to does not. It is also A1's standing rule for this project — compare aggregates, never
a cell. Any later difference between the two modes is therefore the defect, not a bug in new code.

## R2 — two independent code paths agree on the size of the defect

| | method | peak | at |
|---|---|---|---|
| **S1** | Pillow's realised radiance vs the source's own L\* | **+13.1 L\*** | d = 24 |
| **S2** | legacy mode vs linear mode, using neither Pillow nor the source | **+13.9 L\*** | d = 24 |

Same peak level exactly, magnitudes within 0.8 L\*, sharing no code beyond `eink_color`. Either would
have refuted the other.

## Conservation — the cleanest statement of the defect

Error diffusion conserves whatever it accumulates in.

```
linear mode:  sum(target radiance) − sum(realised radiance) − error-off-the-edge  <  1e-6   ✅
legacy mode:  the same radiance identity fails, by an amount that IS the defect   ❌
```

Gamma-space diffusion conserves *encoded values*. A fused dither averages *radiance*. Those are
different quantities and the pipeline has been conserving the wrong one since the path was written.

## What it costs on real art

Fused with an 8×8 box (a placeholder for S4's contrast-sensitivity filter — see the caveat), shadow
region = source L\* < 30:

| work | shadow L\*, incumbent | shadow L\*, correct | mean ΔL\* | p95 ΔL\* |
|---|---|---|---|---|
| **The Night Watch** | **41.1** | **19.9** | **+20.5** | +28.2 |
| Olympia | 36.5 | 23.0 | +6.8 | +16.8 |
| Sunflowers | 84.4 | 65.0 | +5.2 | +17.7 |
| Café Terrace at Night | 42.2 | 39.7 | +5.7 | +22.2 |
| Flaming June | 44.7 | 35.0 | +6.4 | +22.4 |
| The Kiss | 58.0 | 45.1 | +7.0 | +18.8 |

🔑 **The Night Watch's shadows render 21 L\* lighter than they should.** L\* is roughly perceptually
uniform, so that is not a subtlety — it is a fifth of the entire tonal range, in the one region S1
confirmed the panel is genuinely starved.

📏 **This resolves a confusion the project has been carrying.** ADR-094 measured 85% of that same
shadow region rendering as *bare black ink* and called it crush. Both are true at once, and they are
the same bug: gamma-space diffusion sends most shadow pixels to pure black while the surviving
mixture overshoots, so the region is simultaneously **crushed in detail and washed in mean**. Measuring
"which ink did this pixel get" sees only the first half; measuring realised radiance sees both.

## Caveats, stated

- **The 8×8 box is a placeholder for fusion.** S4's opponent-channel CSF filter at the real 0.169 mm
  pitch is the honest version. At 0.5 m fusion is *incomplete*, so part of this difference is texture
  a viewer sees as grain rather than as tone. **These numbers are an upper bound on the tone error at
  fusing distances**, not the final figure.
- **Every absolute number inherits the palette**, which is another physical panel's. The *sign*, the
  *shape*, and the conservation argument are palette-independent; the magnitudes are not.
- **This is a diagnostic, not a production change.** `epaper.py` stays PIL-only. The shipping vehicle
  remains a 3-D LUT computed offline feeding Pillow's existing Floyd–Steinberg. Whether the residual
  after an optimal pre-transform justifies re-opening that constraint is an S5 question, and these
  numbers are the reason it is worth asking.

## STATUS: COMPLETE
