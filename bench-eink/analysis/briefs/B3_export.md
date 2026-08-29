# B3 — Normalise and export   [Sonnet]

**Read `briefs/STANDING_RULES.md` first.** This job is mechanical and tightly specified.
**Do not interpret the data. Do not draw conclusions. Do not editorialise.** Other agents do that.
Your output feeds a PUBLIC artifact, so correctness and clear documentation are the whole job.

## Outputs — yours alone: `bench-eink/analysis/export/`

### 1. `conditions.csv`
One row per condition in `bench-eink/panel_profile.jsonl`. Columns:
`cond, kind, white_point, gamma, chroma_gamma, saturation, isolate, v_lo, v_hi, patch_residual,
align_correlation, align_scale, align_dx, align_dy, capture_date, capture_time, geometry_version,
flat_field, camera_gain, ok`
Parse the levers out of the `flags` list. Absent lever = its default (wp 0.0 = off, gamma 1.0,
chroma_gamma 1.0, saturation 1.0). Blank, not zero, where genuinely not applicable.

### 2. `tonefine_steps.csv`
Long format, one row per (condition, step): `cond, white_point, gamma, chroma_gamma, saturation,
input_value, out_lum, out_r, out_g, out_b, chroma, grain, anisotropy, hue_deg`

### 3. `huevalue_cells.csv`
Long format, one row per (condition, value_row, hue_cell): `cond, block, white_point, gamma,
chroma_gamma, saturation, value_in, hue_in_deg, chroma_out, lum_out, hue_out_deg`
`block` = `main` for the v=40..245 conditions, `lowv` for the `huevalue_lowv_*` conditions.
⚠️ Keep these blocks distinguishable — they used different flat fields and camera gains.

### 4. `inkmix_mixtures.csv`
One row per (ink pair, ratio): `ink_a, ink_b, fraction_a, out_r, out_g, out_b`
Plus `inkmix_element_sweep.csv` and `inkmix_metamers.csv`.

### 5. `primaries.csv`, `linepairs.csv`, `edges.csv`, `resample.csv`, `surround.csv`
Flatten each readout to long format on the same pattern.

### 6. `panel_profile_public.json`
The whole corpus as one nested JSON, minus anything machine-local: strip absolute filesystem paths,
keep relative ones. Keep every `conditions` block — provenance is the point.

### 7. `SCHEMA.md`
Document every file and every column: units, meaning, valid range, how it was derived, and which
target produced it. **Lead with the units caveat verbatim from the standing rules** — anyone reading
these CSVs without it will misuse them. Note the error bars once A1 has published them
(`bench-eink/analysis/error_bars.json`); if that file does not exist yet, say so and point at it.

## Verification before you finish
- Row counts match the source (119 conditions; report the per-target counts you emitted).
- Spot-check 3 values per CSV back against `panel_profile.jsonl` and show the comparison.
- No NaN or `null` written as the string "nan"/"None" — use empty cells.
- Every column in every CSV appears in `SCHEMA.md`. No undocumented columns.

Close with `## STATUS: COMPLETE`.
