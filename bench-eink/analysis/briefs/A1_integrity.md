# A1 — Data integrity and error bars   [Opus]

**Read `bench-eink/analysis/briefs/STANDING_RULES.md` first. All of it applies.**

You are the FOUNDATION agent. Every later phase reports its findings against the error bars you
produce. If you get these wrong, every downstream conclusion and both published skills inherit the
error silently. Nothing you do matters more than being right about what this corpus can and cannot
resolve.

## Outputs — yours alone
- `bench-eink/analysis/A1_integrity.md` — the report. Write it incrementally.
- `bench-eink/analysis/error_bars.json` — machine-readable, consumed by every later agent.

## Tasks

### 1. Re-derivation consistency
Re-derive all rows with time-interpolated flat fields (see STANDING_RULES for the command; use
`--out bench-eink/analysis/A1_rederived.jsonl`, NOT the existing file). Diff against the in-run values
in `panel_profile.jsonl`. Where do they disagree, by how much, and does the disagreement correlate with
anything (target kind, capture time, patch_residual, alignment correlation)? Decide and state plainly:
**which file should downstream agents use**, and why.

### 2. Pure error — the headline number
There are 3 centre replicates per target: `tonefine_wp0.75_g1.4_k1.5_s1.0_rep{1,2,3}` and the huevalue
equivalents. Identical settings, different times. Compute the pure error PER READOUT METRIC — not one
global number. `collapsed_step_pairs`, `grain_peak`, per-step `out_lum`, per-cell `chroma_out`, etc.
each have their own noise. Downstream agents need to know "an effect in grain_peak must exceed X".

Also compute the refresh-to-refresh floor from `primaries#1` vs `primaries#2`.

### 3. Is precision time-dependent?
The daylight fell ~37% across the session; late captures have roughly 1.6x more noise. Run order was
RANDOMISED, so this should be noise and not bias. **Test both halves of that claim:**
- Does readout precision correlate with capture timestamp? (Use the centre replicates and any repeated
  structure you can find.)
- Does any lever value correlate with capture time? If randomisation worked, it should not. **If it
  did not work, say so loudly** — that would mean drift is aliased onto a lever and findings are at risk.
Produce a time-dependent error bar if warranted.

### 4. Null checks — these bound everything
Physics says these must be ~zero. Any residual is measurement error and is an upper bound on trust:
- **Saturation cannot change a neutral ramp.** Compare `tonefine` at s=0.7 / 1.0 / 1.3, same wp/gamma.
- **Chroma-gamma cannot change a neutral ramp.** Same, varying k.
Quantify the residual. If it substantially exceeds the pure error, something systematic remains and you
must say what you think it is.

### 5. Alignment wobble
Tone ramps come back 17-22/25 monotone, not 25/25. A tone ramp is monotone by construction, so every
non-monotone step is measurement error. Quantify it, determine whether it correlates with the `align`
correlation value stored per row, and **determine whether it is fixable offline** — if you can improve
it, say exactly how; if not, fold it into the error bars.

### 6. Which rows should be excluded, if any
State explicitly. Bias toward keeping rows and widening error bars over silently dropping data. If you
exclude anything, list it by `cond` with the reason.

## Report shape
Lead with a table of error bars per metric. Then each task's finding with the number that supports it.
End with **"What downstream agents must assume"** — a short, unambiguous list. Then `## STATUS: COMPLETE`.
