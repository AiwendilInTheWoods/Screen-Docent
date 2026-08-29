# Panel findings, 2026-08-29 afternoon — black crush, and the two-ended problem

Recorded because these arrived AFTER the corpus was captured and after the agent briefs were written.
They came from the panel and from digital simulation, with the judge confirming on glass.

## 1. The panel is starved at BOTH ends of the tone range

The same palette fact, viewed twice:

| end | inks available | consequence |
|---|---|---|
| light | ONE above luminance 101 (white, 163) | highlight collapse — ADR-090 / 091 |
| **dark** | **ONE below luminance 71 (black, 0)** | **shadow collapse — NEW, unnamed** |

blue 71 · green 73 · red 101. Everything from 0-71 must be built from black plus dithered specks of
blue and green, so shadow modelling is structurally starved. This is the mirror of ADR-091 and it had
no name until today.

## 2. White-point is the WRONG lever for shadows, and barely a lever at all

Compression multiplies every input by wp, so it pushes shadows DOWN. Measured on The Night Watch,
fraction of the sub-input-60 region rendering as bare black ink:

    wp 1.00 (off)  67.4%      wp 0.88  71.0%      wp 0.75  74.8%      wp 0.64  78.1%

The entire useful span of the lever is ~11 points and the best value is "off". Only FOUR distinct
inks appear in the shadow region at ANY white-point — the lever cannot change what is not there.

⚠️ **PRODUCTION TODAY IS THE WORST CASE.** `_adaptive_gamma` picks 1.40 for that painting and gamma>1
darkens: 85.0% bare black. What ships is worse than anything we tested.

## 3. Gamma below 1.0 lifts shadows strongly — and the corpus never tested it

    gamma      1.0     0.9     0.75    0.6
    shadow->black    70.9%   —      49.5%   29.7%     (Night Watch)

⚠️ **The corpus gamma axis was 1.0 / 1.4 / 1.8 / 2.2. Every value darkens or is neutral.** The
direction that fixes dark paintings is entirely unmeasured. ADR-090 established that gamma cannot fix
HIGHLIGHT collapse and the design followed that downward, quietly assuming the shadow end needed no
lever.

## 4. The lift costs highlights AND pale colour, roughly one for one

On Manet's Olympia — 50,509 pixels that are pale AND chromatic, 9.2% of the work (the rose in her
hair, warmth in the sheets and skin), mean source chroma 27.5/255:

    gamma                  1.0     0.9     0.75    0.6
    rendered pure white   75.1%   82.5%   95.3%   99.9%
    mean surviving chroma  31.6    27.0    10.3     4.2

⚠️ **The cliff is between 0.9 and 0.75, not spread across 1.0-0.6.** A dark-end round should probe
1.0 / 0.92 / 0.85 — levels chosen from the measured response, which is precisely the fix for ADR-093's
recorded lesson that a badly chosen endpoint flatters a bad model.

**Confirmed independently on the panel.** The judge, with no knowledge of these numbers, reported the
attendant's face recovered, the body and sheets lost detail, and the pale pink rose reading "more or
less ecru" — all three predicted effects, including the chroma one.

📏 A flat synthetic swatch said the rose was never there. The real pixels said it was. **The swatch
was not representative of the painting; the judge's eye was the accurate instrument.**

## 5. Why this was invisible for two days — a blind spot in the sampler

Neither The Night Watch nor Olympia is in the 60-work bench corpus. The corpus was built by
farthest-point sampling over a feature set that contains NO measure of shadow structure, so it could
not sample that axis. The `dark-field` class exists but is astrophotography — small bright subjects on
near-black grounds, which is a different problem from a dark painting whose shadows carry modelling.

📏 Same shape as the nine instrument defects: **the selection could only find what its features could
see.** Not a bad decision — an instrument that could not fail.

## 6. Where this points: an S-CURVE, and possibly a single global one

A power function moves everything in one direction, so it MUST trade one collapse for the other. That
is a limitation of the curve family, not of the panel. An S-curve — lift the bottom, compress the top,
leave the midtones — has the freedom to address both ends at once. **Nothing in the 119-condition
corpus has this shape; every condition is a pure power function plus a linear scale.**

If the panel's two shortages are stable properties (they are — they are palette facts, true for every
image), then a single global S-curve may serve the entire library, because it corrects the PANEL, not
the PAINTING. That would make per-work profiles unnecessary.

It also offers a re-reading of ADR-093: a constant beat a per-work fit on cross-validation, and that
may have been right for a reason we misread — the per-work variation may have been the curve family
failing rather than the works genuinely differing.

## 7. The methodology from here (Josh, 2026-08-29)

Replaces A/B/C, which is bounded by whatever levels are offered:
1. Derive a theoretical ideal across ALL levers from the panel response functions.
2. Show the judge the reference and that ideal.
3. Take qualitative feedback as an ERROR SIGNAL, not a selection.
4. **Do NOT immediately re-render.** Find support in the data and recommend a specific correction only
   if the data predicts it.

⚠️ **Guard, and it is not optional:** the proposed correction and its magnitude must be registered
BEFORE checking whether the data supports it. Otherwise "finding support" becomes motivated search —
the judge says "too dark", one goes looking for darkness, and one always finds some. This is the
failure that produced five successive metrics each of which survived only until the next label
arrived. If feedback repeatedly cannot be explained by the model, that is a finding about the MODEL.
