# Analysis run — state and resume point

**Corpus:** 119 conditions, 131 raws, commit `1063f81`. Rig torn down 2026-08-29.
**Plan:** `/home/josh/.claude/plans/good-morning-get-eager-puffin.md`
**Standing rules (every agent):** `bench-eink/analysis/briefs/STANDING_RULES.md`

## ⚠️ RULE: "RUNNING" MEANS LAUNCHED, NOT INTENDED
A phase may only be marked RUNNING once the agent has actually been dispatched and returned an id.
On 2026-08-29 Phase B was marked RUNNING, committed, and reported to Josh as running — while no agent
had been launched at all. It sat idle ~40 min and the state file asserted a falsehood the whole time.
This is the same failure mode as the instrument defects this project keeps finding: a record that
could only say "yes". Record evidence, not intent.

## How to resume after a session limit
1. Read this file. 2. Read the last completed phase's report. 3. Launch the next phase using its brief
in `bench-eink/analysis/briefs/`. The briefs are self-contained — no conversation context is required.
An output file without a trailing `## STATUS: COMPLETE` means that agent died mid-run; its partial
findings are usable, re-run it.

## Phases

| # | agent | model | brief | output | status |
|---|---|---|---|---|---|
| A | integrity | Opus | `briefs/A1_integrity.md` | `A1_integrity.md`, `error_bars.json`, `A1_rederived.jsonl` | **COMPLETE** |
| B1 | tone-structure | Opus | `briefs/B1_tone_structure.md` | `B1_tone_structure.md` | **RUNNING** (launched 17:10) |
| B2 | colour-gamut | Opus | `briefs/B2_colour_gamut.md` | `B2_colour_gamut.md` | **RUNNING** (launched 17:10) |
| B3 | normalise-export | Sonnet | `briefs/B3_export.md` | `export/` | **RUNNING** (launched 17:10) |
| C | skeptic | Fable | `briefs/C1_skeptic.md` | `C1_verdicts.md` | pending |
| D | findings | Opus | `briefs/D1_findings.md` | `docs/eink-findings-2026-08-29.md` | pending |
| E1 | project-skill | Opus | `briefs/E1_project_skill.md` | `.claude/skills/eink-panel/` | pending |
| E2 | community-skill | Opus | `briefs/E2_community_skill.md` | `.claude/skills/spectra6-field-guide/` | pending |
| F | final-review | Fable | `briefs/F1_review.md` | `F1_review.md` | pending |

## Gate verdicts

### Gate A — PASSED, with three consequences that bind every later phase
1. **Use `A1_rederived.jsonl`, NOT `panel_profile.jsonl`.** The shipped file mixes two alignment
   regimes (48/119 rows carry a stale global prior) and marks two blown-affine rows `ok: true`
   (verified: `huevalue_wp0.75_g1.0_k2.0_s1.0_hf0.5` has gain 255000).
2. **The published ~16/255 floor is SUPERSEDED** — it was itself measured through a defect. Use the
   per-metric bars in `error_bars.json`. A single tonefine step needs 26/255 to be a finding; a mean
   over 26 steps needs 11.
3. ⚠️ **RANDOMISATION ONLY PARTIALLY HELD.** Rows 3-47 ran in literal design order with white-point as
   the OUTER loop, so wp is aliased with capture time at r = +0.883 there. Rows 48-112 are properly
   randomised (max |r| = 0.118). **Any white-point finding must be established on rows 48-112**, where
   every wp level was re-measured under randomisation. Gamma, chroma and saturation are clean throughout.

**Instrument defect #8, found by A1 and provable:** `read_panel` returns the ALIGNED image but the
strip readout samples nominal coordinates on it, so the affine's own anchors — which must read 0 and
255 by construction — read black [61.6, 52.5, 37.3] and white [226, 235, 230]. That invalidates
`strip`, `field_vs_strip`, `worst_disagreement` and `linearity_error` in both shipped files. Same
signature as the other seven: a check that could only pass.

## Human judgements banked alongside this run
Round 4 (2026-08-29): 12 blinded three-level white-point judgements on OIL PAINTINGS.
Result: wp 0.64 x3, 0.76 x2, **0.88 x7**, mean 0.800 — against yesterday's mixed-class mean of 0.727
and the shipped constant of 0.75. n goes 11 -> 23. Anchoring tested by re-fitting without the three
prior-exposure works: n=9, mean 0.813, i.e. FURTHER from 0.75, so anchoring is not the explanation.
⚠️ Not a clean comparison — today was blinded and lit at 5000K D50, yesterday's protocol was neither.
Files: `bench-eink/wp3_labels.jsonl`, `wp3_round2_blinding.json`, `wp3_round2_notes.json`.

## NEXT ACTION
Phase B running (B1, B2, B3 in parallel). When all three end with `## STATUS: COMPLETE`, read them,
check their claims against `error_bars.json`, record a gate verdict, then launch C (skeptic, Fable)
using `briefs/C1_skeptic.md`.
