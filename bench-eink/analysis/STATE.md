# Analysis run — state and resume point

**Corpus:** 119 conditions, 131 raws, commit `1063f81`. Rig torn down 2026-08-29.
**Plan:** `/home/josh/.claude/plans/good-morning-get-eager-puffin.md`
**Standing rules (every agent):** `bench-eink/analysis/briefs/STANDING_RULES.md`

## How to resume after a session limit
1. Read this file. 2. Read the last completed phase's report. 3. Launch the next phase using its brief
in `bench-eink/analysis/briefs/`. The briefs are self-contained — no conversation context is required.
An output file without a trailing `## STATUS: COMPLETE` means that agent died mid-run; its partial
findings are usable, re-run it.

## Phases

| # | agent | model | brief | output | status |
|---|---|---|---|---|---|
| A | integrity | Opus | `briefs/A1_integrity.md` | `A1_integrity.md`, `error_bars.json` | **RUNNING** |
| B1 | tone-structure | Opus | `briefs/B1_tone_structure.md` | `B1_tone_structure.md` | pending |
| B2 | colour-gamut | Opus | `briefs/B2_colour_gamut.md` | `B2_colour_gamut.md` | pending |
| B3 | normalise-export | Sonnet | `briefs/B3_export.md` | `export/` | pending |
| C | skeptic | Fable | `briefs/C1_skeptic.md` | `C1_verdicts.md` | pending |
| D | findings | Opus | `briefs/D1_findings.md` | `docs/eink-findings-2026-08-29.md` | pending |
| E1 | project-skill | Opus | `briefs/E1_project_skill.md` | `.claude/skills/eink-panel/` | pending |
| E2 | community-skill | Opus | `briefs/E2_community_skill.md` | `.claude/skills/spectra6-field-guide/` | pending |
| F | final-review | Fable | `briefs/F1_review.md` | `F1_review.md` | pending |

## Gate verdicts
_(recorded by the orchestrator after reading each phase)_

## NEXT ACTION
Phase A is running. When `A1_integrity.md` ends with `## STATUS: COMPLETE`, read it, record a gate
verdict above, then launch B1+B2+B3 in parallel.
