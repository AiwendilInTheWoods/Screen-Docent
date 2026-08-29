# SCHEMA — e-ink panel measurement export (B3, normalise-export)

> Written incrementally; a missing closing `## STATUS: COMPLETE` line at the bottom means this agent
> died mid-run and the export is still partial.

## ⚠️ READ THIS BEFORE USING ANY FILE BELOW (verbatim from the standing rules)

> **UNITS: camera-RGB normalised to THIS panel's own black=0 / white=255. NOT sRGB.** No absolute
> colour claims. **Never propose rewriting `SPECTRA6_DITHER_PALETTE`.** A camera's filters are not
> human vision and over-saturate by construction: directions may be meaningful, magnitudes are not.

Every numeric colour value in every file here (`out_r`/`out_g`/`out_b`, `out_lum`, `chroma`,
`lum_out`, `chroma_out`, ink RGB triplets, etc.) is in this camera-relative 0–255 scale, measured
through one uncalibrated webcam with no ColorChecker reference. It is **not sRGB**, not a colorimeter
reading, and not comparable to any other device's numbers. Use it to compare conditions **within this
corpus** (this render setting vs. that one, this panel region vs. another) — never as an absolute
colour specification.

## Provenance

- **Source of truth: `bench-eink/analysis/A1_rederived.jsonl`** (119 rows), **not**
  `bench-eink/panel_profile.jsonl`. Per foundation agent A1: the shipped `panel_profile.jsonl` mixes
  two alignment regimes across 48 of its 119 rows (a stale global alignment prior applied to part of
  the corpus) and marks two blown-affine rows `ok: true` when they are not usable. `A1_rederived.jsonl`
  is the corrected re-derive and is what every CSV/JSON in this directory was built from.
- Corpus: 119 conditions, single capture session 2026-08-29 13:12:21 → 15:47:05 local, corpus commit
  `1063f81`. Rig has since been torn down.
- Error bars: `bench-eink/analysis/error_bars.json`, produced by A1_integrity. Headline numbers below;
  see that file for the full breakdown (per-metric σ / 95% MDD, basis, randomisation caveats).
  **Report nothing smaller than the relevant MDD as a real difference — compare aggregates, not single
  steps/cells: per-step/per-cell differences up to 143/255 were measured between BIT-IDENTICAL
  renders.**

  | quantity | σ (single) | 95% MDD (single) | σ (aggregate) | 95% MDD (aggregate) |
  |---|---|---|---|---|
  | `tonefine` `out_lum` | 9.22 | 25.8 | 4.09 (mean of 26 steps) | 11.5 |
  | `tonefine` `grain` | 13.05 | 36.5 | 2.32 (mean of 26) | 6.5 |
  | `tonefine` `chroma` | 4.06 | 11.4 | — | — |
  | `huevalue` `chroma_out` (1 cell) | 5.4 | 15.1 | 1.2 (mean of 72 cells) | 3.3 |
  | `huevalue` `lum_out` (1 cell) | 9.0 | 25.2 | — | — |
  | undithered patch (primaries), refresh-to-refresh | 0.88 (mean) / 3.5 (worst) | — | — | — |

  `huevalue` bars carry an extra caveat: they are a measurement of only 4 centre replicates inflated
  by an estimated ×1.45 factor (not itself a direct measurement) — see `error_bars.json:huevalue._basis`.

- **Randomisation held only partially.** Conditions captured as rows 3–47 of the session (a `tonefine`
  and `huevalue` block, literal design order, white-point as the outer loop) carry a white-point ↔
  capture-time confound (r = 0.883). Rows 48–112 are properly randomised. A white-point effect should
  be checked against the randomised block before being trusted. This export does not filter or flag
  rows by this criterion — `capture_time` is exported precisely so a consumer can reconstruct it
  (session order = sort by `capture_date`, `capture_time`).

## Fields marked INVALID by A1 — do not export as trustworthy, and this export does not

Instrument defect #8 (found by A1): the calibration strip is read from the aligned/rectified image at
nominal coordinates, but that image has already been shifted to register the *content*, so the strip
readout samples the wrong patch. The following fields, present in the source JSONL, are **not
included in any CSV in this export** for that reason:

- `readout.strip.*`
- `readout.field_vs_strip.*`
- `readout.worst_disagreement`
- `readout.linearity_error.*` and `readout.worst_linearity_error` (inkmix)
- `readout.spread` / `readout.sd` / `readout.centre_out` **specifically on the two `uniformity` rows**
  (the general `surround.csv` `spread`/`sd`/`centre_out` columns are a *different* target and are not
  affected by this defect)
- `gain` and `offset`, wherever they appear on a re-derived row (stale, carried over from the original
  capture-time record, not recomputed by the re-derive)

`panel_profile_public.json` (item 12 below) is the one exception: it keeps every field verbatim,
including these, because it is meant to be the full corpus of record — but it carries the same warning
in its own `_meta` block, and this document is the canonical place that warning points back to.
**Do not use these fields from `panel_profile_public.json` as measurements; they are preserved for
provenance only.**

## Two condition blocks needing special handling

- **`huevalue_lowv_*` (4 conditions).** Shot after the panel was rotated back to default orientation,
  using a different flat field (`flat_final.png` vs. `flat.png` for the rest of the corpus) and an
  explicit camera gain of 255 (vs. not explicitly logged for the rest of the corpus). Their
  `value_in` labels in the source JSONL are **wrong**: the readout code hard-codes the label set
  `40, 81, 122, 163, 204, 245` regardless of the actual render, but these four conditions were
  rendered with `--v-lo 20 --v-hi 100`, so the true input values are `20, 36, 52, 68, 84, 100` in the
  same row order. `huevalue_cells.csv` exports `value_in` **as recorded in the source** (i.e. the
  wrong labels) because this export does not silently rewrite source data — use the `block` column
  (`lowv` vs. `main`) to identify these rows, and substitute the corrected sequence above by position
  if you need the true input value. **Do not pool `lowv` rows with `main` rows** in any analysis: they
  used a different flat field, gain, and alignment reference.
- **`uniformity@0` / `uniformity@180`.** Not part of any CSV in this export (the B3 brief does not
  request one), but both appear as rows in `conditions.csv` and `panel_profile_public.json`. Per
  `error_bars.json`: `uniformity@180`'s affine registration failed (its `patch_residual` here is 17.99
  vs. 2.18 for its `@0` partner) and it has no valid 180°-rotated partner to compare against;
  `uniformity@0` alone is "not interpretable alone" (no partner to establish rotational uniformity).
  Both are marked `ok: true` in the source regardless — **`ok` is passed through mechanically from the
  source field and must not be read as "this row is trustworthy."**

## Column conventions used throughout

- **Absent lever = its default**, for conditions whose kind actually passes through the render
  pipeline (`tonefine`, `huevalue`, `edges`, `linepairs`, `surround`, `resample`): `white_point=0.0`
  (off), `gamma=1.0`, `chroma_gamma=1.0`, `saturation=1.0`.
- **Blank (empty cell), not a default value, where the lever is genuinely not applicable**: the
  `primaries`, `inkmix`, and `uniformity` targets are captured undithered/without the render pipeline
  (see `tools/eink_target.py`; their capture functions take no render-pipeline argument at all), so
  `white_point`/`gamma`/`chroma_gamma`/`saturation` are blank, not `0.0`/`1.0`, for those rows.
  `isolate`/`v_lo`/`v_hi` only apply to `huevalue` conditions (the only target with those parameters)
  and are blank for every other kind; within `huevalue`, an absent `--isolate` flag is `False` (its
  off default), and absent `--v-lo`/`--v-hi` default to `40`/`245` (the target's documented default
  value range).
- One condition, `huevalue_wp0.75_g1.0_k2.0_s1.0_hf0.5`, additionally passes `--chroma-floor-max 0.5`.
  This lever is **not** one of the columns specified for `conditions.csv` and is not exported as a
  column anywhere; it is visible in the `cond` name suffix (`_hf0.5`) and in the full flag list kept
  in `panel_profile_public.json`.
- Empty cell = the value was absent, `null`, or (for `tonefine` `anisotropy` only, 5 of 1248 rows)
  `NaN` in the source, meaning column variance was ~0 and the ratio underlying `anisotropy` is
  undefined. **No CSV in this export contains the literal string `"nan"` or `"None"`.**

---

## File-by-file

### 1. `conditions.csv` — 119 rows (one per condition; matches the source's 119 conditions exactly)

| column | meaning |
|---|---|
| `cond` | condition identifier, as captured (e.g. `tonefine_wp0.75_g1.4_k1.5_s1.0_rep1`) |
| `kind` | target type: `primaries`, `inkmix`, `tonefine`, `huevalue`, `edges`, `linepairs`, `surround`, `resample`, `uniformity` |
| `white_point` | `--white-point` lever, 0.0–1.0 (0.0 = off); blank if not applicable (see conventions above) |
| `gamma` | `--gamma` lever (tone curve); blank if not applicable |
| `chroma_gamma` | `--chroma-gamma` lever; blank if not applicable |
| `saturation` | `--saturation` lever; blank if not applicable |
| `isolate` | `--isolate` boolean flag; `huevalue` only, blank elsewhere |
| `v_lo` | `--v-lo`, bottom of the value axis swept by `huevalue`; blank elsewhere |
| `v_hi` | `--v-hi`, top of the value axis swept by `huevalue`; blank elsewhere |
| `patch_residual` | within-patch uniformity residual reported by the re-derive, panel-relative units. ⚠️ per the standing rules, this check "can only pass" — it measures internal patch consistency, not correctness of the affine/alignment, and read a healthy 2–3 on at least one row later found to have a destroyed corrected image |
| `align_correlation` | homography fit correlation, `align[0]` in the source |
| `align_scale` | homography scale factor, `align[1]` |
| `align_dx` | homography x-translation (px), `align[2]` |
| `align_dy` | homography y-translation (px), `align[3]` |
| `capture_date` | ISO date of capture |
| `capture_time` | local time of capture (HH:MM:SS); combined with `capture_date` this reconstructs session order, needed to check the randomisation caveat above |
| `geometry_version` | rig/target geometry version tag (all rows are version 3) |
| `flat_field` | flat-field reference image used to correct this capture: `bench-eink/reference/flat.png` (opening, most rows), `flat_close.png` (uniformity rows, pre-rotation), or `flat_final.png` (the 4 `huevalue_lowv_*` rows, post-rotation) |
| `camera_gain` | explicit camera gain setting, only logged for the 4 `huevalue_lowv_*` rows (255); blank for all other rows (gain was not explicitly logged/varied for the rest of the session) |
| `ok` | pass-through of the source's own `ok` flag. **Mechanical pass-through only** — see the `uniformity` caveat above for a case where `ok: true` does not mean trustworthy |

### 2. `tonefine_steps.csv` — 1,248 rows (48 `tonefine` conditions × 26 steps each)

Long format, one row per (condition, tone-ramp step). Columns `cond`, `white_point`, `gamma`,
`chroma_gamma`, `saturation` as in `conditions.csv` (levers apply to every row here — `tonefine` is
always in the render pipeline). Then:

| column | meaning |
|---|---|
| `input_value` | requested input tone level fed to the ramp (0–255 scale, source key `in`) |
| `out_lum` | measured output luminance (camera-relative, 0–255) |
| `out_r`, `out_g`, `out_b` | measured output RGB (camera-relative, 0–255; source key `out_rgb`) |
| `chroma` | measured chroma of the output patch |
| `grain` | measured local graininess (std. dev. of a flat-field-corrected patch) |
| `anisotropy` | row-variance / column-variance of patch texture; ~1.0 = isotropic grain, far from 1.0 = directional dither structure. Empty cell where the source had `NaN` (5 rows total, column variance ~0) |
| `hue_deg` | measured output hue in degrees; empty where the source had `null` (near-neutral patch, hue undefined) |

Use the aggregate error bars in the units caveat table above — a single step's `out_lum` needs a
26/255 difference to be a finding; only a mean over many steps supports an 11/255 claim.

### 3. `huevalue_cells.csv` — 3,888 rows (54 `huevalue` conditions × 6 value-rows × 12 hue-cells)

Long format, one row per (condition, value row, hue cell). 50 of the 54 conditions are the `main`
block (v = 40…245); 4 are the `lowv` block (v = 20…100, see the caveat above).

| column | meaning |
|---|---|
| `cond` | condition identifier |
| `block` | `main` or `lowv` — **keep these separate in any analysis**, see caveat above |
| `white_point`, `gamma`, `chroma_gamma`, `saturation` | render-pipeline levers, as in `conditions.csv` |
| `value_in` | nominal input "value" (HSV-style lightness) for this row of the grid, **as recorded in the source** — wrong for the 4 `lowv` conditions, see caveat above |
| `hue_in_deg` | requested input hue for this cell, 0–330° in 30° steps |
| `chroma_out` | measured output chroma |
| `lum_out` | measured output luminance |
| `hue_out_deg` | measured output hue in degrees; empty where the source had `null` (chroma too low for hue to be meaningful) |

Not exported: `readout.saturation_in` (a fixed 0.55 input saturation baked into every `huevalue`
target — not the `saturation` lever column, which is a separate multiplier applied by the render
pipeline) and `readout.rows[].n_collapsed` (a per-row summary count) — both are in
`panel_profile_public.json` if needed, just not repeated on every one of this file's long-format rows.

### 4. `inkmix_mixtures.csv` — 75 rows (15 ink pairs × 5 ratios); one `inkmix` condition in the whole corpus

Undithered target, no render-pipeline levers apply (captured once, panel-invariant — see
`tools/eink_target.py:target_inkmix`). `cond` is always `inkmix` and is not repeated as a column here,
per the brief's exact column spec.

| column | meaning |
|---|---|
| `ink_a`, `ink_b` | the two pure inks mixed in this tile: `black`, `white`, `red`, `yellow`, `blue`, `green` |
| `fraction_a` | area fraction of `ink_a` in the deterministic checker mix (0.125, 0.25, 0.5, 0.75, 0.875) |
| `out_r`, `out_g`, `out_b` | measured output RGB of the mixed tile |

Not exported: `readout.linearity_error.*` / `worst_linearity_error` — INVALID, see above (its pure-ink
reference comes from the also-invalid calibration strip; per `tools/eink_readout.py`'s own docstring
this was already "provisional" even before defect #8, since strip patches measure differently from
large ink fields by 40–81/255 in the chromatic inks).

### 5. `inkmix_element_sweep.csv` — 15 rows

Same `inkmix` condition, row 5 of its target grid: a fixed 1:1 mix repeated at 5 checker-element sizes
for 3 ink pairs, testing dot pitch / dither fineness. Column order decoded from
`tools/eink_target.py:target_inkmix()` (the element size and ink-pair sequence used to draw the row)
and `tools/eink_readout.py:readout_inkmix()` (which reads the same cells back in the same order) —
not present as labels in the source JSONL itself, which stores only a flat 15-element RGB list.

| column | meaning |
|---|---|
| `cond` | `inkmix` |
| `ink_a`, `ink_b` | the two inks for this group: (`black`,`white`), (`white`,`yellow`), (`black`,`red`), 5 rows each |
| `element_px` | checker-square size in panel px: 1, 2, 4, 8, 16 |
| `fraction_a` | always 0.5 (fixed 1:1 ratio for this sweep) |
| `out_r`, `out_g`, `out_b` | measured output RGB |

### 6. `inkmix_metamers.csv` — 15 rows

Same `inkmix` condition, row 6 of its target grid: 15 specific ink-mixture pairs the renderer predicts
to land on the same colour, testing whether optical mixing is additive. `ink_a`/`ink_b`/`fraction_a`
decoded the same way as above, from the explicit `metamers` list in
`tools/eink_target.py:target_inkmix()`.

| column | meaning |
|---|---|
| `cond` | `inkmix` |
| `pair_index` | 0–14, position in the metamer test row (preserves source order; there is no natural pairing key otherwise) |
| `ink_a`, `ink_b`, `fraction_a` | the mixture recipe for this tile |
| `out_r`, `out_g`, `out_b` | measured output RGB |

### 7. `primaries.csv` — 12 rows (2 conditions × 6 inks)

`primaries#1` and `primaries#2` are two separate refreshes of the same undithered target, 46 s apart —
the pair A1 used to measure refresh-to-refresh repeatability (worst 3.5/255, mean 0.88/255 on this
correctly-sampled field; see `error_bars.json:undithered`).

| column | meaning |
|---|---|
| `cond` | `primaries#1` or `primaries#2` |
| `white_point`, `gamma`, `chroma_gamma`, `saturation` | blank — `primaries` is undithered, no render pipeline applies |
| `ink` | `black`, `white`, `red`, `yellow`, `blue`, `green` |
| `out_r`, `out_g`, `out_b` | measured RGB from `readout.fields` (the large-field measurement, **not** the calibration strip) |

Not exported: `readout.strip.*`, `readout.field_vs_strip.*`, `readout.worst_disagreement` — INVALID,
see above.

### 8. `linepairs.csv` — 144 rows (4 conditions × 36 blocks: 6 periods × 3 orientations × 2 contrasts)

| column | meaning |
|---|---|
| `cond`, `white_point`, `gamma`, `chroma_gamma`, `saturation` | as above |
| `period_px` | line-pair period in panel px: 8, 12, 16, 24, 32, 48 |
| `orientation` | `h` (horizontal), `v` (vertical), `d` (diagonal) |
| `contrast_in` | requested input contrast (grey-level difference), 80 or 180 |
| `modulation_out` | measured output modulation (90th–10th percentile of the profile) |
| `retained` | `modulation_out` normalised against the coarsest period of the same orientation/contrast (1.0 = resolved as well as the coarsest period; 0 = gone). Empty where the source had `null` (reference modulation ~0) |

### 9. `edges.csv` — 32 rows (4 conditions × 8 blocks)

| column | meaning |
|---|---|
| `cond`, `white_point`, `gamma`, `chroma_gamma`, `saturation` | as above |
| `block_index` | 0–7, position in the target's edge-block grid (source capture/read order; the exact grey-level pair per block is set in `tools/eink_target.py:target_edges` but is not itself present in the JSONL readout, so it is not reconstructed here) |
| `left` | mean level just left of the edge |
| `right` | mean level just right of the edge |
| `asymmetry` | `right − left`; tests whether Floyd–Steinberg error-diffusion residue (which propagates right/down) makes the trailing edge measurably different from the leading edge |

### 10. `resample.csv` — 6 rows (2 conditions × 3 source scales)

| column | meaning |
|---|---|
| `cond`, `white_point`, `gamma`, `chroma_gamma`, `saturation` | as above |
| `source_scale` | render-time downscale factor before dithering: 1, 2, 4 |
| `modulation` | measured texture modulation (90th–10th percentile) |
| `sd` | standard deviation of the same patch |

### 11. `surround.csv` — 50 rows (2 conditions × 25 grid positions)

| column | meaning |
|---|---|
| `cond`, `white_point`, `gamma`, `chroma_gamma`, `saturation` | as above |
| `centre_in` | the fixed input level (170) placed in 25 different surrounding contexts |
| `position_index` | 0–24, position in the 5×5 grid of surrounding contexts |
| `centre_out` | measured output level of the identical centre patch at this position |
| `spread` | `max(centre_out) − min(centre_out)` over all 25 positions — repeated on every row of this condition |
| `sd` | standard deviation of `centre_out` over all 25 positions — repeated on every row |

Note: this is the `surround` target (25 different neighbourhoods around one fixed patch) and is
**not** affected by the `uniformity`-row invalidation noted above, which is a different target
entirely (`uniformity@0`/`@180`, ink response at 9 physical panel positions, not exported to any CSV).

### 12. `panel_profile_public.json`

The full 119-condition corpus as one JSON document, `{"_meta": {...}, "conditions": [...]}`, built
directly from `A1_rederived.jsonl` with two mechanical changes:

1. The `device` key (a webcam device-node path, e.g. `/dev/video0` — machine-local, meaningless off
   the capture machine) is removed from every row's `conditions` block. No other key is removed or
   renamed; every `conditions` block is otherwise kept intact, including capture date/time, flat
   field, geometry version, and the per-row units/colour-reference disclaimer — this is deliberate,
   since provenance is the point of shipping the raw corpus alongside the flattened CSVs.
2. 5 `NaN` floats (`tonefine` `anisotropy` where column variance was ~0) are converted to JSON `null`,
   since `NaN` is not valid JSON.

Everything else, **including the fields marked INVALID above**, is preserved verbatim — this file is
the corpus of record, not a cleaned view. Read the INVALID-fields section of this document (and
`error_bars.json`) before using anything from `readout.strip`, `readout.field_vs_strip`,
`readout.worst_disagreement`, `readout.linearity_error`, `readout.worst_linearity_error`, or `gain`/
`offset` in this file.

---

## Verification performed

- **Row counts**, computed directly from `A1_rederived.jsonl` (119 rows total) and matched to CSV
  output:

  | file | rows | how derived |
  |---|---|---|
  | `conditions.csv` | 119 | 1 per condition |
  | `tonefine_steps.csv` | 1,248 | 48 `tonefine` conditions × 26 steps |
  | `huevalue_cells.csv` | 3,888 | 50 `main` conditions × 72 cells + 4 `lowv` conditions × 72 cells |
  | `inkmix_mixtures.csv` | 75 | 15 pairs × 5 ratios |
  | `inkmix_element_sweep.csv` | 15 | 3 pairs × 5 element sizes |
  | `inkmix_metamers.csv` | 15 | 15 defined metamer triples |
  | `primaries.csv` | 12 | 2 conditions × 6 inks |
  | `linepairs.csv` | 144 | 4 conditions × 36 blocks |
  | `edges.csv` | 32 | 4 conditions × 8 blocks |
  | `resample.csv` | 6 | 2 conditions × 3 scales |
  | `surround.csv` | 50 | 2 conditions × 25 positions |
  | `panel_profile_public.json` | 119 conditions | full corpus |

- **No `"nan"`/`"None"` strings**: `grep -rniE '\bnan\b|\bnone\b' *.csv` over every CSV in this
  directory returned no matches.
- **Spot checks, 3+ values per file, traced back to `A1_rederived.jsonl`** (all matched exactly):
  - `conditions.csv`: `primaries#1` (align `[0.793, 0.96, 32, -48]`, residual 1.98, flat field
    `bench-eink/reference/flat.png`); `tonefine_wp0.75_g1.4_k1.5_s1.0_rep1` (levers 0.75/1.4/1.5/1.0,
    align `[0.708, 0.98, 88, -28]`); `huevalue_lowv_wp0.64_g1.0` (v_lo/v_hi 20/100, camera_gain 255,
    flat field `flat_final.png`).
  - `tonefine_steps.csv`: `tonefine_wp0.0_g1.0_k1.0_s1.0`, input_value 104 → out_lum 130.61, out_rgb
    [140.66, 138.51, 112.66], grain 3.53, anisotropy 2.095, hue 56.0.
  - `huevalue_cells.csv`: `huevalue_wp0.0_g1.0_k1.0_s1.0` main block, value_in 40, hue_in 210° →
    chroma_out 29.88, lum_out 42.72, hue_out 227.8; `huevalue_lowv_wp0.0_g1.0` lowv block, value_in 40
    (as-recorded), hue_in 0° → chroma_out 11.63, lum_out 95.71, hue_out 3.3.
  - `inkmix_mixtures.csv`: black+blue at fraction_a 0.5 → [0.0, 14.44, 59.53].
  - `inkmix_element_sweep.csv`: black/white, element_px 1 → [93.5, 103.5, 113.45] (source
    `element_sweep[0]`).
  - `inkmix_metamers.csv`: pair_index 0 (white/black, 0.5) → [117.09, 117.7, 123.48] (source
    `metamer_row[0]`, def `(1, 0, 0.5)`).
  - `primaries.csv`: `primaries#2` red → [231.2, 4.2, 1.6] (source `readout.fields.red`).
  - `linepairs.csv`: period 24, `h`, contrast 80 → modulation_out 52.7, retained 0.583.
  - `edges.csv`: block_index 0 → left 0.71, right 254.48, asymmetry 253.77.
  - `resample.csv`: source_scale 2 → modulation 99.33, sd 38.62.
  - `surround.csv`: position_index 19 → centre_out 224.56.
  - `panel_profile_public.json`: `primaries#1.readout.fields` matches the source object exactly;
    `device` key absent from every row's `conditions` block; the 5 source `NaN` anisotropy values
    (e.g. `tonefine_wp0.88_g1.0_k1.0_s0.7`, step 25) come through as JSON `null`.
- **Every column above appears in this document.** No undocumented column exists in any CSV.

## STATUS: COMPLETE
