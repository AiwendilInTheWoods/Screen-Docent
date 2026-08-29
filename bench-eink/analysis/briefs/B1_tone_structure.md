# B1 — Tone and structure   [Opus]

**Read `briefs/STANDING_RULES.md` first, then `A1_integrity.md` and `error_bars.json`.**
A1's error bars are binding: report nothing below them as a finding.

Targets you own: `tonefine` (48 conditions), `edges` (4), `linepairs` (4), `resample` (2), `surround` (2).
**Do not touch `huevalue`, `inkmix` or `primaries` — B2 owns those.**

## Outputs — yours alone
`bench-eink/analysis/B1_tone_structure.md` (incremental) and `bench-eink/analysis/B1_findings.json`.

## The design you are analysing
A central composite over four levers: white-point (0 / 0.64 / 0.75 / 0.88 / 1.0), gamma (1.0 / 1.4 /
1.8 / 2.2), chroma-gamma (1.0 / 1.5 / 2.0 / 2.5), saturation (0.7 / 0.85 / 1.0 / 1.15 / 1.3).
Axial points give curvature; a full 2^4 of corners gives main effects and two-factor interactions
UNCONFOUNDED; 3 centre replicates give pure error. Run order randomised.

## Tasks

### 1. The wp x gamma response surface
Fit `collapsed_step_pairs` (tone detail destroyed) and `grain_peak` (dither texture) against
white-point and gamma. Report main effects AND the interaction. State which effects clear A1's bars.
Preliminary read from the raw data, to be confirmed or overturned: collapse falls as either lever is
applied (7 -> 0-1) while grain climbs steeply with gamma (30 -> 63 -> 98).

### 2. The detail-versus-grain trade — quantify it
This is the trade a human judge described qualitatively on a bronze statue ("trading less grain on the
statue for more grain on the background"). Give it numbers: how much grain does each unit of recovered
detail cost, and is there a knee? **This is the single most decision-relevant output of your analysis** —
it is what tells Pieria what to ship.

### 3. Do the chroma and saturation levers touch a NEUTRAL ramp?
They must not — saturation cannot change a grey. A1 measured the residual; you have more conditions.
Confirm across the full lever range and report any value where it breaks down.

### 4. Structure
- `edges`: error-diffusion smear. FS pushes error right and down, so trailing edges should carry a
  residue leading edges do not. Is the asymmetry above the floor? Does white-point change it?
- `linepairs`: detail retention vs period (8-48 px), 3 orientations, 2 contrasts. The DIAGONAL is the
  one that matters — FS is direction-biased. Note the readout normalises against the coarsest period
  of the same orientation and contrast.
- `resample`: is texture lost to the resampler before the dither sees it, or to the panel?
- `surround`: does an identical input measure the same in 25 different surrounds? If not, every
  dithered grid target in this corpus carries an unquantified surround term. Digitally this was ~0.1/255
  on cell means; confirm on glass.

### 5. What you could not resolve
Explicit section. Include the alignment wobble's effect on your numbers.

Close with `## STATUS: COMPLETE`.
