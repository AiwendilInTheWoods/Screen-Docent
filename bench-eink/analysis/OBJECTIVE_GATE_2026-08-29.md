# The objective gate — the S-curve cost function fails it, and so does everything built on its terms

**Run:** 2026-08-29 evening, in a Jupyter kernel (`~/notebooks/pieria-objective-gate.ipynb`).
**Runnable form:** `python tools/eink_objective_gate.py --ceiling` — no panel, no rig, no camera.
**What was gated:** `tools/eink_scurve.py`'s `cost()`, against the 23 usable human white-point calls
in `bench-eink/wp3_labels.jsonl`. This is step 2 of `NEXT_SESSION.md`, done before fitting anything.

## 0. The bar in NEXT_SESSION.md was wrong — 34.8%, not 61%

The picks split **0.64 x7 · 0.76 x8 · 0.88 x8** over 23 usable rows (one excluded at the judge's
request). Always guessing the mode scores **8/23 = 34.8%**; chance is 33.3%. `NEXT_SESSION.md` said
61%, which would have failed an objective scoring 45% — a genuine result — as a failure.
📏 A wrong bar is worse than no bar: it is a check that can only return the wrong verdict.

## 1. The incumbent objective is DEGENERATE, and its accuracy hides that

    agreement 7/23 = 30.4%   base rate 34.8%   chance 33.3%
    predicts  {0.64: 23, 0.76: 0, 0.88: 0}     <-- one answer for every work

It is below the base rate *and* below chance, but the number that matters is the distribution:
**`cost()` picks the heaviest compression on 23 of 23 works.** It is a constant function of its input.
Its 7 hits are exactly the 7 works where Josh happened to agree with a constant — it is not choosing,
it is guessing one answer and collecting the coincidences.

Every work's cost rises monotonically from wp 0.64 to 0.88, with no exceptions. Decomposed as mean
weighted contribution (value at 0.88 minus value at 0.64):

    highlight (w1)   +37.2      shadow (w1)      -11.6
    tone (w3)        +10.6      grain (w0.35)     -3.7
    pale-chroma (w1) +10.2
    contrast (w40)    +1.0      ------------------------
    pushing to compress: +59.0  pushing back: -15.3

**The highlight-collapse term alone outweighs everything opposing it by nearly 4x.** Compression
monotonically reduces highlight collapse, so compression always wins.

🔑 **This is the same degeneracy as the grey-rectangle S-curve, seen from the other side.** There the
objective ran the toe to the grid edge; here it runs the white-point to the low edge. One mechanism:
*the objective rewards the absence of the failure it can measure, and the cheapest way to remove a
failure is to remove the picture.*

## 2. The null hypothesis loses to the grey rectangle by 2.5x

Scoring the identity LUT with the same `cost()` on the six works the curve was fitted on
(the check the Hermes note proposed, now measured):

    stored fit (pivot .30 toe 11.66 shoulder 8.75)   90.53
    IDENTITY (toe = shoulder = 0, the null)         224.20
    wp0.75 g1.0                                     184.23
    no correction                                   224.20

`scurve_lut`'s docstring says the family contains "do nothing" precisely so the optimiser can fall
back to it. It can — and the objective prefers erasing the midrange **by a factor of 2.5**. The family
was never the problem.

## 3. Re-weighting cannot rescue it — that is a finding about the TERMS

**Registered before testing:** *if every term is monotone in wp per work, then any non-negative
weighting is monotone too, 0.76 becomes inexpressible, and the ceiling is 15/23 = 65.2%.*

**Partly refuted, and recorded as such.** `lost_contrast` is non-monotone on 16 of 23 works and
`pale_chroma_loss` on 12, so a middle pick *is* expressible. The ceiling therefore had to be measured:

    incumbent cost()                          30.4%
    chance                                    33.3%
    base rate (always the mode)               34.8%
    BEST-POSSIBLE weighting, leave-one-out    34.8%-43.5%  (mean 38.4%, 6 searches)
    BEST-POSSIBLE weighting, in-sample        65.2%        <- 6 free params on 23 points

The in-sample 65.2% landed on the predicted ceiling by coincidence, not for the predicted reason.
The honest number is the LOO range, and it is **indistinguishable from the base rate**. Even at its
ceiling the feature set does not generalise.

⚠️ The LOO figure moves with the search seed (34.8-43.5%). It is reported as a range; a single run of
it quoted as one number would be the same defect as any other figure that changes when you look again.

📏 Per ADR-096: *if feedback repeatedly cannot be explained by the model, that is a finding about the
model.* **Do not re-weight. The six terms cannot see what the judge sees.**

## 4. What the terms are blind to — the one signal, and why it still is not a rule

The single feature ADR-092 built on — fraction of the work above the white ink's luminance (163) —
was re-tested on n=23 (ADR-093 had n=11):

    Spearman rho = -0.507, permutation p = 0.021     <- real, and now significant where it was not
    picked 0.64: n=7  mean 44.1% above ceiling
    picked 0.76: n=8  mean 37.2%
    picked 0.88: n=8  mean  8.6%

The trend is genuine and in the direction Josh's own label notes assert. But no decision rule on it
generalises: a two-threshold ordinal rule scores 60.9% in-sample and **34.8% LOO — exactly the base
rate**; even the easier two-class split (0.88 vs the rest) reaches only 56.5% LOO against a 65.2%
base rate, permutation p = 0.82. The feature separates *nothing above the ceiling* from *plenty above
it* and then stops; 0.64 and 0.76 are not separated at all (44.1% vs 37.2%, overlapping end to end).

**This CONFIRMS ADR-093 at more than twice the sample size and does not reopen ADR-092.** The constant
stands. What is new is that the variation is now measurable as a rank trend while still not being a
predictor — so a better feature is worth looking for, and this one is not it.

## 5. Provenance defect found in passing: `scurve_fit.json` is not self-reproducible

The recorded per-work metrics could not be reproduced at the tool's default `--max-px 700`. Sweeping
the parameter, **`--max-px 380` reproduces all six works to zero error in every metric**. The file
records pivot/toe/shoulder/cost/LUT but not the render size it was measured at, so its numbers cannot
be checked from the file alone. Same family as the eleven instrument defects: a record that omits the
condition it was measured under.

*(Also verified, so it is not assumed later: Pillow 12.3.0 in the notebook env and 10.3.0 in the
project venv produce byte-identical ink indices and identical thumbnails on this path — sha1 match
over 2 works x 4 white-points. Re-check before trusting a third version.)*

## 6. Where this leaves the S-curve

Step 1 of `NEXT_SESSION.md` — constrain the family so degenerate curves are inexpressible — is still
worth doing, but it is now clearly **not sufficient**. A slope bound stops the grey rectangle; it does
not give the objective the ability to tell a good render from a bad one, because §3 shows that ability
is not in these terms at any weighting. Fitting a constrained curve against this cost would produce
the best curve *by a measure that disagrees with the judge*.

**The order that follows from the measurement:**
1. Do NOT fit a curve against `cost()` in its current form. It is not a scoring function, it is a
   preference for compression.
2. The missing ingredient is a term that responds to what the judge is actually trading. §4 says the
   ceiling feature captures part of it and stops; the 0.64-vs-0.76 boundary is unexplained by anything
   measured so far. **That boundary is the open question**, and it is where new labels or a new
   feature would pay.
3. `tools/eink_objective_gate.py` now runs this check on any candidate in one command. Nothing should
   be fitted against an objective that has not passed it.

## 7. Follow-up: the trade is two-ended, and a second feature moves the ceiling — but not the boundary

**Registered before testing:** white-point is one knob trading highlight recovery against shadow crush
(ADR-094), so the optimum must depend on BOTH ends and a one-ended feature cannot express it.
Predicted: among 0.64-vs-0.76 pickers, the 0.76 group carries >=1.5x the content below the shadow
floor (ink luminance 71.3), and a two-feature rule clears the 34.8% base rate leave-one-out.

**Direction supported.** Each feature isolates ONE class and is blind to the other two:

    picked   above ceiling (>163)   below floor (<71.3)
    0.64          44.1%                 15.7%
    0.76          37.2%                 28.9%      <- 1.85x the 0.64 group (predicted >=1.5x)
    0.88           8.6%                 26.0%

A two-threshold cascade (`above < t1 -> 0.88`; else `below < t2 -> 0.64`; else `0.76`) reaches
**60.9% leave-one-out, permutation p = 0.010** — the first thing all day to clear the base rate.

⚠️ **AND THE CONFUSION MATRIX SAYS IT DID NOT SOLVE THE STATED PROBLEM.**

    human 0.64 -> 2/7 correct      <- the boundary this was supposed to explain
    human 0.76 -> 6/8
    human 0.88 -> 7/8

The fitted second threshold is `below_floor < 1%`, satisfied by **3 works of 23**, so the 0.64 branch
is nearly empty: the rule is in practice *"0.88 when there is nothing above the ceiling, otherwise
0.76"*. Its 60.9% is one genuine discrimination (0.88 vs the rest, which the ceiling feature already
did) plus defaulting to the mode of the remainder. **Group means separating is not the same as a
decision boundary existing** — 1.85x on the means yielded no usable cut.

📏 Quoted alone, 60.9% at p = 0.010 would have read as "per-work is back". The distribution is what
says otherwise — the same lesson as §1, one section later, on my own result.

⚠️ **This does NOT reopen ADR-093.** What is now defensible is a BINARY question — *does this work
have anything above the ceiling to rescue?* — at 7/8, not the three-level per-work rule ADR-093
withdrew. ADR-092's binary was 0.64-vs-0.88 and ADR-093's criticism was that it learned "which works
tolerate a bad option least"; this is a different question on different data and must be argued
explicitly against ADR-093 before anything is shipped, not slipped back in.

**Conclusion for per-work.** Two features and six terms now fail to split 0.64 from 0.76. That is not
a feature-hunting problem any more — it needs an objective that can rank renders, and §3 shows the
current terms cannot at any weighting. **The missing mechanism is perceptual fusion:** every term
counts pixels one at a time, and none models that dither fuses at viewing distance, which is the whole
reason dithering works and is exactly what the judge sees from a wall. A spatial/perceptual difference
metric (S-CIELAB style — filter render and reference by human contrast sensitivity at the real viewing
distance, then compare in a perceptual space) is the standard answer to that question rather than
another invented one, and it can be gated against these same 23 labels with no panel time.

## STATUS: COMPLETE
