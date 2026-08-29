# Standing rules — every analysis agent reads this FIRST

Repo: `/home/josh/ai-workspace/Pieria`. Corpus commit: `1063f81`. Python: `.venv/bin/python`.

## The data
- `bench-eink/panel_profile.jsonl` — 119 conditions, one JSON object per line.
  Keys: `cond, kind, flags, ok, patch_residual, gain, offset, align, conditions, readout`.
- `bench-eink/panel_profile_rederived.jsonl` — 113 rows re-derived after the pipeline fixes.
- `bench-eink/vault/raw/*.png` — 131 raw captures. **READ ONLY. NEVER MODIFY OR DELETE.** Irreplaceable:
  the rig is torn down and cannot be rebuilt cheaply.
- `bench-eink/vault/*.jpg` — rectified, corrected, downscaled previews. Look at these; they are how
  three separate bugs were finally found.
- Flat fields: `bench-eink/reference/flat.png` (opening), `flat_close.png` (pre-rotation),
  `flat_final.png` (post-rotation, for the `huevalue_lowv_*` block only).
- Re-derive: `.venv/bin/python -m tools.eink_vault rederive --flat bench-eink/reference/flat.png
  --flat-close bench-eink/reference/flat_close.png --profile <in> --out <out>`

## Context you must read before starting
- `docs/eink-panel-characterisation.md` — the vault index: units, error bars, the seven instrument
  defects already found and fixed. **Do not re-derive those defects; assume the pipeline is correct.**
- `docs/eink-measurement-rig.md` — the rig and its limits.
- `.ai/decision_log.md`, ADR-090 through ADR-093 — the claims this corpus must adjudicate.

## Rules — non-negotiable
1. **UNITS: camera-RGB normalised to THIS panel's own black=0 / white=255. NOT sRGB.** No absolute
   colour claims. **Never propose rewriting `SPECTRA6_DITHER_PALETTE`.** A camera's filters are not
   human vision and over-saturate by construction: directions may be meaningful, magnitudes are not.
2. **Report nothing below the error bar as a finding.** Say "below resolution". Until Phase A sets the
   bars, the working floor is ~16/255 worst / 6.7 mean (refresh-to-refresh).
3. **Every agent writes ONLY to its own output paths.** Never touch another agent's files.
4. **Verify, don't assert.** If a claim can be tested against the data, test it and show the number.
   Register your prediction before you check it where that is possible.
5. **Report what you could not resolve, explicitly.** An honest gap beats a confident guess. A finding
   you could not confirm is a finding about the limits of the data, and belongs in the report.
6. ⚠️ **Beware checks that can only pass.** All seven instrument defects found on capture day shared
   this signature: the camera `lock` verified a value the video stream then discarded; a four-point
   homography reports zero error because it fits four points by construction; `patch_residual` measures
   within-patch uniformity so it read a healthy 2-3 while the corrected image was destroyed. If your
   validation cannot fail, it is not a validation.

## Checkpointing — MANDATORY, the session may hit a limit mid-run
- **Write your output file INCREMENTALLY**, appending findings as you compute them. Do not hold results
  in your head and write once at the end.
- **Close with a literal `## STATUS: COMPLETE` line.** A file lacking it means you died mid-run and your
  partial findings are still usable.
- **Cache expensive intermediates as JSON** next to your report so a re-run need not recompute them.
