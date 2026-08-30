# S6 — the labels as a diagnostic, and the finding it produced about the objective

**Run:** 2026-08-29/30. 23 human white-point calls scored with the S-CIELAB objective through the exact
bench framing at 1600×1200. **Diagnostic, not a gate** — the rules below were registered before running.

## Registered rules, and what happened to them

| registered | outcome |
|---|---|
| A degenerate spread is a defect **regardless of accuracy**. Non-negotiable. | fired — see below |
| Above base rate is encouraging and is **not** a licence to tune. | n/a |
| Near base rate with a healthy spread is the **expected** outcome. | not what happened |
| *My prediction:* accuracy near/below base rate, **≥2 distinct predictions**. | half refuted |

**Result: 7/23 = 30.4%, against a 34.8% base rate — and the spread is `{0.64: 23}`.** Degenerate, by
the letter of the rule. My "≥2 distinct predictions" was wrong.

## But the rule was written for a different failure, and the difference is measurable

`eink_scurve.cost` was degenerate *pathologically*: more compression was always better, running to the
grid edge. A constant answer can also mean the correct answer genuinely is constant. Those are
distinguishable, so they were distinguished — sweep the white point continuously and look for an
interior optimum:

```
   n     0.20   0.30   0.40   0.50   0.60   0.641  0.72   0.76   0.88   1.00   argmin
  54    34.92  26.18  20.11   9.23   2.63   3.86   7.95  10.10  15.08  17.01   0.600
  49    16.45  12.42  10.78  10.91  11.63  12.32  14.21  15.27  18.64  22.09   0.400
   7    16.06  11.40   7.43   6.68   8.19   9.22  12.19  14.17  20.71  26.09   0.500
   2     6.57   6.99   8.51  10.20  12.13  12.79  14.29  15.10  16.85  18.49   0.200
  42    25.06  21.00  19.07  16.26  14.64  15.10  16.85  17.89  21.10  23.93   0.600
```

**4 of 5 have a genuine interior optimum.** The objective is *not* pathological. The three-level test
simply has no power here, because every optimum lies **below the lowest level offered** — which is
ADR-093's own lesson recurring: *"if neither level is near the optimum, the winner tells you about the
losers."*

## ⛔ The real finding: the objective prefers renders that are measurably too dark

Two registered predictions, both refuted, and chasing them is what produced the result.

**Refuted (0/8):** *"the exact derived curve `e(d)` will beat every linear white-point."* It loses on
every work — a linear scale near 0.4–0.6 wins.

**And the objective is not simply broken.** On raw realised **mean lightness** — no filtering, no
ΔE00, so it cannot be an S-CIELAB artefact — the derived **0.641 is closest on every work tested**:

```
work    reference L*    wp 0.50   wp 0.641   wp 0.76   wp 0.88   closest
  54           80.68      66.64      82.00     94.13     98.88     0.641
   7           46.70      38.36      48.96     56.99     64.24     0.641
  42           49.77      41.27      51.71     60.21     67.74     0.641
  25           27.90      23.37      29.85     35.18     40.37     0.641
```

Decomposing resolves it. On work 7, at wp 0.641 the *mean* lightness is right (48.96 vs 46.70) but
per-pixel |ΔL\*| is **6.18**; at wp 0.50 the mean is far too dark and per-pixel |ΔL\*| is **2.37**.

🔑 **The objective minimises MEAN PER-PIXEL error, and will accept a globally-too-dark image in
exchange for less highlight clipping. The judge does the opposite — Josh picks 0.76, lighter than
both. They disagree about AGGREGATION, not about colour.** Mean-per-pixel metrics systematically
under-weight global shifts; that is a known property, not a surprise, and it is exactly what ADR-096
says to conclude when feedback repeatedly cannot be explained by the model: *a finding about the model*.

**And it makes the objective the outlier, not the judge:**

```
derived e(d) asymptote (mean-lightness correct)   0.641
judge, n=23                                       0.727 - 0.800
shipped constant                                  0.75
the objective's preference                        0.40 - 0.60   <- the odd one out
```

## What this scopes, and what it does not

⚠️ **Do NOT use this objective to choose the white point.** ADR-084 governs and the panel decides;
Josh's 23 labels remain the best evidence we have for that number, and they sit close to the
physics-derived 0.641 — much closer than the objective does. **This vindicates keeping wp 0.75 shipped.**

✅ **The quantiser comparisons stand.** S5's largest step — LUT+precomp 12.71 → linear-light dither
9.20 — is measured at **matched tone**, so this aggregation bias does not touch it. The same is true of
S2's radiance-conservation result and S4's degeneracy audit.

⚠️ **The white-point step of S4/S5's ladder (16.48 → 14.01) does NOT hold tone fixed and therefore
inherits this bias. Treat that 2.48 ΔE00 as provisional** — the direction is supported by the
mean-lightness check, the magnitude is not.

📏 **The preference layer is not fitted here, deliberately.** The planned last step was to fit one
parameter — a lightness elevation over media-relative correct — to the 23 labels. Doing that now would
be fitting a preference constant on top of an objective whose aggregation is known to be wrong in the
same axis. **The honest move is to leave the shipped 0.75 alone and let the panel decide**, which is
what ADR-084 said before any of this started.

## STATUS: COMPLETE
