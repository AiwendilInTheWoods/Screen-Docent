# B2 — Colour and gamut

**Agent B2.** Corpus commit `1063f81`. Source of truth: `bench-eink/analysis/A1_rederived.jsonl`
(119 rows). Targets owned: `huevalue` (54), `inkmix` (1), `primaries` (2).
Units throughout: **camera-RGB normalised to THIS panel's own black = 0 / white = 255. NOT sRGB.**
A camera's filters are not human vision and over-saturate by construction — **directions are
meaningful, magnitudes are not**, and nothing here licenses touching `SPECTRA6_DITHER_PALETTE`.

> Written incrementally. Absence of a closing `## STATUS: COMPLETE` line means the run died and
> everything above the cut still stands.

---

## 0. CORRECTION TO A1 — instrument defect #9: `--v-lo` / `--v-hi` are INERT, and there is no low-value block

**This has to come first because it removes the corpus's only claimed evidence on the dark end and it
changes the headline.**

A1 scoped the four `huevalue_lowv_*` rows as a separate sub-corpus with three defects, the first being
"`value_in` labels are WRONG: the readout hard-codes 40..245; the render used `--v-lo 20 --v-hi 100`
so the true inputs are 20/36/52/68/84/100."

**That is not what happened. The render also produced 40..245.** `target_huevalue` honours `v_lo`/`v_hi`
**only on the `isolate` branch**; the default *joint* branch — the one that was used, since none of the
four rows carries `--isolate` — hard-codes its own value ladder:

```python
    if isolate:
        ...  v = round(v_lo + (v_hi - v_lo) * r / max(values - 1, 1))     # honours the flags
    canvas = Image.new("HSV", ...)
    for r in range(values):
        v = round(40 + (245 - 40) * r / max(values - 1, 1))               # <-- IGNORES them
```

Verified digitally, and the check can fail (the `isolate` arm shows it firing correctly):

```
target_huevalue(1600,1200, v_lo=20,v_hi=100)  vs  (v_lo=40,v_hi=245)
   joint path (used)      identical? True    max pixel difference 0
   isolate path (unused)  identical? False   max pixel difference 255
```

So the four `lowv` rows are **not a low-value block**. They are a fourth capture of the *same*
v = 40..245 grid at wp 0.0 / 0.64 / 0.75 / 0.88, shot at 15:44–15:47 after the panel was rotated back,
with the camera re-locked at gain 255. The shipped `value_in` labels (40/81/122/163/204/245) are
**correct**; A1's relabelling to 20/36/…/100 is the thing to discard. A1's caveat (c) — "the alignment
reference was rebuilt at the default v-range, so it does not match the render" — also dissolves: the
reference matches exactly, because the render *is* the default v-range. Only A1's caveat (b) survives:
the correct flat field is `flat_final.png` and `A1_rederived.jsonl` gives them the interpolated
pre-rotation flat.

### Consequences

1. **The dark end of the hue×value table was never measured.** The characterisation doc's own
   reasoning — "the bottom row lands at 4–8/255 of chroma, below the dark floor, so *chroma collapses
   at low value* cannot be distinguished from *the instrument cannot see chroma down here*, and
   re-shooting at v_lo=20 is the only way to separate the two" — is still entirely open. The re-shoot
   was commanded, was reported as done, and did not happen. **This is the eighth member of the
   documented family of defects: a knob that is offered, accepted, tagged into the filename and does
   nothing** (ADR-089's `EINK_SATURATION` defect, and the same defect `_pre` was already fixed for on
   `--saturation`/`--chroma-gamma`).
2. **The brief's preliminary read is therefore void.** "At v=36 the low-value block measures 44.0/255
   with 1 of 12 hues collapsed against 8.2/255 and 8 collapsed at v=40 in the main block" is comparing
   the reshoot's **second** row (nominal v=81) with the main block's **first** row (v=40). Row-for-row
   the two blocks agree (§1.3). There was no low-value artefact to overturn, because there was no
   low-value measurement.
3. **A gift in exchange:** four pairs of *bit-identical renders* captured 2.3 h apart with independent
   alignment solutions — the only such pairs `huevalue` has. They give a directly measured
   cross-capture error bar (§0.1) where A1 could only transfer one from `tonefine`.

Re-derived with the correct flat (`--flat bench-eink/reference/flat_final.png`, no interpolation) the
four rows improve on the one criterion that can fail — `patch_residual` 4.97/4.97/5.06/5.02 → **3.20 /
3.15 / 3.17 / 3.09**, the best in the corpus. But two of the four then jump to a different alignment
branch (dy −28 → +80), so the re-flat-fielded readouts are *not* uniformly better. Both versions are
used below and both are named wherever they differ.

### 0.1 A measured cross-capture bar for `huevalue`, and why A1's is ~5x too tight per cell

A1's `huevalue` bars come from four centre replicates that **all share one alignment solution**,
inflated by a factor 1.45 transferred from `tonefine`. The four bit-identical `lowv`/main pairs do not
share an alignment, so they measure the thing the transfer was standing in for:

| statistic | A1 (replicates ×1.45) | measured here, bit-identical renders, independent alignments |
|---|---|---|
| a single cell's `chroma_out` | σ 5.4, MDD **15** | rms(pair) 41.6/63.3/24.5/16.6 → σ **28.8**, MDD **81** |
| mean chroma over a value row (12) | MDD **15** | σ 8.1, MDD **23** |
| `mean_chroma_all` (72 cells) | MDD **3.3** | σ 3.5, MDD **9.7** |
| `n_collapsed_total` (of 72) | MDD **9** | MDD **6.2** |
| 4 centre replicates, same alignment, for contrast | — | cell r = 0.990, rms **7.3** |

> **Operative rule for everything below: a single `huevalue` cell cannot be compared between two
> conditions at all** — its cross-capture floor is ~80/255, which is a third of the range. Only
> aggregates are used. `mean_chroma_all` is treated as MDD **10**, a value row as **23**,
> `n_collapsed_total` as **9** (A1's, the more conservative of the two).
> The one place a single cell is quoted is where a *hue* is compared to itself across many
> conditions, i.e. as a mean over conditions.

---

## 1. ADR-091 on glass — VERDICT: the mechanism is confirmed, the table is wrong by ~7.6x, and "not hue-limited" does not survive

### 1.1 First, ADR-091's simulated table is reproducible — and it was computed in the wrong colour space

Re-running the ADR's simulation with the shipped code reproduces its numbers **exactly**, which
identifies how it was made:

```
chroma kept (out/in), s=0.55, flat patch, FS-dithered
                        v=100   v=140   v=180   v=220
ADR-091 as published
  cyan   (157 deg)       419%    186%     58%      0%
  blue   (180 deg)       218%     64%      0%      0%
  violet (292 deg)       308%     95%      2%      0%

reproduced, measuring the OUTPUT-palette encoding  (pure 255/0/0-style primaries)
  cyan                   419%    187%     58%      0%      <-- exact match
  blue                   219%     63%      0%      0%
  violet                 308%     95%      2%      0%

reproduced, measuring the DITHER palette  (the inks the panel actually lays down)
  cyan                    55%     26%     10%      3%
  blue                    30%     11%      4%      3%
  violet                  44%     14%      3%      3%
```

`SPECTRA6_OUTPUT_PALETTE` is the **encoding handed to the client**, not what the panel shows: it
re-labels every ink as a fully saturated primary so a dumb client's own quantiser cannot snap the
muted blue/green to black. Measuring chroma in it treats each ink as ~2–8x more colourful than it is.

> **ADR-091's headline sub-claim — "at v=100 the panel OVER-saturates badly, 250–460% of input
> chroma" — is an artefact of measuring the encoding instead of the ink. In ink terms the panel
> *under*-saturates at v=100 (30–55%).** The "predicted next problem" the ADR hands forward — that a
> chroma *attenuation* is the right tool once luminance is compressed — rests entirely on that
> inverted number and **must not be actioned as written**. §2 shows what the panel actually asks for.

The *shape* of the ADR's table (chroma-kept falls monotonically as value rises, and the dark inks'
hues fall first) is unaffected by the space error and is confirmed below.

### 1.2 On glass: chroma dies at BOTH ends, and white-point acts on it purely through value

Baseline `huevalue_wp0.0_g1.0_k1.0_s1.0` — no levers at all, i.e. exactly the condition ADR-091
simulated (12 hues x 6 values, s=0.55, joint FS dither):

```
 v_in   mean chroma   n collapsed (of 12)   mean lum
   40       11.3            7                55.9
   81       56.2            0                77.7
  122       80.1            0               128.9
  163       97.1            0               172.6
  204       74.4            1               200.2
  245       49.2            4               223.5
```

**ADR-091's "every hue survives at v=100" is CONFIRMED** (0 of 12 collapsed at v=81 and v=122, against
an `n_collapsed` MDD of 4 per row). **Its "six collapse to zero by v=220" is directionally confirmed
but overstated**: 1 of 12 at v=204 and 4 of 12 at v=245, and 4 is at the edge of the MDD. The collapse
is real, it is later than simulated, and it is smaller.

**The strongest result in this section is a superposition test that could have failed.** ADR-091's
mechanism says white-point buys chroma *only* by moving content down the value axis — so chroma should
be a function of the **effective** value `v_eff = v_in x wp` alone, no matter which (v, wp) pair
produced it. Eight independent captures (wp 0.0/0.64/0.75/0.88, each shot twice, 2.3 h apart, at
g=1.0 k=1.0 s=1.0), 48 value-rows:

```
model                         R^2      residual rms    corr(residual, wp)
chroma ~ f(v_eff = v * wp)   0.891         9.02            -0.117   (n.s.)
chroma ~ f(v nominal)        0.772        13.03            -0.113

pure replicate scatter, the two captures at identical v_eff:  mean |diff| 8.6
```

The `v_eff` model's residual (9.0) is the replicate scatter (8.6). **There is nothing left for
white-point to explain once value is accounted for**: regressing the residual on wp gives a slope of
−2.8/255 across the whole wp range, against a value-row MDD of 23. Collapsing on `v_eff` is also
strictly better than collapsing on nominal value, so the wp scaling is doing real work rather than
nothing.

> **VERDICT on "the gamut is luminance-limited and white-point is the gamut fix": ACCEPTED.**
> Measured, not simulated; the mechanism is confirmed by a superposition test with a failure mode, on
> 8 captures. At a fixed nominal v=245 the wp lever moves mean chroma from 43.6 (wp 0.88) to 49.2
> (wp 0.0, i.e. none) to **105.7 (wp 0.64)** — a 2.4x gamut gain from a luminance operator.
> ⚠️ Those particular rows are block-1 and wp is aliased with capture time there; the claim is
> re-established on the randomised block-2 rows in §2.3, where it holds.

### 1.3 ADR-091's "nothing is missing from the hue circle" does NOT survive

The ADR's mechanism sentence is *"EVERY CHROMATIC INK ON THIS PANEL IS DARK... the only light ink is
neutral white at 163."* That is false even of the palette it was reasoning from — **yellow is 156,
within 7 of white** — and the panel confirms it is false. Measured pure inks on this panel
(`primaries`, mean of the two refreshes, panel-relative):

```
ink        hue     camera-RGB mean ("lum")     chroma
black       -             0.0                    0.0
blue      221.0          49.7                  111.0
red         0.5          79.4                  232.3
green     134.7          89.2                   36.2
yellow     58.8         190.3                  189.3
white       -           248.4                    7.2
```

There is a **light** chromatic ink (yellow, 190/248 = 0.77 of white) and a **very dark** one (blue,
0.20 of white). So the luminance limit is not one number — it is **per hue**, set by whichever ink
serves that hue. Registered prediction before testing: *if the limit is set by the serving ink's
luminance, the value at which a hue carries its chroma must track that ink's luminance.* Measured, as
the chroma-weighted centroid of each hue's chroma over `v_eff`, across the same 8 captures:

```
hue   serving ink  ink lum  |  centroid v_eff  mean chroma  |  chroma at v_eff 215-260
  0   red             79    |      141.2          114.7     |     108.0
 30   yellow         190    |      140.9          103.1     |      91.0
 60   yellow         190    |      146.5           76.3     |      79.5
 90   yellow         190    |      151.1           51.4     |      72.4
120   green           89    |      132.5           29.0     |      26.9
150   green           89    |      113.0           23.8     |       3.5
180   blue            50    |      109.4           43.0     |      18.9
210   blue            50    |      113.7           59.2     |      19.6
240   blue            50    |      117.1           60.7     |      32.1
270   blue            50    |      113.0           26.3     |      16.2
300   red             79    |      137.3           60.4     |      21.4
330   red             79    |      139.4           16.9     |       1.2

corr(serving ink luminance, chroma centroid) = +0.764   permutation p = 0.0023 (N=200 000)
```

**The prediction holds.** Yellow-served hues (30–90) carry their chroma ~35 units of value higher than
blue-served hues (180–270), and are the only hues with usable chroma left in the top band. So:

> **CORRECTION TO ADR-091: the gamut is luminance-limited, but the limit is HUE-DEPENDENT.**
> "Nothing is missing from the hue circle" is wrong in two ways. (a) The bright end is not lost
> uniformly — hues 0–90 keep 72–108/255 of chroma at `v_eff` 215–260 while hues 150/330 keep 1–4.
> (b) Two hues are weak at **every** value: **150° (yellow-green) and 330° (magenta-pink)**, peaking at
> 40 and 43 against 182 for hue 0. 330° has no ink within 30°; 150° sits 15° from green, but green is
> the panel's *least* chromatic ink (chroma 36 against red's 232). Those two are gamut holes in the
> ordinary sense and no luminance operator will fill them.
>
> Practical consequence: **a single global white-point cannot put every hue in its best band.** The
> optimum for a yellow-dominated work sits ~35 units of value above the optimum for a blue-dominated
> one. That is a real, measured argument for a hue-aware white-point that ADR-093's constant does not
> capture — but see §2.4 before acting on it.

### 1.4 The dark end is STILL unmeasured — and the corpus cannot say whether chroma survives there

At `v_eff` below 45 the panel measures 3–22/255 of mean chroma with 5–10 of 12 hues collapsed. That
looks like a hard low-value gamut floor and it is **not safe to report as one**, for three reasons:

1. The re-shoot commissioned to answer exactly this **never happened** (§0). The lowest `v_eff` in the
   corpus is 25.6, and it is reached by *scaling* v=40 by wp 0.64 — not by rendering a dark value.
2. The digital predictor, driven by this panel's own measured inks, expects **31.6** mean chroma at
   v=40 where the panel measures **11.3**; and it expects mean luminance **31.5** where the panel
   measures **55.9**. The panel's black end is lifted ~24/255 above prediction. A veiling-glare /
   lateral-scatter lift of that size mechanically destroys chroma in a `max−min` readout.
3. `chroma_out = max − min` is geometrically capped at `min(3L, 765−3L, 255)`. At a measured
   luminance of 20 the ceiling is 60 whatever the panel does.

> **Not a finding. The low-value half of ADR-091's table remains unmeasured**, and the instrument as
> built cannot separate "the panel loses colour in shadow" from "the rig cannot see colour in shadow".
> The fix is the re-shoot that was already specified — and now it must be run with `--isolate`, or
> with the joint branch's hard-coded ladder repaired.

### 1.5 The digital predictor is at the panel's own reproducibility ceiling

A quantiser simulator was built for this analysis: render each condition through the exact `_pre`
chain, map the output-palette result onto **this panel's measured inks**, compose, and run the same
`readout_huevalue`. Across all 50 main `huevalue` conditions (3600 cells):

```
                                   per cell            per value-row mean
predictor vs glass, measured inks  r 0.596  rms 41.9   r 0.932  rms 12.3   bias  -2.0
predictor vs glass, Pimoroni ink   r 0.636  rms 42.2   r 0.927  rms 23.2   bias +15.3
GLASS vs GLASS, bit-identical      r 0.75 avg rms 36.5 r 0.96 avg rms 10.1
  renders, independent alignments
```

**The simulator predicts the panel as well as the panel predicts itself.** That is the licence the
`inkmix` target was built to grant: sweeps can now be predicted offline instead of photographed one
refresh at a time. It also means the per-cell disagreement is registration, not model error.

---

## 2. The white-point x chroma interaction — REAL, confirmed on the randomised rows, and it is a 3x effect

### 2.1 The block-1 read (aliased — shown for continuity, not as evidence)

The crossed `wp x chroma-gamma` design lives at rows 18–29, which A1 proved ran in literal design
order with white-point as the outer loop (r = +0.883 with capture time). Mean chroma of the **top**
value row (v=245), from `A1_rederived.jsonl`:

```
                k=1.0          k=1.5          k=2.0        (mean chroma / n collapsed of 12)
  wp 0.0 (off)  49.2 /  4     15.4 /  5      3.7 /  8
  wp 0.64      105.7 /  0     85.3 /  0     51.2 /  1
  wp 0.75       61.1 /  2     41.9 /  2     25.9 /  4
  wp 0.88       43.6 /  3     21.2 /  4      8.8 /  5
```

The brief's preliminary read reproduces (3.7 with 8 of 12 collapsed at wp off, against 51.2 with 1 at
wp 0.64 — a **14x** ratio, where the same wp comparison without the chroma lever is only 2.1x).
**But these rows cannot carry the claim.** Hence:

### 2.2 The block-2 test — a complete 2^4 factorial, randomised

Block 2 (rows 48–112, randomisation verified by A1) contains **all 16 corners** of
wp {0.64, 0.88} x gamma {1.0, 1.8} x chroma-gamma k {1.0, 2.0} x saturation {0.7, 1.3}, plus 4 centre
replicates and 6 axial points. Effects are stated as the change from the low to the high level;
each averages 8 rows against 8, so with the measured value-row σ of 8.1 an effect is a finding above
**11.3** (2.8 x σ/2).

```
response: mean chroma of the TOP value row (v=245), grand mean 67.6
   chroma-gamma k  1.0 -> 2.0      -29.31   ****
   gamma         1.0 -> 1.8        +24.80   ****
   wp x gamma                      +19.58   ***
   wp x chroma-gamma               -14.34   **      <-- the interaction under test
   white-point  0.64 -> 0.88       -13.73   **
   gamma x k                       +12.16   (marginal)
   saturation   0.7 -> 1.3          +9.19   (below resolution)

response: mean chroma over the whole 72-cell grid, grand mean 44.4
   k -11.91 | sat +11.78 | gamma x k +6.62 | wp x k -3.91 | everything else < 3
```

The 2x2 cell means make the interaction concrete (top value row, averaged over gamma and saturation,
n=4 each):

```
                     k = 1.0      k = 2.0      cost of the chroma lever
   wp 0.64             81.9         67.0            -14.9
   wp 0.88             82.6         38.9            -43.7
```

> **VERDICT: the interaction is REAL and it is a 3x effect. The chroma-gamma lever costs 43.7/255 of
> top-end chroma at wp 0.88 and only 14.9 at wp 0.64.** Confirmed on randomised rows, effect −14.34
> against a resolution of 11.3. The two white-points are *indistinguishable* when the chroma lever is
> off (81.9 vs 82.6) and differ by more than a factor of two when it is on.
>
> Note the direction relative to the brief: white-point does not so much "protect colour against the
> chroma lever" as **remove the conditions under which that lever is destructive**. With no chroma
> lever, wp 0.64 and 0.88 give the same top-end chroma.

### 2.3 The re-established wp claim, and why the interaction exists

§1.2's `v_eff` superposition explains it without a new mechanism. `wp x gamma = +19.58` is the giveaway:
**wp and gamma are the same lever** (both move content down the value axis), so they substitute for
each other — that interaction is redundancy, not synergy. What the chroma lever then meets depends on
where the content landed:

* At **wp 0.88** the top of the range sits at `v_eff` 216, above every ink but yellow and white. Chroma
  there is built from a handful of scattered chromatic dots, and reducing requested saturation removes
  them entirely — 82.6 → 38.9.
* At **wp 0.64** the top sits at `v_eff` 157, inside the band where the dark inks are usable, so the
  same relative reduction still leaves plenty — 82.6 → 67.0.

This also **re-establishes ADR-091's wp claim on randomised data**: the `wp` main effect on the top
value row is −13.73 (0.64 → 0.88 loses chroma) with `wp x gamma` +19.58 — i.e. lowering the white-point
buys top-end chroma exactly when gamma has not already done it. The block-1 result in §2.1 is
therefore not an artefact of the illumination alias.

### 2.4 What this says about ADR-088, and what it does NOT license

**It supports the brief's hypothesis for why the ADR-088 chroma work failed.** ADR-088's A/Bs were run
with no white-point at all. At wp off, this corpus measures top-of-range chroma at 49.2 with the lever
off and **3.7 with the lever at k=2.0, 8 of 12 hues collapsed**. There was nothing left to modulate:
the chroma lever was being judged at a luminance where the colour had already gone. That is the same
diagnosis ADR-091 reached ("it attacked chroma while the binding constraint was luminance"), now
measured on the crossed design rather than inferred.

**But the corpus does not license the follow-on that ADR-091 proposed.** ADR-091 predicts that once
luminance is compressed, a chroma *attenuation* becomes the right tool, because the panel
"over-saturates 250–460%" at low value. §1.1 showed that number is an encoding artefact. On glass, in
ink terms, **every measurement in this corpus says the chroma lever only ever subtracts**: the k main
effect is −29.3 at the top row and −11.9 over the whole grid, the saturation main effect is +11.8, and
there is no (wp, k) combination anywhere in the block-2 design where k = 2.0 beats k = 1.0.

⚠️ **This is a chroma measurement, not a preference measurement.** `huevalue` is a synthetic s=0.55
grid; a human judge looking at art may still prefer less chroma because false colour on skin is
objectionable in a way that missing chroma on a test patch is not. What can be said is that the
*mechanism* offered for attenuation — over-saturation at low value — is not there.

### 2.5 Bonus: the dither-bleed term A1 left open, measured for `huevalue`

A1 reported that the `surround` readout is a registration artefact and that "every dense grid target
in this battery still carries an unquantified surround term." For `huevalue` it can be quantified,
because the corpus contains the `_iso` condition — the same levers, each cell dithered **on its own**
and then composited, so the difference from the joint render *is* the Floyd–Steinberg cross-cell
contamination. And the digital simulator gives the same difference with no camera involved:

```
mean chroma per value row     v=40   81    122   163   204   245   |  ALL
  glass, joint                 5.8  37.1  59.5  69.3  78.2  61.1   | 51.8
  glass, isolated              6.2  30.0  53.7  72.6  96.5  87.8   | 57.8
  glass, isolated - joint      0.4  -7.1  -5.9  +3.3 +18.3 +26.7   | +6.0
  SIMULATED, isolated - joint  0.4  +1.4  +4.2  +7.2 +11.1 +11.9   | +6.1
```

> **The FS bleed costs 6.0/255 of grid-mean chroma, and the simulator predicts 6.1** — a match to
> 0.1/255 on a quantity nobody tuned. So the dither-bleed term on `huevalue` is small relative to
> everything measured above, it is **in the render rather than the instrument**, and it is
> **predictable offline**. The per-row figures diverge at the top (glass +26.7 vs digital +11.9) but a
> single value-row difference has an MDD of 23, so that gap is not a finding.
> This does **not** close A1's `surround` question in general — it bounds it for this one target.

---

## 3. THE CHROMA-SURVIVAL FUNCTION — a panel property, and the colour cost of any tone curve

*(Added after the brief, on the coordinator's request, in response to the Olympia evidence.)*

### 3.1 How it was extracted

`huevalue` looks like a 12x6 grid at fixed saturation, but it is not: the lever chain moves every cell
in **both** value and saturation before the quantiser sees it. Running the exact `_pre` chain over the
nominal HSV of every cell in every condition converts the corpus into 3528 samples of

```
   (L_in, C_in, hue)  ->  C_out
   L_in = mean RGB of what the quantiser was ASKED for      (a tone curve moves this)
   C_in = max-min of the same                                (a chroma lever moves this)
   C_out = measured chroma on glass, panel-relative
```

The structure that emerged is simple and it is the deliverable: **within the covered domain, output
chroma is proportional to requested chroma, with a gain that depends on input luminance and hue.**

```
survival ratio  S = C_out / C_in,  pooled over hue, C_in >= 25
  L_in    15    25    35    45    55    65    75    85    95   105   115
  S     0.82  1.01  1.04  1.18  1.35  1.32  1.43  1.27  1.39  1.31  1.28
  L_in  125   135   145   155   165   175   185   195
  S     1.05  0.93  0.72  0.58  0.48  0.26  0.21  0.18
```

Fitted (weighted least squares over 19 luminance bins, R² = 0.845, residual rms 0.161):

> ### **S(L) = 1.30 / (1 + exp((L − 152) / 15))**
>
> `C_out ≈ S(L_in) · C_in`, in panel-relative camera units, for `C_in` between 25 and 200.
> Half-survival at **L_in = 152**; 90% of plateau at 119; 25% at 168; 10% at **185**.

**Per hue**, the same fit (the parameter that matters is `L50`, the luminance at which that hue loses
half its colour):

```
hue    0    30    60    90   120   150   180   210   240   270   300   330
L50  150   169   170   154   143   132*  120   117   123   100    89*  144
        \___ yellow-served ___/              \______ blue-served ______/
corr(serving-ink luminance, per-hue L50) = +0.770,  permutation p = 0.0033  (n=12)
      * hues 150 and 300 fit a near-flat curve (w = 36 and 59) and their L50 is poorly determined.
```

This is §1.3's hue-dependence stated as a usable number: **a yellow-served hue keeps its colour ~50
luminance units higher than a blue-served one.**

### 3.2 The chroma ceiling, stated precisely

For a **flat** pale-chromatic patch (no dither texture to borrow from), simulated on the validated
measured-ink predictor at C_in = 27.5 — the Olympia figure:

```
mean OUTPUT chroma of a flat patch, C_in = 27.5, no tone curve
  L_in     100   120   140   160   180   200   220   240
  hue   0  85.6  88.1  81.1  31.5   7.2   7.2   7.2   7.2
  hue  30  81.1  72.7  62.3  25.7   7.2   7.2   7.2   7.2
  hue 210  55.8  45.2  25.0   7.2   7.2   7.2   7.2   7.2
  hue 300  30.0  34.2  27.7   7.2   7.2   7.2   7.2   7.2
                                              (7.2 = the white ink's own chroma, i.e. DEAD)
```

> **The ceiling is a cliff, not a slope.** Between L_in 140 and 180 a red-family pale colour goes from
> 81/255 of chroma to **exactly the white ink** — the patch quantises to 100% white and there is no
> colour left to measure. A blue-family one is already dead at 160. The transition occupies **~40
> luminance units** and there is nothing beyond it: no tone curve, chroma lever or dither can recover
> chroma from a patch that is 100% white ink.
>
> **Practical answer to "how far can a tone curve lift before colour dies": to L_in ≈ 152 (measured
> half-life), and not past ≈ 185 (10% of plateau) for any hue; ≈ 170 for yellow-served hues and
> ≈ 120 for blue-served ones.**

### 3.3 Reconciliation with ADR-091 — yes, it is the same finding, and that is the point

**They are one mechanism, read in opposite directions.** ADR-090 measured the *ink ceiling* at input
luminance **163** — the palette's white ink, above which no ink exists. This corpus measures the
*chroma* half-life at input luminance **152**, on glass, from a completely independent target and
readout. Those are the same number to within the fit's resolution.

```
ADR-090 (digital render):   input > 163 has no ink to be built from -> tone collapses to flat white
ADR-091 (simulated):        compressing luminance moves content down into the inks -> colour returns
B2      (on glass):         S(L) = 1.30/(1+exp((L-152)/15)) -> chroma half-life at L_in = 152
new evidence (panel judge):  lifting pushes content up past that line -> the rose "is more or less ecru"
```

**Why saying it twice matters.** ADR-091 is written as a *permission* — compression buys gamut. Read
the other way it is a *constraint*: **every operator that raises luminance spends colour, and the
exchange rate is knowable in advance.** A shadow-lift, a highlight recovery, an `_adaptive_gamma`
below 1.0, an exposure correction — each is a chroma transaction, and until now none of them had a
price tag.

**The price tag.** A tone curve of exponent γ maps `L -> 255(L/255)^γ`, so it pushes a pixel over the
163 ceiling as soon as `L > 255·(163/255)^(1/γ)`. Everything with colour between that line and 163
loses it:

```
  gamma   colour now survives only below L_in     band of pixels that lose their colour
   1.00              163.0                        (none — the reference)
   0.95              159.2                        159 .. 163
   0.90              155.1                        155 .. 163
   0.85              150.6                        151 .. 163
   0.80              145.7                        146 .. 163
   0.75              140.4                        140 .. 163
   0.60              121.0                        121 .. 163
```

**The steep nonlinearity between γ 1.0 and 0.75 is not a panel effect — it is the work's histogram
crossing a fixed threshold.** The band widens roughly linearly in γ, but the *number of chroma-bearing
pixels inside it* is whatever the painting puts there, and a bright work like Olympia puts a lot in
140–163. Verified by simulating a population of pale chromatic pixels (C_in = 27.5, hues 0/30/300,
L_in uniform on 110–210) through the same curves on the measured-ink predictor:

```
gamma           1.00    0.90    0.75    0.60
simulated mean surviving chroma   24.3    19.3    11.4     3.3
Olympia, measured on the render   31.6    27.0    10.3     4.2
simulated bare-white fraction    79.8%   85.2%   92.4%   97.8%
Olympia, measured                75.1%   82.5%   95.3%   99.9%
```

An uncalibrated toy population reproduces both curves. **The Olympia failure is fully explained by the
ink ceiling and needs no new mechanism** — and, more usefully, it was predictable from
`SPECTRA6_DITHER_PALETTE` plus a histogram, with no panel refresh at all.

> **Recommendation (stated as a measurement, not a decision — ADR-084 governs):** any tone operator
> should carry a **chroma budget**: the fraction of the work's chroma-bearing pixels the curve moves
> across L=163. It is one histogram query, it costs nothing, and this corpus says it predicts what the
> judge will see. The same arithmetic prices white-point compression **positively** — wp 0.64 moves
> the top of the range from 245 to 157, i.e. from far above the ceiling to just below it.

### 3.4 Where this runs out — be careful with it

1. **The `L_in > 140, C_in < 30` corner is EMPTY in the corpus.** `huevalue` renders at s=0.55, so
   input chroma is proportional to input value; the levers can lower chroma but nothing raises
   luminance. Coverage at L_in ≥ 140 starts at C_in = 30, and there are no cells at all above
   L_in = 220. **Exactly Olympia's regime — pale and chromatic — is the corner the corpus does not
   contain.** §3.2's flat-patch numbers there come from the *simulator*, which is validated against
   the corpus (§1.5) but is being extrapolated, and §3.3's population figures are simulation, not
   measurement.
2. **S(L) is fitted on `C_in >= 25`.** Below that the ratio is dominated by division noise. It should
   not be used for near-neutral input.
3. **The dark end is unmeasured** (§0, §1.4). S(L) turns over below L_in ≈ 50 in the data, but that
   turnover is at least partly the rig's dark-end lift, so the low-L half of the curve is a
   description of *this instrument's* view, not established panel behaviour.
4. **The corpus contains no lifting curve.** Its gamma levels are 1.0 / 1.4 / 1.8 / 2.2 — all
   darkening. γ < 1 was never photographed. The survival function is measured across the luminance
   range a lift would reach, but "a lift produces the same result as arriving at that luminance by
   another route" is the §1.2 superposition assumption; it held for white-point (R² 0.891, no residual
   wp term) and is *expected* to hold for gamma, but for gamma it is untested below 1.0.
5. **`n = 12` hues, 30° apart.** The per-hue `L50` values for hues 150 and 300 are poorly determined,
   and the two hue-circle holes identified in §1.3 sit near them.
6. **Nothing here is a white-point claim**, so the block-1 alias does not apply — the survival function
   pools all 50 main conditions and its predictor is input luminance, not a lever. The one wp claim in
   §3.3 (wp 0.64 moves 245 → 157) is arithmetic on the lever definition, and its chroma consequence
   was established on the randomised rows in §2.2.

The function, its per-hue parameters and the coverage map are in `B2_findings.json`
(`chroma_survival`).

---

## 4. The optical mixing law (`inkmix`) — instrument defect #10 blocks it, and the shipped reading is inverted

### 4.1 The keystone target is misregistered, and the check that proves it cannot pass by accident

`inkmix` is 15 ink pairs x 5 ratios, undithered by construction — the one pure panel invariant in the
battery. Its shipped readout (in `A1_rederived.jsonl`, which is the *re*-derive, so this is not a
stale-file problem) contains this:

```
blue+green, five ratios:   [254 253 252] [254 253 253] [254 252 252] [252 253 250] [252 252 251]
```

**Two dark inks in a checkerboard cannot produce paper white at every ratio.** Six more columns are
similarly wrong: `white+green` reads a yellow-to-red ramp, `red+yellow` reads a blue-to-red ramp, and
the `black/red` element cell at pitch 16 reads [252, 253, 252]. The sampling windows are on the wrong
cells.

**The mechanism, and it is a live version of a defect the code already documents.** `grid_offsets`
searches **±30 px** for each cell's own offset. Its docstring is calibrated for the old geometry —
*"inkmix cells are 101x98, ~22 px of margin"* — but `content_box` was since narrowed from 1528 to
1192 px (correctly, so that sampling stays inside the fiducials). **`inkmix` cells are now 67 x 86**,
so at a 0.30 inset the sample window is 27 px wide with 20 px of margin, and a ±30 px search **can
reach the neighbouring cell**. It then finds it, because a neighbouring flat cell minimises interior
variance just as well as the right one. This is the documented signature exactly: the check —
minimum interior variance — cannot fail.

`huevalue` is not affected in the same way: its cells are 93 x 108 with a 12-hue period, and its
readout was independently validated against the digital predictor in §1.5.

### 4.2 Best-effort re-registration, and why it is not enough

I re-derived `inkmix` from the raw with the affine solved before alignment (as `read_panel` does) and
then searched a global (scale, dx, dy) on a criterion **independent of the mixing law**: each pair
column, in each channel, must be monotone across the five ratios, because any two-ink mixture is a
monotone path between two fixed colours. Digital truth is 45/45.

```
arm                                            monotone/45   columns reading their own design
shipped free per-row search  (0.447,1.02,+60,-44)     37             8 of 15
best global on monotonicity  (0.91, +22, -64)         41            10 of 15
   + bounded per-cell refinement (radius 12 px)       38            11 of 15
gutter-lattice registration (purely geometric)        13             not evaluated
no alignment                                          16             2 of 15
```

⚠️ **Two law-free criteria disagree.** Optimising monotonicity lands at scale 0.91, dx +22, dy −64;
locating the white gutter lattice by comb-correlation lands at scale 0.88–0.99, dx +9, dy +51 — and
that solution scores 13/45 on monotonicity, i.e. worse than not aligning at all. **A registration
that two independent geometric criteria cannot agree on is not a registration.**

> **VERDICT: the optical mixing law is NOT MEASURABLE from this corpus as it stands.** The keystone
> target's cells cannot be located to better than a fraction of a cell, no arm reaches 45/45, and 4–5
> of 15 pair columns are demonstrably reading the wrong cell under every arm tried. Everything in §4.3
> is **provisional**, conditional on the best-effort registration, and is reported because it points
> in the opposite direction to the shipped numbers — which matters more than its own precision.

### 4.3 What the best-effort registration says — provisional, and it inverts the shipped reading

Under (scale 0.91, dx +22, dy −64) with a bounded per-cell refinement, restricted to the 8 columns
that both read their own design and are 3/3 monotone (`black+white`, `black+red`, `black+yellow`,
`black+blue`, `white+red`, `white+yellow`, `white+blue`, `white+green`):

**(a) The mixture space is already close to LINEAR — the linear-light transform makes it worse.**
Scanning the exponent γ in `meas^γ = f·A^γ + (1−f)·B^γ`, with A and B the measured pure inks:

```
  gamma    0.6    0.7    0.8    0.9    1.0    1.1    1.2    1.4    1.8    2.2
  mean|e| 22.5   19.1   16.4   14.6  14.15  14.8   15.9   18.4   24.1   29.2
```

**The optimum is γ ≈ 0.9–1.0, not 2.2.** The brief's instruction to work in linear light comes from a
1:1 black/white cell reading 178.9 against a linear-light prediction of 188 — but under this
registration that same cell reads **110.3** against an *encoded*-linear prediction of 124.2 and a
γ=2.2 prediction of 181.3. The reason is mundane and worth recording: **the readout averages encoded
pixel values over a window, and the affine correction is applied to encoded values**, so a resolved
checkerboard is an arithmetic average by construction. `panel-relative camera RGB` behaves as a
mixing-linear space for this rig, and applying a 2.2 decode to it introduces error rather than
removing it.

**(b) The dot gain is SMALL — the "far darker than linear" reading is a registration artefact.**
Fitting the effective ink-a coverage `g` at each nominal ratio by least squares over the accepted
columns (γ = 1):

```
  nominal f    0.125   0.250   0.500   0.750   0.875
  effective g  0.013   0.192   0.526   0.766   0.873
  dot gain    -0.112  -0.058  +0.026  +0.016  -0.002
  residual     24.5    13.1    10.9     7.2     4.3      (mean |meas-pred| over 8 pairs x 3 channels)

  black+white column alone (the widest-contrast pair):
  effective g  0.060   0.223   0.556   0.796   0.884
  dot gain    -0.065  -0.027  +0.056  +0.046  +0.009
```

Against the brief's expectation (*"high coverage measures far darker than linear — 66 where linear
predicts 137"*), the corrected reading at f = 0.875 black is **28.9 measured against 31.0 predicted**.
**Coverage is within ±0.06 of nominal everywhere**, with a weak S-shape: slight *under*-coverage at
the sparse end (a lone dark dot renders lighter than its area) and slight *over*-coverage at 0.5–0.75.
There is no large dot gain to characterise. ⚠️ Provisional — see §4.2.

**(c) The element-size sweep says dot spread is NOT the patch-size effect.**

```
  mean over channels, 1:1 mixture      pitch  1     2     4     8    16 px    spread
  black / white                             130.3 137.8 133.9 126.7 115.7      22.1
  white / yellow                            214.7 213.9 214.3 214.9 218.9       5.0
  black / red   (columns in the rejected zone — do not use)  79.8 106.9 109.5 54.0 46.6  62.8
```

`white/yellow` moves **5.0/255 across a 16x change in element pitch** — below the undithered
field-repeatability bar (11.6 worst). `black/white`, the maximum-contrast pair, moves 22.1 with the
finest pitches reading *lighter*, which is the expected sign for optical averaging happening partly in
the lens.

> **This answers the brief's question negatively and usefully: element size does not explain the
> strip-vs-field patch-size effect.** A 16x change in element pitch moves a chromatic mixture by 5/255,
> while the strip-vs-field disagreement on chromatic inks is 42–78/255 (A1's corrected figures). Two
> orders of magnitude apart. **The patch-size effect is about the SIZE OF THE PATCH and its
> surround — lateral scatter in the glass, exactly the mechanism `STRIP_ORDER` was reordered for
> — not about how finely the dither is divided.** A corollary worth having: a Floyd-Steinberg dither
> at 1–8 px granularity costs nothing measurable in mixture accuracy.

**(d) The metamer block cannot be read.** Only one designed pair falls within 20/255 of another under
the additive model (`0.50 white+black` vs `0.75 green+white`, predicted 19.7 apart). They measure 72.4
apart — but that is one pair of single cells, in a target whose registration is unresolved, against a
single-cell bar that is already tens of 255ths. **n = 1 and the instrument is not trusted: no verdict.**
The metamer question — *do different mixtures the renderer computes to the same target measure the
same?* — is **unanswered**, and it is the question that decides whether the quantiser's distance model
is sound. It needs a re-shoot at a larger cell pitch, not more analysis.

---

## 5. Measured inks vs `SPECTRA6_DITHER_PALETTE` — direction only

`SPECTRA6_DITHER_PALETTE` is Pimoroni's measurement of a **different** EL133UF1. Both sets normalised
to the same convention (per channel, this panel's own black = 0 and white = 255; for the palette,
its own `(0,0,0)` and `(161,164,165)`):

```
             THIS PANEL (primaries fields, mean of 2 refreshes)   PIMORONI, normalised
ink        R     G     B    mean   chroma   hue        R     G     B    mean  chroma   hue
black      0     0     0     0.0     0.0     -         0     0     0     0.0    0.0     -
blue       0    37   112    49.7   111.0   221.0      97    92   145   111.2   53.5   244.7
red      234     4     1    79.4   232.3     0.5     247   112   116   158.3  135.1   358.5
green     74   110    84    89.2    36.2   134.7      92   142   108   113.8   49.6   138.8
yellow   255   250    66   190.3   189.3    58.8     255   255   110   206.6  145.3    60.0
white    244   252   249   248.4     7.2     -       255   255   255   255.0    0.0     -
```

**Directions that survive the camera's over-saturation (which inflates every chroma by construction):**

1. **Hue directions agree, closely.** Yellow 58.8 vs 60.0, green 134.7 vs 138.8, red 0.5 vs 358.5 —
   all within 4°. Only **blue** differs materially: 221.0 measured against 244.7 in the palette, a
   24° swing toward cyan. The renderer's distance model places blue ~24° from where this panel puts
   it.
2. **Every chromatic ink is DARKER relative to white than the palette says**, and by different amounts:
   blue 0.20 of white here against 0.44 in the palette, red 0.32 against 0.62, green 0.36 against
   0.45, yellow 0.77 against 0.81. ⚠️ Part of this is the camera: a saturated ink drives two of three
   channels toward zero, dragging a *mean of camera RGB* down harder than a luminance would fall. So
   the direction is credible and the magnitude is not.
3. **The luminance ORDER is not the same.** Palette: blue 71 < green 73 < red 101 < yellow 156.
   Measured: blue < red < green < yellow. **Red and green swap.** ADR-091's mechanism reasoning quoted
   the palette order; on this panel red is the *darker* of the two.
4. **ADR-091's "the only light ink is neutral white" is wrong in both sets.** Yellow is 156 of 163 in
   the palette and 190 of 248 here — a genuinely light chromatic ink, which is why the yellow-served
   hues are the only ones with colour left at the top of the value range (§1.3, §3.1).

**Which set predicts this panel better — tested, not asserted.** Both were run through the identical
quantiser simulator against all 50 `huevalue` conditions (§1.5):

```
                             per value-row mean, vs glass
   measured inks     chroma  r 0.932  rms 12.3  bias  -2.0   |  luminance  rms 34.3  bias +30.1
   Pimoroni palette  chroma  r 0.927  rms 23.2  bias +15.3   |  luminance  rms 20.5  bias  -1.6
```

The measured inks predict **chroma** roughly twice as accurately (rms 12.3 vs 23.2, and unbiased);
the Pimoroni palette predicts **camera-mean luminance** better. That split is exactly what a
camera-over-saturation artefact looks like, and it is why the comparison cannot be pushed further.

> ⚠️ **NOTHING HERE PROPOSES CHANGING `SPECTRA6_DITHER_PALETTE`, and this corpus cannot support such a
> change.** The measurement is camera-RGB with no colour reference in frame; the session's own
> conditions record `colour_reference: "none (no ColorChecker) — absolute colour claims are NOT
> supported"`. A proper answer needs **either a spectrophotometer / colorimeter reading of the six
> inks under the panel's own diffuse illumination, or a ColorChecker in every frame** so the camera's
> own transform can be inverted. Until one of those exists, the honest statement is: *this panel's
> inks are probably darker and its blue probably sits ~24° from where the renderer thinks, and the
> size of both effects is unknown.*
>
> One thing above **is** actionable without any of that, because it is a hue and not a magnitude:
> `_chromatic_ink_hues()` derives the chroma rule's ink hues from the palette. If the panel's blue
> really sits 24° away, that rule's "distance to the nearest serving ink" is wrong for the whole
> cyan–violet arc — the arc §1.3 and §3.1 independently identify as the weakest on this panel.
> Worth a check; not worth a rewrite on this evidence.

---

## 6. What I could not resolve

1. **The optical mixing law — the keystone target's whole question.** Two registration criteria that
   are both independent of the mixing law disagree (§4.2), so additivity, dot gain and the metamer
   test are all provisional or unanswered. The one thing I am confident of is the *sign*: the shipped
   reading (large dot gain, high coverage far darker than linear) is a registration artefact, and the
   corrected reading is close to additive. **Needs a re-shoot with a coarser `inkmix` grid** (fewer
   pairs per frame, or two frames) so the cell margin exceeds the per-cell search radius.

2. **The dark end of the gamut — the corpus's one deliberate gap, and it is still open.** The
   re-shoot commissioned to answer it did not happen (defect #9, §0). Nothing here can separate "the
   panel loses colour in shadow" from "the rig cannot see colour in shadow": the digital predictor
   expects mean luminance 31.5 at v=40 where the panel measures 55.9, a ~24/255 black-end lift that
   mechanically destroys a `max − min` chroma readout. **The re-shoot must be run with `--isolate`**,
   or the joint branch's hard-coded value ladder fixed first.

3. **The pale-AND-chromatic corner is not in the corpus** (§3.4). `huevalue` renders at s=0.55, so
   input chroma is tied to input value and no lever raises luminance; there are no cells at
   `L_in > 140` with `C_in < 30`. Everything §3 says about Olympia's regime is the *simulator*
   extrapolating, validated elsewhere but not measured there. A `huevalue`-style target with an
   independent value axis and saturation axis would close this in one frame.

4. **Lifting curves (γ < 1) were never photographed.** The corpus's gamma levels are 1.0/1.4/1.8/2.2,
   all darkening. §3.3's price list assumes that arriving at a luminance by lifting is equivalent to
   arriving there by any other route. That superposition is *verified* for white-point (R² 0.891, no
   residual wp term) and is untested for gamma below 1.0.

5. **Whether the panel's blue really sits 24° from the palette's.** It is the one hue direction where
   measurement and palette disagree materially, and it falls on the arc this panel is weakest across.
   The camera cannot arbitrate: no colour reference was in frame. **Needs a colorimeter or an in-frame
   ColorChecker** — this is the single measurement that would unlock everything §5 has to leave as a
   direction.

6. **Whether the chroma lever is ever *preferable*, as opposed to ever *additive*.** §2.4 shows the
   chroma lever only ever subtracts chroma, and that ADR-091's stated reason for wanting it is an
   encoding artefact. It does **not** show that a judge prefers more chroma on art; `huevalue` is a
   synthetic grid and false colour on skin is objectionable in a way a missing test patch is not.
   That is a panel-judgement question and ADR-084 governs it.

7. **Whether a hue-aware white-point is worth having.** §1.3 and §3.1 measure a ~50-unit spread in the
   luminance at which different hues die, which is a real argument that one global wp cannot be
   optimal for every work. It is not an argument that a per-hue wp is *implementable* — the lever is a
   scalar on luminance and hue-selective compression is a different operator, unmeasured here. It also
   sits against ADR-093, which chose a constant on **human judgements of whole works**, a criterion
   this corpus does not measure at all.

8. **A1's `huevalue` per-cell error bar is ~5x too tight for cross-condition use** (§0.1) and my
   replacement rests on **4 pairs**, two of which changed alignment branch when re-flat-fielded. The
   aggregate bars (row mean 23, grid mean 10) are the trustworthy part; the single-cell figure (σ 28.8)
   should be read as "single cells are unusable", not as a precise number.

9. **`n_collapsed`'s threshold is below its own error bar.** `readout_huevalue` calls a cell collapsed
   at `chroma_out < 6.0`, while a single cell's cross-condition σ is 28.8. The count is therefore a
   *noisy* statistic that happens to aggregate well over 72 cells; it should not be read cell by cell,
   and a threshold tied to the measured bar would be a better readout.

---

## Summary of what changes

| claim | status after this corpus |
|---|---|
| ADR-091: gamut is luminance-limited, wp is the gamut fix | **ACCEPTED** — superposition test, R² 0.891, no residual wp term; re-confirmed on randomised rows |
| ADR-091: chroma-kept table (419%/186%/58%/0%) | **WRONG SPACE** — reproduces exactly in the client *encoding*; in ink terms 55%/26%/10%/3% |
| ADR-091: panel over-saturates at low value → attenuate chroma next | **DO NOT ACTION** — the over-saturation is the encoding artefact above |
| ADR-091: "nothing is missing from the hue circle" | **OVERTURNED** — hues 150° and 330° are weak at every value; the limit is hue-dependent (corr +0.764, p 0.0023) |
| ADR-090: renderable ceiling at input 163 | **CORROBORATED INDEPENDENTLY** — measured chroma half-life at input 152 |
| wp x chroma interaction | **REAL, 3x** — chroma lever costs 43.7/255 at wp 0.88, 14.9 at wp 0.64 (randomised block 2) |
| the low-value block overturns the dark-end collapse | **VOID** — there is no low-value block (defect #9); the dark end is unmeasured |
| A1: `huevalue_lowv_*` value labels are wrong | **REVERSED** — the shipped labels are right; A1's relabelling must be discarded |
| inkmix: additivity fails, large dot gain | **REGISTRATION ARTEFACT** (defect #10); corrected reading is near-additive, coverage within ±0.06 |
| `SPECTRA6_DITHER_PALETTE` | **UNTOUCHED, and untouchable on this evidence** |

## STATUS: COMPLETE
