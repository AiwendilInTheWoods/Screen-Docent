# B1 — Tone and structure

**Agent B1.** Corpus commit `1063f81`. Source of truth: `bench-eink/analysis/A1_rederived.jsonl`
(119 rows). Error bars: `bench-eink/analysis/error_bars.json` (binding).
Units: **camera-RGB normalised to this panel's own black = 0 / white = 255. NOT sRGB.**

Targets owned here: `tonefine` (48), `edges` (4), `linepairs` (4), `resample` (2), `surround` (2).
`huevalue`, `inkmix`, `primaries` belong to B2 and are not touched.

> Written incrementally. Absence of a closing `## STATUS: COMPLETE` line means the run died and
> everything above the cut still stands.

Cached intermediates: `B1_tonefine_metrics.json` (48 rows x 22 derived fields),
`B1_findings.json` (machine-readable findings).

---

## 0. Three things established before any effect was fitted

### 0a. `--white-point 0` and `--white-point 1.0` are the SAME RENDER — and they measure the same

`eink_bench.py` applies white-point as `im.point([min(255, round(i*wp))])` **only when `wp > 0`**.
So `wp=0` means *lever off* — it is not a white point of zero — and `wp=1.0` is the identity LUT.
**`wp0` and `wp1.0` are bit-identical bitmaps.** The corpus contains both (`wp0_g1.4` x2, `wp1.0_g1.4`
x1), which makes this a null check the data can fail. It does not fail:

| metric | `wp0_g1.4` (n=2) | `wp1.0_g1.4` (n=1) | Δ | 95% MDD |
|---|---|---|---|---|
| `collapsed_step_pairs` | 4.50 | 4.00 | −0.50 | 3.8 |
| `grain_peak` | 70.19 | 71.32 | +1.13 | 30 |
| mean `grain` (26 steps) | 17.36 | 18.96 | +1.59 | 6.5 |
| mean `out_lum` (26 steps) | 172.20 | 178.07 | +5.87 | 11 |
| `ramp_span` | 225.74 | 233.04 | +7.30 | 17 |
| `max_chroma_on_neutral_axis` | 37.61 | 29.29 | −8.32 | 10.6 |

Every difference is inside the bar. **Consequence, and it is used throughout below: the white-point
axis is not five levels but four —** `1.0` (no compression, = `wp0`) → `0.88` → `0.75` → `0.64`.
Pooling `wp0` with `wp1.0` gives the uncompressed corner **three** rows instead of one, and it is the
only reason the "no compression" end of the surface has any replication at all.

### 0b. The pure-error bars reproduce independently

Re-deriving A1's pooled within-identical-render-group σ from my own metric table (groups = every
`(wp_eff, gamma)` cell with n≥3, since saturation and chroma-gamma are exact no-ops on a neutral
target; 32 df):

| metric | B1 σ | A1 σ | B1 MDD | A1 MDD |
|---|---|---|---|---|
| `collapsed_step_pairs` | 1.33 | 1.37 | 3.7 | 3.8 |
| `grain_peak` | 10.40 | 10.74 | 29.1 | 30.1 |
| mean `grain` | 2.29 | 2.32 | 6.4 | 6.5 |
| mean `out_lum` | 4.08 | 4.09 | 11.4 | 11.5 |
| `ramp_span` | 5.83 | 5.91 | 16.3 | 16.6 |
| monotone pairs | 1.72 | 1.75 | 4.8 | 4.9 |
| `max_chroma_on_neutral_axis` | 3.92 | 3.78 | 11.0 | 10.6 |

Agreement to <5% on every metric. **A1's bars are adopted as-is.** Where a comparison is between two
*cell means* rather than two single rows the bar shrinks to `2.8·σ·√(1/n₁+1/n₂)`, and that smaller
bar is quoted explicitly wherever it is used.

### 0c. `collapsed_step_pairs` has a FLOOR of ≈2.6 that is not tone collapse

The `tonefine` ramp is input **100 → 200 in 26 steps** (`readout_tonefine(lo=100, hi=200)`), and
`collapsed_step_pairs` counts adjacent pairs with |Δ out_lum| < 2.0 out of 25. Rendering the digital
ramp through the real lever chain shows how many pairs are collapsed **before the panel ever sees
it**:

| wp_eff | γ | digital ramp | steps above the 163 white-ink ceiling | digital collapsed pairs |
|---|---|---|---|---|
| 1.0 | 1.0 | 100 → 200 | **10 / 26** | 0 |
| 1.0 | 1.4 | 69 → 181 | **4** | 0 |
| 1.0 | 1.8 | 47 → 165 | 1 | 0 |
| 0.88 | 1.0 | 88 → 176 | **4** | 0 |
| 0.88 | 1.4/1.8 | 57→152 / 38→131 | 0 | 0 |
| 0.75 | 1.0…2.2 | 75→150 … 17→79 | 0 | 0–1 |
| 0.64 | 1.0…1.8 | 64→128 … 21→74 | 0 | 0–2 |

Measured `collapsed_step_pairs` in the ten cells where the digital ramp has **zero** steps above the
ink ceiling ranges 2.00–4.60, mean **2.63**. That is the floor: with per-step σ = 9.2 and a true step
of ~8/255, adjacent measured means land within 2.0 of each other by chance a few times per ramp.

> **Rule used from here on: only excursions of `collapsed_step_pairs` ABOVE ≈2.6 are highlight
> collapse.** A reading of 2–3 is the instrument, not the render. Reporting "collapse fell from 7 to
> 1" as a 6-pair recovery overstates it: the recoverable range is 7 → 2.6, i.e. **≈4.4 pairs**.

### 0d. NEW instrument defect (#9): the tonefine grid's last two columns read the SURROUND, not the patch

Found while profiling the ramp step-by-step, and it is the reason several headline numbers below
differ from the brief's preliminary read.

`readout_tonefine` samples a **13 x 2** grid (26 steps, left→right then a second row). Mean measured
`out_lum` by step index, over all 48 rows:

```
step   ...  9     10     11     12  |  13     14   ...  22     23     24     25
lum         ...   90.4  212.6  235.9 | 123.0  ...        190.0  180.4  238.2  230.8
```

A ramp that is monotone by construction **cannot** run 90 → 213 → 236 → 123. Steps **11, 12, 24, 25**
— the last two columns of each grid row — read near paper-white in most conditions
(`out_lum > 240` in 23/48, 33/48, 36/48, 30/48 rows respectively) while their neighbours do not.
The deviation from a 5-step rolling median is 91 and 101 units at steps 11 and 12 against 0–16
everywhere else, and step 12 / step 25 correlate with the row's alignment `dx` at r = −0.55 / −0.66.
The sample window has slid off the rightmost patches onto the white field beside the grid.

**What this contaminates in the shipped readouts:**

| field | damage |
|---|---|
| `ramp_span` | **47 of 48 rows** take their maximum from step 11, 12, 24 or 25. Unusable. |
| `grain_peak` | 15 of 48 rows take their peak from step 24/25 and 9 more from step 0. ~50% contaminated. |
| `collapsed_step_pairs` | pairs (11,12) and (24,25) are two adjacent near-white cells, so they read as collapsed whatever the render did. Inflates the count, and inflates it MOST in the conditions that genuinely have bright ramps. |
| per-step `out_lum` / `grain` at 11,12,24,25 | invalid |

**Fix applied throughout this report: steps 11, 12, 24 and 25 are dropped.** All metrics prefixed
`c_` are computed on the surviving **22 steps** as two contiguous runs — **LOWER** = steps 0–10
(input 100–140) and **UPPER** = steps 13–23 (input 152–192) — giving 20 adjacent pairs. This is not
cosmetic: it halves the noise.

| σ | shipped 26 steps | clean 22 steps |
|---|---|---|
| mean `grain` | 2.29 | **1.42** |
| `grain_peak` | 10.40 | **8.43** |
| collapsed pairs | 1.33 | **1.28** (of 20, not 25) |
| monotone pairs | 1.72 | **1.31** |
| mean `out_lum` | 4.08 | **3.49** |

Splitting the ramp into its LOWER and UPPER runs is better still, because it turns the headline
question into a **within-row paired contrast**:

| clean metric | σ | 95% MDD |
|---|---|---|
| `coll_lo` — collapsed pairs in the lower run (of 10) | 0.91 | 2.54 |
| `coll_hi` — collapsed pairs in the upper run (of 10) | 0.86 | 2.41 |
| `coll_excess` = `coll_hi − coll_lo` | 1.22 | 3.41 |
| `grain_lo` / `grain_hi` — mean grain per run | 2.64 / **1.11** | 7.41 / **3.10** |
| `c_med_step` — median &#124;Δ out_lum&#124; per adjacent pair | 1.62 | 4.52 |

`grain_hi` at σ = 1.11 is the most precise instrument in the whole corpus, and it happens to measure
exactly the thing the human judge complained about: **grain in the bright parts of the picture.**

Cached: `B1_tonefine_clean.json` (48 rows, clean + split metrics).

---

## 1. The wp x gamma response surface

### 1a. The surface (cell means; `wp_eff` pools `wp0` with `wp1.0` per §0a)

| wp_eff | γ | n | `coll_hi` | `coll_lo` | `coll_excess` | `c_med_step` | `grain_hi` | `grain_lo` | mean grain | mean lum |
|---|---|---|---|---|---|---|---|---|---|---|
| **1.0** | 1.0 | 1 | **5.00** | 0.00 | **+5.00** | 4.24 | **3.27** | 14.76 | 9.02 | 209.2 |
| 1.0 | 1.4 | 3 | 2.00 | 0.67 | +1.33 | 8.68 | 6.75 | 33.22 | 19.98 | 161.8 |
| 1.0 | 1.8 | 1 | 0.00 | 0.00 | 0.00 | 10.52 | 14.58 | 45.87 | 30.22 | 124.4 |
| **0.88** | 1.0 | 5 | **3.80** | 1.00 | **+2.80** | 5.27 | 5.11 | 23.02 | 14.06 | 191.5 |
| 0.88 | 1.4 | 2 | 1.00 | 0.50 | +0.50 | 9.74 | 15.36 | 40.21 | 27.78 | 129.4 |
| 0.88 | 1.8 | 5 | 0.00 | 2.20 | −2.20 | 10.57 | 23.83 | 49.99 | 36.91 | 97.0 |
| **0.75** | 1.0 | 5 | **0.40** | 2.20 | **−1.80** | 7.44 | 12.55 | 31.62 | 22.09 | 152.5 |
| 0.75 | 1.4 | 11 | 0.82 | 1.64 | −0.82 | 11.29 | 25.19 | 49.55 | 37.37 | 95.2 |
| 0.75 | 1.8 | 2 | 0.00 | 2.00 | −2.00 | 11.79 | 29.02 | 43.87 | 36.44 | 61.4 |
| 0.75 | 2.2 | 1 | 0.00 | 3.00 | −3.00 | 13.15 | 29.98 | 39.85 | 34.91 | 43.6 |
| **0.64** | 1.0 | 5 | **1.40** | 1.20 | **+0.20** | 8.88 | 18.25 | 41.90 | 30.07 | 117.1 |
| 0.64 | 1.4 | 2 | 1.00 | 1.00 | 0.00 | 9.75 | 24.97 | 39.23 | 32.10 | 65.9 |
| 0.64 | 1.8 | 5 | 0.80 | **3.80** | −3.00 | 9.81 | 21.43 | 27.03 | 24.23 | 39.5 |

### 1b. Main effects and interaction, and which clear the bars

OLS on the raw rows with both levers mean-centred, `y ~ wp + γ + wp:γ`; p by 6 000-shuffle
permutation of the response. "over-range" is the fitted change across the observed lever range
(wp 0.64→1.0, γ 1.0→2.2) and is what must be compared with the MDD. **Every row is refitted on
block-2 rows alone** (rows 48–112, the properly randomised block) because A1 established that a
white-point claim from block 1 carries an illumination-drift term.

| response | lever | over-range (all 48) | p | over-range (block 2, n=33) | p | 95% MDD | verdict |
|---|---|---|---|---|---|---|---|
| `coll_hi` | **wp** | +1.42 | 0.0033 | +1.61 | 0.0048 | 2.41 | real, **confirmed randomised** |
| `coll_hi` | **γ** | −2.75 | 0.0002 | −3.51 | 0.0002 | 2.41 | real, clears |
| `coll_hi` | **wp:γ** | −7.55 | 0.0002 | −8.61 | 0.0002 | 2.41 | **real and large** |
| `coll_lo` | wp | −1.64 | 0.0012 | −1.53 | 0.0145 | 2.54 | sign confirmed, size below bar |
| `coll_lo` | γ | +1.73 | 0.0018 | +2.30 | 0.0017 | 2.54 | sign confirmed, size at bar |
| `coll_excess` | wp | +3.06 | 0.0002 | +3.14 | 0.0010 | 3.41 | confirmed; cell contrast clears (below) |
| `coll_excess` | γ | −4.48 | 0.0002 | −5.81 | 0.0002 | 3.41 | **clears** |
| `coll_excess` | wp:γ | −4.66 | 0.069 | −3.73 | 0.234 | 3.41 | **not resolved** |
| `grain_hi` | **wp** | −13.52 | 0.0002 | −11.71 | 0.0003 | 3.10 | **clears 4x** |
| `grain_hi` | **γ** | +18.48 | 0.0002 | +17.77 | 0.0002 | 3.10 | **clears 6x** |
| `grain_hi` | wp:γ | +24.54 | 0.0010 | +33.82 | 0.0013 | 3.10 | **clears** |
| mean grain | wp | −8.91 | 0.0010 | −5.53 | 0.101 | 3.98 | **weakens when randomised** — use `grain_hi` |
| mean grain | γ | +16.73 | 0.0002 | +14.47 | 0.0007 | 3.98 | clears |
| mean grain | wp:γ | +53.45 | 0.0002 | +65.98 | 0.0002 | 3.98 | clears |
| `c_grain_peak` | wp | −2.40 | 0.72 | +5.86 | 0.52 | 23.6 | **nothing** |
| `c_grain_peak` | γ | +17.14 | 0.028 | +12.75 | 0.199 | 23.6 | **below bar — do not report** |
| `c_med_step` | wp | −2.07 | 0.025 | −1.84 | 0.165 | 4.52 | below bar |
| `c_med_step` | γ | +5.66 | 0.0002 | +5.52 | 0.0010 | 4.52 | clears |
| mean lum | wp | +93.19 | 0.0002 | +95.94 | 0.0002 | 9.76 | clears 10x |
| mean lum | γ | −127.53 | 0.0002 | −125.23 | 0.0002 | 9.76 | clears 13x |
| monotone | wp | +4.84 | 0.0002 | +5.74 | 0.0002 | 3.67 | clears |
| monotone | γ | −5.83 | 0.0002 | −6.67 | 0.0002 | 3.67 | clears |

### 1c. What the surface says — and where the brief's preliminary read was wrong

1. **Collapse is TWO opposite phenomena and the shipped metric adds them together.** `coll_hi` and
   `coll_lo` move in **opposite** directions under both levers (wp: +1.42 vs −1.64; γ: −2.75 vs
   +1.73, all confirmed on block 2). Compression and gamma cure **highlight** collapse and cause
   **shadow** collapse. `collapsed_step_pairs`, being their sum, therefore shows almost no main
   effect at all — only the interaction survives (over-range −10.4, p = 0.0002) — and the brief's
   "collapse falls as either lever is applied, 7 → 0-1" is an artefact of reading a single row of the
   contaminated 26-step metric. Corrected: it falls **5.0 → 0.4 in the highlights** while rising
   **0.0 → 2.2–3.8 in the shadows**.
2. **The wp x γ interaction on `coll_hi` is the single largest structured effect in the tone data**
   (over-range −7.55 vs a bar of 2.41; −8.61 on block 2 alone). Its meaning: **white-point only buys
   you anything at low gamma.** At γ = 1.0 the wp ladder runs `coll_hi` 5.00 → 3.80 → 0.40 → 1.40. At
   γ = 1.8 it runs 0.00 → 0.00 → 0.00 → 0.80 — gamma has already pulled the whole ramp below the ink
   ceiling and there is nothing left for white-point to fix. The two levers are **substitutes, not
   complements**, on highlight collapse.
3. **Grain climbs with gamma, exactly as the brief expected — but the honest number is `grain_hi`,
   not `grain_peak`.** `grain_peak` shows **no** main effect of either lever once its contaminated
   steps are removed (p = 0.20–0.72, σ = 8.4). The brief's "30 → 63 → 98" is the contaminated
   `grain_peak` on a single un-randomised row. The effect is real and is carried cleanly by
   `grain_hi`: over-range +18.5 for gamma and −13.5 for white-point, both at p = 0.0002 and both
   confirmed on block 2.
4. **Gamma costs mean luminance at a brutal rate** (−127/255 over γ 1.0→2.2, the largest single
   coefficient in the table). Anything gamma buys, it buys by making the picture darker.

---

## 2. The detail-versus-grain trade — the numbers, and where the knee is

### 2a. The mechanism: grain is a function of WHERE the tone lands, not of which lever put it there

1 056 clean step observations (48 rows x 22 steps), binned by the **digital** render value the dither
actually received (computed from the lever chain, no measurement involved):

| digital level | n | mean `grain` | s.e. |
|---|---|---|---|
| 20–39 | 85 | 32.5 | 2.3 |
| **40–59** | 176 | **43.9** | 2.0 |
| 60–79 | 192 | 40.5 | 1.8 |
| 80–99 | 219 | 30.8 | 1.2 |
| 100–119 | 188 | 18.4 | 0.7 |
| 120–139 | 101 | 12.7 | 0.6 |
| 140–159 | 60 | 5.2 | 0.4 |
| 160–179 | 29 | 2.2 | 0.1 |

A cubic in the digital level alone explains the group structure: **grain peaks at digital ≈ 55 and
decays to ~2 by the white ink at 163.** Tested against the levers directly — regressing mean grain on
the *predicted mid-point of the digital ramp* and its square gives **R² = 0.824** (resid sd 3.68),
against **R² = 0.628** (resid sd 5.41) for the full `wp + γ + wp:γ` model on the same rows. Adding wp
and γ on top of the position terms lifts R² only to 0.877.

> **The levers do not have grain characters of their own.** They have one job — they move content
> down the level axis — and grain is whatever the level axis says at the place they put it. That is
> why they are substitutes (§1c.2) and it is what makes the trade predictable.

This also explains why `grain_lo` at the far corner (wp 0.64, γ 1.8) *falls* to 27.0 from 49.6 at
(0.75, 1.4): that corner has pushed the shadows **past** the grain ridge toward the black ink.
**Grain is not monotone in compression.** Anything sitting above digital ~55 gets grainier as you
compress; anything already below it gets *cleaner*.

### 2b. The trade, priced

All comparisons are cell means with the aggregate bar `2.8·σ·√(1/n₁+1/n₂)` quoted; verdicts are
against that bar.

**The white-point ladder at γ = 1.0** (the ladder Pieria actually has to choose on):

| step | `coll_hi` (highlight detail lost, of 10) | `grain_hi` (grain in the bright half) | mean lum | price |
|---|---|---|---|---|
| wp off (1.0) | **5.00** | **3.27** | 209.2 | — |
| → 0.88 | 3.80 (−1.20, *below bar 2.64*) | 5.11 (+1.83, *below bar 3.40*) | 191.5 | nothing measurable happens |
| → 0.75 | **0.40 (−3.40, bar 1.52 — CLEARS)** | **12.55 (+7.44, bar 1.96 — CLEARS)** | 152.5 | **2.19 grain per pair** |
| → 0.64 | 1.40 (**+1.00**, below bar — no gain, wrong sign) | **18.25 (+5.70, bar 1.96 — CLEARS)** | 117.1 | **grain only. no return.** |
| **net, off → 0.75** | **−4.60 (bar 2.64 — CLEARS)** | **+9.28 (bar 3.40 — CLEARS)** | −56.7 | **2.02 grain per pair** |

> ### THE KNEE IS AT WHITE-POINT 0.75, AND IT IS BRACKETED ON BOTH SIDES BY MEASUREMENTS THAT CLEAR THE BARS.
> - **0.88 is too weak**: it leaves `coll_hi` at 3.80 against 0.40 at wp 0.75 — a 3.40-pair gap
>   against a bar of 1.52, and `coll_excess` +2.80 vs −1.80, a 4.60 gap against a bar of 2.16.
> - **0.64 is pure cost**: it buys **+5.70 units of highlight grain** (bar 1.96, certain) and
>   **−35 units of panel luminance** for a detail change of +1.00 pairs *in the wrong direction* and
>   inside the bar. Every detail metric — `coll_hi`, `coll_lo`, total collapse, `c_med_step`,
>   monotone pairs — is inside the bar between 0.75 and 0.64. This is the check that could have
>   failed and did not: had 0.64 been genuinely better, `c_med_step` and `coll_hi` had the resolution
>   to say so.
>
> **This is an independent confirmation of ADR-093 (`ship a single white-point of ~0.75`) from a
> completely different measurement** — the panel's tone structure rather than the human A/B fit. The
> two lines of evidence share no data and land on the same number.

**The trade rate is linear over the productive segment, not curved.** off→0.75 costs 2.02 units of
highlight grain per recovered highlight pair; 0.88→0.75 costs 2.19. There is **no knee inside the
useful range** — the knee is at its end, where the detail return goes to zero and the grain keeps
coming.

**In the whole-frame currency:** off → 0.75 costs +13.07 mean grain for 4.60 recovered pairs =
**2.84 grain units per pair**, and −56.7/255 of panel luminance = 12.3 luminance per pair.

### 2c. Gamma buys the same detail in a different currency — and it is the worse currency

| route from (wp off, γ 1.0) | `coll_hi` recovered | `grain_hi` cost | `grain_lo` cost | lum cost |
|---|---|---|---|---|
| **white-point → 0.75** | **4.60** (clears) | **+9.28** (clears) = 2.02/pair | +16.87 (clears) = 3.67/pair | −56.7 = 12.3/pair |
| **gamma → 1.4** | **3.00** (clears) | +3.47 (below bar) = 1.16/pair | **+18.46** (clears) = 6.15/pair | −47.4 = 15.8/pair |
| gamma → 1.8 | 5.00 (n=1) | +11.31 | +31.11 | −84.8 |

Gamma is *cheaper in highlight grain* per pair recovered and **1.7x more expensive in shadow grain**
and **1.3x more expensive in luminance**. And it does not finish the job: at (wp off, γ 1.4)
`coll_excess` is still **+1.33**, against **−1.80** at (0.75, γ 1.0) — a 3.13 gap on a bar of 2.49,
i.e. gamma 1.4 leaves measurable highlight collapse that white-point 0.75 has removed. That is
ADR-090's structural claim (gamma preserves endpoints, so the top still clips) reproduced on glass
with an error bar attached.

### 2d. What to ship — (wp 0.75, γ 1.0) Pareto-dominates every other cell tested

| against | `coll_hi` | `grain_hi` | mean grain | mean lum | verdict |
|---|---|---|---|---|---|
| (0.88, γ1.4) | tie | −2.81 **better** | −5.70 better | **+23.1 brighter** | dominated |
| (0.64, γ1.4) | tie | −12.42 **better** | −10.01 better | **+86.6 brighter** | dominated |
| (0.75, γ1.4) | tie | −12.64 **better** | −15.28 better | **+57.3 brighter** | dominated (γ1.4 wins only `c_med_step`, +3.85) |
| (wp off, γ1.8) | tie | −2.03 (tie) | −8.14 better | **+28.1 brighter** | dominated on luminance, tied on grain |

Nothing tested beats (wp 0.75, γ 1.0) on any resolvable axis. **Once the white-point is at 0.75, gamma
above 1.0 is a pure cost on this panel: it buys no measurable highlight recovery
(`coll_hi` 0.40 → 0.82, bar 1.30), costs +12.6 highlight grain and −57 luminance.** Its one genuine
gain is `c_med_step` +3.85 (bar 2.44) — more separation between adjacent tones — bought against
monotone pairs −3.42 (bar 1.98), i.e. it spreads the steps out and simultaneously scrambles their
order. On the neutral ramp that is a wash.

⚠️ **Scope limit.** This is measured on a *neutral* ramp spanning input 100–200. It says nothing
about what gamma does to colour (B2's territory) and nothing about works whose content sits outside
100–200.

---

## 3. What the panel measurement says about the 23 human white-point judgements

`bench-eink/wp3_labels.jsonl` holds 24 three-level judgements (1 excluded by the judge), 12 from
round 3 and 12 fresh blinded round-4 calls on oil paintings, each a best-of-three over
wp {0.64, 0.76, 0.88}. Picks overall: **0.76 x8, 0.88 x8, 0.64 x7** — mean pick **0.765**.

The measured knee (§2b) and the judge's mean land on the same value. But two things in the labels are
worth putting numbers on.

### 3a. Round 4 shifted toward LESS compression, and the panel data says why that is possible

| round | n usable | 0.64 | 0.76 | 0.88 | mean pick |
|---|---|---|---|---|---|
| 3 (mixed corpus) | 11 | 4 | 6 | 1 | **0.727** |
| 4 (oil paintings, blinded) | 12 | 3 | 2 | 7 | **0.800** |

That is a 0.073 shift, the same size as the per-work sd (0.074) ADR-093 recorded as real-but-
unpredicted. It is **not** contradicted by anything I measured — §2a says the optimum depends only on
where a work's tonal mass sits relative to the grain ridge (digital ≈55) and the ink ceiling (163),
and oil paintings sit lower than illustration. **This is an observation for whoever owns the wp
constant, not a re-decision of ADR-093** — a 0.073 shift measured once, unreplicated, on a corpus
chosen for one material class, is exactly the shape of evidence ADR-093 warns against acting on.

### 3b. Work 16 is NOT an anomaly — the grain-vs-level curve predicts it. **Prediction registered before checking.**

Work 16 (round 4, blinded) picked **0.64, the most compressed level, rejecting both lighter levels
for GRAININESS** ("both B and C have the excessive graininess"). Under a naive reading — grain rises
monotonically with compression — that is backwards.

§2a says it is not backwards at all. Grain peaks at digital ≈55 and **falls** below it. A work whose
tonal mass already sits near the ridge under light compression gets *cleaner*, not grainier, when
compressed harder, because compression pushes it past the peak toward the black ink.

> **Registered prediction, made before opening `corpus.json`:** if that mechanism is the explanation,
> work 16 must be **DARK** — well below the round-4 median luminance and with a **small** fraction of
> its area above the ink ceiling. Note that this is the **opposite** of what ADR-092's rule predicts:
> that rule sends works with a *large* fraction above the ceiling (≥0.48) to heavy compression, so
> ADR-092 requires work 16 to be **BRIGHT**. The two mechanisms make opposite, falsifiable
> predictions about the same single work.

**Outcome — the prediction FAILED, and work 16 stays an anomaly.**

Work 16 is **Flaming June** (`symbolism-romance__flaming-june__073d442f.jpg`).

| | value | round-4 median | verdict on the prediction |
|---|---|---|---|
| mean luminance | 90.5 | 112.1 | **darker than median — as predicted, but only 3rd darkest of 12**, and the two works *darker* than it (25 at 67.3, 52 at 82.7) **both picked 0.88** |
| `wash_pct` (fraction above the 163 ink ceiling) | **0.19 %** | 0.035 % | essentially nothing above the ceiling — **so ADR-092's rule is wrong here too**: at 0.19 % ≪ 48 % it prescribes the *light* setting (0.88), and the judge chose 0.64 |

The load-bearing half of the prediction is refuted by my own curve. Modelling each work's luminance
as `N(mean_lum, lum_stddev)` and integrating the measured grain-vs-level curve of §2a over it:

```
work 16   predicted mean grain   wp0.64  36.6      wp0.76  33.9      wp0.88  30.2
```

Work 16's mass sits at digital 58–80 under wp 0.64 — that is **ON the grain ridge, not past it**.
Compressing it harder moves it *toward* the peak. The mechanism predicts 0.88 for work 16, which is
what the judge rejected. Across all 23 judgements an "argmin predicted grain" rule scores **10/23**
against a 8/23 base rate — no better than the constant, and it misses work 16.

**Two candidate explanations I cannot test with the targets I own, recorded for whoever can:**
1. **It may not be luminance grain at all.** Work 16 is the **2nd most chromatic** work in the label
   set (`mean_chroma` 86.5, median 41.5) and the **most chromatically variable** (`chroma_stddev`
   43.3, highest of 23). Everything I measured is luminance grain on a *neutral* ramp, which cannot
   see false colour. On my own neutral ramp the false-chroma metric does fall from 23.1 (wp 0.75) to
   21.1 (wp 0.64) in the direction the judge's complaint would need — but that is **2.0 against a bar
   of 3.4, i.e. below resolution, so my data does not support the story either.** It is B2's chroma
   territory.
2. It may be the `lum_stddev` axis rather than the mean: the 0.64 pickers average `lum_stddev` 45.8
   and the 0.88 pickers 32.8 (n = 7 and 8, no test performed). Not a finding — an observation.

> **Honest verdict: work 16 is a real anomaly that neither ADR-092's feature nor the panel's
> grain-vs-level curve accounts for.** The value of running the check was that it *could* have
> rescued the anomaly and did not, and it independently falsified ADR-092's rule on this work in the
> process.

---

## 4. Do the chroma and saturation levers touch a neutral ramp?  **NO — over the FULL lever range**

### 4a. Digitally: 190 lever combinations, max pixel difference **0**

A1 verified this at s ∈ {0.7, 1.0, 1.3} and k ∈ {1.0, 2.0}. Extended here to **every k and s value
that appears in the corpus** (k ∈ {1.0, 1.5, 2.0, 2.5} x s ∈ {0.7, 0.85, 1.0, 1.15, 1.3}, minus the
identity) crossed with **ten** (wp, γ) cells spanning the whole design — 190 comparisons:

```
WORST max pixel difference on the quantised target, over all 190      0
non-zero cases                                                        0
CAPABILITY CHECK  γ 1.0 vs 1.4   max diff 255,  52.3% of pixels differ
CAPABILITY CHECK  wp off vs 0.64 max diff 255,  49.1% of pixels differ
```

⚠️ **Method note, because I got this wrong first and the wrong version looked like a discovery.**
`target_tonefine` calls `_quantize(img, pre)` — **`pre` is applied BEFORE the dither**. Applying the
lever chain to the *already-dithered* target instead (which is what I did on the first pass) reports
saturation 0.7 changing **41% of the grid-band pixels by up to 68/255**, because a dithered "grey" is
built from coloured inks and `ImageEnhance.Color` desaturates those inks. That number is an artefact
of the test, not a property of the render, and it is precisely the failure `eink_bench.py:866`'s
comment warns about. **The null must be evaluated on the quantised output of a `pre`-fed generator.**

### 4b. On glass: no measured k or s effect clears its bar, on any metric

39 rows in the 7 replicated (wp, γ) cells, `y ~ cell fixed effects + k + s`. "over-range" is the
fitted change across k 1.0→2.5 and s 0.7→1.3:

| metric | over k-range | over s-range | 95% MDD |
|---|---|---|---|
| `coll_hi` | +0.64 | +0.49 | 2.41 |
| `coll_lo` | −0.00 | +0.39 | 2.54 |
| `grain_hi` | +1.12 | −0.74 | 3.10 |
| `grain_lo` | +0.34 | −0.69 | 7.41 |
| mean grain | +0.73 | −0.72 | 3.98 |
| `c_med_step` | +0.40 | −1.25 | 4.52 |
| mean lum | +1.72 | +2.60 | 9.76 |
| monotone | −1.22 | +0.78 | 3.67 |
| mean chroma | −1.73 | +0.92 | 5.35 |
| **`max_chroma_on_neutral_axis`** | **−5.27** (t = −2.8) | +2.47 | **10.73** |

**Every effect is inside its bar.** The largest, `max_chroma` falling 5.3 across the chroma-gamma
range, has a nominal t of −2.8 but is half the MDD, is one of 24 coefficients tested, and is a
max-statistic on a heavy-tailed quantity. **It is not a finding, and it does not break down at any
lever value** — there is no k or s where the residual jumps. **The nulls hold across the full range,
which is what licenses A1's use of the (wp, γ) cells as identical-render replicate groups, and hence
every error bar in this corpus.**

---

## 5. ADDED TASK — the transfer function, black crush, and whether an S-curve helps

Requested mid-run after the black-crush finding on Night Watch and Olympia. Deliverable:
**`bench-eink/analysis/B1_transfer_function.json`**.

### 5a. The shadow-end limit of this corpus, stated plainly first

`target_tonefine` spans **input 100–200**. It contains **no input below 100**, so the corpus cannot
be asked directly about the shadow region of a picture.

**But that is not the same as having no shadow data**, and the distinction matters. The levers ran
*after* the target was generated, so the **digital levels that actually reached the dither** span
**17 → 192**, not 100 → 200:

```
digital-level coverage of the 1056 clean step observations
  17- 19   n=   2   1 (wp,gamma) cell        70- 79   n=  82   9 cells
  20- 29   n=  32   3 cells                  ...
  30- 39   n=  53   5 cells                 150-159   n=  24   4 cells
  40- 49   n=  79   7 cells                 160-169   n=  24   3 cells
  50- 59   n=  97   8 cells                 170-179   n=   5   2 cells
  60- 69   n= 110  10 cells                 180-192   n=   4   1 cell
```

So the corpus **does** measure the panel at digital 17–40 — thinly (3–5 conditions, 87 observations)
but really. What it cannot do is tell you what a *picture's* shadows look like, because it has no
shadow **content**: no shadow-shaped structures, no dark chromatic passages, and no per-work
histograms. **Everything below is about the panel's transfer, never about a painting.**

### 5b. Deliverable 1 — the DIGITAL transfer function. Exact, needs no rig, and it is the one that answers black crush

For each level d, a flat patch is dithered through `SPECTRA6_DITHER_PALETTE` and rendered through
`SPECTRA6_OUTPUT_PALETTE`. No camera, no panel, no alignment, no error bar — it is arithmetic.

| digital in | output lum | **bare black ink** | white ink |
|---|---|---|---|
| 0 | 0.0 | **100 %** | 0 % |
| 10 | 13.1 | **85 %** | 0 % |
| 20 | 23.3 | **73 %** | 0 % |
| 30 | 34.7 | **59 %** | 0 % |
| 40 | 44.0 | **48 %** | 0 % |
| 50 | 55.0 | 35 % | 0 % |
| 60 | 66.1 | 22 % | 0 % |
| 70 | 76.9 | 9 % | 0 % |
| **80** | 87.9 | **0 %** | 2 % |
| 100 | 127.4 | 0 % | 25 % |
| 130 | 187.4 | 0 % | 60 % |
| 160 | 247.1 | 0 % | 95 % |
| **165 and above** | **255.0** | 0 % | **100 %** |

Two hard walls, and they are the same fact seen from both ends — **the palette has one ink below 71
(black at 0) and one above 101 (white at 163):**

- **The ceiling at input 165.** Everything brighter is one flat value. ADR-090, confirmed.
- **The floor is not a wall but a *dilution*.** There is no cliff at the bottom: the mean response is
  **linear all the way down** (d 20 → 23.3, d 40 → 44.0, slope ≈ 1.03). What degrades is *what the
  tone is made of* — below d 80 the render must reach for the black ink, and by d 20 it is 73 %
  bare black. Shadow modelling is starved not because the mean is wrong but because there is nothing
  to model *with*.
- **This is the exact mirror of ADR-091** ("the gamut is luminance-limited because every chromatic
  ink is dark") at the other end: *tone* is ink-limited because only one ink is dark.

⚠️ **The corpus never tested gamma below 1.0** — the axis was 1.0/1.4/1.8/2.2, every value neutral or
darkening. Everything said below about shadow-lifting curves comes from the **digital** transfer,
which does not need the corpus, and is flagged as such.

### 5c. Deliverable 2 — the MEASURED rig transfer function, and its honest accuracy

Binning the 1056 clean observations by digital level (`B1_transfer_function.json`
→ `measured_rig_transfer`):

| digital in | measured out_lum | s.e. | mean grain |
|---|---|---|---|
| 20–29 | **14.5** | 2.3 | 33.9 |
| 30–39 | **15.2** | 1.6 | 31.6 |
| 40–49 | 33.1 | 1.8 | 47.5 |
| 50–59 | 36.9 | 2.5 | 41.0 |
| 60–69 | 59.2 | 2.7 | 39.8 |
| 80–89 | 97.1 | 2.6 | 34.4 |
| 100–109 | 165.0 | 2.2 | 22.5 |
| 120–129 | 197.4 | 1.4 | 14.9 |
| 140–149 | 227.6 | 1.3 | 6.1 |
| 160–169 | 249.2 | 0.7 | 2.1 |

**Is it a genuine transfer function — i.e. does the measured output depend only on the level, and not
on which lever put the content there?** A spline in the digital level **alone** gives
**R² = 0.933** on per-step `out_lum` (resid sd 19.6). Adding wp and gamma as free terms raises R² to
0.955 — **0.2 percentage points.** So yes: to a good approximation the levers have no character of
their own, they only move content along one curve. That is what makes the object reusable.

**Its accuracy is much worse than the per-step σ, and here is why — a second position defect.**
Within a fixed digital bin the *grid row* the step came from shifts the reading by **+26/255**:

```
digital bin      row-0 mean    row-1 mean    row1 - row0
 40- 59              29.1          55.1          +26.0
 60- 79              51.3          83.2          +31.9
 80- 99              96.7         125.2          +28.4
100-119             152.6         179.4          +26.7
120-139             194.2         208.0          +13.8
```

Consistent across five bins with n = 100–200 each, so it is certainly real. **I cannot say whether it
is a spatial gradient the flat field failed to remove, a surround effect, or a condition effect** —
because inside any single condition the ramp is monotone, so row and level are **perfectly
confounded** and the target cannot separate them. Per-condition systematic bias is a further
−10.3 to +13.1. **Quote the measured transfer with a ±26/255 position band and a ±13/255 condition
band.** It is fit for *shape* and for *ranking curves*; it is not fit for absolute prediction.

### 5d. Where the measured and digital curves DISAGREE — and the disagreement is the finding

```
digital level      DIGITAL out_lum      MEASURED out_lum
   20-29                 23-33                14.5
   30-39                 35-44                15.2
   40-49                 44-55                33.1
```

The **digital** curve is linear through the shadow region. The **measured** curve is **flat**:
14.5 → 15.2 across digital 20 → 39. Confirmed *within single conditions*, where the row confound
cannot operate — the slope `dT/dd` fitted on the 11 steps of one grid row:

| condition | digital range covered | fitted slope | r |
|---|---|---|---|
| wp0.75 γ2.2 | 17 → 36 | **−0.29** | −0.13 |
| wp0.75 γ1.8 | 28 → 52 | **−0.07** | −0.04 |
| wp0.64 γ1.8 | 21 → 39 | +0.53 | 0.41 |
| wp0.64 γ1.4 | 37 → 59 | +0.88 | 0.50 |
| wp0.88 γ1.8 | 38 → 69 | +1.54 | 0.73 |
| wp0.75 γ1.4 | 46 → 74 | +1.26 | 0.60 |
| wp0.64 γ1.0 | 64 → 90 | +2.74 | 0.90 |
| wp0.75 γ1.0 | 75 → 105 | +3.50 | 0.94 |
| wp0.88 γ1.0 | 88 → 123 | +3.22 | 0.99 |
| wp1.0 γ1.4 | 69 → 110 | +3.51 | 0.98 |

The slope standard error on 11 points at σ = 9.2 is ≈0.46, so a true slope above ~0.6 would have been
detected. **Below digital ≈40 the rig registers no tone gradient at all; from 40 to 65 it registers
about half; above 65 it registers 2.6–3.5.**

> **What that disagreement is, honestly: I cannot tell whether the shadow flat is the PANEL or the
> CAMERA.** The render demonstrably carries the gradient (the digital curve is linear there), so
> something between the render and the number destroys it. Candidates are the panel's own dark
> response and the camera's toe, and **this corpus cannot separate them** — there is no reflectance
> standard in frame, only the panel's own black and white as affine anchors, and A1 recorded that the
> camera already exaggerates collapse (ADR-090's method note: the camera put highlight collapse at
> input 112 where the render puts it at 159). **Do not quote "the panel crushes everything below
> digital 40" as a panel fact. The honest statement is that the RIG cannot resolve tone below digital
> 40, and the render says there is tone there to resolve.**

### 5e. Can an S-curve do what no power function can?  **Structurally yes — and the arithmetic is not in doubt**

The structural argument needs no data at all:

- A **power function** `y = 255·(x/255)^γ` fixes both endpoints: 0→0 and 255→255. It can move nothing
  off either wall. γ>1 pulls midtones toward the black floor (worse crush, better ceiling); γ<1 pushes
  them toward the white ceiling (better crush, worse clipping). **It always trades one collapse for
  the other.** Confirmed numerically below.
- A **scale** (white-point) multiplies: it pulls the top off the ceiling and *drags the bottom further
  into the black-ink region in the same proportion*. That is precisely the coordinator's measurement.
- Only a map with **two free endpoints** — lift the black point, pull the white point, shape the middle
  — can place *both* ends inside the usable window `[≈40, 163]`. That is levels/S-curve, and **no
  condition in this corpus has that shape**, so everything from here is *prediction from the digital
  transfer*, not measurement.

Evaluated on the **real luminance histograms** of Night Watch and Olympia, through the exact digital
transfer (bare-black fraction, clipping) and the **measured** grain curve of §2a (the cost side, which
only the rig can supply). "midtone slope" is the curve's gradient at input 128:

| curve | midtone slope | Night Watch bare-black % | clip % | pred. grain | Olympia bare-black % | clip % | pred. grain |
|---|---|---|---|---|---|---|---|
| **production today** (wp off, γ1.4) | 1.05 | **78.0** | 0.3 | 19.0 | **38.1** | 0.6 | 16.1 |
| wp 0.75, γ1.0 (§2d pick) | 0.75 | 66.6 | 0.0 | 27.5 | 32.4 | 0.0 | 22.9 |
| wp 0.64, γ1.0 | 0.64 | 70.7 | 0.0 | 25.4 | 34.8 | 0.0 | 26.3 |
| wp 0.75, γ1.4 | 0.79 | 84.6 | 0.0 | 15.0 | 43.9 | 0.0 | 24.1 |
| **γ 0.75 alone** (shadow lift, untested by the corpus) | 0.83 | 36.4 | 1.3 | 37.6 | 16.7 | **39.1** | 24.2 |
| levels, linear → [60, 163] | 0.40 | 8.2 | 0.0 | 38.1 | 3.6 | 0.0 | 25.4 |
| **S-curve → [60, 163]**, slope matched | **0.75** | **16.3** | **0.0** | 40.3 | **7.9** | **0.0** | 25.0 |
| **S-curve → [80, 163]**, slope matched | **0.75** | **0.0** | **0.0** | **34.1** | **0.0** | **0.0** | **20.2** |

Three things fall out, and the second is the one that answers the question:

1. **γ < 1 does exactly what it is accused of.** On Night Watch it cuts bare black 78 → 36 %; on
   Olympia it drives **39.1 % of the picture above the ceiling**. One collapse traded for the other,
   in one table row. It is not a fix.
2. **An S-curve at the SAME midtone slope as wp 0.75 removes both collapses at once.** Night Watch
   bare-black **66.6 % → 0.0 %** and clipping stays at 0.0 %, with midtone contrast unchanged. No
   power-plus-scale curve in the corpus, or constructible, does that — because both of its endpoints
   are free and theirs are not.
3. **The bill is grain, and it is paid at the worst place on the curve.** Lifting shadows out of the
   black-ink region deposits them at digital 60–90, and §2a's measured ridge peaks at **digital 55**.
   Predicted mean grain rises 27.5 → 34.1 on Night Watch (the [60,163] variant is worse still, 40.3
   — it lands almost exactly on the ridge). On **Olympia** the S-curve is a **Pareto improvement** —
   0 % bare black, 0 % clipped **and** grain 20.2 against 22.9 — because its mass is already high
   enough that the lift moves it *past* the ridge rather than onto it.

**And a cost my metrics cannot see, which must be said.** The [80, 163] S-curve maps the darkest
pixel in the picture to digital 80, whose rendered output is ~88/255. **The image then contains no
black at all.** The black ink is the one place this panel is unambiguously good, and refusing to use
it buys shadow *modelling* by spending shadow *depth*. Whether that is the right trade is a judge
question of exactly the kind ADR-084 reserves for the panel, not a measurement question, and nothing
in this corpus can settle it.

### 5f. What this needs that the corpus cannot give — stated as a finding

1. **A shadow target.** `target_tonefine` mirrored to **input 0–100**, so the shadow region is
   measured with its own steps and its own replicates rather than reached sideways through the levers
   by 3–5 conditions. Without it the shadow transfer rests on 87 observations.
2. **A reflectance standard in frame.** Until the camera's toe can be divided out, "the panel crushes
   below digital 40" and "the camera cannot see below digital 40" are the same measurement. A
   neutral step wedge of known reflectance photographed beside the panel separates them; nothing in
   the current rig does.
3. **At least one condition with a non-power curve.** Every corpus condition is scale∘power. The
   specific risk an S-curve carries — that parking a large flat shadow region on the grain ridge at
   digital 55–80 makes Floyd–Steinberg lock into visible periodic worming — is **untestable here**,
   because the one readout built for it (`anisotropy`) is heavy-tailed to max 52 and A1 ruled it
   unusable per step.
4. **A gamma axis that includes values below 1.0.**

> **Honest verdict on the added task: the transfer function is delivered and is genuinely reusable
> (digital: exact; measured: shape-only, ±26/255). The S-curve question is answered in the direction
> it was asked — an S-curve can do what no power function can, and the arithmetic showing it needs no
> rig at all. But the shadow half of the answer rests on a region this corpus reaches only sideways,
> with an unresolvable camera confound, and the grain cost of the shadow lift lands on the worst part
> of the curve. Confirming it needs a new target and a re-shot rig.**

---

## 6. Structure — `edges`, `linepairs`, `resample`, `surround`

Every one of these four has **n = 1 per lever setting and no replicate anywhere in the corpus**, so
**no error bar can be derived for any of them from the data itself.** The method used instead is to
compute each readout on the **DIGITAL render at the same lever settings** — a truth the rig cannot
contaminate — and to treat `measured − digital` as the quantity of interest. Where that residual is
larger than the whole effect being looked for, the readout cannot answer its question.

### 6a. `edges` — **the readout does not measure error diffusion. NEW instrument defect (#10).**

`readout_edges` reports `asymmetry = right_strip − left_strip` and is documented as the FS smear
test ("FS pushes error right and down, so the trailing sides carry a residue the leading sides do
not"). Computed on the **perfectly registered digital render**, with no camera and no panel:

```
block        0      1      2      3      4      5      6      7
bg,fg    30,220 190,60 90,170 200,120 140,175 120,20 150,255 163,100
digital
asymmetry +220.1 -189.0 +147.6  -87.5  +12.9 -100.0  +27.4 -125.0
```

The sign alternates **exactly** with block parity, and each value is exactly
`rendered(fg) − rendered(bg)`:  +220.1 = 255.0 − 34.9;  −189.0 = 66.0 − 255.0;  +147.6 = 255.0 −
107.4; and so on for all eight. **The left strip lands wholly in the background and the right strip
wholly in the foreground.** So `asymmetry` is a *contrast* readout — it reports the block's designed
bg→fg step, whose sign alternates because `target_edges` alternates polarity with `i % 2` — and it
carries no information about diffusion at all. `worst_asymmetry` (253.8, 248.5, 254.3, 211.0 in the
four measured rows) is the largest designed contrast in the target, not a smear.

The `measured − digital` residual is mean +107.5 / +74.6 / +112.9 / −11.9 with sd **127–171/255**
across the four conditions — i.e. even the contrast it *does* report is swamped by sub-cell
registration.

**Direct attempt to answer the question anyway,** from the rectified vault previews (800x600, so
1 preview px = 2 panel px): the 10–90 % transition width of each block's **left** (leading) and
**right** (trailing) inner-rectangle edge, 8 blocks x 4 conditions:

| condition | mean leading width | mean trailing width |
|---|---|---|
| wp off, γ1.0 | 2.7 px | 3.0 px |
| wp 0.75, γ1.0 | 3.3 px | 3.4 px |
| wp off, γ1.8 | 3.4 px | 3.0 px |
| wp 0.75, γ1.8 | 3.5 px | 3.0 px |

Pooled: leading 3.2, trailing 3.1 preview px (≈6 panel px either way). **No asymmetry, and the
quantisation is ±1 preview px = ±2 panel px, which is a third of the transition itself.**

> **Verdict: `edges` is UNUSABLE as shipped, and the asymmetry question is UNANSWERED.** The
> readout needs the two strips placed on the *same polarity* transition — e.g. compare the run-in
> profile on the left of the inner rectangle against the mirrored profile on its right — and the
> capture needs more than 0.86 camera px per panel px to resolve a few-pixel residue. **No
> white-point effect on edge asymmetry can be reported, in either direction.**

### 6b. `linepairs` — **the dither loses nothing; everything measured is the rig**

The decisive number is the digital truth, and it is emphatic:

```
DIGITAL retained (modulation relative to the 48 px period, same orientation and contrast)
                       8px    12px   16px   24px   32px   48px
  horizontal  c=80    0.997  0.988  1.000  0.865  0.988  0.988
  vertical    c=80    1.000  0.796  0.816  0.816  0.816  0.816
  DIAGONAL    c=80    1.000  1.000  1.000  1.000  1.000  1.000
  DIAGONAL    c=180   0.667 -> 1.000 depending on lever setting
```

**Floyd–Steinberg preserves line pairs down to an 8 px period at essentially full modulation, in
every orientation.** And the orientation the brief flagged as the one that matters — the diagonal,
because FS is direction-biased — is the **best**-preserved of the three digitally (1.000 at every
period, against 0.80–0.99 for the vertical). **The predicted diagonal penalty does not exist in the
render.**

Measured on glass, retention collapses:

```
MEASURED retained, contrast 80, wp off / gamma 1.0
                       8px    12px   16px   24px   32px   48px
  horizontal          0.006  0.166  0.401  0.583  0.813  1.000
  vertical            0.000  0.058  0.587  0.889  1.000  0.970
  diagonal            0.000  0.162  0.381  0.651  0.730  1.000
```

Half-modulation falls between **16 and 24 px** period. Since the digital render carries full
modulation at 8 px, **that entire roll-off is the panel + camera + registration chain, not the
dither** — exactly what `readout_linepairs`' own docstring anticipated ("periods below 8 px would
report the camera's MTF rather than the panel's"). It is now measured: the rig's MTF is the limit
from 32 px downward, not the dither's.

⚠️ **Cross-condition comparison of `retained` is unsafe** and I do not make one. The metric normalises
against the 48 px block *of the same condition*, and the levers change that normaliser — at
wp 0.75 / γ1.8 the digital 48 px reference itself falls to 0.79, which inflates every ratio in that
row (its measured 12 px diagonal reads 0.916 against 0.307 at γ1.0, which is the normaliser moving,
not detail appearing). **With one row per setting and a moving normaliser there is no white-point or
gamma finding available here in either direction.**

### 6c. `resample` — the resampler is exonerated; the loss is downstream

```
                        source scale 1   scale 2   scale 4
  DIGITAL   modulation      170.0         170.0     170.0     (wp off AND wp 0.75 — identical)
  MEASURED  wp off          101.3          97.0      85.3
  MEASURED  wp 0.75          97.0          99.3      87.7
```

**The digital modulation is identical to the decimal at all three source scales.** The LANCZOS
downscale in `render_for_epaper`'s fit stage loses nothing on this texture. So the ~41 % modulation
deficit measured on glass (≈100 against 170) is **entirely panel + camera + registration**, and the
diagnosis "the resampler ate it before the dither saw it" is ruled out for texture of this scale.

The one residual: **scale 4 measures ~13 units lower than scale 1** (−16.0 and −9.3 in the two rows,
same sign both times) with the digital reference flat. With n = 2 and no replicates **there is no bar
to test it against**; it is suggestive of an optical rather than algorithmic loss at the finest
rendered texture, and it is not a finding.

### 6d. `surround` — no evidence of a surround term, but the target cannot prove its absence

25 identical centres (input 170) in 25 different surrounds. Digital truth: spread **0.00** at wp off
(input 170 > the 165 ceiling, so all 25 render as flat white) and **8.15** at wp 0.75 (input 170 →
digital 128, genuinely dithered, so the spread is the dither's own stochasticity).

A1 correctly declared the shipped `spread` / `sd` / `centre_out` fields invalid: measured `spread` is
150.5 / 158.3 because it is a max−min over 25 cells of which **2–4 are destroyed by registration**.
The robust replacement:

| | digital spread | measured full spread | measured **IQR** | IQR, 3 worst cells dropped |
|---|---|---|---|---|
| wp off (undithered white) | 0.00 | 150.5 | 8.4 | **4.9** |
| wp 0.75 (dithered) | 8.15 | 158.3 | 14.5 | **14.0** |

4.9/255 on an undithered flat patch sits at the refresh-to-refresh floor (A1: worst 3.5, mean 0.88).
14.0/255 on dithered cells sits *below* the dithered per-cell floor of 9–13 combined with the
digital 8.15. **So there is no surround term above the noise, and the dithered grid targets in this
corpus do not carry a large hidden surround bias.**

⚠️ **But the target cannot establish that, and the reason is a design flaw worth recording.**
`target_surround` assigns its 25 surrounds **in index order**, so surround *value* is perfectly
confounded with grid *position*: the black (0) and near-black (40) surrounds are cells 0 and 1 — the
**top-left corner**, which is exactly where the registration failure lands. Measured
`r(centre_out, surround luminance)` = +0.51 / +0.57, and `r(centre_out, distance from frame centre)`
= −0.29 / −0.36; dropping the 3 worst cells moves them to +0.33 / +0.42 and −0.36 / −0.35. **The two
explanations cannot be separated with this layout and two rows.** Shuffling the surround assignment
across the grid, and taking three replicates, would settle it for the cost of three refreshes.

---

## 7. The alignment wobble — I re-derived the whole `tonefine` block to measure its effect on my numbers

A1 measured that forcing each kind's median alignment as a prior would cut `tonefine`'s σ roughly in
half, but did **not** apply it (`A1_rederived.jsonl` is the standard re-derive). Because the headline
result of this report is a *knee*, and a knee is exactly the kind of claim registration noise could
manufacture, I re-derived all 48 `tonefine` rows from the banked raws with the alignment forced to
`(scale 0.98, dx +72, dy −28)` and re-ran the decisive comparisons.
Cached: **`B1_tonefine_prior_aligned.jsonl`**.

**Pure error, per-search vs forced prior** (clean 22 steps, 32 df):

| metric | searched (A1_rederived) | forced prior |
|---|---|---|
| `coll_hi` | 0.86 | **0.75** |
| `coll_lo` | 0.91 | 0.92 |
| `coll_excess` | 1.22 | **1.18** |
| monotone pairs | 1.31 | **0.84** |
| mean grain | 1.42 | 1.53 |
| `grain_hi` | **1.11** | 1.46 |
| `grain_lo` | 2.64 | **2.22** |
| mean lum | 3.49 | **3.47** |

**The knee is unchanged, and the bracketing is if anything cleaner:**

| step (γ = 1.0) | `coll_hi` | verdict | `grain_hi` | verdict | grain per pair |
|---|---|---|---|---|---|
| wp off → 0.75 | 6.00 → 0.80 (**−5.20**, bar 2.31) | **CLEARS** | 3.49 → 12.50 (**+9.01**, bar 4.47) | **CLEARS** | **1.73** |
| wp 0.88 → 0.75 | 3.00 → 0.80 (**−2.20**, bar 1.33) | **CLEARS** | 6.08 → 12.50 (**+6.42**, bar 2.58) | **CLEARS** | 2.92 |
| wp 0.75 → 0.64 | 0.80 → 1.20 (+0.40, bar 1.33) | **tie** | 12.50 → 18.66 (**+6.16**, bar 2.58) | **CLEARS** | **∞ — grain only** |

Under the better alignment **every detail metric** — `coll_hi`, `coll_lo`, `coll_excess`, `med_step`
and monotone pairs — is still inside the bar between wp 0.75 and 0.64, while all three grain metrics
still clear it. **The knee at 0.75 is not an artefact of registration noise.** The trade rate shifts
from 2.02 to 1.73 units of highlight grain per recovered highlight pair, which is the honest width of
that number: **≈1.7–2.2**.

---

## 8. What I could not resolve

1. **Panel versus camera at the shadow end.** The single biggest gap. The digital render carries a
   linear tone gradient below digital 40; the rig registers none (§5d). With no reflectance standard
   in frame, "the panel crushes below 40" and "the camera cannot see below 40" are the same
   measurement. Everything in §5e about shadow lifting therefore rests on the *digital* transfer,
   which is exact, and not on the measured one.
2. **The +26/255 grid-row term in the transfer function** (§5c). Row and level are perfectly
   confounded inside any one condition because the ramp is monotone, so I cannot say whether it is a
   residual illumination gradient, a surround effect, or a condition effect. It bounds the absolute
   accuracy of the measured transfer function and it is larger than the per-step σ.
3. **Work 16 (Flaming June).** Neither ADR-092's feature (it has 0.19 % above the ceiling and should
   therefore take the *light* setting) nor my grain-vs-level curve (which predicts 0.88 for it)
   explains why the judge picked the heaviest compression **to escape graininess** (§3b). A registered
   prediction failed. The most likely place to look is chroma, which I do not own.
4. **`edges` — the asymmetry question is unanswered**, and the readout as shipped cannot answer it
   (§6a). No statement about error-diffusion smear, or about white-point's effect on it, is available
   from this corpus in either direction.
5. **`linepairs` cross-condition comparison.** One row per setting, and `retained` normalises against
   a 48 px reference that the levers themselves move (§6b). No white-point or gamma effect on detail
   retention can be reported.
6. **`surround`.** Surround *value* is perfectly confounded with grid *position* by the target's own
   index-order layout, and the two rows contain no replicate (§6d). The robust spread is small
   enough that no surround term matters in practice, but the design cannot prove it.
7. **`resample`'s scale-4 deficit** (~13 units, same sign in both rows, digital reference flat) has
   no error bar because there is no replicate (§6c).
8. **Anisotropy / worming.** A1 declared per-step `anisotropy` unusable (median σ 0.07, mean 2.53,
   max 52.1). Since §5e's S-curve proposal parks shadows on the grain ridge — exactly where FS is
   most likely to lock into periodic structure — this is the readout that matters most for the next
   decision and it is the one that does not work.
9. **`grain_peak` and `ramp_span`** are not recoverable from this corpus at all: 47/48 rows take
   `ramp_span`'s maximum from a contaminated end cell, and ~50 % of `grain_peak` values likewise
   (§0d). I report neither.
10. **Everything here is one panel, one camera, one session, one neutral ramp.** No absolute colour
    claim is made and `SPECTRA6_DITHER_PALETTE` is not touched.

---

## 9. Summary — what this tells Pieria to ship

1. **White-point 0.75, gamma 1.0.** It Pareto-dominates every other cell in the design: tied on
   highlight collapse with the best of them, lower highlight grain, and 23–87/255 brighter. The knee
   at 0.75 is bracketed on both sides by measurements that clear the bars and it survives a full
   re-derivation under forced alignment. This confirms **ADR-093** from evidence that shares no data
   with it.
2. **Do not spend gamma on highlight collapse once the white-point is set.** Gamma above 1.0 buys no
   measurable highlight recovery at wp 0.75 and costs +12.6 highlight grain and −57 luminance.
3. **The price of the fix, in the currency the judge actually uses:** **≈1.7–2.2 units of highlight
   grain per recovered highlight step-pair**, over the productive part of the ladder. The rate is
   linear; there is no knee inside it. The knee is at its end, where the return goes to zero.
4. **`_adaptive_gamma` at 1.40 is the worst of both worlds** and the corpus now says so from two
   directions: on the neutral ramp it leaves highlight collapse *and* adds grain, and on the real
   histograms (§5e) it is the highest bare-black condition tested — 78.0 % on Night Watch. ADR-090
   already retired its rationale; this adds the numbers.
5. **The reusable asset is `B1_transfer_function.json`, not any single lever result.** The digital
   half is exact and predicts the effect of any curve offline. Use it, and stop shooting the panel to
   answer questions that are arithmetic.
6. **The next measurement session needs a shadow target and a reflectance standard** before the
   S-curve question can be answered on glass rather than on paper.

## STATUS: COMPLETE
