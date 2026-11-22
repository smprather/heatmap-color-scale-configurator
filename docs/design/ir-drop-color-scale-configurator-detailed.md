# IR-drop heat-map colour-scale configurator — detailed design

Status: complete description of the design as of 2026-11-21. Target toolkit: **Qt 5**.

This is the long-form, normative description. The short overview is
[`ir-drop-color-scale-configurator.md`](ir-drop-color-scale-configurator.md); where the two
disagree, this document wins. The clickable reference is [`mockup/index.html`](../../mockup/index.html).
Prior art (how an established EMIR tool configures its colour scales) is in
[`../research/prior-art-color-scales.md`](../research/prior-art-color-scales.md).

## 0. How to read this document

- **Mock-up = reference implementation.** Every behaviour below was read out of the mock-up's
  source; every number was produced by running it. §18 records the 18 places where the first
  mock-up differed from this design and how each was fixed.
- **Normative words.** *Must* = the native product has to match this exactly (mostly the
  colour mapping, §4). *Should* = intended behaviour; deviations need a reason. *Mock-up:*
  marks a fact about the mock-up that is not a requirement.
- **Units.** Values are in mV unless a sentence says `% VDD`. "Axis" means `min…max`;
  "window" means `cool…hot`.
- **Numbers in examples** come from the synthetic fixture (§16) in its default state
  (metric IR, view `dvd_vectorless`, layer `INST`, all domains, mV).
- **Section map.** 1 purpose · 2 glossary · 3 data model · 4 colour mapping · 5 statistics ·
  6 window layout · 7 distribution & range widget · 8 tabs · 9 live preview · 10 KPI cards ·
  11 insights · 12 presets · 13 commands, undo, apply · 14 render pipeline · 15 formatting ·
  16 fixture · 17 Qt 5 implementation · 18 resolved mock-up deviations · 19 acceptance
  checklist · 20 open questions · 21 decision log.

## 1. Purpose and context

### 1.1 The problem

An IR-drop heat map paints every standard-cell instance (or wire segment, or tile) by its
voltage drop. The colour scale decides what the map says. The same data can look clean or
alarming depending on four numbers and a palette, so a reviewer has to trust the scale before
trusting the picture. Two different jobs pull the scale in opposite directions:

| job | needs from the scale |
|---|---|
| **Find hotspots** (PDN designer iterating on the grid) | a window tight around where the data is, so gradients inside the hot regions stay visible; non-linear transfer for long-tailed distributions; perceptually ordered palettes |
| **Argue signoff** (signoff engineer, reviewer) | the top colour pinned to the budget so "red" provably means "over limit"; a distinct over-limit colour; counts of offending instances; the same scale across layers and views so maps are comparable |

The configurator serves both and makes it obvious which one the current scale is set up for.

### 1.2 Users and situations

- Signoff and PDN engineers working a full-chip run with millions of instances, often on a
  compute farm through VNC or X forwarding (affects animation and repaint policy, §14, §17).
- Reviewers reading a dumped map or a screenshot; the legend has to carry the scale's meaning
  on its own (extend triangles, threshold ticks, units, scope title).
- Multi-supply designs (here VDD_CORE 0.75 V and VDD_SRAM 0.85 V) where one mV number means
  different margins in different domains.

### 1.3 Lineage

| | Classic banded tool | Reference ramp dialog | This design |
|---|---|---|---|
| scale type | banded table, 6 rows by default | continuous ramp, two handles | both: Ramp mode and Bands mode |
| parameters | band boundaries | `min, cool, hot, max` | `min, cool, hot, max` + band edges |
| out of range | rows can be disabled | clamp; `cool_is_black` | clamp / black / transparent / custom below; clamp / custom above |
| direction | fixed (red = worst) | `flip` | `flip` |
| guidance | none | full-chip histogram behind the bar | histogram coloured by the scale, log counts, CDF, percentiles, threshold flags, live counts |
| scope | per metric | per metric × layer × view | range per metric × view × layer (with "link layers" and "lock across views"); style per metric |
| multi-VDD | `% VDD` against one reference domain (manual calls this a trap) | unknown | `% VDD` per instance against **its own** supply |
| scripting | `config colormap …`, `dump gif` | unknown | JSON scale definition; classic Tcl export as approximation |

### 1.4 Goals, non-goals, anti-goals

Goals

1. Tune a scale against the data it paints: histogram, CDF, percentiles and thresholds share
   the bar's x-axis; dragging a handle shows the effect on the map at once.
2. Make thresholds first-class: evaluated per instance against its own supply, drawn on the
   axis and legend, counted live, and usable as snap targets, band anchors and the centre of a
   diverging map.
3. Make maps comparable: one scale shared across layers and/or locked across views.
4. Never make a colour decision silently: clamped regions are hatched, disabled bands show the
   layout through, saturation is counted, the legend shows clamp colours.
5. Produce a portable scale definition (JSON) and a best-effort classic Tcl equivalent.
6. Port to a native Qt 5 GUI without redesign: every visual is QPainter-reproducible.

Non-goals

- Rendering the production full-chip map. The configurator previews; the host's renderer
  paints the real map, using the same mapping code (§4, §17.2).
- A bridge into the existing EMIR tool (its colormap scripting interface is unknown).
- Editing analysis data, thresholds policy files or the design.

Anti-goals (failure even if it "works")

- A saturated region that reads as "hottest in range" rather than "off the scale".
- Multi-VDD maps silently normalised to a single reference.
- A palette that draws false edges (lightness reversals) without the tool saying so.
- A preview whose colours differ from the production renderer's for the same scale.

### 1.5 Design principles

1. **One mapping function.** Histogram bars, the ramp strip, the legend, the preview and the
   production renderer all call the same `colour(v)` (§4.9). Nothing is painted from a
   separately computed gradient.
2. **Numbers and pictures agree.** Every count on screen (flags, KPIs, band rows, saturation)
   is computed from the same filtered value set the histogram shows.
3. **Show the consequence where the cause is.** Threshold counts sit on the threshold flag;
   band counts sit in the band row; the hover on a histogram bin lights those instances on the
   map, and hovering the map marks the value on the histogram.
4. **Suggest, don't force.** Insights (§11) explain problems and offer one-click fixes; they
   never change the scale by themselves.
5. **Preview freely, commit deliberately.** Edits apply to the preview at once; **Apply** hands
   the scale to the host; the title says when the two differ.

## 2. Glossary

| term | meaning |
|---|---|
| instance | a placed cell with one IR-drop value; the unit of all counts |
| tile | a cell of the data raster: value, domain and instance count. *Mock-up:* 10 µm tiles (§16) |
| metric | the quantity mapped: `IR` (effective VDD + VSS drop), `IR_VDD`, `GB_VSS` (ground bounce) |
| layer | `INST` (instances) or a metal/RDL layer (`M1`, `M2`, `M4`, `M7`, `M10`, `AP`) |
| view | one analysis result: `static_ir`, `dvd_vectorless`, `dvd_vcd_func` |
| analysis kind | `static` or `dynamic`, derived from the view; selects the threshold set |
| domain | a supply net group with its own VDD (`VDD_CORE` 750 mV, `VDD_SRAM` 850 mV) |
| `% VDD` | drop divided by the instance's **own** domain VDD × 100 |
| reference VDD | the single VDD used to draw a %-threshold on an mV axis and to convert range numbers between units: VDD_SRAM's when the domain filter is VDD_SRAM, otherwise VDD_CORE's |
| scope key | `metric \| view \| layer`, with `*` for view when locked and for layer when linked |
| scale | the per-scope range object `{min, max, cool, hot, edges}` (§3.5) |
| style | palette, interpolation, mode, transfer, out-of-range rules, flip, band settings; one per metric (§3.7) |
| axis | `min…max`, the extent of the histogram and the ramp track |
| window | `cool…hot`, where the ramp (or bands) is painted |
| clamp | values outside the window take an end colour (or an out-of-range colour) |
| band | in Bands mode, one interval of the window with its own colour and enable flag |
| edge | an interior band boundary (absolute value) |
| LUT | 1024-entry RGB table sampled from the palette |
| transfer | function from normalised window position to ramp position (§4.2) |
| threshold | named %-VDD level with colour and enable flag; `warn` and `limit` always exist |
| violation | an instance with `pct > limit.pct` |
| cluster | a 4-connected group of violating tiles |
| brushing | highlighting on the map the instances in a hovered histogram bin |
| snap | magnetic attraction of a dragged handle/edge to statistically meaningful values |
| commit | record the current state as one undo step |
| apply | hand the current state to the host renderer; becomes the Revert target |

## 3. Data model

### 3.1 What the configurator reads

The user confirmed on 2026-11-21 that the mock-up's data is close to what the real GUI will
read: a tile raster per analysis result. Per (metric, view, layer):

| item | type | used for |
|---|---|---|
| tile value | float, mV, NaN = no data | everything |
| tile domain | small int → domain table | `% VDD`, domain filter, threshold evaluation |
| tile instance count | integer, the same in every view | weighting every statistic and count |

Per design: domain table `{id, vdd_mV, colour}`; floorplan / hierarchy (owning block per
tile); bump map including missing bumps; layout backdrop (die and core outline, macros,
keep-outs, pads); design name and run id. Per view: analysis kind.

**Counting rule.** Every instance in a tile counts as having the tile's value, so all counts,
percentiles and fractions are instance-weighted (§3.4). What the tile value means (worst
instance in the tile, mean, …) is the source's definition; with worst-instance tiles,
"instances over the limit" is an upper bound (§20).

*Mock-up:* 200 × 150 tiles of 10 µm. Instance counts come from floorplan block densities
(§16), 12 170 017 instances in all. A `DESIGN` descriptor supplies the name `demo_soc`, run
`r01` and `synthetic: true` (shown as a `synthetic data` tag in the status bar).

### 3.2 Domains

| id | VDD | colour | where (mock-up) |
|---|---|---|---|
| `VDD_CORE` | 750 mV | `#41e2ff` | everything not SRAM |
| `VDD_SRAM` | 850 mV | `#c792ff` | tiles x 96–137, y 94–139 |

Domain filter values: `all`, `VDD_CORE`, `VDD_SRAM`. Filter index: −1 / 0 / 1.

### 3.3 Metrics, layers, views

Metrics (combo label → code): `IR drop · effective (VDD + VSS)` → `IR`;
`IR drop · VDD rail only` → `IR_VDD`; `Ground bounce · VSS rail` → `GB_VSS`.

Layers (combo label): `Instances` (`INST`), `M1`, `M2`, `M4`, `M7`, `M10`, `AP (RDL)` (`AP`).

Views (combo label): `static_ir  (static)`, `dvd_vectorless  (dynamic)`,
`dvd_vcd_func  (dynamic)`. The kind picks the threshold set (§3.8).

### 3.4 Derived statistics

Computed when metric, view, layer, units or domain filter changes, and **cached per that
key** (12 entries, oldest dropped first), so switching back is instant.

- `vals` = tile `mv` or `pct` depending on units.
- Filtered tiles = finite value, instance count > 0, and (filter = all or domain = filter).
- `n` = sum of the filtered tiles' instance counts; `domCount[d]` = the same per domain.
- Sorted arrays: filtered tile values ascending with cumulative instance counts (`sv`, `cw`,
  `cw[0] = 0`), and the same for `pct` (`sp`, `cp`).
- `below(v)` = instances with value < v = `cw[lowerBound(sv, v)]`;
  `atMost(v)` = instances with value ≤ v = `cw[upperBound(sv, v)]`;
  `pctAbove(t)` = instances with pct > t = `n − cp[upperBound(sp, t)]`.
- `valueAtRank(r)` = value of the tile holding instance rank `r` (0-based): the smallest `k`
  with `cw[k+1] > r`, then `sv[k]`.
- **Percentile (nearest rank over instances):**
  `q(p) = valueAtRank(clamp(round(p/100 · (n−1)), 0, n−1))`. No interpolation. Must match in
  the product so snap targets and P-labels agree.
- `min = sv[0]`, `max = sv[last]`, `mean` = instance-weighted mean.
- **Worst:** the filtered tile with the largest **`pct`** (own-domain % VDD), regardless of
  display units. In mV it is therefore the worst-against-budget tile, which need not hold the
  largest mV value. This is deliberate (an 80 mV drop on an 850 mV rail is less serious than
  78 mV on a 750 mV rail).
- **Fraction below a value** (hover P-labels): `below(v) / n`.

### 3.5 The scale object

One per scope key, created lazily on first visit.

| field | meaning | default (`defaultScale`) |
|---|---|---|
| `min` | axis start | `floor(D.min / step) · step` |
| `max` | axis end | `ceil(D.max / step) · step` |
| `cool` | window start, ramp cold end | `= min` (the nice-rounded axis start, not the data min) |
| `hot` | window end, ramp hot end | `q(99.5)` rounded to 3 significant digits |
| `edges[]` | `bandCount − 1` interior band boundaries, absolute values | generated by the current generator (§8.3) |

`step = niceStep((D.max − D.min) / 24)`, where `niceStep(raw)` picks 1, 2, 5 or 10 × 10^k:
with `p = 10^floor(log10 raw)` and `f = raw/p`: `f < 1.5 → 1`, `< 3 → 2`, `< 7 → 5`, else 10.
Example: data 1.015…83.62 mV → raw 3.44 → step 5 → axis 0…85; P99.5 = 62.618 → hot 62.6.

Invariants, enforced by `clampScale` after every change and on every render:

```
if not (max > min): max = min + 1
eps  = (max − min) · 1e-3
cool = clamp(cool, min, max − eps)
hot  = clamp(hot, cool + eps, max)
```

Edge invariant (`clampEdges`): walking left to right, each edge is clamped into
`[previous, hot]`, starting with `previous = cool`. So edges never cross and never leave the
window; when a handle is dragged past them they are **pushed** and pile up against it.

Edge rescaling (`rescaleEdges(old_cool, old_hot)`): maps each edge proportionally from the
old window to the new one. Applied when cool/hot change through the number fields, a
histogram range selection, or an auto-range action (except with the Equal generator, which
regenerates instead). Not applied while dragging handles (push semantics above). Presets
regenerate edges with their own generator (§12).

### 3.6 Scope keys and sharing

- Key: `${metric}|${lockViews ? '*' : view}|${linkLayers ? '*' : layer}`.
- Defaults: `linkLayers = true`, `lockViews = false` → default key `eff|dvd_vectorless|*`.
  (`eff` is the mock-up's internal id for metric `IR`.)
- Switching metric, view or layer selects the scale under the new key, creating a default
  one if absent. Previously visited keys keep their ranges.
- Toggling Link layers or Lock across views: the current range is deep-copied to the new key
  **only if that key has no scale yet**. Then a toast: `One scale now shared by all layers` /
  `Scale locked across views — switch views to compare` (turning on) or `Scale is now per
  layer` / `Scale is now per view` (turning off).
- Scope chip text in the distribution header:

| link layers | lock views | chip |
|---|---|---|
| on | on | `shared · all layers, all views` |
| on | off | `shared · all layers` |
| off | on | `shared · all views` |
| off | off | `this layer & view only` |

The chip describes the **range**. Style is per metric (§3.7) and the tab bar says so
(`style applies to every IR map`); thresholds are per analysis kind (§3.8).

### 3.7 Style (per metric)

`cfg.styles[metric]`, created from the defaults below the first time a metric is shown. All
views and layers of a metric share it. Rationale: ranges genuinely differ by view and layer
(static ≈ ⅓ of dynamic, AP ≈ 1/10 of INST), while palette, transfer and out-of-range rules are
conventions of the quantity being mapped; classic EMIR tools also had one colour dialog per
metric. (Claude's call, 2026-11-21, open to change.)

| key | values | default |
|---|---|---|
| `mode` | `ramp`, `bands` | `ramp` |
| `flip` | bool | false |
| `under` | `clamp`, `black`, `transparent`, `custom` | `black` (mirrors `cool_is_black` = on in the reference screenshot) |
| `underColor` | `#rrggbb` | `#26303d` |
| `over` | `clamp`, `custom` | `clamp` |
| `overColor` | `#rrggbb` | `#ff3df0` |
| `palette` | `{name, stops: [{p ∈ [0,1], c: #rrggbb}]}`, kept sorted by `p` | Turbo (copy) |
| `interp` | `oklab`, `rgb` | `oklab` |
| `transfer` | `linear`, `sqrt`, `log`, `gamma`, `equalize`, `center` (two-slope) | `linear` |
| `gamma` | 0.25…4, step 0.05 | 0.6 |
| `bandCount` | 2…12 | 6 |
| `bandGen` | `equal`, `quantile`, `thresholds` | `equal` |
| `bandFollow` | bool (band colours follow palette) | true |
| `bands.colors[]` | `#rrggbb` × bandCount, storage order = un-flipped | sampled from LUT |
| `bands.enabled[]` | bool × bandCount | all true |

### 3.8 Thresholds

Two lists, keyed by analysis kind. Each entry `{id, name, pct, color, on}`.

| kind | warn | limit |
|---|---|---|
| static | 3 % VDD, `#ffb547` | 5 % VDD, `#ff4d6d` |
| dynamic | 8 % VDD, `#ffb547` | 10 % VDD, `#ff4d6d` |

- `warn` and `limit` always exist (no delete button); they can be disabled.
- Custom thresholds: added with name `Custom`, `pct = round(0.6 · limit.pct, 2 dp)`, colour
  `#7cf0b0`, enabled, id `u` + base-36 timestamp; deletable.
- A threshold's **value in display units**: `pct` in % VDD mode; `pct/100 · refVDD` in mV
  mode. Counting always uses each instance's own `pct` (§5.4); only drawing uses refVDD.
- Thresholds are shared across all scopes, layers and metrics (policy, not presentation).

Where thresholds are used:

| use | which thresholds |
|---|---|
| dashed line + flag with count on the histogram | enabled |
| snap targets | enabled |
| legend ticks | enabled, inside the window |
| band generator "Threshold-anchored" | enabled, strictly inside the window |
| tooltip badge | limit, then warn (enabled) |
| KPI cards | limit, warn |
| violation mask, clusters, contour, glow | limit (enabled) |
| `0 → limit`, `Margin @ limit`, presets that need the limit, two-slope transfer, "Set hot = limit" | limit, **enabled**: these controls are disabled while it is off; a two-slope map that is already selected falls back to the window midpoint |
| counts table | all |

### 3.9 View state

Not undoable, not applied, not exported. Changing it never sets the modified indicator.

| key | meaning | default |
|---|---|---|
| `metric`, `layer`, `view` | which map is shown | `eff` (IR), `INST`, `dvd_vectorless` |
| `domain` | domain filter | `all` |
| `cvd` | colour-vision simulation of the preview: `none`, `deutan`, `protan`, `tritan`, `gray` | `none` |
| `legend`, `bumps`, `domains` | preview overlays | on |
| `blocks` | floorplan block outlines on the preview | off |
| `smooth` | bilinear preview scaling | off |
| `opacity` | heat-layer opacity 0.2…1 | 0.92 |
| `logY`, `showCdf`, `showPctl` | histogram decorations | on |
| `pulse`, `contour` | limit-violation glow / outline | on |
| `snap` | magnetic snapping | on |

### 3.10 State partitions

| part | contents | undo | Apply | export |
|---|---|---|---|---|
| **scale set** (`cfg`) | `units`, `linkLayers`, `lockViews`, `numFmt` (`eng`), `legendFont` (11 px; 7…32, the classic tools' range), thresholds, styles per metric, ranges per scope key | yes | yes | yes (current scope) |
| **view state** (`vw`) | §3.9 | no | no | no |
| **session UI** (`ui`) | tab, export format, active handle, selected stop, drag, hover, brush, selection, snap label, locate time | no | no | no |

Units belong to the scale set because range numbers are stored in display units.

**Automatic defaults are not edits.** A range or style created because a scope or metric was
shown for the first time is recorded as *pristine* after the render that created it.
Snapshots (§13) leave out entries that still equal their pristine form, so browsing never
creates an undo step or the modified indicator, and editing an entry back to its default
counts as unchanged.

## 4. Colour mapping (normative)

Everything in this section is a *must*. The production renderer and the configurator share
one implementation (§17.2) and are tested against the vectors in §4.10.

### 4.1 Palette → LUT

1. Sort stops by `p` (the editor keeps them sorted; sorting again is harmless).
2. For `i = 0…1023`, `t = i / 1023`:
   - `t ≤ p_first` → first stop's colour; `t ≥ p_last` → last stop's colour;
   - else find `k` with `p_k ≤ t ≤ p_{k+1}`, `u = (t − p_k)/(p_{k+1} − p_k)` (0 if equal),
     interpolate stop colours `k`, `k+1` by `u`:
     - `oklab`: convert both to OKLab, lerp L, a, b, convert back, clamp to 0…255;
     - `rgb`: lerp the 8-bit sRGB components directly.
3. Store as `LUT[i] = (r, g, b)`, components stored in 8-bit with clamping and rounding to
   nearest (the mock-up uses a `Uint8ClampedArray`, which rounds half to even).
4. Also build `DLUT[i] = cvd(LUT_unrounded[i], st.cvd)` for the preview (§4.8).

sRGB ↔ linear (per channel, c in 0…255):

```
s2l(c) = (c/255 ≤ 0.04045) ? (c/255)/12.92 : ((c/255 + 0.055)/1.055)^2.4
l2s(x) = 255 · (x ≤ 0.0031308 ? 12.92·x : 1.055·x^(1/2.4) − 0.055)
```

OKLab (Björn Ottosson's matrices, exact constants used by the mock-up):

```
rgbToOklab(R,G,B):
  r,g,b = s2l(R), s2l(G), s2l(B)
  l = cbrt(0.4122214708 r + 0.5363325363 g + 0.0514459929 b)
  m = cbrt(0.2119034982 r + 0.6806995451 g + 0.1073969566 b)
  s = cbrt(0.0883024619 r + 0.2817188376 g + 0.6299787005 b)
  L = 0.2104542553 l + 0.7936177850 m − 0.0040720468 s
  a = 1.9779984951 l − 2.4285922050 m + 0.4505937099 s
  b = 0.0259040371 l + 0.7827717662 m − 0.8086757660 s

oklabToRgb(L,a,b):
  l = (L + 0.3963377774 a + 0.2158037573 b)^3
  m = (L − 0.1055613458 a − 0.0638541728 b)^3
  s = (L − 0.0894841775 a − 1.2914855480 b)^3
  R = l2s( 4.0767416621 l − 3.3077115913 m + 0.2309699292 s)
  G = l2s(−1.2684380046 l + 2.6097574011 m − 0.3413193965 s)
  B = l2s(−0.0041960863 l − 0.7034186147 m + 1.7076147010 s)
  clamp each to [0, 255]
```

### 4.2 Transfer functions

Input `t0 = (v − cool) / span`, `span = max(1e-9, hot − cool)`, for in-window values
(`0 ≤ t0 ≤ 1`). Output `t` (ramp position before flip).

| transfer | UI label | formula | note text shown under the curve |
|---|---|---|---|
| `linear` | Linear | `t = t0` | Equal value steps → equal color steps. |
| `sqrt` | √x | `t = sqrt(t0)` | Expands the low end; good for long-tailed IR maps. |
| `log` | Log | `t = ln(1 + 60·t0) / ln(61)` | Strong low-end expansion (log1p, k = 60). |
| `gamma` | γ | `t = t0^γ`, γ ∈ [0.25, 4] | Power curve; γ < 1 expands lows, γ > 1 expands highs. |
| `equalize` | Equalize | `t = EQ(t0)`, see below | Each color covers the same number of instances (histogram equalization). |
| `center` | Two-slope | see below | Lower half of the palette spans cool → limit, upper half limit → hot. Pair with a diverging palette. |

**Equalize.** A 1025-entry table from binary searches on the sorted, instance-weighted
values: `base = below(cool)`, `total = atMost(hot) − base`,
`EQ[k] = (below(cool + k/1024 · span) − base) / total` for `k = 0…1023` (or `k/1024` if no
in-window instances), `EQ[1024] = 1`. Lookup: `f = t0 · 1024`, `k = floor(f)`; `t = 1` if
`k ≥ 1024`, else `EQ[k] + (EQ[k+1] − EQ[k]) · (f − k)`. Rebuilt on every render while
`equalize` is active (1025 binary searches, no pass over the data).

**Two-slope.** `C` = the limit threshold's value in display units while the limit is enabled;
with the limit disabled, the window midpoint. `tc = clamp((C − cool)/span, 0.02, 0.98)`.
`t = t0 < tc ? 0.5 · t0/tc : 0.5 + 0.5 · (t0 − tc)/(1 − tc)`.
So the palette midpoint lands exactly on the limit.

### 4.3 Ramp mode

```
v NaN                → nodata (transparent)
v < cool             → below colour   (kind 1)
v > hot              → above colour   (kind 2)
else                 → t = transfer(t0); if flip: t = 1 − t
                       LUT[round(clamp(t,0,1) · 1023)]   (kind 3)
```

`v == cool` and `v == hot` are in range.

### 4.4 Bands mode

`E = [cool, edges…, hot]`, `n = bandCount = len(E) − 1`.

```
v NaN                → nodata
v < E[0]             → below colour (kind 1)
v > E[n]             → above colour (kind 2)
k = largest index in 0…n−1 with v ≥ E[k]   (lower-inclusive; v == hot falls in band n−1)
band k disabled      → transparent (kind 4), like classic -disable
else                 → bandColour'[k]      (kind 3)
```

`bandColour' = flip ? reverse(bands.colors) : bands.colors`. The enable flags are **not**
reversed: they belong to value intervals, colours belong to the palette.

### 4.5 Out-of-range colours

| setting | below `cool` | above `hot` |
|---|---|---|
| `clamp` | ramp: `LUT[flip ? 1023 : 0]`; bands: `bandColour'[0]` | ramp: `LUT[flip ? 0 : 1023]`; bands: `bandColour'[n−1]` |
| `black` (= the reference tool's `cool_is_black`) | `#000000` | — |
| `transparent` (UI "None") | alpha 0: layout shows through | — |
| `custom` | `underColor` | `overColor` |

### 4.6 Flip

Flip reverses the ramp (and band colours). It is applied **after** the transfer function, so
`sqrt` still expands the low *values*, whichever colour they get.

### 4.7 No data

NaN maps to alpha 0 everywhere (preview, ramp, legend). The ramp and legend draw a
checkerboard under transparent colour so "transparent" is visible as such.

### 4.8 Colour-vision simulation (preview only)

Machado, Oliveira & Fernandes (2009), severity 1.0, applied in **linear** RGB, results
clamped to [0, 1] before re-encoding:

```
protan = [ 0.152286  1.052583 −0.204868 ;  0.114503  0.786281  0.099216 ; −0.003882 −0.048116  1.051998 ]
deutan = [ 0.367322  0.860646 −0.227968 ;  0.280085  0.672501  0.047413 ; −0.011820  0.042940  0.968881 ]
tritan = [ 1.255528 −0.076749 −0.178779 ; −0.078411  0.930809  0.147602 ;  0.004733  0.691367  0.303900 ]
gray   : Y = 0.2126 r + 0.7152 g + 0.0722 b (linear), R = G = B = l2s(Y)
```

Applied to: preview heat layer, histogram bars, ramp strip, legend, stop-editor bar, transfer
curve strip, band and out-of-range colours (through `disp(hex)`). **Not** applied to: the
lightness plot (uses `LUT`), band colour sampling (uses `LUT`), exports. An insight says
`Previewing as <mode>. Export is unaffected.`

### 4.9 Reference pseudocode

```
prepare():                                   # once per render, after LUT build
  s = scale(scopeKey)
  M.cool, M.hot, M.span = s.cool, s.hot, max(1e-9, s.hot − s.cool)
  M.flip, M.mode, M.tf, M.gamma = flip, mode, transfer, gamma
  C = value(limit) if limit enabled else (s.cool + s.hot)/2
  M.tc = clamp((C − s.cool)/M.span, 0.02, 0.98)
  if mode == bands:
    M.edges = [s.cool, s.edges…, s.hot]
    cols = map(disp, bands.colors); if flip: reverse(cols)
    M.bandCols, M.bandOn = cols, copy(bands.enabled)
    M.lo, M.hi = cols[0], cols[last]
  else:
    M.lo, M.hi = DLUT[flip ? 1023 : 0], DLUT[flip ? 0 : 1023]
  M.under = {clamp: M.lo, black: (0,0,0,255), transparent: (0,0,0,0), custom: disp(underColor)}[under]
  M.over  = {clamp: M.hi, custom: disp(overColor)}[over]

transfer(t0):
  linear: t0 · sqrt: √t0 · log: ln(1+60 t0)/ln 61 · gamma: t0^γ
  equalize: EQ lookup (§4.2) · center: two-slope (§4.2)

colour(v) → (rgba, kind):                    # kind 0 nodata, 1 below, 2 above, 3 in, 4 disabled
  if v is NaN: return (transparent, 0)
  if M.mode == bands:
    if v < M.edges[0]: return (M.under, 1)
    if v > M.edges[n]: return (M.over, 2)
    k = 0; while k < n−1 and v ≥ M.edges[k+1]: k += 1
    if not M.bandOn[k]: return (transparent, 4)
    return (M.bandCols[k], 3)
  if v < M.cool: return (M.under, 1)
  if v > M.hot:  return (M.over, 2)
  t = transfer((v − M.cool)/M.span); if M.flip: t = 1 − t
  return (DLUT[round(clamp(t, 0, 1) · 1023)], 3)
```

The production renderer calls the same code with `cvd = none` (so `DLUT == LUT`).

### 4.10 Golden vectors

Scale set to axis 0…85, cool 0, hot 59.5 (mV) — the vectors fix this window; the fixture's
default hot is 62.6 — with Turbo, OKLab, `under = black`, `over = clamp`, unless the case
says otherwise. Results from the mock-up; a native
implementation must reproduce them, with at most ±1 per 8-bit channel where its maths library
rounds differently from the browser's (record any such case in the test).

| case | v (mV) | kind | colour |
|---|---|---|---|
| default | NaN | nodata | transparent |
| default | −1 | below | `#000000` |
| default | 0 | in | `#23171b` |
| default | 5 | in | `#4a4dc6` |
| default | 14.875 (t0 = .25) | in | `#26bcdf` |
| default | 29.75 (t0 = .5) | in | `#96fa50` |
| default | 44.625 | in | `#fd811d` |
| default | 59.5 | in | `#900d00` |
| default | 59.51 | above | `#900d00` |
| default | 90 | above | `#900d00` |
| flip | 0 / 14.875 / 59.5 | in | `#900d00` / `#fd811d` / `#23171b` |
| sqrt | 14.875 / 29.75 | in | `#96fa50` / `#fe9e22` |
| log | 14.875 / 29.75 | in | `#fbb326` / `#df4612` |
| gamma 0.6 | 14.875 / 29.75 | in | `#66fc6f` / `#f9bc28` |
| two-slope, cool 0, hot 85, limit 75 | 37.5 / 75 / 80 | in | `#26bcdf` / `#96fa50` / `#fd811d` |
| OKLab vs sRGB interp | 31.875 | in | `#b4f145` (OKLab) / `#b1f145` (sRGB) |
| Inferno, OKLab | 0 / 29.75 / 59.5 | in | `#000004` / `#ba3955` / `#fcffa4` |
| Inferno, sRGB | 29.75 | in | `#ba3853` |
| Inferno, OKLab, deutan preview | 29.75 | in | `#7a7252` |
| Inferno, OKLab, gray preview | 29.75 | in | `#696969` |
| bands, equal ×6 (edges 9.9167, 19.833, 29.75, 39.667, 49.583) | 0 / 9.9 / 9.9167 / 29.75 / 59.5 | in | `#4a4cc5` / `#4a4cc5` / `#26bcdf` / `#d5e237` / `#ad1b05` |
| same | 59.6 | above | `#ad1b05` |
| same, band 2 disabled | 25 | disabled | transparent |
| `under = clamp` | −1 | below | `#23171b` |
| `under = transparent` | −1 | below | transparent |
| `under = custom #26303d` | −1 | below | `#26303d` |
| `over = custom #ff3df0` | 60 | above | `#ff3df0` |

Band colours (Turbo, 6 bands, follow palette) sampled at band centres
`LUT[round((k + 0.5)/6 · 1023)]`: `#4a4cc5 #26bcdf #5afa7b #d5e237 #fd811d #ad1b05`.

## 5. Statistics and derived overlays

### 5.1 Histogram

- Bin count `nb = clamp(round(plotWidth / 4.5), 60, 240)` (≈ one bin per 4.5 px; 215 at the
  1720 px test window).
- Bins span the **axis** `[min, max]`, `bin = (max − min)/nb`. Counts are instance counts from
  binary searches at the bin edges: bin `b` = `below(edge_{b+1}) − below(edge_b)`, the last bin
  = `atMost(max) − below(edge_{nb−1})`.
- Bin counts depend only on data and axis and are **cached** per (data key, min, max, nb);
  dragging handles never recounts.
- Counters, all by binary search: `below` = `below(min)`, `above` = `n − atMost(max)`,
  `under` = `below(cool)`, `over` = `n − atMost(hot)`, `inr = n − under − over`.
- `lowQuarter` = fraction of in-window instances whose transfer position `t < 0.25`: the value
  where `t` reaches 0.25 is found by bisection (40 steps; every transfer is monotonic), then
  `(below(v25) − under) / inr`.

### 5.2 CDF

Cumulative count starting at `below`, normalised by `n`, one vertex per bin edge across the
plot. Right-side labels `0%`, `50%`, `100%`.

### 5.3 Percentile markers

P50, P90, P99, P99.9 (nearest rank, §3.4), drawn only if inside the axis.

### 5.4 Threshold evaluation

For each threshold `t` (enabled or not): `count = pctAbove(t.pct)` — instances with
`pct > t.pct`, strict, using each tile's own domain VDD. The domain filter applies.

### 5.5 Violation mask, clusters, contour

- Mask: tiles with instances whose `pct > limit.pct` (limit enabled), domain filter applied.
- Clusters: connected components of the mask, **4-connectivity** (no diagonals), iterative
  flood fill.
- Contour: for every masked tile, one unit segment along each side whose neighbour is
  unmasked or outside the grid; together they outline every cluster on tile boundaries.
  Drawn scaled to the preview (§9.3).
- Fixture default: 37 tiles, 22 050 instances (22.1 k), 1 cluster.

### 5.6 Worst tile and cause hint

Worst = §3.4. The cause hint is built the way the host would build it:

1. **Owning block** from the floorplan / instance hierarchy (mock-up: first floorplan block
   containing the tile, nested blocks listed before their parents; otherwise `soc`).
2. **Bump check:** append ` · missing bump` if any of the four bump sites around the tile (the
   bump cell containing it) is missing.

Text format: `<block path>[ · missing bump]`, e.g. `soc/npu · missing bump`.

## 6. Window and layout

### 6.1 Structure and sizes

Mock-up window: minimum 1440 × 900 px; rows 38 (title) / 54 (scope bar) / flexible (main) /
32 (status). Main: two columns, left flexible, right fixed **568 px**, 12 px gap and padding.

```
┌────────────────────────────────────────────────────────────────────────────────────────────┐
│ ▣ Heatmap Color Scale  IR::INST…dvd_vectorless           ● modified — not applied         ? │ 38
├────────────────────────────────────────────────────────────────────────────────────────────┤
│ METRIC        LAYER       [Link layers] │ VIEW            [Lock across views] │ SUPPLY DOMAIN │ 54
│ IR drop · ef▾ Instances ▾               │ dvd_vectorless ▾                    │ All domains ▾ │
│                                                                      SCALE PRESET  Apply… ▾ │
├───────────────────────────────────────────────────────────┬────────────────────────────────┤
│ DISTRIBUTION & RANGE  [shared · all layers] log CDF pctl   │ LIVE PREVIEW  Legend Bumps     │
│                                n 12.17 M mean P99 max      │  Domains Blocks Smooth Vision ▾│
│ ┌ 240 px canvas: percentile labels, histogram, flags,     ┐ │ ┌ 4:3 preview ───────────────┐ │
│ │ CDF, ramp strip with handles, axis                      │ │ │ legend │ heat map, bumps,  │ │
│ └─────────────────────────────────────────────────────────┘ │ │        │ worst, glow …     │ │
│ [min][cool][hot][max] │ mV|%VDD  Ramp|Bands  flip  cool_is_black │ └────────────────────────────┘ │
│ AUTO RANGE  Fit data · min→P99.5 · P1→P99 · 0→limit · Margin@limit │ x … µm  y … µm  mV  % of …  │
├───────────────────────────────────────────────────────────┤ ┌Worst drop┐ ┌Over limit┐       │
│ Palette 1 │ Mapping 2 │ Bands 3 │ Thresholds 4 │ Export 5    │ ┌Over warn ┐ ┌Saturation┐       │
│  (tab body scrolls)                                         │ SCALE INSIGHTS (scrolls)        │
├───────────────────────────────────────────────────────────┴────────────────────────────────┤
│ demo_soc · run r01 · 12.17 M instances · synthetic data   scale IR::INST…  ↶ ↷ │Reset range│Revert│Apply│ 32
└────────────────────────────────────────────────────────────────────────────────────────────┘
```

### 6.2 Title bar

- App icon (conic gradient in Turbo colours), title `Heatmap Color Scale`, scope subtitle
  `<CODE>::<layer>…<view>` (e.g. `IR::INST…dvd_vectorless`, the reference tool's title format).
- `● modified — not applied` in amber, visible when the current state differs from the last
  applied one (§13.2). Qt: `setWindowModified()` with `[*]` in the window title.
- Help button (tooltip `Shortcuts (?)`) opens the shortcut overlay. Window buttons are the
  native decorations' job; the mock-up draws none.

### 6.3 Scope bar

Left to right: Metric combo; Layer combo; `Link layers` toggle (tooltip `One scale shared by
every layer (fair layer-to-layer comparison)`, link icon); separator; View combo;
`Lock across views` toggle (tooltip `Keep the scale fixed when switching views (A/B
compare)`, lock icon); separator; Supply domain combo (`All domains`, `VDD_CORE · 0.750 V`,
`VDD_SRAM · 0.850 V`); stretch; Scale preset combo (placeholder `Apply a preset…`, groups
`Built-in` and `Saved`; resets to the placeholder after use). Field labels are 9.5 px
uppercase, letter-spaced, dim.

### 6.4 Status bar

Left: `demo_soc · run r01 · 12.17 M instances · 200×150 tiles @ 10 µm` from the data
source's design descriptor, plus a `synthetic data` tag when the source says so. Then `scale <title> · <mode> · <palette> · <transfer>`.
Right: Undo (tooltip `Undo (Ctrl+Z)`), Redo (`Redo (Ctrl+Shift+Z)`), both disabled when their
stack is empty; separator; `Reset range`; `Revert`; `Apply` (primary button).

### 6.5 Overlays

- **Toast**: bottom-centre, 48 px above the window bottom, cyan border, fades in/out over
  0.2 s, visible 1.9 s, one at a time (a new one replaces the old).
- **Help**: modal card 560 px wide on a 70 % black scrim; closes with `?`, `Esc` or a click on
  the scrim. Content:

| gesture | effect |
|---|---|
| drag handle | move cool / hot (snaps to P-values & limits) |
| drag gradient body | pan the cool–hot window |
| drag on histogram | select a range → becomes cool…hot |
| wheel on ramp / hist | zoom the window / zoom the axis |
| hover histogram | light up those instances on the map |
| ← → (Shift) | nudge the last-used handle |
| F · B · U | flip · ramp/bands · mV/%VDD |
| A · L · 0 | fit data · toggle legend · 0 → limit |
| 1 – 5 | switch tabs |
| Ctrl+Z · Ctrl+Shift+Z | undo · redo |
| Ctrl+Enter | apply |

### 6.6 Visual language

Dark theme, chosen because heat colours read best against a dark neutral surround. The plot
areas keep their own dark background even if the host app is light (§17.4).

| token | value | use |
|---|---|---|
| `bg0` | `#06080b` | outside the window |
| `win` | `#0b0f14` | window |
| `panel` / `panel-hi` | `#0f141b` / `#131a23` | panel body / top of panel gradient |
| `inset` | `#0a0e13` | input fields, wells |
| `line` / `line-hi` | `#1b2330` / `#263142` | separators / control borders |
| `text` / `muted` / `dim` | `#d6dfea` / `#8190a3` / `#56647a` | text hierarchy |
| `accent` | `#41e2ff` | focus, selection, cool handle, toggles on |
| `hot` | `#ff9c6b` | hot handle |
| `amber` | `#ffb547` | warnings, modified indicator, warn threshold |
| `danger` | `#ff4d6d` | limit threshold, violations, missing bump |
| `ok` | `#3ddc97` | healthy states |
| `violet` | `#c792ff` | VDD_SRAM |

Typography: sans `Inter`, `Fira Sans`, `Noto Sans`, system UI at 12.5 px base; mono
`JetBrains Mono`, `Fira Code`, `Noto Sans Mono` for every number, axis label, field and code.
Section labels 9.5–10 px uppercase bold with 1.2–1.5 px letter spacing. Radii: controls 6 px,
panels 9 px, window 10 px, chips 12 px. Controls 26 px tall (mini 20–22 px).

Toggles: off = plain bordered button; on = cyan-tinted fill, cyan border, faint glow.
Segmented controls: one bordered group, selected segment with a 2 px cyan underline.

## 7. Distribution & range widget

One custom-painted canvas (Qt: `ScaleAxisWidget`) holding histogram, CDF, percentiles,
threshold flags, ramp, handles and axis on **one shared x transform**. Height 240 px.

### 7.1 Geometry

| constant | value | meaning |
|---|---|---|
| `padL` / `padR` | 50 / 46 px | plot inset (room for y labels / CDF labels) |
| `hTop` / `hBot` | 22 / 166 px | histogram well |
| `rTop` / `rH` | 182 / 24 px | ramp strip (bottom `rB` = 206) |
| `pw` | width − 96 | plot width |

`x(v) = padL + (v − min)/(max − min) · pw`; `v(x)` is its inverse. Axis labels sit at
`rB + 16`; tick marks at `rB + 10…14`.

### 7.2 Paint order

1. Histogram well: `#090d12`, radius 4, border `#18202c`.
2. Y grid + labels (instances).
3. Histogram bars.
4. Brush highlight (hovered bin).
5. Overflow text (`◂ N below min`, `N above max ▸`).
6. CDF line + right labels.
7. Percentile ticks + labels.
8. Threshold lines, flags and ramp triangles.
9. Range selection (during histogram drag).
10. Ramp strip: checkerboard, colour columns, clamp dimming + hatch, disabled-band hatch,
    gloss, border.
11. Band-edge ticks (Bands mode).
12. Cool and hot handles (+ pills).
13. X axis ticks, labels, unit.
14. Hover marker (from histogram or map) unless dragging.

### 7.3 Layer details

**Y axis.** `maxC` = largest bin count. Bar height
`= (logY ? log10(1 + c)/log10(1 + maxC) : c/maxC) · (hBot − hTop − 12)`. Ticks in instances:
log → 10, 100, 1k, … up to `maxC` instances; linear → multiples of `niceStep(maxC/4)`. A tick
is skipped if within 6 px of the top or within 16 px of the previous label. Labels right
aligned at `padL − 7`, formatted `1k`, `10k`, `1M`; grid lines `rgba(140,160,190,.06)`.
Rotated label `instances` at x = 11 in `#3d4a5c`.

**Bars.** Each bar is filled with `colour(bin centre)` — the histogram is literally painted
by the scale. Below/above-window bins are drawn at 50 % opacity. Bars whose colour is
transparent or near black (`r + g + b < 50`) are drawn `#273041` so they stay visible. When a
bin is brushed, other bars drop to 35 % of their opacity. Bars wider than 3 px have a 0.5 px
gap each side; every bar gets a 1 px top highlight `rgba(255,255,255,.28)`.

**Brush.** Hovered bin column: fill `rgba(255,255,255,.06)`, stroke `rgba(255,255,255,.5)`.

**Overflow.** If any values lie outside the axis: `◂ <count> below min` at top-left,
`<count> above max ▸` at top-right (y = `hTop + 10`), muted.

**CDF.** Stroke `rgba(225,238,255,.55)`, 1.25 px. Labels `0%`, `50%`, `100%` at
`padL + pw + 7`; rotated `CDF` at `x = width − 8`.

**Percentiles.** For P50, P90, P99, P99.9 inside the axis: solid tick
`rgba(129,144,163,.55)` from y = 13 to `hTop`; dashed `[2,3]` line `rgba(129,144,163,.10)`
through the histogram; label at y = 10 (baseline) unless within 34 px of the previous label.

**Thresholds.** For each enabled threshold inside the axis (index `k` among enabled ones):
dashed `[4,3]` line in the threshold colour at 90 % alpha from `hTop` to `rB + 2`; a flag
`<NAME> <value>[ @<domain>] · <count>` (e.g. `LIMIT 75.0 mV @VDD_CORE · 22.1 k`; `@<domain>`
names the reference domain when supplies with different VDD share an mV axis) in 600-weight 10 px mono, box
height 15 px at `y = hTop + 20 + 19k`, background `rgba(8,11,16,.85)` + 18 % colour tint,
70 % colour border; placed right of the line, or left if it would overflow the plot. A
filled triangle (apex at `rB + 2`, 9 px wide, 6.5 px tall) marks the threshold under the
ramp.

**Range selection.** Fill `rgba(65,226,255,.12)` from `hTop` to `rB`, stroke
`rgba(65,226,255,.8)`, pill `<a> … <b>` at `y = hTop + 60`.

**Ramp strip.** Checkerboard 4 px (`#161d27` / `#212a37`) underneath. One colour per
**device pixel column** across the whole axis: `colour(min + (i + 0.5)/cols · (max − min))`,
drawn unsmoothed. So the clamped regions show the actual out-of-range colours (black,
checkerboard for transparent, custom colour). Then: clamped regions (left of cool, right of
hot) dimmed by `rgba(6,9,13,.30)` and hatched with 45° lines `rgba(255,255,255,.14)` every
5 px; disabled bands hatched `rgba(255,255,255,.35)` every 4 px; gloss gradient (white 26 % →
5 % at 45 %, transparent at 50 %, black 22 % at bottom); clip radius 5; border `#2c3849`.

**Band edges** (Bands mode, interior edges only). 2 px white bar (85 %, 100 % when hovered
or dragged) from `rTop − 3` to `rB + 3`, triangle with apex at `rTop − 8`; hovered/dragged
edge shows a pill with its value at `rTop − 20`.

**Handles.** 10 × 38 px (from `rTop − 7` to `rB + 7`), radius 4, fill `#eef8ff`, glow in
the handle colour (blur 9, 18 when hovered or dragged); a 5 px cap in the handle colour
(cool `#41e2ff`, hot `#ff9c6b`); three 5 px grip lines. A pill `cool 12.3 mV` (or `hot …`)
at `rTop − 20` shows while hovered or dragged, and on the active handle while the widget has
keyboard focus. While dragging and snapped, the pill reads `hot 75.0 mV  → limit`.

**X axis.** `step = niceStep((max − min) / max(2, pw/70))` (≈ one label per 70 px); labels
with `max(0, −floor(log10 step))` decimals; unit `mV` or `%VDD` right of the plot.

**Hover marker.** White line from `hTop` to `rB + 4` at the hovered value, with a pill
`<value> · P<pp.p>` at `hBot − 14` (two decimals above P99). Source: histogram hover or map
probe. Hidden while dragging.

### 7.4 Hit testing

| zone | condition | hit |
|---|---|---|
| ramp band | `rTop − 11 ≤ y ≤ rB + 11` | see below |
| — handle | within 8 px of cool or hot x | nearer handle; on a tie, cool if x < cool x, else hot |
| — band edge (Bands mode) | within 5 px of an interior edge | `edge k` |
| — pan | strictly between the handles | `pan` |
| — track | elsewhere within the plot | `track` |
| histogram | `hTop ≤ y < rTop − 11`, x within plot | `hist` |

Cursors: handle/edge `ew-resize`, pan `grab`, track `pointer`, histogram `crosshair`.

### 7.5 Pointer gestures

| hit | press | move | release |
|---|---|---|---|
| handle | grab; becomes active handle | `cool = clamp(snap(v), min, hot − eps)` / `hot = clamp(snap(v), cool + eps, max)` | commit |
| track | nearer handle (by value) jumps to the press point (clamped with `eps`) and is grabbed | as handle | commit |
| pan | remember start value and window | shift both by Δv, keeping width; stop at the axis ends | commit |
| edge k | grab | `edge = clamp(snap(v), neighbour below, neighbour above)` | commit |
| hist | start selection at v | extend selection, clamped to the axis | if wider than 4 px: `cool, hot = sorted(a, b)`; Bands mode rescales edges; commit |

Pointer capture is held for the whole drag. Brushing is cleared on press. Edges are
re-clamped after every move (push semantics, §3.5).

### 7.6 Snapping

On (default) unless `snap` is off or **Alt** is held. Candidates: P50, P90, P99, P99.5,
P99.9, data min, data max, zero, and every enabled threshold's display value. The candidate
whose x is nearest the pointer **within 7 px** wins; its label (`P99.5`, `data min`, `zero`,
threshold name lower-cased) is shown in the handle pill. Applies to handles and band edges,
not to selection or pan.

### 7.7 Wheel

Each notch scales by 1.12 (wheel down = zoom out), around the value under the cursor
(clamped to the axis).

- Over the ramp band (`y ≥ rTop − 11`): scales the **window**:
  `cool' = max(min, v + (cool − v)·f)`, `hot' = min(max, v + (hot − v)·f)`; ignored if the
  result is narrower than 0.4 % of the axis.
- Over the histogram: scales the **axis**: `min' = v + (min − v)·f`, `max' = v + (max − v)·f`
  (if wider than 1e-6). The window is then clamped into the new axis.

Commit 350 ms after the last notch (one undo step per wheel gesture).

### 7.8 Double-click

On the histogram (above the ramp band): reset the axis to the nice data extent
(`autorange('axis')`, §7.11), committed.

### 7.9 Keyboard (widget focused or anywhere outside text fields)

`←`/`→` nudge the active handle (last grabbed; default hot) by 0.5 % of the axis, 5 % with
Shift, with the same clamps as dragging. Commit 350 ms after the last key.

### 7.10 Panel header

`DISTRIBUTION & RANGE`, scope chip (§3.6), three independent mini toggles `log`, `CDF`,
`pctl`, and stats right-aligned: `n 12.17 M  mean 20.1 mV  P99 56.4 mV  max 83.6 mV`.

### 7.11 Fields row and auto range

**Number fields** `min`, `cool`, `hot`, `max`, each with a unit suffix (`mV` / `%`); cool's
label is cyan with a cyan inset bar, hot's orange. Focus selects all. Enter commits and
blurs; Escape blurs and reverts; blur commits. Parsing per §15.2; an unparsable entry reverts.
Side effects:

| field | rule |
|---|---|
| min | `cool = max(cool, min)` |
| max | `hot = min(hot, max)` |
| cool | if below min, min follows |
| hot | if above max, max follows |

Then `clampScale`; Bands mode rescales edges when cool or hot changed; `clampEdges`; commit.

Then: `mV | % VDD` segmented (units, §13.4 for conversion), `Ramp | Bands` segmented, `flip`
toggle (icon), `cool_is_black` toggle (tooltip `Values below cool paint black`): shown on when `under == black`; click sets `under = black`, or `clamp` if it was
black.

**Auto range** chips. `step = niceStep((D.max − D.min)/24)`, "nice extent" =
`[floor(D.min/step)·step, ceil(D.max/step)·step]`, `L` = limit value in display units.

| chip / key | effect |
|---|---|
| `axis` (double-click) | axis = nice extent; window unchanged except clamping |
| `Fit data` / `A` | axis = nice extent; cool = D.min; hot = D.max |
| `min → P99.5` | cool = axis min; hot = P99.5 (the default rule, unrounded) |
| `P1 → P99` | cool = P1; hot = P99 |
| `0 → limit` / `0` | min = min(0, min); cool = 0; hot = L; max = max(max, 1.05·L, D.max); disabled while the limit is off |
| `Margin @ limit` | applies the `Margin to limit` preset (§12.1): complete style plus its range rule; disabled while the limit is off |

After any of them except `axis`: Bands mode regenerates edges if the generator is `equal`,
else rescales them proportionally. Then clamp and commit. Hint text at the row end:
`drag handles · drag histogram to select · wheel zooms · Alt no snap · ?`.

## 8. Tabs

Tab bar: `Palette 1`, `Mapping 2`, `Bands 3`, `Thresholds 4`, `Export 5` (the digit is the
shortcut, drawn dim). Right-aligned hint: `style applies to every <CODE> map`. The body
scrolls. Only the visible tab's canvases are repainted.

### 8.1 Palette

**Preset chips**: 5 × 2 grid, each chip a 16 px gradient swatch + name + kind tag. Clicking
loads a fresh copy of that palette (selected stop resets to 0) and commits. The chip whose
name equals the current palette is highlighted; any edit marks the palette custom by
appending `*` to its name, which un-highlights all chips.

| name | kind tag | stops (evenly spaced unless given) |
|---|---|---|
| Turbo | `rainbow+` | 15 samples of the Turbo polynomial at i/14: `#23171b #4a44bc #4076f5 #2ba7f0 #26d0cc #38ee9e #60fc72 #96fa50 #cde839 #f6c62a #ff9921 #fa6719 #d4370f #a61503 #900d00` |
| Inferno | `uniform` | `#000004 #1b0c41 #4a0c6b #781c6d #a52c60 #cf4446 #ed6925 #fb9b06 #f7d13d #fcffa4` |
| Viridis | `uniform` | `#440154 #482878 #3e4989 #31688e #26828e #1f9e89 #35b779 #6ece58 #b5de2b #fde725` |
| Magma | `uniform` | `#000004 #180f3d #440f76 #721f81 #9e2f7f #cd4071 #f1605d #fd9668 #feca8d #fcfdbf` |
| Cividis | `cvd-safe` | `#00224e #123570 #3b496c #575d6d #707173 #8a8779 #a69d75 #c4b56c #e4cf5b #fee838` |
| Plasma | `uniform` | `#0d0887 #46039f #7201a8 #9c179e #bd3786 #d8576b #ed7953 #fb9f3a #fdca26 #f0f921` |
| Blackbody | `thermal` | `#000000 #3d0a0a #8a1414 #c8361b #e86a10 #f5a300 #ffd84a #fff3b0 #ffffff` |
| Traffic | `bands` | `#0e3b2e #1f7a4d #3fb35f #a8d84e #f2e14c #f7a738 #ef5a33 #d61f45` |
| Classic | `legacy` | 0 `#1414ff`, .25 `#00b0ff`, .5 `#10e050`, .72 `#f0f000`, 1 `#ff1414` |
| Budget ± | `diverging` | `#2166ac #4393c3 #92c5de #d1e5f0 #f2f2f2 #fddbc7 #f4a582 #d6604d #b2182b` |

Turbo polynomial (Mikhailov 2019 approximation), `t ∈ [0,1]`, channel = clamp(·, 0, 1)·255:

```
r = 0.13572138 + t(4.61539260 + t(−42.66032258 + t(132.13108234 + t(−152.94239396 + t·59.28637943))))
g = 0.09140261 + t(2.19418839 + t(4.84296658 + t(−14.18503333 + t(4.27729857 + t·2.82956604))))
b = 0.10667330 + t(12.64194608 + t(−60.58204836 + t(110.36276771 + t(−89.90310912 + t·27.34824973))))
```

Lightness reversals (§8.1 check) per palette: Turbo 1, Traffic 1, Classic 1, Budget ± 1
(expected for a diverging map); all others 0.

**Gradient stops header**: `GRADIENT STOPS <name> · <n> stops`; `OKLab | sRGB` segmented
(interpolation); `Reverse` (every `p → 1 − p`, re-sorted, selection kept); `Even out` (sort, then `p_k = k/(n−1)`).

**Stop editor** (62 px canvas). A 24 px bar (inset 10 px each side) shows the CVD-applied
LUT over a checkerboard. Each stop: a triangle under the bar pointing at its position and a
14 px colour swatch below; the selected stop is cyan with a glow.

| gesture | effect |
|---|---|
| click the bar | add a stop there with the colour currently at that position (no visible change until edited); select it |
| press a stop (within 9 px) | select it |
| drag horizontally | move it (`p` clamped to 0…1); the list re-sorts as it passes other stops and the selection stays on it |
| drag > 34 px vertically (only if > 2 stops) | stop fades to 30 %; on release it is removed |
| release | commit |

Hover cursor: over a stop `ew-resize`, over the bar `copy`. Under the canvas: `Stop k/n`,
`at` position field (0…1, 3 decimals), colour picker, hex field (`#rrggbb` or `rrggbb`,
otherwise ignored), `Remove` (refused when only 2 stops remain; the neighbour becomes selected), hint
`click the bar to add a stop · drag a stop off to remove it`.

**Perceived lightness** plot (118 px). OKLab L of 256 LUT samples, drawn as a 2.5 px line
coloured by the LUT itself; y grid 0, 0.5, 1; x label `ramp position →`. Tag: `monotonic ✓`
(green) or `<n> reversal(s)` (red). Reversal count: walk the samples keeping a reference L;
whenever `|L − ref| > 0.02`, take `dir = sign(L − ref)`, count a reversal if `dir` differs
from the previous non-zero direction, then set `ref = L`.

**Color-vision check** (118 px). Five strips labelled `normal`, `deutan`, `protan`,
`tritan`, `print` (grayscale), each the LUT simulated in that mode; the label matching the
preview's current simulation is cyan.

### 8.2 Mapping

Three columns (1.1 : 1 : 1).

**Transfer function**: segmented `Linear | √x | Log | γ | Equalize | Two-slope` (wraps).
Two-slope is disabled while the limit threshold is off (unless already selected; its note then
reads `The limit threshold is disabled, so the two slopes meet at the window midpoint
instead.`).
γ slider (0.25…4, step 0.05, value shown with 2 decimals) visible only for γ; live while
dragging, committed on release. **Curve plot** (150 px): x = value from cool to hot, y = ramp
position; faint in-window histogram behind it (60 bins, heights ∝ sqrt(count/max) · 90 %);
a vertical strip of the ramp along the y axis; a faint diagonal reference; the curve in cyan
with glow from 201 samples; labels `cool <v>` and `<v> hot` under the ends. Note text per
transfer (§4.2).

**Out of range**: `Below cool` segmented `Clamp | Black | None | Color` + swatch;
`Above hot` segmented `Clamp | Color` + swatch. Swatches are at 35 % opacity unless their mode
is `custom`; picking a colour switches the mode to `custom` (live; commit on close). Note:
`“None” lets the layout show through, like classic -disable. A distinct above-hot color makes
“saturated” impossible to confuse with “hottest in range”.` Then `LAYER` → `Heat opacity`
slider 20…100 % (step 1 %; view state, not undoable).

**Legend**: `Numbers` segmented `Eng | Sci | Fixed`; `Font size` slider 7…32 px.
**Handles**: `Snapping` toggle `Snap to P-values & limits`; note `Hold Alt while dragging to
place freely. Arrow keys nudge the last-used handle, Shift for ×10.`

### 8.3 Bands

Header: `BANDS` stepper `− n +` (2…12); `GENERATE` segmented `Equal | Quantile |
Threshold-anchored`; toggle `Colors follow palette`.

Callout (only in Ramp mode): `Bands are shown in Bands mode — the classic six-row band table,
with a live instance count per row.` + button `Switch to bands`.

Generators (all produce `n − 1` interior edges inside `[cool, hot]`, `span = hot − cool`):

| generator | edges |
|---|---|
| Equal | `cool + k/n · span`, k = 1…n−1 |
| Quantile | from the in-window sorted values `sub`: `sub[round(k/n · (len − 1))]`; if empty, Equal |
| Threshold-anchored | anchors = enabled threshold values strictly inside `(cool, hot)`, ascending, at most n−1; `free = n − 1 − #anchors` edges spaced evenly from cool to the first anchor (`cool + k/(free+1) · (a₀ − cool)`), then the anchors. No anchors → Equal |

Example: 6 bands, window 0…86.25, anchors 60 and 75 → edges 15, 30, 45, 60, 75.

When generation happens: changing the generator; changing the band count (manual edge edits
are lost); switching to Bands mode if any gap is < 0.1 % of the window (degenerate edges left
over from ramp-mode handle drags); a missing or wrong-length edge list on render; auto range
with the Equal generator; built-in band preset.

Band colours: with `Colors follow palette` on, colour k = `LUT[round((k + 0.5)/n · 1023)]`
(band centre), resampled whenever the LUT changes. Editing a band colour turns follow off.
Changing the band count with follow off keeps custom colours whose index survives; new bands
sample the palette at their centres.

Table (one row per band, top = lowest values):

| column | content |
|---|---|
| ☐ | enable (disabled rows at 45 % opacity; band paints transparent) |
| swatch | colour as displayed (flip-aware); editing writes the storage index `flip ? n−1−k : k` |
| from / to | editable, unit-aware parse; first row's from = cool, last row's to = hot, others = edges |
| instances | `countIn(from, to)`: lower-inclusive, upper-exclusive, except the last band which includes `hot` |
| share | bar, width = fraction × 100 %, in the band colour |
| % | `fmtFrac(count / n)` |

Edits are clamped (`clampScale`, `clampEdges`) and committed.

### 8.4 Thresholds

Header: `SIGNOFF THRESHOLDS · static IR analysis` or `… · dynamic (DvD) analysis`; toggles
`Pulse violations`, `Contour at limit`; button `Add`.

| column | content |
|---|---|
| ☐ | enabled |
| swatch | colour |
| name | text; empty input keeps the old name |
| value | in display units (`fmtIn` of `pct/100 · refVDD` in mV); editing in mV converts back to pct with refVDD |
| % of VDD | `pct`, plain number |
| instances over | `count` (§5.4) |
| share | bar, width = `max(fraction × 2000 %, 2 % if count > 0)` (×20 exaggeration, capped by the cell) |
| 🗑 | custom thresholds only |

Note under the table: `Violations are counted per instance against its own supply
(VDD_CORE 0.750 V, VDD_SRAM 0.850 V), avoiding the classic single-reference trap in
multi-VDD designs. Lines on an mV axis are drawn at the reference domain's value.`

### 8.5 Export

Segmented `Scale JSON | Classic Tcl (bands) | Import JSON` (UI state, not undoable). For the
two export formats: preset name field (placeholder `preset name`), `Save as preset`, `Copy`,
and a read-only, syntax-highlighted code view of the export for the current state,
regenerated on every change while the tab is open. Copy → clipboard, toast `Copied to
clipboard` (or `Clipboard unavailable here — select the text instead`).

#### 8.5.1 JSON scale definition, format `heatmap-color-scale/1`

| field | type | content |
|---|---|---|
| `format` | string | `"heatmap-color-scale/1"` |
| `scope.metric` | string | metric code (`IR`, `IR_VDD`, `GB_VSS`) |
| `scope.layer` | string | layer id or `"*"` when linked |
| `scope.view` | string | view id or `"*"` when locked |
| `units` | string | `"mV"` or `"pct_vdd"` |
| `normalization` | string | `"absolute"` (mV) or `"per_instance_domain_vdd"` (% VDD) |
| `range.min/cool/hot/max` | number | 6 significant digits |
| `flip` | bool | |
| `mode` | string | `"ramp"` or `"bands"` |
| `transfer` | object | `{"type": "linear" \| "sqrt" \| "log" \| "equalize"}`, `{"type": "gamma", "gamma": γ}`, or `{"type": "two_slope", "center": "threshold:limit" \| "window_midpoint"}` (midpoint when the limit is disabled) |
| `palette.name` | string | may end in `*` (edited) |
| `palette.interpolation` | string | `"oklab"` or `"rgb"` |
| `palette.stops` | `[[p, "#rrggbb"], …]` | sorted by p, p to 6 significant digits |
| `out_of_range.below` | string | `"clamp"`, `"black"`, `"transparent"` or `"#rrggbb"` |
| `out_of_range.above` | string | `"clamp"` or `"#rrggbb"` |
| `bands` | array, Bands mode only | `{"from", "to", "color", "enabled"}` per band, colours in storage (un-flipped) order |
| `thresholds` | array | current analysis kind: `{"name", "pct_vdd", "color", "enabled"}` |
| `legend` | object | `{"font_px": 7…32, "numbers": "eng" \| "sci" \| "fixed"}` |

Arrays of scalars are written on one line (cosmetic). Fixture default export:

```json
{
  "format": "heatmap-color-scale/1",
  "scope": {"metric": "IR", "layer": "*", "view": "dvd_vectorless"},
  "units": "mV",
  "normalization": "absolute",
  "range": {"min": 0, "cool": 0, "hot": 62.6, "max": 85},
  "flip": false,
  "mode": "ramp",
  "transfer": {"type": "linear"},
  "palette": {
    "name": "Turbo",
    "interpolation": "oklab",
    "stops": [
      [0, "#23171b"], [0.0714286, "#4a44bc"], [0.142857, "#4076f5"], [0.214286, "#2ba7f0"],
      [0.285714, "#26d0cc"], [0.357143, "#38ee9e"], [0.428571, "#60fc72"], [0.5, "#96fa50"],
      [0.571429, "#cde839"], [0.642857, "#f6c62a"], [0.714286, "#ff9921"], [0.785714, "#fa6719"],
      [0.857143, "#d4370f"], [0.928571, "#a61503"], [1, "#900d00"]
    ]
  },
  "out_of_range": {"below": "black", "above": "clamp"},
  "thresholds": [
    {"name": "Warn", "pct_vdd": 8, "color": "#ffb547", "enabled": true},
    {"name": "Limit", "pct_vdd": 10, "color": "#ff4d6d", "enabled": true}
  ],
  "legend": {"font_px": 11, "numbers": "eng"}
}
```

(Reflowed here for width; the mock-up prints objects one key per line.)

#### 8.5.2 Classic Tcl (bands)

A best-effort translation into the classic banded tools' `config colormap` command. Only
metrics with a known classic map id are exported: `IR → IR`. Any other metric is **refused**
with two comment lines and no command:

```tcl
# No classic banded colormap equivalent is known for metric GB_VSS (Ground bounce · VSS rail).
# Export refused rather than written as an IR map. Use the scale JSON instead.
```

Otherwise, line by line:

1. `# Classic banded equivalent of <title>`
2. `# "config colormap" syntax; -values boundary count and -disable indexing unverified`
3. Ramp mode: `# ramp approximated by 6 equal bands; transfer "<transfer>" and palette are not
   representable`. Bands mode: `# <n> bands, generated "<generator>"`.
4. % VDD only: `# classic -percent uses a single reference VDD; per-domain normalization is
   lost`
5. `config colormap -map <id> <-instance|-wire> <-absolute|-percent> \` — `-instance` for
   layer `INST`, else `-wire`.
6. `    -min <cool> -max <hot>[ -refvdd <V.VVV>] \` — `-refvdd` (reference VDD in volts, 3
   decimals) only in % VDD.
7. `    -values {<boundaries>}` — ramp: 7 boundaries (6 equal bands cool…hot); bands: all `n+1`
   boundaries. Followed by ` \` if a disable line follows.
8. `    -disable {<indices>}` — 0-based indices of disabled bands, only if any.
9. empty line, then `dump gif -map <id> -o <id lower-case>_<layer lower-case>_<view>.gif`.

Numbers: mV mode writes **volts** with 4 significant digits (`0.06260`); % mode writes the
percentage with 4 significant digits, trailing zeros trimmed. Fixture default:

```tcl
# Classic banded equivalent of IR::INST…dvd_vectorless
# "config colormap" syntax; -values boundary count and -disable indexing unverified
# ramp approximated by 6 equal bands; transfer "linear" and palette are not representable
config colormap -map IR -instance -absolute \
    -min 0.000 -max 0.06260 \
    -values {0.000 0.01043 0.02087 0.03130 0.04173 0.05217 0.06260}

dump gif -map IR -o ir_inst_dvd_vectorless.gif
```

Unverified against the real tool: whether `-values` takes `n+1` boundaries (the classic
manual's EM example lists 9 values) and whether `-disable` indices are 0-based (the manual
example suggests so).

#### 8.5.3 User presets

`Save as preset` stores `{name, style, range}`; empty name → `<palette> · <mode> ·
YYYY-MM-DD HH:MM` (local time, locale-independent). `style` = deep copy of the current
metric's style (§3.7). `range` = `min, max, cool, hot` converted to **% VDD** with the
reference VDD at save time, so a preset works in either unit. Mock-up storage: browser
`localStorage['hcs.presets']` (JSON array; failures ignored). Product: `QSettings` (§17.9).
Toast `Saved preset “<name>”`. Applying: the style replaces the current metric's style
(missing keys from defaults); the range is converted back with the current reference VDD into
the current scope's scale; edges reset (regenerated by the generator).

#### 8.5.4 JSON import

`Import JSON` swaps the code view for a text area (paste) plus `Load file…` (any `.json`) and
`Import into this scope`. The text is validated on every change; messages appear under it in
three kinds: errors (red, block the import), notes (amber) and a green
`Valid heatmap-color-scale/1 definition[ · <k> notes]`.

Required: `format`, `units`, `range`, `palette`. Errors:

| check | message |
|---|---|
| JSON syntax | `Not valid JSON: <parser message>` |
| top level | `The top level must be an object.` |
| `format` | `format must be "heatmap-color-scale/1" (found <value>)` |
| `units` | `units must be "mV" or "pct_vdd"` |
| `range` | finite `min, cool, hot, max`; `min < max`; `cool < hot` |
| `palette.stops` | at least two `[p ∈ 0…1, "#rrggbb"]` |
| `palette.interpolation` | `oklab` or `rgb` if present |
| `flip`, `mode` | boolean; `ramp` or `bands` |
| `transfer` | known type; `gamma` in 0.25…4 |
| `out_of_range` | below: clamp / black / transparent / hex; above: clamp / hex |
| `bands` | 2…12 entries with `from ≤ to`, hex colour, boolean enabled |
| `thresholds` | list of `{name, pct_vdd > 0, hex color, enabled}` |
| `legend` | `font_px` integer 7…32, `numbers` eng / sci / fixed |

Notes (import proceeds): cool/hot outside min…max (axis widened); bands not starting at cool
or ending at hot (cool and hot win); bands not contiguous (each band's `from` is its lower
edge); bands mode without bands (equal bands generated); no threshold named Warn or Limit (the
current one is kept); file scope differs from the current scope (imported into the current
one); file units differ (range converted at the reference VDD).

Applying: the current metric's style is replaced (palette stops sorted, lower-case hex; bands
given explicitly turn "colours follow palette" off); the current scope's range is set (edges
from the bands' `from` values); the current analysis kind's thresholds are replaced — names
`Warn` / `Limit` (case-insensitive) map to the built-in ids, the rest become custom
thresholds; legend settings are applied. One undo step; toast `Imported “<palette name>”[ with
<k> notes]`. Exporting right after importing an exported file reproduces it byte for byte.

## 9. Live preview

Panel header: `LIVE PREVIEW`, mini toggles `Legend`, `Bumps`, `Domains`, `Blocks`, `Smooth`, and the
vision combo (`Vision: normal`, `Deuteranopia`, `Protanopia`, `Tritanopia`, `Grayscale
print`; tooltip `Simulate color-vision deficiency`). The canvas is 4:3 with a 1 px border,
radius 6, crosshair cursor.

### 9.1 Backdrop (cached per size)

Die `#05070a`; core (inset by the 3-tile ring) `#0a0f15`; standard-cell rows every 2.4 px
`rgba(120,140,170,.07)`; PDN grid lines every half bump pitch (7 tiles, from offset 7) in both
directions `rgba(110,150,210,.10)`; SRAM macros 8 × 9 tiles on a 10 × 11 pitch, fill
`#0e1520`, border `rgba(199,146,255,.16)`; analog keep-out `#080b10` hatched
`rgba(140,160,190,.10)` every 6 px; IO pads 3 × 1.8 tiles every 6 tiles along all edges
`rgba(170,185,210,.14)`.

### 9.2 Heat layer

Rebuilt only when its key changes (§14). One pixel per tile (`colour(v)` with CVD; tiles
without instances transparent), scaled to the canvas nearest-neighbour (or
smoothed when `Smooth` is on), drawn at `opacity`. **Dimming**: tiles outside the domain
filter, or outside a brushed histogram bin, are drawn as dark grey
`g = (0.3 r + 0.59 g + 0.11 b) · 0.2 + 8`, alpha 230 (0 if the tile was transparent).

### 9.3 Overlays (paint order)

1. **Domains** (toggle): VDD_SRAM outline dashed `[5,3]` `rgba(199,146,255,.85)` 1.2 px; tag
   `VDD_SRAM 0.85 V` at its top-left; keep-out label `ANALOG` / `no instances` (two lines,
   ±6 px around the centre) in `rgba(140,160,190,.55)`.
2. **Blocks** (toggle, off by default): dashed `[3,3]` outlines `rgba(235,242,255,.45)` of
   every floorplan block, labelled with the last segment of its path.
3. **Limit glow** (toggle `Pulse violations`, only with violations): the violation mask in
   the limit colour, blurred (7 px twice, then 2 px, device pixels), composited additively
   with alpha `0.3 + 0.55 · (0.5 + 0.5 · sin(2π · 0.8 Hz · t))`.
4. **Limit contour** (toggle `Contour at limit`): §5.5 segments, 1.4 px, limit colour with a
   6 px glow.
5. **Bumps** (toggle): dots r = 2.1 px, fill `rgba(0,0,0,.35)`, stroke `rgba(235,242,255,.5)`;
   missing bump: dashed `[2,2]` red (`#ff4d6d`) circle r = 5 with an X (±3 px).
6. **Worst marker**: white circle r = 7 (1.5 px) with four ticks from 9 to 14 px; tag
   `WORST <value>` 16 px to the right (left if it would overflow). **Locate** animation
   (1.6 s): three cyan rings with phases 0, ⅓, ⅔, radius `8 + 60p`, alpha `0.9 (1 − p)`, 2 px.
7. **Hover crosshair**: full-width/height lines `rgba(255,255,255,.22)` through the tile
   centre and a white box 1.5 px outside the tile.
8. **Legend** (toggle, §9.4).

### 9.4 Legend

Top-left at (10, 10). `fs = legendFont`.

| element | spec |
|---|---|
| box | `rgba(5,8,12,.78)`, radius 6, border `rgba(255,255,255,.10)`; width = max(2·8 + barW + 8 + label width + 8, title width + 16); height = 16 + 2.6·fs + 2·tri + barH + 6 |
| title | `<CODE>::<layer>` (e.g. `IR::INST`), 600 weight, at y = 18 + 0.6·fs |
| unit | `[mV]` or `[% VDD]`, muted, at y = 18 + 1.85·fs |
| bar | width `max(10, 1.1·fs)`, height `clamp(13·fs, 110, canvasH − 70)`; painted one row per pixel with `colour(v)`, `v` from hot (top) to cool (bottom); transparent rows as 4 px dark stripes `#1a212c` / `#242d3a`; border `rgba(255,255,255,.3)` |
| extend triangles | height `max(7, 0.7·fs)`: top = above colour, bottom = below colour, outline `rgba(255,255,255,.35)`, unfilled when transparent (matplotlib `extend='both'`) |
| labels | Ramp: 5 values evenly spaced cool…hot; Bands: every boundary. Positioned linearly by value; 4 px tick each; a label is skipped if closer than 1.05·fs to the previous one; formatted by §15.1 legend rules |
| threshold ticks | enabled thresholds inside the window: 2 px line in the threshold colour across the bar, 4 px beyond the left edge |

### 9.5 Probe

Hovering a tile sets the hover cell (crosshair), marks its value on the histogram, and shows
a tooltip offset 16 px right/below the pointer (flipped left/up at the preview edges):

```
■ 83.6 mV                                  ← swatch in the mapped colour + value
11.15 % of VDD · above hot (clamped)        ← other unit · state
VDD_CORE · 0.750 V · <n> instances         ← domain colour; instances in the tile
(1215, 1245) µm · soc/npu · missing bump    ← position · cause hint
[OVER LIMIT +1.15 %]                        ← badge (limit colour) or [over warn]
```

State text: `below cool`, `above hot (clamped)`, `band disabled`, `band k / n`, or
`ramp <round(t·100)> %` (`t` before flip). Empty tile: `no instances` + position.
Coordinates: origin at the die's lower-left, y up: `x = (col + 0.5)·10 µm`,
`y = (rows − row − 0.5)·10 µm`.

Readout line under the preview: `x 1215 µm  y 1245 µm  83.62 mV  11.15 % of VDD_CORE  <n> inst`, or
when not hovering: `hover the map to probe · hover the histogram to light up instances on the
map`.

## 10. KPI cards

2 × 2 grid; each card has a 3 px left accent bar.

| card | big number | subtitle | accent / colour |
|---|---|---|---|
| Worst drop (+ `Locate` button) | worst value in display units | `<other unit> · <cause hint> · (x, y) µm`, tooltip = that text + domain id | neutral |
| Over limit | `<count> inst` or `0`; `off` when disabled | `<frac> of instances > <pct> % VDD · <k> cluster(s)` / `none above <pct> % VDD ✓` / `threshold disabled` | accent = limit colour; number red if > 0, green if 0 |
| Over warn | same for warn | same, no cluster count | accent = warn colour; number amber if > 0 |
| Saturation | two rows: `below cool` bar + %, `above hot` bar + % | — | below bar `#5d6b80` (width = fraction), above bar `hot` colour (width = 10 × fraction, capped); accent amber when above > 2 % |

Fixture default: `83.6 mV` / `11.15 % VDD · soc/npu · missing bump · (1215, 1245) µm`;
`22.1 k inst` / `0.18 % of instances > 10 % VDD · 1 cluster`; `73.4 k inst` /
`0.60 % of instances > 8 % VDD`; saturation `0 %` / `0.50 %`.

`Locate` (and the insight action of the same name) starts the ring animation on the worst
marker. Product: also pan/zoom the host's main map there.

## 11. Scale insights

Header `SCALE INSIGHTS`, hint `live, from the data under this scale`. Rules are evaluated in
this order on every render; **the first four that fire are shown**; if none fire, one green
`Scale looks healthy for this data.` Each item: severity icon (bad = bolt/red, warn =
triangle/amber, info = i/cyan, ok = check/green), rich text, optional action button. The list
is only re-rendered when its content changes (so buttons don't flicker under the pointer).

| # | condition | sev | text | action |
|---|---|---|---|---|
| 1 | limit enabled and instances over it > 0 | bad | `<count> instances exceed the <pct> % limit in <k> cluster(s); worst in <cause hint>.` | `Locate` |
| 2a | limit enabled, Ramp mode, transfer ≠ two-slope, `hot < 0.98·L` and `D.max > hot` | warn | `Hot (<hot>) sits below the <L> limit: the hottest color does not mean “violation”.` | `Set hot = limit`: if `L > max` then `max = 1.05·L`; `hot = L`; commit |
| 2b | else, same preconditions, `\|hot − L\| ≤ max(1e-6, 0.02·L)` | ok | `Hot = limit: the top color marks exactly the signoff budget[, and anything over it gets its own color].` (suffix when above = custom) | — |
| 3 | above-hot fraction > 2 % | warn | `<frac> of instances clamp at hot — gradients inside hotspots are flattened.` | — |
| 4 | `lowQuarter > 0.8`, linear, Ramp mode | info | `<frac> of in-range instances use only the bottom quarter of the ramp. Try √x, Log or Equalize.` | `Try √x` → transfer sqrt |
| 5 | below = black or transparent and below-cool fraction > 30 % | info | `<frac> of instances fall below cool and render as background.` | — |
| 6 | palette has lightness reversals, Ramp mode, transfer ≠ two-slope | warn | `<palette> has <n> lightness reversal(s), which can draw false edges. Inferno, Viridis or Cividis do not.` | — |
| 7 | mV units and the filtered data contain domains with different VDD | info | `<k> supplies share one mV scale; <highest-VDD domain> has <ΔVDD> mV more headroom than <lowest-VDD domain>. % VDD normalizes each instance to its own rail.` (computed from the domain table) | `Use % VDD` → units % |
| 8 | CVD preview active | info | `Previewing as <mode>. Export is unaffected.` | — |

Fixture default shows rules 1, 2a, 6, 7; rule 7 reads `2 supplies share one mV scale;
VDD_SRAM has 100 mV more headroom than VDD_CORE. …` and disappears with the VDD_SRAM filter.
After the Signoff preset: rules 1, 2b, 6, 7.

## 12. Presets

### 12.1 Built-in

Every built-in preset sets the **complete** style of the current metric (defaults of §3.7
plus the listed changes), then applies its range rule, then regenerates band edges with its
own generator on the new window. Presets marked *needs limit* are disabled in the combo
(suffix ` (needs limit)`) while the limit threshold is off.

| name | style (changes from defaults) | range rule | fixture result |
|---|---|---|---|
| `Signoff · 0 → limit, over = magenta` *needs limit* | Turbo; below clamp; above custom `#ff3df0` | `0 → limit` | 0 / 0 / 75 / 85 |
| `Hotspot hunt · P50 → P99.9 (Inferno)` | Inferno; √x; below black | cool = P50; hot = P99.9 | 0 / 17.28 / 78.62 / 85 |
| `Legacy look · rainbow + cool_is_black` | Classic; sRGB interpolation; below black | cool = axis min; hot = P99.5 | 0 / 0 / 62.62 / 85 |
| `Colorblind-safe · Cividis, equalized` | Cividis; equalize; below clamp; above custom `#ff3df0` | cool = axis min; hot = P99.5 | 0 / 0 / 62.62 / 85 |
| `Threshold bands · 6, anchored at warn/limit` *needs limit* | Traffic; bands; 6; threshold-anchored; follow palette; below transparent; above custom `#ff3df0` | cool = 0; min = min(min, 0); hot = max(1.15·L, min(D.max, max)); max grows to hot if needed | 0 / 0 / 86.25 / 86.25, edges 15 30 45 60 75 |
| `Margin to limit · two-slope diverging` *needs limit* | Budget ±; two-slope; below clamp; above clamp | axis = nice floor of D.min … max(nice ceil of D.max, 1.1·L); cool = D.min; hot = axis max | 0 / 1.015 / 85 / 85 |

(Results are min / cool / hot / max in mV.) Applying a preset shows toast
`Preset: <name>` and is one undo step. The `Margin @ limit` auto-range chip applies the last
preset.

### 12.2 User presets

§8.5.3. Listed under `Saved` in the preset combo.

## 13. Commands, undo, apply

### 13.1 Commit points

Only the scale set (§3.10) is undoable.

| action | when the undo step is recorded |
|---|---|
| segmented controls and toggles that change the scale set (units, mode, transfer, interpolation, out-of-range, flip, `cool_is_black`, numbers, snap is *not* one), `F`, `B`, `U`, `Switch to bands`, Link layers, Lock across views | immediately |
| auto-range chips, `A`, `0`, double-click axis reset | immediately |
| handle / pan / edge / track / selection drag | on release |
| wheel, arrow-key nudges | 350 ms after the last event |
| number fields | on Enter or blur |
| sliders (γ, legend font) | live while dragging; recorded on release |
| colour pickers (stop, below, above, band, threshold) | live; recorded on close |
| stop editor (add / move / remove) | on release; Reverse, Even out, Remove, hex, position: immediately |
| palette chip, preset, import | immediately |
| band / threshold table edits, Add, delete | immediately |
| insight actions | immediately |
| Reset range | immediately |
| Revert | restores, then records (so Revert is undoable) |
| **never**: metric / layer / view / domain / vision combos, preview overlay toggles (Legend, Bumps, Domains, Blocks, Smooth), histogram toggles, Pulse, Contour, Snapping, heat opacity, `L` | — |

A snapshot is the scale set serialised without pristine defaults (§3.10). A commit whose
snapshot equals the previous one is dropped. New commits clear the redo stack. Depth 200.
Undo restores the scale set and leaves the view state alone: undoing a palette change after
switching views keeps the new view. If the units differ, statistics are recomputed (or taken
from the cache).

### 13.2 Apply, Revert, Reset range

- **Apply** (button or Ctrl+Enter): the current snapshot becomes the applied state and goes
  to the host renderer. Toast `Applied to <CODE> · <layer|all layers> · <view|all views>`.
- **Modified indicator**: shown while the current snapshot ≠ the applied one, by value
  comparison (editing back to the applied values clears it; browsing never sets it).
  Initially the startup state is applied.
- **Revert**: restore the applied state as a new undo step. Toast `Reverted to last applied
  scale`.
- **Reset range**: delete the current scope's range; a fresh default (§3.5) is created at once.
  Toast `Range reset to the default: cool = axis min, hot = P99.5`.

### 13.3 Keyboard

| key | action | notes |
|---|---|---|
| `Ctrl+Z` | undo | works everywhere, also in text fields |
| `Ctrl+Shift+Z`, `Ctrl+Y` | redo | |
| `Ctrl+Enter` | apply | |
| `Esc` | close help | |
| `?` | toggle help | |
| `F` | toggle flip | |
| `B` | Ramp ↔ Bands | |
| `U` | mV ↔ % VDD | |
| `L` | toggle legend | view state |
| `A` | Fit data | |
| `0` | 0 → limit | refused with toast while the limit is off |
| `1`–`5` | tabs | |
| `←` / `→` (`Shift`) | nudge active handle 0.5 % (5 %) of the axis | |

Single keys are ignored while an input, combo or text area has focus, and when Ctrl, Meta or
Alt is held. In Qt, `QLineEdit` already overrides shortcuts for printable keys
(`ShortcutOverride`), so `QAction` shortcuts with `Qt::WidgetWithChildrenShortcut` behave the
same way.

### 13.4 Units conversion

Switching units converts **every** scope's `min, max, cool, hot, edges` with the current
reference VDD (`x·100/refVDD` to %, `x·refVDD/100` back), then switches the statistics. Data
values themselves use each tile's own VDD, so after conversion the window covers
approximately, not exactly, the same instances when domains differ. Because that is easy to
miss, a toast says so when the design has supplies with different VDD:
`Ranges converted at the VDD_CORE reference (0.750 V); instances use their own supply`.
Fixture: 0 / 0 / 62.6 / 85 mV → 0 / 0 / 8.347 / 11.33 %.

### 13.5 Toasts (complete list)

`One scale now shared by all layers` · `Scale locked across views — switch views to compare` ·
`Scale is now per layer` · `Scale is now per view` · `Preset: <name>` · `Enable the limit
threshold first` · `Range reset to the default: cool = axis min, hot = P99.5` · `Reverted to
last applied scale` · `Applied to <where>` · `Ranges converted at the <domain> reference
(<V> V); instances use their own supply` · `Copied to clipboard` · `Clipboard unavailable
here — select the text instead` · `Saved preset “<name>”` · `Imported “<name>”[ with <k>
notes]`.

## 14. Render pipeline and performance

Renders are coalesced: any number of invalidations within one frame cause one render
(mock-up: `requestAnimationFrame`; Qt: a zero-timeout single-shot timer or `update()`
coalescing). Expensive work is keyed so that it only runs when its inputs change:

| work | key / trigger | cost when unchanged |
|---|---|---|
| derived statistics (§3.4) | (metric, view, layer, units, domain); 12-entry cache | lookup |
| LUT, lightness, reversal count | (palette stops, interpolation, CVD) | none |
| histogram bin counts | (data key, axis min/max, bin count) | none |
| under/over/lowQuarter, equalize table | every render | binary searches only |
| violation counts, mask, clusters, contour, glow | `viol` flag: thresholds, data, undo/redo, presets, import | none |
| heat image | (size, data key, LUT key, mapping parameters, brush, opacity, smooth) | none |
| insights list | its HTML | none |
| overlay (crosshair, pulse, locate rings, legend) | every frame while animating | cheap vector drawing |

Render order: statistics → style → LUT → scope range (create, clamp, band arrays, edges) →
prepare mapping (§4.9) → equalize table (if active) → histogram → violations → distribution
widget → heat layer → overlays → visible tab's canvases/tables → KPIs → insights → widget
sync → export text (if tab open) → record pristine defaults created by this render.

The overlay animates (glow pulse, locate rings) only while there are violations with pulse
on, or during the 1.6 s locate animation; otherwise nothing repaints when idle.

Measured in the mock-up (headless Chromium, 1720 × 1010, fixture default): full render 4.6 ms;
render on map hover 1.7 ms; per handle-drag step 3.3 ms; switching view 19 ms; switching back
(cached) 9 ms.

Product budgets (should):

| interaction | target | how |
|---|---|---|
| handle drag, pan, wheel | repaint at display rate (≤ 16 ms) on a full-chip run | as above: recolour the tile raster only; histogram counts cached; counters by binary search |
| metric/view/layer/domain/units switch | < 1 s, non-blocking | sort and cumulative counts in a worker (`QtConcurrent::run`), cached per key |
| threshold edit | < 100 ms | counts by binary search on the sorted `pct` array; mask/clusters on the raster |
| idle | 0 repaints | animation only as above; pulse stays a user toggle (remote displays) |

## 15. Number formatting and parsing

### 15.1 Formatters

| formatter | rule |
|---|---|
| value (`fmtV`), Eng | mV: < 10 → 2 dp, < 100 → 1 dp, else 0 dp, suffix ` mV`; %: < 10 → 2 dp, else 1 dp, suffix ` %` |
| value, Sci | mV: volts, `toExponential(4)` + ` V`; %: `toExponential(3)` + ` %` |
| value, Fixed | 3 dp + unit |
| legend label, Eng | mV: < 10 → 2 dp else 1 dp; %: 2 dp; no unit (the legend states it) |
| legend label, Sci / Fixed | as value, without unit |
| input field (`fmtIn`) | 5 significant digits, trailing zeros trimmed |
| count (`fmtCount`) | instances: ≥ 1e6 → `x.xx M`; ≥ 1e4 → `x.x k`; ≥ 1e3 → `x.xx k`; else integer |
| fraction (`fmtFrac`) | 0 → `0 %`; < 0.01 % → 3 dp; < 1 % → 2 dp; else 1 dp |
| axis tick | decimals = `max(0, −floor(log10(step)))` |
| non-finite | `—` |

### 15.2 Parsing (fields, band from/to, threshold value)

Case-insensitive, whitespace removed; grammar `number [unit]`, number with optional sign,
decimals and exponent; unit ∈ `mv`, `uv`, `µv`, `v`, `%`.

| display units | input | result |
|---|---|---|
| mV | `45` | 45 mV |
| mV | `45mV`, `0.045V`, `45000uV` | 45 mV |
| mV | `6%` | `6/100 · refVDD` mV (45 at 750 mV) |
| % | `6`, `6%` | 6 % |
| % | `45mV`, `0.045V` | `45/refVDD · 100` % (6 at 750 mV) |
| any | anything else | invalid → field reverts |

## 16. Synthetic fixture

The mock-up's data generator is deterministic; the native build can port it to get the same
numbers for tests (§19). All coordinates in tiles; x right, row index y **down**.

### 16.1 Geometry

- Grid 200 × 150 tiles of 10 µm (2.0 × 1.5 mm die).
- Invalid (NaN, no instances): the 3-tile ring at the die edge, and the analog keep-out
  x 160–195, y 112–145. Valid tiles: 26 712.
- VDD_SRAM: x 96–137, y 94–139 (1932 tiles); the rest VDD_CORE.
- Bumps: pitch 14, offset 7 → 14 × 11 = 154 bumps at `(7 + 14i, 7 + 14j)`; bump `(i 8, j 1)`
  (tile 119, 21) is **missing**.
- Bump field per tile: `d` = distance to the nearest non-missing bump among the 5 × 5 bump
  neighbourhood of the nearest bump index; `BUMPF = 4 + 13 · min(d / (14 · 0.72), 1.8)^1.6`.

### 16.2 Floorplan and instance counts

Blocks (tile rectangles, `x0 ≤ x < x1`, `y0 ≤ y < y1`; first match wins) with density factors:

| block | x | y | density |
|---|---|---|---|
| `soc/cpu/core0` | 24–49 | 22–46 | 1.3 |
| `soc/cpu/core1` | 54–79 | 22–46 | 1.3 |
| `soc/cpu/core2` | 24–49 | 51–75 | 1.3 |
| `soc/cpu/core3` | 54–79 | 51–75 | 1.3 |
| `soc/cpu/l2` | 24–79 | 22–75 | 0.9 |
| `soc/npu` | 108–137 | 12–41 | 1.25 |
| `soc/gpu/sc0` | 160–176 | 43–56 | 1.2 |
| `soc/gpu` | 136–185 | 42–89 | 1.1 |
| `soc/ddr_phy` | 3–37 | 102–146 | 0.8 |
| `soc/sram_ss` | 96–137 | 94–139 | 0.12 |
| `soc/noc` | 60–95 | 92–129 | 0.95 |
| `soc` (everything else) | | | 1.0 |

Instance count per valid tile, drawn once per design in row-major order with
`rnd = mulberry32(20261121)`: `count = round(470 · density · (0.75 + 0.5 · rnd()))`.
Total 12 170 017. Per block: `soc` 7 299 926, `soc/gpu` 1 117 015, `soc/noc` 613 752,
`soc/ddr_phy` 591 988, `soc/npu` 527 564, `soc/cpu/core1` 405 539, `soc/cpu/core0` 399 385,
`soc/cpu/core2` 396 969, `soc/cpu/core3` 393 829, `soc/cpu/l2` 181 338, `soc/gpu/sc0` 134 095,
`soc/sram_ss` 108 617.

### 16.3 Value parameters

| metric | code | wb | wg | wh | seed |
|---|---|---|---|---|---|
| `eff` | IR | 1 | 1 | 1 | 1 |
| `vdd` | IR_VDD | .62 | .55 | .64 | 2 |
| `vss` | GB_VSS | .38 | .45 | .36 | 3 |

| layer (index) | f | smooth radius | noise |
|---|---|---|---|
| INST (0) | 1.00 | 0 | 1.0 |
| M1 (1) | 0.93 | 0 | 0.75 |
| M2 (2) | 0.86 | 1 | 0.5 |
| M4 (3) | 0.70 | 2 | 0.3 |
| M7 (4) | 0.48 | 3 | 0.15 |
| M10 (5) | 0.27 | 4 | 0.06 |
| AP (6) | 0.11 | 6 | 0.02 |

| view | kind | scale | seed | hotspot mode |
|---|---|---|---|---|
| static_ir | static | 0.36 | 11 | avg |
| dvd_vectorless | dynamic | 1.00 | 23 | vl |
| dvd_vcd_func | dynamic | 0.92 | 37 | vcd |

Hotspots `(x, y, σ, amplitude, id)` shape the values (they are not used for cause hints):
(38, 36, 9, 30, cpu0) · (66, 36, 9, 27, cpu1) · (38, 62, 9, 25, cpu2) · (66, 62, 9, 40, cpu3) ·
(52, 49, 15, 10, l2) · (122, 26, 7, 63, npu) · (156, 64, 20, 19, gpu) · (168, 50, 7, 13,
gpu_sc) · (20, 118, 10, 13, ddr) · (117, 117, 18, 7, sram) · (80, 110, 16, 11, fabric).

Hotspot amplitude by mode: `avg` → a × 0.6 if σ < 10, else a; `vcd` → a × {cpu2 1.6, cpu3 0.7,
npu 0.8, gpu 1.4, ddr 1.3}, others × 1; `vl` → a. Then × metric `wh`.

### 16.4 Value generator

```
mulberry32(a):                      # returns rnd() in [0, 1)
  a = (a + 0x6D2B79F5) | 0
  t = imul(a ^ (a >>> 15), 1 | a)
  t = (t + imul(t ^ (t >>> 7), 61 | t)) ^ t
  return ((t ^ (t >>> 14)) >>> 0) / 4294967296

rnd = mulberry32(view.seed·1009 + layerIndex·97 + metric.seed·7919)
waves = 6 × {kx = (2 rnd() − 1)·0.12, ky = (2 rnd() − 1)·0.12, ph = rnd()·6.283, a = 0.5 + 0.8 rnd()}
                                    # draws in exactly this order: kx, ky, ph, a per wave
for y in 0..149, for x in 0..199:   # row-major; invalid tiles: NaN, no rnd() draw
  nx = x/199·2 − 1 ; ny = y/149·2 − 1
  glob = 7 (1 − nx²)(1 − ny²)
  hot  = Σ_h a_h · exp(−((x − x_h)² + (y − y_h)²) / (2 σ_h²))
  lf   = Σ_w a_w · sin(kx_w·x + ky_w·y + ph_w)
  v  = (BUMPF·wb + glob·wg + hot)·view.scale·layer.f + lf·0.6·view.scale
  v += (rnd() − 0.5)·3.2·layer.noise·sqrt(view.scale)
  v += 0.9·layer.noise·cos(x·π/2)·view.scale
  if SRAM: v ×= 0.85
  mv = max(0.05, v)
if layer.smooth = r > 0:            # NaN-aware separable box blur, horizontal then vertical
  each pass: dst = mean of finite values within ±r along the axis; NaN stays NaN
pct = mv / vdd(domain) · 100
```

### 16.5 Reference numbers

Default state (IR, dvd_vectorless, INST, all domains, mV): n = 12 170 017 instances in
26 712 tiles; min 1.015, mean 20.114, P1 4.9523, P50 17.279, P90 35.783, P99 56.368, P99.5
62.618, P99.9 78.623, max 83.621 mV; worst tile (121, 25) = 83.621 mV = 11.149 % VDD; default
scale 0 / 0 / 62.6 / 85; over warn 73 431 instances, over limit 22 050 instances in 37 tiles,
1 cluster; above hot 60 968 instances (0.50 %); lowQuarter 0.417; Turbo 1 reversal.

Per metric/view/layer (mV; columns min, P50, P99, P99.5, max, worst % VDD, instances over
warn, instances over limit, limit tiles, clusters):

```
IR      static_ir       INST   0.4   6.1  15.2  16.6  21.4   2.86       0      0    0  0
IR      static_ir       M1     0.6   5.7  14.2  15.3  20.2   2.69       0      0    0  0
IR      static_ir       M4     1.2   4.3  10.3  11.2  14.3   1.91       0      0    0  0
IR      static_ir       M10    0.1   1.6   4.0   4.2   5.2   0.69       0      0    0  0
IR      static_ir       AP     0.1   0.8   1.9   2.0   2.2   0.29       0      0    0  0
IR      dvd_vectorless  INST   1.0  17.3  56.4  62.6  83.6  11.15   73431  22050   37  1
IR      dvd_vectorless  M1     2.2  16.2  52.5  58.4  78.8  10.51   56672   7645   13  1
IR      dvd_vectorless  M4     1.9  12.2  38.6  43.4  56.7   7.57       0      0    0  0
IR      dvd_vectorless  M10    0.3   4.7  13.6  15.6  19.9   2.65       0      0    0  0
IR      dvd_vectorless  AP     0.1   2.1   5.4   5.6   5.9   0.79       0      0    0  0
IR      dvd_vcd_func    INST   2.0  16.8  50.8  54.9  66.9   8.91   23004      0    0  0
IR      dvd_vcd_func    M1     1.7  15.5  46.3  49.8  62.1   8.28    4783      0    0  0
IR      dvd_vcd_func    M4     1.5  11.7  34.8  37.7  44.5   5.93       0      0    0  0
IR      dvd_vcd_func    M10    0.2   4.5  12.7  14.3  17.7   2.36       0      0    0  0
IR      dvd_vcd_func    AP     0.1   2.0   5.1   5.3   6.1   0.82       0      0    0  0
IR_VDD  static_ir       INST   0.1   3.8   9.7  10.6  14.3   1.91       0      0    0  0
IR_VDD  dvd_vectorless  INST   0.1  10.7  34.9  40.1  54.0   7.20       0      0    0  0
IR_VDD  dvd_vcd_func    INST   0.1  10.2  31.3  34.2  43.6   5.81       0      0    0  0
GB_VSS  static_ir       INST   0.1   2.4   6.0   6.5   9.1   1.21       0      0    0  0
GB_VSS  dvd_vectorless  INST   0.1   7.0  22.2  24.1  32.4   4.32       0      0    0  0
GB_VSS  dvd_vcd_func    INST   0.1   6.4  19.5  21.2  27.3   3.64       0      0    0  0
```

VDD_SRAM filter (IR, dvd_vectorless, INST): 108 617 instances in 1932 tiles, 6.309…26.5 mV,
worst 3.117 % VDD, no violations.

## 17. Qt 5 implementation guide

Assumed: Qt 5.15 LTS, Widgets (see the overview's port notes for what changes on 5.9–5.14,
under Qt Quick, or in Python).

### 17.1 Module split

```
hcs_core     (C++17 or numpy; no QtWidgets — QtCore at most)
  Palette, Lut            stops, OKLab/sRGB interpolation, 1024-entry table, lightness, reversals
  Cvd                     Machado matrices, gray
  Transfer                linear/sqrt/log/gamma/equalize/two-slope
  ScaleRange, ScaleStyle, Thresholds, ScaleSet (scope-keyed ranges + style + thresholds)
  ColorMapper             prepare() + colour(v) → (rgba, kind)        ← §4.9, golden-tested (§4.10)
  Stats                   sorted values, nearest-rank percentiles, histogram, under/over counts
  Violations              counts, mask, 4-connected clusters, contour segments
  Bands                   generators, clamp/rescale
  JsonIo                  heatmap-color-scale/1 read + write
  TclExport               classic config colormap text
hcs_gui      (QtWidgets)
  ScaleConfiguratorDialog, controllers, widgets (§17.3)
host renderer links hcs_core and calls the same ColorMapper.
```

### 17.2 One mapping implementation

`ColorMapper` is the only place a value becomes a colour. The dialog's histogram, ramp,
legend and preview and the host's full-chip renderer all use it. Its unit tests use §4.10
and a port of the fixture (§16) to check statistics against §16.4.

### 17.3 Widget tree

```
ScaleConfiguratorDialog : QDialog (non-modal; title "Heatmap Color Scale — <scope>[*]")
├─ ScopeBar : QWidget
│  ├─ QComboBox metric, QComboBox layer, QToolButton linkLayers (checkable)
│  ├─ QComboBox view, QToolButton lockViews (checkable)
│  ├─ QComboBox domain
│  └─ QComboBox preset (with separators/headers for Built-in, Saved)
├─ left column
│  ├─ DistributionPanel : QFrame
│  │  ├─ header: QLabel title, ScopeChip (QLabel), 3 × checkable QToolButton, QLabel stats
│  │  ├─ ScaleAxisWidget : QWidget            (custom paint, §7; focus policy StrongFocus)
│  │  ├─ RangeFieldsRow: 4 × UnitLineEdit (QLineEdit + UnitValidator), 2 × segmented
│  │  │                  (QButtonGroup of QToolButton), flip + cool_is_black QToolButtons
│  │  └─ AutoRangeBar: 5 × QToolButton (chip style) + hint QLabel
│  └─ QTabWidget
│     ├─ PaletteTab: PresetChipGrid, segmented interp, Reverse, Even out, StopEditorWidget,
│     │              stop fields, LightnessPlot, CvdStrips
│     ├─ MappingTab: segmented transfer, QSlider γ, TransferCurvePlot, out-of-range rows
│     │              (segmented + ColorSwatchButton → QColorDialog), QSlider opacity,
│     │              segmented numbers, QSlider legend font, snap toggle
│     ├─ BandsTab: QSpinBox (2–12), segmented generator, follow toggle, callout,
│     │            QTableView + BandTableModel + delegates (checkbox, swatch, UnitLineEdit)
│     ├─ ThresholdsTab: toggles, Add, QTableView + ThresholdTableModel + delegates
│     └─ ExportTab: segmented format, QLineEdit name, Save, Copy,
│                   QPlainTextEdit (read-only) + QSyntaxHighlighter
├─ right column (fixed width ≈ 568 px at 1×)
│  ├─ PreviewPanel: header toggles + QComboBox vision, PreviewWidget (heightForWidth 3:4),
│  │                QLabel readout
│  ├─ QGridLayout 2 × 2 KpiCard : QFrame
│  └─ InsightsPanel : QScrollArea of InsightItem (QLabel rich text + QPushButton)
└─ status row: QLabel ×2, QToolButton undo/redo (from QUndoStack::createUndoAction/RedoAction),
               QPushButton Reset range, Revert, Apply (default)
```

Toast: a child `QLabel` overlay positioned bottom-centre, hidden by a `QTimer` after 1.9 s
(fade optional). Help: a small `QDialog` or overlay widget; `?` and `F1`.

### 17.4 Painting recipes

| mock-up | Qt 5 |
|---|---|
| canvas with device-pixel columns | `QImage(cols, 1, Format_ARGB32_Premultiplied)` filled from `ColorMapper`, `drawImage` into the ramp rect with `SmoothPixmapTransform` off |
| heat layer | `QImage(gridW, gridH, …)`, nearest by default, smooth toggle |
| hatch | clipped diagonal `drawLine`s (pattern brushes are cosmetic in Qt 5) |
| shadow glow on handles, contour, stop selection | stroke 2–3 times underneath with wider, more transparent pens |
| limit glow | box-blur the mask at grid resolution (3 passes), scale up smoothly, `CompositionMode_Plus` |
| dashed lines | `QPen` with `setDashPattern` |
| rounded rects, pills, tags | `QPainterPath::addRoundedRect` |
| rotated axis labels | `save()`, `translate`, `rotate(±90)`, `drawText`, `restore()` |
| checkerboard | small tiled `QPixmap` brush with `setDevicePixelRatio` |
| contour | one `QPainterPath` of unit segments in grid coordinates, drawn under a `QTransform` scale with a cosmetic pen |
| fonts | `QFontDatabase::systemFont(QFontDatabase::FixedFont)` for numbers |
| dark plot areas | each custom widget fills its own background; the rest follows the host style |

### 17.5 High DPI and remote displays

`Qt::AA_EnableHighDpiScaling` and `Qt::AA_UseHighDpiPixmaps` before `QApplication`; on 5.14+
`setHighDpiScaleFactorRoundingPolicy(PassThrough)`. Allocate offscreen images at
`size · devicePixelRatioF()` and set their device pixel ratio. Over VNC/X forwarding: no idle
animation, partial `update(QRect)` for pulse and crosshair, the heat image rebuilt only on
data/mapping changes.

### 17.6 Model, controller, signals

- `ScaleSetModel` owns the scale set (§3.10 partition 1); emits `rangeChanged(scopeKey)`,
  `styleChanged()`, `thresholdsChanged()`.
- `ViewState` owns partition 2; emits `scopeChanged()`, `filterChanged()`, `displayChanged()`.
- `DataProvider` (host interface, §17.8) delivers values asynchronously.
- `ConfiguratorController` maps signals to the dirty flags of §14 and schedules one render
  per event-loop turn.

### 17.7 Undo

`QUndoStack` with `setUndoLimit(200)`. Simplest faithful design: one `SnapshotCommand`
holding before/after serialisations of the scale set (a few KB) — mirrors the mock-up.
Continuous gestures (wheel, arrow nudges) use `id()` + `mergeWith()` within a 350 ms window
instead of a debounce timer; drags push one command on release. "Modified" compares the
current serialisation to the applied one (not `QUndoStack::isClean()`, which cannot see
"edited back to the same values" and is lost when history branches). Revert pushes a
`SnapshotCommand` to the applied state.

### 17.8 Host interface

Shaped like the mock-up's data, which the user confirmed is close to what the real GUI reads.

```
class IrDataSource {                               // implemented by the host
  design()  → {name, run, domains: [{id, vdd_mV, colour}]}
  views()   → [{id, label, kind}]
  metrics(), layers()
  instanceCounts() → raster<uint32>                 // per tile, the same for every view
  tiles(metric, view, layer) → future<Raster>       // value mV (NaN = empty) + domain per tile
  blockAt(x, y) → path                              // floorplan / hierarchy
  bumps(), backdrop()
}
signal scaleApplied(ScaleSet)                       // dialog → host renderer
```

### 17.9 Persistence

User presets in `QSettings` (group `heatmapColorScale/presets`, JSON per preset, same
`{name, style, range-in-%VDD}` shape as §8.5.3). Scale-set persistence (per run / per user)
is deferred by the user (§20).

## 18. Resolved mock-up deviations

The first mock-up differed from this design in 18 places. All are fixed in the current
mock-up; the table keeps the record.

| # | was | now |
|---|---|---|
| 1 | every tile stood for a constant ≈ 463.6 instances | each tile carries its own instance count (data); every count and percentile is instance-weighted (§3.1, §3.4) |
| 2 | undo and Apply included the view selection and display options | scale set / view state / session UI split; only the scale set is undone, applied and exported (§3.10, §13.1) |
| 3 | style was global while ranges were per scope | style is per metric, ranges per scope; the tab bar says what a style edit affects (§3.7) |
| 4 | built-in presets set only some keys; others leaked from the previous preset | every preset sets the complete style (§12.1) |
| 5 | presets left band edges pushed against the old window | presets regenerate edges with their own generator (§12.1) |
| 6 | changing the band count resampled custom band colours | custom colours are kept where their index survives; new bands sample the palette (§8.3) |
| 7 | a track click could set a handle without the `eps` gap | clamped with `eps` immediately (§7.5) |
| 8 | palette stops could cross; the selection was an index into the unsorted list | stops stay sorted; the selection follows the stop; removal selects the neighbour (§8.1) |
| 9 | Tcl export wrote `-map IR` for every metric | metric → classic map id; metrics without one are refused (§8.5.2) |
| 10 | insight 7 said "100 mV more headroom" literally and fired whenever the filter was "all" | computed from the domains present in the filtered data (§11) |
| 11 | threshold lines on an mV axis gave no hint of their reference domain | flags read `@<domain>` when supplies with different VDD share the axis (§7.3) |
| 12 | units conversion of ranges used the reference VDD silently | same conversion, now announced by a toast (§13.4) |
| 13 | two-slope, `0 → limit` and "Set hot = limit" used the limit even when disabled | these controls and the presets that need the limit are disabled while it is off; an already selected two-slope map falls back to the window midpoint (§3.8) |
| 14 | export only | validated JSON import with errors and notes; round-trips byte for byte (§8.5.4) |
| 15 | automatic preset names used the browser locale's time | `YYYY-MM-DD HH:MM` (§8.5.3) |
| 16 | fake window buttons, `MOCK-UP` badge, "demo design" text | no fake chrome; design name and run from a data descriptor; a `synthetic data` tag in the status bar (§6.2, §6.4) |
| 17 | histogram and heat image recomputed on every render | keyed caches; hover renders in 1.7 ms (§14) |
| 18 | cause hint from the synthetic hotspot list | floorplan block lookup plus a bump-cell check, as the host would do it (§5.6) |

## 19. Acceptance checklist

Against the fixture (§16), default state unless noted. Each line is a test.

Mapping and statistics

1. All §4.10 vectors reproduce.
2. §16.5 statistics reproduce (n, percentiles, worst tile, counts, clusters).
3. Default scope key `eff|dvd_vectorless|*`; default scale 0 / 0 / 62.6 / 85.

Distribution widget

4. Bars are coloured by `colour(bin centre)`; bars beyond hot draw at 50 % opacity.
5. Flags read `WARN 60.0 mV @VDD_CORE · 73.4 k` and `LIMIT 75.0 mV @VDD_CORE · 22.1 k`; with
   the VDD_SRAM filter the `@` part disappears.
6. Dragging hot near 75 mV snaps to exactly 75 with pill `→ limit`; with Alt it doesn't.
7. Dragging on the histogram from 20 to 40 mV sets cool 20, hot 40 (one undo step).
8. Wheel over the ramp narrows/widens the window around the cursor; over the histogram it
   zooms the axis; double-click resets the axis to 0…85.
9. Hovering a histogram bin dims all map tiles outside it; hovering the map draws the value
   marker with its P-value on the histogram.

Preview

10. Worst marker at tile (121, 25), tooltip `83.6 mV`, `11.15 % of VDD · above hot
    (clamped)`, `VDD_CORE · 0.750 V · <n> instances`, `(1215, 1245) µm · soc/npu · missing
    bump`, badge `OVER LIMIT +1.15 %`.
11. Limit contour outlines one cluster of 37 tiles; glow pulses at 0.8 Hz; both toggle off.
12. Legend shows `IR::INST`, `[mV]`, labels `0.00 15.7 31.3 47.0 62.6`, the top extend
    triangle in the clamp colour `#900d00` and a black bottom triangle (`cool_is_black`).
13. `Blocks` outlines the floorplan; block labels match the cause hints.

KPIs and insights

14. KPIs as in §10.
15. Insights show rules 1, 2a, 6, 7 in that order; `Set hot = limit` sets hot 75 and rule 2a
    becomes 2b.
16. `0 → limit` → 0 / 0 / 75 / 85.

Modes, units, scope

17. `% VDD` → 0 / 0 / 8.347 / 11.33 with the conversion toast; KPIs keep the same counts;
    insight 7 disappears.
18. Domain VDD_SRAM → 108 617 instances, 6.309…26.5 mV, worst 3.117 %, no violations;
    threshold lines at 68 and 85 mV; VDD_CORE tiles dimmed; no undo step.
19. Threshold-bands preset → bands mode, edges 15 30 45 60 75, hot 86.25; band rows show live
    counts; disabling the third row (30–45 mV) makes those tiles transparent.
20. Unlinking layers, switching to M1, changing the range, switching back to INST shows the
    INST range unchanged; locking views and switching view keeps the range.
21. Choosing Inferno on IR, then switching to GB_VSS shows Turbo; switching back shows Inferno.

Commands and export

22. Every action in §13.1 creates exactly one undo step; view-state changes create none and
    never set the modified indicator; undoing a palette change after a view switch keeps the
    view.
23. Modified indicator appears after any scale edit and clears after Apply or after editing
    back to the applied values; Revert restores and is itself undoable.
24. Legacy-look then Colorblind-safe leaves OKLab interpolation (complete preset styles);
    Hotspot hunt then Bands shows evenly spaced edges from P50.
25. With the limit disabled: `0 → limit`, `Margin @ limit`, Two-slope and the three limit
    presets are disabled; `0` shows `Enable the limit threshold first`.
26. JSON and Tcl exports of the default state equal §8.5.1 / §8.5.2 (modulo JSON
    whitespace); the Tcl export of GB_VSS is refused.
27. Importing an exported file and exporting again gives identical text; a file in % VDD is
    converted with a note; malformed files list their errors and cannot be imported.
28. A saved preset re-applied in the other unit system yields the same window in % VDD; its
    default name ends in `YYYY-MM-DD HH:MM`.

## 20. Open questions

1. **Tile record** (decides §3.1, §17.8): the user says the real data is close to the
   mock-up's. Still to confirm: what a tile's value is (worst instance or mean — with
   worst-instance tiles, instance counts over a threshold are upper bounds) and whether
   per-tile instance counts are available (if not, counts become tile counts and the UI says
   "tiles").
2. **Persistence**: scale sets per design/run, per user, or both — **deferred by the user**
   (2026-11-21).
3. **JSON versioning** policy for `heatmap-color-scale/1` now that import exists.
4. **Per-net colormaps**: needed? (Unknown in the reference tool too.)
5. **Host language**: C++ or Python (PyQt5/PySide2)? Doesn't change the design.
6. **Classic Tcl**: verify `-values` boundary count and `-disable` indexing against the real
   tool; find classic map ids for non-IR metrics, if any.
7. Prior-art unknowns still open: the reference tool's log/sqrt support, its default rule for
   `hot`, its scripting access, saved-scale semantics.

## 21. Decision log

| decision | by | date |
|---|---|---|
| Build a mock-up from the prior-art research ("make it super cool"), end target native Linux GUI | user | 2026-11-21 |
| Toolkit: Qt 5 | user | 2026-11-21 |
| Real data is close to the mock-up's tile data | user | 2026-11-21 |
| Saved-scale persistence deferred | user | 2026-11-21 |
| No vendor or product names in the repository | user | 2026-11-21 |
| GitHub repo `smprather/heatmap-color-scale-configurator`, private | user (repo), Claude (visibility) | 2026-11-21 |
| Web mock-up, QPainter-reproducible drawing only | Claude | 2026-11-21 |
| Qt 5.15 + Widgets assumed | Claude | 2026-11-21 |
| Four-number ramp model as the core; classic bands as second mode | Claude (from prior art) | 2026-11-21 |
| Default palette Turbo, OKLab interpolation | Claude | 2026-11-21 |
| Default `below = black` (mirrors the reference screenshot), `hot = P99.5` | Claude | 2026-11-21 |
| Thresholds static 3 / 5 %, dynamic 8 / 10 % VDD; per-instance own-domain evaluation | Claude | 2026-11-21 |
| Range per scope key; style per metric; thresholds per analysis kind | Claude | 2026-11-21 |
| Scale set / view state / session UI split; only the scale set is undone and applied | Claude | 2026-11-21 |
| Statistics instance-weighted from per-tile counts | Claude | 2026-11-21 |
| Band edges stored as absolute values, pushed by handles, rescaled by field/auto edits | Claude | 2026-11-21 |
| Worst = largest own-domain % VDD | Claude | 2026-11-21 |
| Classic Tcl export approximates a ramp with 6 equal bands; refuses metrics without a classic map id | Claude | 2026-11-21 |
