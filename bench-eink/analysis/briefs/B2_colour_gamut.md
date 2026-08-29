# B2 — Colour and gamut   [Opus]

**Read `briefs/STANDING_RULES.md` first, then `A1_integrity.md` and `error_bars.json`.**
A1's error bars are binding.

Targets you own: `huevalue` (54 conditions incl. the low-value block), `inkmix` (1), `primaries` (2).
**Do not touch `tonefine`, `edges`, `linepairs`, `resample`, `surround` — B1 owns those.**

## Outputs — yours alone
`bench-eink/analysis/B2_colour_gamut.md` (incremental) and `bench-eink/analysis/B2_findings.json`.

## Tasks

### 1. ADR-091 on glass — the headline
ADR-091 claims the gamut is LUMINANCE-limited, not hue-limited, and that white-point is therefore the
gamut fix. Its supporting table was produced BY SIMULATION and never checked against the panel until
this corpus. Adjudicate it. The simulated claim: every hue survives at v=100, six collapse to zero by
v=220.

⚠️ **The `huevalue_lowv_*` block (v=20..100) is separate and was shot LAST, after the panel was
rotated and with the camera re-locked at gain 255 — it has its own flat field (`flat_final.png`).**
Treat it as its own block; do not pool it naively with the main v=40..245 block. It exists because the
main block's bottom row measured 4-8/255 of chroma, BELOW the dark floor, so "chroma dies at low value"
could not be distinguished from "the instrument cannot see chroma there". Preliminary read: at v=36 the
low-value block measures 44.0/255 with 1 of 12 hues collapsed, against 8.2/255 and 8 collapsed at v=40
in the main block — i.e. the apparent low-value collapse was an ARTIFACT. Confirm or overturn this.

### 2. The white-point x chroma interaction
Preliminary read to confirm or overturn: with chroma-gamma 2.0 at the top of the value range,
`wp off` leaves chroma 2.5 with all 12 hues collapsed while `wp 0.64` leaves 49.7 with one — white-point
protecting colour against the chroma lever by ~20x. If real, this is a plausible reason the ADR-088
chroma work failed: chroma was judged at a luminance where it had already lost. Test it properly across
the crossed design, not just at the extremes.

### 3. The optical mixing law (`inkmix`)
15 ink pairs x 5 known ratios, plus an element-size sweep and a metamer block. Undithered by
construction, so this is a pure panel invariant.
- **Additivity must be tested in LINEAR LIGHT, not in gamma-encoded values.** A 1:1 black/white
  checkerboard measures 178.9 against a linear-light prediction of 188; computing this in encoded values
  produced nonsense (errors to 234/255).
- Characterise the **dot gain**: high coverage measures far darker than linear (66 where linear predicts
  137). Give the curve.
- **Element-size sweep** (1/2/4/8/16 px checkers): differences across pitch are panel dot-spread plus
  camera MTF. This also bears on the unexplained strip-vs-field patch-size effect (40-81/255 on
  chromatic inks) — see if it explains it.
- **Metamer block**: do different mixtures the renderer computes to the same target measure the same?
  If not, additivity fails and the quantiser's distance model is on sand.

### 4. Measured inks vs the palette
`SPECTRA6_DITHER_PALETTE` is Pimoroni's measurement of a DIFFERENT EL133UF1. Compare — **direction
only, never magnitude**, and normalise the palette to the same black=0/white=255 convention before
comparing. ⚠️ Rule 1 is absolute: do not propose rewriting the palette. State what a proper answer
would require (a colorimeter or a ColorChecker) rather than reaching past the data.

### 5. What you could not resolve
Explicit section.

Close with `## STATUS: COMPLETE`.
