# A1 — Data integrity and error bars

**Agent A1 (foundation).** Corpus commit `1063f81`, 119 conditions, 131 raw captures, session
2026-08-29 13:12:21 → 15:47:05 local (154.7 min).
Units throughout: **camera-RGB normalised to this panel's own black = 0 / white = 255. NOT sRGB.**

> This file is written incrementally. If it lacks a closing `## STATUS: COMPLETE` line, the run died
> mid-way and everything above the cut is still valid.

## The error bars — lead table

**σ = 1 standard deviation of a single measurement. MDD = smallest difference between two single
measurements that is credible at 95% (2.8σ). Report nothing below the MDD as a finding.**

| readout | σ | 95% MDD | basis |
|---|---|---|---|
| `tonefine` `out_lum`, **a single step** | 9.2 | **26** | 6 identical-render groups, 30 df |
| `tonefine` `out_lum`, **mean over 26 steps** | 4.1 | **11** | same |
| `tonefine` `grain`, single step / mean | 13.1 / 2.3 | 37 / 6.5 | same |
| `tonefine` `grain_peak` | 10.7 | 30 | same |
| `tonefine` `collapsed_step_pairs` (of 25) | 1.4 | 3.8 | same |
| `tonefine` `chroma` per step | 4.1 | 11 | same |
| `tonefine` `neutral_hue_range_deg` | 33 | 92 | **unusable** |
| `huevalue` `chroma_out`, **a single cell** | 5.4 | **15** | 4 replicates ×1.45 |
| `huevalue` `lum_out`, a single cell | 9.0 | 25 | same |
| `huevalue` mean chroma over a value row (12 cells) | 5.3 | 15 | same |
| `huevalue` `mean_chroma_all` (72 cells) | 1.2 | 3.3 | same |
| `huevalue` `n_collapsed_total` (of 72) | 3.2 | 9 | same |
| undithered flat patch, refresh-to-refresh | ~1 (mean) / 3.5 (worst) | — | `primaries#1` vs `#2`, correctly sampled |

The doc's working floor of **~16 worst / 6.7 mean is superseded**: it was measured through instrument
defect #8 and is simultaneously ~4x too pessimistic for an undithered patch and, for a *single step*
of a dithered ramp, ~1.6x too optimistic.

---

## Task 3a — Did randomisation hold?  **NO — and only for part of the corpus**

This is reported first because it changes how everything else must be read.

The run was **not one invocation**. The capture times split the corpus into four blocks:

| rows (index) | wall clock | what it is | order |
|---|---|---|---|
| 0–2 | 0.0–1.6 min | `primaries#1`, `primaries#2`, `inkmix` | design (intended) |
| 3–47 | 2.3–36.5 min | the crossed / one-at-a-time blocks: `tonefine` wp×gamma, tone nulls, `huevalue` wp×chroma, `edges`, `linepairs`, `surround`, `resample` | **LITERAL DESIGN ORDER — not shuffled** |
| 48–112 | 51.8–104.4 min | the central-composite block (axial + 2⁴ corners + centre replicates) | **properly randomised** |
| 113–114 | 128.1–129.8 min | `uniformity@0`, `uniformity@180` | separate invocation |
| 115–118 | 151.8–154.7 min | `huevalue_lowv_*` | separate invocation, ascending wp |

Rows 3–47 reproduce the order of `_rows()` in `tools/eink_vault.py` exactly, element for element.
`_randomised()` shuffles the tail, so this block cannot have passed through it: it was captured by an
earlier invocation of the tool that did not yet randomise (or with the composite block not yet in the
matrix). **The doc's claim "run order is randomised" is true of block 2 and false of block 1.**

### The measurement — Pearson r of lever value against capture time, permutation p (20 000 shuffles)

```
BLOCK 2, randomised (n=65)          BLOCK 1, whole (n=45)
  wp      r=+0.084  p=0.51            wp      r=-0.036  p=0.82
  gamma   r=+0.093  p=0.47            gamma   r=-0.112  p=0.46
  chroma  r=+0.118  p=0.35            chroma  r=-0.030  p=0.84
  sat     r=-0.110  p=0.38            sat     r=+0.011  p=0.95

BLOCK 1, tonefine wp×gamma cross     BLOCK 1, huevalue wp×chroma cross
  (rows 3–14, n=12, 2.3→10.8 min)      (rows 18–29, n=12, 13.9→22.5 min)
  wp      r=+0.883  p<0.0001  <--      wp      r=+0.883  p<0.0001  <--
  gamma   r=+0.236  p=0.48             chroma  r=+0.236  p=0.48
```

**White-point is aliased onto capture time inside both crossed blocks.** The sweep ran
wp 0.0, 0.0, 0.0, 0.64, 0.64, 0.64, 0.75, 0.75, 0.75, 0.88, 0.88, 0.88 in that order — exactly the
failure mode the design note warned about. Gamma and chroma are the inner loop and are therefore
*protected* (r=+0.24, n.s.); wp is the outer loop and is not.

Whole-session correlations are worse still and are an artefact of the block split — block 1 sits
mostly at chroma 1.0 / sat 1.0 and block 2 is centred at chroma 1.5 — giving `chroma` r=+0.54
(p=0.0002) on `tonefine` and `gamma` r=+0.59 (p<0.0001) on `huevalue` when all rows are pooled.

### What this costs, and the mitigation
The confounded window is **8.5 min inside block 1**, not the 154-min session. The global illumination
level is absorbed by the per-photograph black/white affine, so only the *shape* drift leaks through,
and that was measured at 1.5% median / 6.9% p95 over the full 192 min between the two flat fields —
pro-rata ≈0.07% median over 8.5 min. So the alias is real but small.

**The mitigation is that block 2 re-measures the same wp levels in randomised order.** wp
0.0 / 0.64 / 0.75 / 0.88 / 1.0 all appear in block 2 on both `tonefine` and `huevalue`.

> **Downstream rule:** a white-point effect estimated *only* from rows 3–14 or 18–29 carries a drift
> term and must be confirmed against the block-2 rows before it is called a finding. Effects on
> gamma, chroma and saturation are not aliased anywhere.

---

## NEW: instrument defect #8 — the calibration-strip readout samples the wrong pixels

Found while computing the refresh-to-refresh floor. It has the documented signature exactly: it
always returns six plausible numbers and can never fail.

**The mechanism.** `read_panel` (correctly, per defect #7) solves the affine on the *rectified*
image, where the homography has put the calibration furniture at its nominal coordinates, and uses
`strip_dy()` to find the strip. It then calls `align_to_reference`, which **shifts and rescales the
whole image to line the CONTENT up** — which necessarily moves the furniture *off* nominal. The
returned `corrected` image is the aligned one. `readout_primaries_from_strip()` then samples
`patch_rects(w, h, 6)` at **nominal dy=0** on that aligned image, i.e. at coordinates the strip no
longer occupies.

**The check that can fail, and does.** Black and white are the affine's anchors, so after correction
they *must* read ≈0 and ≈255 wherever they are correctly sampled. On `primaries#1`:

```
                                             black                white
sampled on corrected+UNALIGNED, strip_dy      [2.1, 1.5, 1.8]     [252.7, 253.5, 253.2]   <- correct
sampled on corrected+ALIGNED, nominal dy=0   [61.6, 52.5, 37.3]   [226.1, 235.2, 229.6]   <- shipped
                                                                     align = (corr 0.799, scale 0.96, dx +32, dy -48)
```

A 96 px-tall patch read with a 22% inset has ~21 px of margin; the alignment moved it 48 px. The
sample window is largely off the patch.

**What this invalidates** (in BOTH `panel_profile.jsonl` and `panel_profile_rederived.jsonl`):

| field | status |
|---|---|
| `readout.strip.*` (every target's `primaries` rows) | **invalid — do not use** |
| `readout.field_vs_strip`, `worst_disagreement` | **invalid** |
| `inkmix.linearity_error`, `worst_linearity_error` | **invalid** — its pure-ink reference is the strip |
| `readout.fields` (`primaries`) | valid (content, aligned correctly) |
| `inkmix.mixtures`, `element_sweep`, `metamer_row` | valid (content, via `_cells`) |
| every `_cells`-based readout (tonefine, huevalue, surround, edges, linepairs, resample) | valid |
| `gain`, `offset`, `patch_residual` | valid — solved before alignment |

It is not a bug in the alignment: aligning the content necessarily de-aligns the furniture. It is a
bug in reading the furniture off the aligned image. **Recomputable offline** — sample the strip on
`corrected-before-align`. That is a one-line fix in `read_panel` (return both images) and is the
single highest-value pipeline change left.

### Consequence: the published refresh-to-refresh floor is ~4x too pessimistic

`docs/eink-panel-characterisation.md` quotes **16/255 worst, 6.7 mean** for refresh-to-refresh. That
figure is the `primaries#1` vs `#2` strip comparison — i.e. it is dominated by defect #8, not by the
panel. Recomputed with the strip sampled correctly:

```
UNDITHERED flat patch, refresh #1 vs refresh #2
  250x96 strip patch, correctly sampled     worst 3.5   mean 0.88   rms 1.40
  509x345 ink field   (as shipped)          worst 12.8  mean 2.63
     (the 12.8 is green alone; every other ink is <= 4.0. The two refreshes aligned to
      dx +32/dy -48 and dx +12/dy -40, so some of the green field's spread is align jitter.)
  as published (strip, defect #8)           worst 20.7  mean 9.85
```

⚠️ **This does NOT license using ~1/255 as a general error bar.** It is the floor for a *large flat
undithered* patch only. Dithered targets are stochastic and are several times noisier — see Task 2.

**What survives unchanged:** the field-vs-strip disagreement on the *chromatic* inks is real and is
not defect #8. Correctly sampled it is red 78, yellow 64, blue 42, green 13 — while black is 2.1 and
white 6.4. The doc's warning that this is a patch-size/context effect and **not** an error bar stands;
only the black (56.7 → 2.1) and white (13.1 → 6.4) figures were corrupted.

---

## Task 1 — Re-derivation consistency, and which file to use

Re-derived all 119 rows to `bench-eink/analysis/A1_rederived.jsonl` with the time-interpolated flat
field (`flat.png` 12:05 → `flat_close.png` 15:17, weight from each raw's mtime — mtimes survived, they
are the real capture times). **119 re-derived, 0 failed.**

### The disagreement is large, and it is not the flat field — it is the ALIGNMENT

| metric | median |in-run − rederived| | p90 | max |
|---|---|---|---|
| `tonefine` per-step `out_lum` | **7.5** | 24.4 | 46.0 |
| `tonefine` per-step `grain` | 3.7 | 15.8 | 23.5 |
| `huevalue` per-cell `chroma_out` | **5.2** | 21.6 | 68.8 |
| `huevalue` per-cell `lum_out` | 7.4 | 35.9 | 84.9 |
| `tonefine` `collapsed_step_pairs` | −1.0 (median shift) | | ±5 |
| `huevalue` `n_collapsed_total` | +1.0 (median shift) | | ±11 |

**The cause: `panel_profile.jsonl` mixes two alignment regimes.** 48 of its 119 rows carry the
*identical* align tuple `(0.353, 0.94, 6, −42)` — that is the `inkmix`-derived **global prior**, not a
per-row search (the tell is that the stored correlation is `inkmix`'s own 0.353, on rows of every
other kind). The remaining 71 rows carry per-row search results. The prior-aligned rows are
`primaries`(2), `inkmix`(1), `edges`(4), `linepairs`(4), `surround`(2), `resample`(2),
`tonefine`(15), `huevalue`(18).

Split the disagreement by that regime and it is entirely explained:

```
                        median |in-run - rederived|, per-element
  tonefine  out_lum     prior-aligned rows  22.8      searched rows  5.6
  huevalue  chroma_out  prior-aligned rows  17.5      searched rows  4.9
```

The searched rows agree to ~5/255, which is inside the pure error (Task 2). The prior rows disagree
by 4x that.

### Which alignment is right — tested against a criterion that can fail
Correlation cannot arbitrate (the search maximises it by construction). **Monotonicity can**: a tone
ramp is monotone by construction, so counting monotone step pairs is an independent test.

```
tonefine, mean monotone step pairs out of 25
  in-run,   prior-aligned rows (n=15)     16.53   ->  re-derived  19.07
  in-run,   searched rows      (n=33)     15.94   ->  re-derived  16.15
```

Re-derivation improves the prior rows by 2.5 pairs out of 25 and leaves the rest alone. It is better
by an independent measure, not merely different.

### DECISION
> **Downstream agents must use `bench-eink/analysis/A1_rederived.jsonl`.**
> Reasons, in order: (1) `panel_profile.jsonl` mixes a global alignment prior with per-row searches
> across 48/119 rows, so its rows are not mutually comparable; (2) the re-derive applies the
> time-interpolated flat field to every row; (3) it is the only file covering all 119 conditions.
>
> `bench-eink/panel_profile_rederived.jsonl` (the shipped one) is **113 rows** — it predates
> `uniformity@0`, `uniformity@180` and the four `huevalue_lowv_*` rows, and it was built without the
> `--flat-close` interpolation. Its `tonefine` readouts are equivalent to mine (identical monotone
> means), so it is not *wrong*; it is just incomplete. Prefer `A1_rederived.jsonl`.

### ⚠️ Traps inside the re-derived file, in BOTH copies
- **`gain` and `offset` are STALE.** `cmd_rederive` overwrites `readout`, `patch_residual` and
  `align`, and carries `gain`/`offset` through from the in-run record. They were solved against the
  in-run flat field. Do not read them as belonging to the re-derived numbers.
- **`readout.strip`, `field_vs_strip`, `linearity_error` are invalid** — defect #8 above.
- **The four `huevalue_lowv_*` rows are mislabelled and mis-flat-fielded.**
  (a) They were rendered with `--v-lo 20 --v-hi 100`, but `readout_huevalue` hard-codes its value
  labels as `40 + 205·r/5`, so the `value_in` field reads 40/81/122/163/204/245 when the actual
  inputs were 20/36/52/68/84/100. (b) They were captured at 15:44–15:47, **after the panel was
  rotated 180°** for `uniformity@180`, and their correct flat field is `flat_final.png` (15:42),
  which the two-flat interpolation cannot reach — they get `flat_close.png` (pre-rotation). (c) The
  alignment reference `_reference()` rebuilds the default v-range target, so it does not match what
  was on the panel. **Treat these four rows as a separate sub-corpus with its own caveats.**
- `uniformity@180` is likewise post-rotation and is corrected with the pre-rotation flat field.

---

## Task 2 — Pure error, per readout metric

### The right replicate set is bigger than 3, and the brief's set understates the error

Two independent facts widen the pure-error estimate beyond the three named centre replicates:

1. **There are FOUR centre replicates per target, not three.** The axial block's centre point
   (`tonefine_wp0.75_g1.4_k1.5_s1.0`, no `_rep` suffix) is lever-identical to `_rep1/2/3`. Same for
   `huevalue`. Times: tonefine 57.5 / 78.7 / 86.9 / 99.3 min; huevalue 55.9 / 81.2 / 83.6 / 88.6 min.

2. **On the NEUTRAL `tonefine` target, saturation and chroma-gamma are inert BY CONSTRUCTION** —
   verified digitally, not assumed: rendering `tonefine` through the full lever chain at
   s ∈ {0.7, 1.0, 1.3} and k ∈ {1.0, 2.0} against s=1.0/k=1.0 gives **max pixel difference 0, over
   0.0000% of pixels**. `ImageEnhance.Color` and the HSV chroma LUT are exact identities on a
   neutral. So every `tonefine` row sharing a `(wp, gamma)` pair is the **same bitmap, pushed to the
   panel again**. That yields six replicate groups of 5–11 rows spanning 2 → 99 min:

   ```
   wp0.64 g1.0 : 5     wp0.75 g1.0 : 5     wp0.88 g1.0 : 5
   wp0.64 g1.8 : 5     wp0.75 g1.4 : 11    wp0.88 g1.8 : 5      (36 rows, 6 groups, 30 df)
   ```

**These groups are the correct pure-error estimate and the centre replicates are not**, because all
four centre replicates happen to receive the *same* alignment solution, while different conditions do
not. The replicates therefore do not sample the alignment lottery, and understate the error by ~1.45x:

```
tonefine per-step out_lum, 1 sigma
   4 centre replicates (all share dx=+88)                        6.36 median
   6 identical-render groups, 30 df (alignment varies)           9.22 median   <- USE THIS
```

### Error bars, `tonefine` (pooled within identical-render groups, 30 df)

| readout | 1σ | median level | 95% min. detectable difference (2.8σ) |
|---|---|---|---|
| `steps[i].out_lum` — **a single step** | **9.2** (mean 12.1, p90 22.2, max 30.7) | 4 → 254 | **26** |
| mean `out_lum` over all 26 steps | **4.1** | 64 → 215 | 11 |
| mean `out_lum`, lower half / upper half | 4.4 / 4.8 | | 12 / 13 |
| `steps[i].grain` — a single step | **13.1** | 1 → 102 | 37 |
| mean `grain` over 26 steps | **2.3** | 8.5 → 41 | 6.5 |
| `grain_peak` | **10.7** | 31 → 107 | 30 |
| `steps[i].chroma` | 4.1 | 1.6 → 30 | 11 |
| `max_chroma_on_neutral_axis` | 3.8 | 18 → 42 | 10.6 |
| `collapsed_step_pairs` (of 25) | **1.37** | 1 → 10 | 3.8 |
| monotone step pairs (of 25) | 1.75 | 11 → 23 | 4.9 |
| `ramp_span` (max − min) | 5.9 | 137 → 255 | 17 |
| `neutral_hue_range_deg` | **33.0** | mean 120 | 92 — **unusable, do not report** |
| `hue_qualifying_steps` | 2.0 | mean 17.5 | 5.7 |
| `steps[i].anisotropy` | 0.07 median but 2.5 mean, max 52 | | heavy-tailed — treat per-step values as unusable, use the median over steps |

### Error bars, `huevalue` (4 centre replicates, ×1.45 for the alignment lottery)

`huevalue` has **no identical-render group** — saturation and chroma are live on a chromatic target —
so its replicates share one alignment and understate the error the same way `tonefine`'s do. The
factor 1.45 is transferred from the `tonefine` measurement above; it is an estimate, not a measurement,
and is flagged as such in `error_bars.json`.

| readout | 1σ (replicates) | 1σ (inflated ×1.45) | 95% MDD |
|---|---|---|---|
| `cells[].chroma_out` — a single cell | 3.7 (p90 9.6, max 13.5) | **5.4** | **15** |
| `cells[].lum_out` — a single cell | 6.2 (p90 13.1, max 18.0) | **9.0** | 25 |
| mean chroma over a value row (12 cells) | 0.6 – 3.7 by row | **5.3** worst | 15 |
| `mean_chroma_all` (72 cells) | 0.81 | **1.2** | 3.3 |
| `n_collapsed` per value row (of 12) | 0.58 – 1.00 | 1.4 | 4 |
| `n_collapsed_total` (of 72) | 2.22 | 3.2 | 9 |

### Refresh-to-refresh floor, undithered (`primaries#1` vs `#2`, 46 s apart)

```
250x96 strip patch, correctly sampled (see defect #8)   worst 3.5   mean 0.88   rms 1.40
509x345 ink field, as shipped in the rederived file     worst 11.6  mean 2.67
                                                          (11.6 is green alone; every other ink <= 4)
```

**An undithered flat patch repeats to ~1–3/255. A dithered grid cell does not** — it repeats to
9–13/255 per step. The difference is the dither's stochasticity plus the per-row alignment. Do not
carry the undithered number into a dithered comparison.

---

## Task 3b — Is precision time-dependent?  **Yes at the sensor, no at the readout**

Two different quantities behave in opposite directions, and conflating them would give the wrong
answer either way.

**At the sensor, the noise really did grow — by 1.75x, matching the ~1.6x claim.** `patch_residual`
is within-patch standard deviation measured on the *rectified, pre-affine* image, i.e. in raw camera
units. It must be multiplied by the affine `gain` to become panel-relative:

```
                 mean patch_residual   mean gain   panel-relative noise
  t   0- 40 min          3.87            3.24            12.8
  t  40- 70 min          6.63            3.39            22.5     <- 1.76x
  t  70-105 min          6.35            3.52            22.4
corr(t, patch_residual) = +0.607     corr(t, gain) = +0.894 (light falling)
```

**At the readout, precision did NOT degrade — it slightly improved.** Using the 36 identical-render
`tonefine` rows and each row's deviation from its own group mean:

```
corr(capture time, |out_lum residual|) = -0.368
  t   0- 40 min  n= 9   mean |residual| 10.60
  t  40- 70 min  n= 9   mean |residual|  7.42
  t  70-105 min  n=18   mean |residual|  6.71
```

The reason is that the readout error is **alignment-dominated, not photon-dominated**: a 1.75x change
in a 12–22/255 sensor term is invisible underneath a registration term worth tens of 255ths.

> **No time-dependent error bar is warranted.** Use one error bar for the whole session. If a later
> agent needs one, the conservative reading is that the FIRST 40 minutes are marginally *worse*, which
> is the opposite of the assumption in the brief — and those are exactly the non-randomised block-1
> rows (Task 3a).

---

## Task 4 — Null checks

**Both nulls are exact by construction, and that was verified rather than assumed.** Rendering the
neutral `tonefine` target through the real lever chain
(`white-point → chroma-gamma → saturation → gamma`, `tools/eink_bench.py:_pre`):

```
wp0.75 g1.0 k1.0 s0.7  vs  s1.0      max pixel difference 0     differing pixels 0.0000%
wp0.75 g1.0 k1.0 s1.3  vs  s1.0      max pixel difference 0     differing pixels 0.0000%
wp0.75 g1.0 k2.0 s1.0  vs  k1.0      max pixel difference 0     differing pixels 0.0000%
(gamma, by contrast, changes 52.3% of pixels — the comparison is capable of showing a difference)
```

So any measured difference across a saturation or chroma sweep on `tonefine` is **100% measurement
error**, and its size is the honest instrument bar.

### The residual

```
group (identical bitmaps)                      n   median |Δ| per step   p90     MAX
NULL  saturation, wp0.75 g1.0, s .7/1/1.3      3         3.88           12.6    37.5
NULL  saturation, wp0.75 g1.4 k1.5, 5 levels   5         5.92           26.9    75.6
NULL  chroma-gamma, wp0.75 g1.0, k 1.0/2.0     2         2.69           16.8    45.1
NULL  chroma-gamma, wp0.75 g1.4, k 1/1.5/2/2.5 4         4.10           42.4   143.1
REF   the 4 true centre replicates             4         6.28           36.4    70.5
---------------------------------------------------------------------------------------
REAL  gamma 1.0 / 1.4 / 1.8 at wp0.75          3        48.47          107.8   124.5
```

**Verdict: the nulls PASS.** The null residual (median 2.7–5.9/255) sits at or below the pure error
measured from true replicates (6.3), and an order of magnitude below a real lever effect (48.5). There
is **no systematic saturation or chroma leakage into the neutral ramp.**

⚠️ **But the tail is not small.** Inside a null group — bit-identical images — individual steps differ
by up to **143/255**. That is the alignment lottery landing a sample window on a neighbouring step.
Two consequences, and they are the operative rule for every later agent:

> **Per-step and per-cell comparisons between two conditions are UNSAFE.** Compare aggregates (mean
> over steps, mean over a value row, a count), where the null residual falls to ~4/255.

A concrete example of the tail: `tonefine_wp0.75_g1.0_k1.0_s1.3` and `..._k2.0_s1.0` report
`grain_peak` 56.4 and 55.7, while their three bit-identical siblings report 83.0, 88.2 and 68.1. A
30-unit spread on a null, from nothing but registration.

---

## Task 5 — Alignment wobble: quantified, diagnosed, and partly fixable

### It is not noise; it is registration
A tone ramp is monotone by construction — confirmed digitally: the rendered target reads **25/25
monotone step pairs** at both wp0.75 g1.4 and wp0.0 g1.0, and `huevalue` reads **60/60** monotone
`lum_out` pairs. On the panel:

```
tonefine monotone step pairs / 25      in-run 16.12 mean   re-derived 17.06 mean   (never 25)
```

If the inversions were photon noise they would sit at the error bar. They do not:

```
381 inverted pairs across 48 rows:  median |Δ| 13.2   p90 77.3   max 211.1
   only 27.6% are below 1 sigma (3.6)   only 45.1% are below 2.8 sigma (10.0)
```

More than half of the non-monotonicity is a **structural** error — the sampling window straddling a
cell boundary — not scatter.

### It correlates with the alignment solution, exactly as that diagnosis predicts

```
corr(monotone_pairs, align correlation) = +0.773
corr(monotone_pairs, |dx|)              = -0.702
corr(monotone_pairs, patch_residual)    = -0.373

rows pinned at the +-88 translation search LIMIT:  22/48 tonefine, 32/119 corpus-wide
   monotone mean, pinned  13.82      not pinned  19.81
rows sitting on a scale-search EDGE (0.96 or 1.04): 45/119
```

**A third of the corpus is aligned by a boundary result, and a boundary result is a request to widen
the search, not an answer.**

### Alignment is NOT behaving as a rig property
Camera and panel did not move, so one alignment should fit every row. It does not:

```
kind        n    align corr (med)   scale (med)   dx (med, range)      dy (med, range)
primaries   2       0.809             0.96        +22  (+12..+32)      -44  (-48..-40)
edges       4       0.558             0.96        +18  (-60..+28)      -28
surround    2       0.548             0.97         -6  ( -8.. -4)      -24
tonefine   48       0.727             0.98        +72  (+40..+88)      -28  (-48..-24)
huevalue   54       0.457             0.98        -72  (-88..+88)      -24  (-44..+88)
linepairs   4       0.346             0.98        -80  (-84..-80)      -24
```

`tonefine` sits **+50 px** from `primaries` — close to half its 91.7 px cell pitch — and `huevalue`
sits **−94 px** from it — close to one full 99.3 px cell pitch. Those are the periodic aliases of each
target's own grid, which is what a cross-correlation against a repeating pattern is expected to find.

### The pre-registered test — and it failed in the informative direction
**Registered before running:** *if the dense targets are aliasing, then forcing the alignment measured
on `primaries` (the only target with no periodic structure, and the highest correlation, 0.81) should
IMPROVE ramp monotonicity.* Re-derived all 48 `tonefine` rows under five forced alignments and one
free search, scoring on two criteria that are independent of the alignment objective:

```
arm                       monotone/25    pooled sd over identical renders: out_lum   grain
free per-row search          17.06                                          9.22     13.05
primaries consensus          15.77   <- WORSE                               8.50     10.10
doc consensus (0.94,6,-42)   15.85   <- WORSE                               7.71      8.23
no alignment at all          14.83                                          8.04      7.63
surround consensus           16.04                                          7.88      7.38
tonefine's OWN consensus     19.21   <- BEST on both                        6.64      5.91
   (0.96, +68, -28) = the median of tonefine's own per-row search results
```

**The prediction was wrong.** The `primaries`-derived alignment is worse, not better. The per-row
search is finding the right neighbourhood for `tonefine`; what it is not doing is finding it
*stably* — and the instability, not the aliasing, is the error. Fixing the alignment to the target's
own consensus improves monotonicity by **+2.15 pairs of 25** and cuts the identical-render scatter by
**28% (out_lum) and 55% (grain)** — on two criteria neither of which is the thing being optimised.

`huevalue`, same treatment, scored on `lum_out` monotonicity (60 pairs, digitally 60/60):

```
free per-row search 46.06/60   its own consensus 44.62   no alignment 48.40   tonefine consensus 52.54
```

For `huevalue` the per-row search is **worse than doing nothing**. Its correlation peaks are low
(median 0.457) and its `dx` spans the entire ±88 search range, which for a rig-fixed quantity is
proof of failure rather than variation.

### So: is it fixable offline?  **Partly, and cheaply**

> **YES — replace the per-row search with a per-kind consensus.** Take the median
> `(scale, dx, dy)` over all rows of a kind, then re-derive that kind with it as `align_prior`.
> Measured gain on `tonefine`: monotone 17.06 → 19.21, per-step `out_lum` σ 9.22 → 6.64,
> per-step `grain` σ 13.05 → 5.91. Three lines in `cmd_rederive`. **I have NOT applied it to
> `A1_rederived.jsonl`** — that file is the standard re-derive from `STANDING_RULES`, so it stays
> reproducible; the error bars I ship describe it as it stands.
>
> **NO — it cannot be fixed all the way.** Even at the best alignment the ramp reaches only 19.2/25,
> not 25/25, and `huevalue` only 52.5/60. A residual ~23% of the wobble is not removable by a global
> similarity transform, because the real distortion is not global (see `grid_offsets`' own docstring:
> the homography fits four fiducials exactly and the distortion lives *between* them).
>
> The proper fix is upstream and out of scope here: **widen the search** (a third of rows are pinned
> at its limit) and **give `huevalue`'s alignment reference the full lever chain** — `_reference()` in
> `eink_vault.py` applies only `--white-point` and `--gamma`, so for a chroma- or saturation-modified
> chromatic target the template does not match what was photographed, which is the likeliest reason
> `huevalue` correlates at 0.46 while `tonefine` correlates at 0.73.

### Consequence already folded into Task 2
The alignment lottery is ~1.45x of the total per-step error. It is *inside* the shipped error bars,
which is why they are 9.2 and not 6.4.

---

## Task 6 — What to exclude

Bias is toward keeping rows and widening bars. Only one row is excluded outright; the rest are
**scoped** — usable for some questions, not for others.

### EXCLUDE (1 row)
| `cond` | reason |
|---|---|
| `uniformity@180` | **The affine failed completely in the re-derive: gain = [255000, 255000, 255000]**, i.e. white − black ≈ 0. The panel was physically rotated 180° for this capture, so the calibration strip is upside down at the top of the frame and `strip_dy`/`patch_rects` sample the wrong region — the two anchors land on the same content. `patch_residual` 17.99, the worst in the corpus by 2.6x. Its readout is garbage. |

**It is recoverable, and here is exactly how** (tested): rotate the RAW 180° before rectifying.
`Image.open(raw).rotate(180)` → gain [2.51, 2.59, 2.64], residual 12.26; with `flat_final.png`
(15:42, the post-rotation flat) → gain [2.65, 2.46, 2.52], residual 11.45. Still the worst residual in
the corpus, but usable. Whoever needs the uniformity pair should do this rather than drop it.

### DO NOT USE THESE READOUTS (the rows are fine, the numbers are not)
| what | reason |
|---|---|
| `readout.strip`, `field_vs_strip`, `worst_disagreement` on every row | defect #8 — sampled off-patch after alignment |
| `inkmix.linearity_error`, `worst_linearity_error` | same — its pure-ink reference is the strip |
| **`surround_wp0.0_*` and `surround_wp0.75_*` `spread` / `sd` / `centre_out`** | see below |
| `tonefine.neutral_hue_range_deg` | σ = 33° on a mean of 120° |
| `steps[].anisotropy` per step | heavy-tailed, σ up to 52 on a single step |

**The `surround` readout is a registration artefact, and this is provable.** Digitally, the 25 centre
patches are *identical* (they must be — the same value 25 times). The digital readout returns
**spread 0.0** at wp0.0 and **8.2** at wp0.75. The panel measured **150.5** and **158.3**. Shifting
the digital image and re-reading reproduces the measured value exactly:

```
digital shift   0px -> spread   8.2      16px -> 100.8
                4px ->         37.1      24px -> 157.7   <- measured on the panel: 150.5 / 158.3
```

The cause is in the readout, not the target: `readout_surround` takes the middle *half* of each cell,
and `target_surround` insets the centre patch by exactly a quarter of the cell — so the sample window
is the centre patch with **zero margin**. Any sub-cell registration error mixes surround straight into
the reading. Fixable offline by insetting to the middle third; until then,
**the surround/dither-bleed question is UNANSWERED, not answered negatively.** Every dense grid target
in this battery still carries an unquantified surround term.

### SCOPE, do not pool (6 rows)
| `cond` | scoping |
|---|---|
| `huevalue_lowv_wp{0.0,0.64,0.75,0.88}_g1.0` (4 rows) | `value_in` labels are wrong (readout hard-codes 40…245; the render used `--v-lo 20 --v-hi 100`, so the true inputs are 20/36/52/68/84/100); corrected with the pre-rotation flat instead of `flat_final.png`; alignment reference rebuilt at the default v-range. **Usable for within-block wp comparison only.** They are the dedicated low-value target the characterisation doc asks for, so they matter — but they are their own sub-corpus. |
| `uniformity@0` | valid capture, but uninterpretable alone: without a working `@180` partner, panel non-uniformity and flat-field residual are perfectly confounded. |

### FIXED BY THE RE-DERIVE — do not exclude, just do not use the in-run file
- `huevalue_wp0.75_g1.0_k2.0_s1.0_hf0.5`: in-run gain **[255000, 255000, 60.2]** — a blown affine.
  Re-derived gain [2.80, 3.31, 3.69], residual 6.74. Fine.
- `edges_wp0.0_g1.0_k1.0_s1.0`: in-run gain [27.8, 23.7, 20.5] (≈7x the normal 3.2–3.7). Re-derived
  [2.83, 3.26, 3.62]. Fine.

Two rows with a physically impossible affine sat in the shipped in-run profile and were reported
`ok: true` with a healthy `patch_residual` of 4.38 and 3.55. **`patch_residual` cannot see a blown
gain** — the same blind spot as defect #7. A one-line sanity assert (`0.5 < gain < 20`) would catch it.

### KEEP EVERYTHING ELSE — 112 of 119 rows are clean
No row is excluded for being an outlier. `patch_residual` across the re-derived corpus is
median 6.52, max 7.02 excluding `uniformity@180` — a tight, unremarkable distribution.

---

## What downstream agents must assume

1. **Use `bench-eink/analysis/A1_rederived.jsonl`.** Not `panel_profile.jsonl` (mixes a global
   alignment prior with per-row searches across 48/119 rows, and carries two rows with a blown
   affine), not `panel_profile_rederived.jsonl` (113 rows, no flat interpolation).
2. **Compare AGGREGATES, never a single step or a single cell.** Between *bit-identical* renders,
   individual steps differed by up to **143/255**. Aggregate `out_lum` MDD is 11; single-step is 26,
   with a tail far beyond it.
3. **The thresholds are in `error_bars.json`.** Headline: `tonefine out_lum` mean **11**,
   `huevalue chroma_out` single cell **15**, `mean_chroma_all` **3.3**, `grain_peak` **30**,
   `collapsed_step_pairs` **3.8**. Anything smaller is below resolution — say so, do not report it.
4. **These fields are INVALID, in every file:** `readout.strip.*`, `field_vs_strip`,
   `worst_disagreement`, `linearity_error`, `worst_linearity_error` (instrument defect #8);
   the `surround` rows' `spread`/`sd`/`centre_out` (a pure registration artefact — digital truth is
   0.0, measured 150); `gain`/`offset` on any re-derived row (stale); `neutral_hue_range_deg`
   (σ=33°); per-step `anisotropy` (σ up to 52).
5. **Randomisation held only for rows 48–112.** Rows 3–47 ran in literal design order with
   white-point as the outer loop, so **wp is aliased onto capture time (r=+0.88) inside both crossed
   blocks.** A white-point claim resting only on `tonefine_wp*_g*_k1.0_s1.0` (rows 3–14) or
   `huevalue_wp*_g1.0_k*` (rows 18–29) must be re-checked against block 2. Gamma, chroma and
   saturation are clean everywhere.
6. **Use one error bar for the whole session.** Sensor noise grew 1.76x as the daylight fell, but
   readout precision did not degrade (it improved slightly) because the readout error is
   alignment-dominated. Do not apply a time-dependent correction.
7. **The nulls pass** — saturation and chroma-gamma do not move the neutral ramp (they are
   bit-identical no-ops, verified digitally). So a lever effect on `tonefine` is real only if it is
   a *gamma* or *white-point* effect; if a chroma/saturation effect appears there, it is your
   alignment, not the panel.
8. **A tone ramp reads 17/25 monotone and a hue grid 46/60, where the digital truth is 25/25 and
   60/60.** That gap is registration, not panel behaviour. Do not interpret non-monotonicity as a
   panel finding.
9. **`uniformity@180` is excluded** (affine failed, gain 255000). `uniformity@0` is uninterpretable
   without it. The four `huevalue_lowv_*` rows are a separate sub-corpus with wrong `value_in`
   labels (true inputs 20/36/52/68/84/100) and the wrong flat field.
10. **The surround question is OPEN.** The battery intended to bound the dither-bleed term and its
    readout is broken, so every dense-grid cell value still carries an unquantified surround term of
    unknown size. Any per-cell claim inherits that.
11. **Direction, not magnitude, and never absolute colour** (unchanged from the standing rules).
    Nothing in this analysis licenses touching `SPECTRA6_DITHER_PALETTE`.

## What I could not resolve
- **The size of the surround / dither-bleed term.** The one target built to measure it returns a
  registration artefact. It needs a readout with margin (middle third, not middle half) — cheap, but
  it is a pipeline change, not an analysis.
- **Which alignment is physically true.** `primaries` says dx +22, `tonefine` says +72, `huevalue`
  says −72, on a rig that did not move. Two falsifiable criteria both prefer each target's *own*
  consensus, which is not a coherent physical picture — it means the residual distortion is
  target-dependent (i.e. not global), and this corpus cannot separate that from cell-pitch aliasing.
- **Whether the panel itself is non-monotone anywhere.** At the best available alignment the ramp
  still reads 19.2/25. Some of the remaining 5.8 could be panel; nothing here can attribute it.
- **The `huevalue` alignment lottery factor (1.45) is transferred, not measured.** `huevalue` has no
  bit-identical replicate group, so its σ carries that assumption. If a later agent needs a tight
  `huevalue` bar, re-derive the 54 rows under a fixed prior and re-measure.
- **Why 48 in-run rows carry a global alignment prior at all.** The shipped `cmd_run` sets
  `args.prior = None`. The corpus was captured by an earlier build; the run is not replayable from
  the current code, which is why re-derivation rather than the in-run numbers is the answer.

## STATUS: COMPLETE
