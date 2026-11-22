# IR-drop heat-map colour-scale configurator — design

Status: mock-up stage, 2026-11-21. Clickable mock-up: `mockup/index.html` (open in any
browser, offline, synthetic data). The end target is a native Linux GUI built on **Qt 5**;
the mock-up only uses drawing operations QPainter can reproduce (see the port notes for the
few that need a workaround in Qt 5).

Prior art (how an established EMIR tool configures its colour scales):
`docs/research/prior-art-color-scales.md`.

This is the overview. The full, normative description (every control, formula, default and
string, golden test vectors, Qt 5 implementation guide, acceptance checklist) is
`ir-drop-color-scale-configurator-detailed.md`; where the two disagree, that one wins.

## Intent

A colour-scale editor for IR-drop heat maps that does two jobs at once:

- **find hotspots**: a continuous ramp tuned so gradients inside the hot regions stay visible;
- **argue signoff**: thresholds (`warn`, `limit`) that the scale can be pinned to, so “red”
  provably means “over budget”, with counts of offending instances.

It keeps the four-number ramp model of the reference dialog (`min / cool / hot / max`,
`flip`, `cool_is_black`, histogram on the bar's axis) and the classic band table, then adds
what those dialogs lack.

## Goals, non-goals, anti-goals

Goals

- Tune a scale against the data: histogram, CDF and percentiles share the bar's x-axis.
- Signoff-aware: thresholds are first-class, evaluated **per instance against its own supply**.
- Comparable maps: one scale can be shared across layers and/or locked across views.
- Every colour decision is visible: clamping, disabled bands and over-limit are never silent.
- Portable output: a JSON scale definition; classic Tcl `config colormap` as best effort.

Non-goals

- Rendering the full-chip heat map (the configurator drives the existing renderer).
- A bridge into the existing EMIR tool (its colormap scripting access is unknown).

Anti-goals (fails even if it “works”)

- A scale whose saturated region looks like “hottest in range”.
- Multi-VDD maps silently normalised to one reference (the classic tools' trap).

## Scale model

Three kinds of state:

- **Range** (`min`, `max`, `cool`, `hot`, band `edges`): one per **scope key**
  `metric | view | layer`; `*` replaces view or layer when locked or linked.
- **Style** (palette, mode, transfer, out-of-range, flip, band settings): one per **metric**,
  shared by all its views and layers (the tab bar says `style applies to every IR map`).
- **Thresholds**: one list per analysis kind (static / dynamic), shared everywhere.

Together with `units`, link/lock and legend settings these form the **scale set**: the only
thing that is undone, applied and exported. Which map is shown, the domain filter and preview
toggles are view state.

| field | meaning |
|---|---|
| `min`, `max` | axis extent of histogram and track (data extent by default, nice-rounded) |
| `cool`, `hot` | ramp ends; values outside clamp. Default `cool = min`, `hot = P99.5` |
| `edges[]` | interior band boundaries, absolute values (Bands mode) |
| `flip` | reverse ramp / band colours |
| `units` | `mV` (absolute) or `% VDD` (per-instance own-domain normalisation) |
| `mode` | `ramp` or `bands` |
| `transfer` | `linear`, `sqrt`, `log` (log1p, k = 60), `gamma` (γ), `equalize`, `two_slope` (centre = limit) |
| `palette` | named stops `[(p, #rrggbb)]`, interpolated in OKLab (or sRGB) into a 1024-entry LUT |
| `below` | `clamp` · `black` (= `cool_is_black`) · `none` (transparent) · custom colour |
| `above` | `clamp` · custom colour (default magenta `#ff3df0` in the signoff preset) |
| `bands` | per band: colour, enabled; colours follow palette until edited |
| `thresholds` | per analysis kind (static / dynamic): name, % VDD, colour, enabled |

Dragging `cool` past an interior edge pushes it; edges never cross.

### Mapping rule (the renderer must match this exactly)

```
colour(v):
  NaN                         → transparent (layout shows)
  bands mode:
    v < edge[0] (= cool)      → below
    v > edge[n] (= hot)       → above
    band k disabled           → transparent      (classic -disable)
    else                      → bandColour[k] (reversed if flip)
  ramp mode:
    v < cool → below ; v > hot → above
    t = transfer((v - cool) / (hot - cool))
    if flip: t = 1 - t
    LUT[round(t * 1023)]
```

`below = clamp` uses the ramp's cold end (after flip). `two_slope` maps cool…limit to the
lower half of the palette and limit…hot to the upper half. `equalize` uses the CDF of
in-range values.

Thresholds count `pct[i] > t.pct` using each instance's own VDD. On an mV axis the line is
drawn at the selected domain's VDD (VDD_CORE when “All”) and its flag says `@VDD_CORE`.

Data: a tile raster per analysis result, each tile with a value, a domain and an instance
count (the user confirmed the mock-up's data is close to what the real GUI reads). All
statistics are instance-weighted.

## Window layout

```
┌ Heatmap Color Scale   IR::INST…dvd_vectorless          ● modified — not applied         ? ┐
│ Metric ▾  Layer ▾ [Link layers]  │ View ▾ [Lock across views] │ Domain ▾      Preset ▾    │
├───────────────────────────────────────────────┬────────────────────────────────────────────┤
│ DISTRIBUTION & RANGE  shared·all layers  log CDF pctl        n · mean · P99 · max         │
│  P50      P90            P99      P99.9                       │ LIVE PREVIEW  Legend Bumps… │
│ ▕▁▂▄▆█▇▆▅▄▃▃▂▂▁▁ ▁ ▁  ┊WARN 60 mV·73k ┊LIMIT 75 mV·22k  CDF  │ ┌────────────────────────┐ │
│ ▕▒▒◆━━━━━━━━━━━━━━━━━━━◆▒▒▒▒▒▒▒▒▒▒▒▏  (hatched = clamped)    │ │ legend │ die heat map    │ │
│  0    10    20    30    40    50    60  ▲  70  ▲  80 mV      │ │        │ bumps, domains, │ │
│ [min][cool][hot][max]  mV|%VDD  Ramp|Bands  flip  cool_is_black│ │ worst, limit glow│ │
│ Auto: Fit · min→P99.5 · P1→P99 · 0→limit · Margin@limit       │ └────────────────────────┘ │
├───────────────────────────────────────────────┤ x · y · mV · %VDD probe readout          │
│ Palette | Mapping | Bands | Thresholds | Export               │ ┌ Worst ┐ ┌ Over limit ┐     │
│  preset chips · stop editor · lightness plot · CVD strips     │ ┌ Warn  ┐ ┌ Saturation ┐     │
│                                                               │ Scale insights (live)        │
├───────────────────────────────────────────────┴────────────────────────────────────────────┤
│ demo_soc · r01 · 12.17 M inst · synthetic   scale IR::INST…  ramp · Turbo  ↶ ↷ Reset Revert Apply │
└──────────────────────────────────────────────────────────────────────────────────────────────┘
```

## Interactions

| where | gesture | effect |
|---|---|---|
| ramp | drag handle | move cool/hot; magnetic snap (7 px) to P50/P90/P99/P99.5/P99.9, data min/max, 0, thresholds; `Alt` disables |
| ramp | drag between handles | pan the window, keeping its width |
| ramp | click empty track | nearest handle jumps there |
| ramp | wheel | zoom the cool–hot window around the cursor |
| ramp (bands) | drag white tick | move an interior edge, bounded by neighbours |
| histogram | hover | **brush**: instances in that bin light up on the map, the rest dim |
| histogram | drag | rubber-band select → becomes cool…hot |
| histogram | wheel / double-click | zoom axis (min/max) / reset axis to data |
| map | hover | crosshair, tooltip (value, other unit, domain, instances in tile, xy µm, block, band/ramp %, over-limit badge); marker on the histogram |
| keys | `← →` (`Shift`) | nudge last-used handle 0.5 % (5 %) of axis |
| keys | `F B U A L 0 1–5` | flip, ramp/bands, units, fit, legend, 0→limit, tabs |
| global | `Ctrl+Z / Ctrl+Shift+Z`, `Ctrl+Enter` | undo/redo (scale set only), apply |

Edits preview live; **Apply** commits to the renderer, **Revert** returns to the last
applied state; the title shows “modified — not applied” in between.

## What the mock-up adds beyond the prior art

- Histogram bars coloured by the scale itself, log counts, CDF, percentile ticks.
- Threshold lines with live counts; one-click `0 → limit`; distinct above-hot colour.
- Brushing histogram ↔ map; worst-drop locator with cause hint (floorplan block, missing bump).
- Per-domain `% VDD`, domain filter that dims other supplies.
- Link layers / lock views for fair comparison.
- Transfer functions incl. two-slope “margin to limit” with a diverging palette.
- Palette lab: OKLab interpolation, lightness-monotonicity check, CVD simulation (deutan,
  protan, tritan, grayscale print), stop editor.
- Legend preview with extend triangles showing clamp colours; font 7–32 px (classic range).
- Insights panel: hot below limit, heavy clamping, ramp under-use, lightness reversals,
  shared mV scale across supplies — each with a one-click fix.
- Saved presets (complete styles); JSON export and validated import; classic Tcl band export
  (refused for metrics without a classic map id).
- Controls that need the limit threshold are disabled while it is off.

## Native port notes (Qt 5 Widgets)

Qt 5 is the target (user, 2026-11-21). Assumed until told otherwise: **Qt 5.15** (the last
Qt 5 LTS, what RHEL 8/9 ship) and **Widgets**, not Qt Quick. Everything below also works on
Qt 5.9 (RHEL 7) except the fractional scale-factor policy, which needs 5.14.

| mock-up part | Qt 5 |
|---|---|
| distribution + ramp + axis | one custom `ScaleAxisWidget` (`QWidget::paintEvent`, shared x transform). Ramp = 1-px `QImage` row from the LUT, scaled; clamped-region hatch as clipped diagonal lines (not a pattern brush, see below); handle glow = stacked translucent strokes |
| live preview | `QImage(Format_ARGB32_Premultiplied)` per tile grid → `drawImage` (nearest = default, smooth toggle = `SmoothPixmapTransform`); overlays painted on top; limit glow = box-blurred mask with `CompositionMode_Plus`; pulse via `QVariantAnimation` |
| scope bar | `QComboBox` ×4, checkable `QToolButton` ×2 |
| number fields | `QLineEdit` + validator accepting `45`, `45mV`, `0.045V`, `6%` (unit-aware parse) |
| segmented controls | `QButtonGroup` of checkable `QToolButton`s |
| tabs | `QTabWidget`; bands / thresholds = `QTableView` + delegates (colour swatch → `QColorDialog`) |
| stop editor, lightness, CVD, curve | small custom-painted widgets |
| KPIs, insights | `QFrame` cards; insights as `QLabel` rich text + action buttons |
| undo / apply | `QUndoStack` (in QtWidgets in Qt 5; Qt 6 moved it to QtGui) with one command per committed gesture (drag end, field commit) |
| number font | `QFontDatabase::systemFont(QFontDatabase::FixedFont)` for values and axis labels |

Qt 5 specifics

- **High DPI is off by default.** Set `Qt::AA_EnableHighDpiScaling` and
  `Qt::AA_UseHighDpiPixmaps` before constructing `QApplication`; on 5.14+ also
  `QGuiApplication::setHighDpiScaleFactorRoundingPolicy(Qt::HighDpiScaleFactorRoundingPolicy::PassThrough)`
  so 1.25× / 1.5× desktops are not rounded. Allocate offscreen images at
  `size * devicePixelRatioF()` and call `setDevicePixelRatio()`.
- **Pattern brushes are cosmetic.** Qt 5 draws `Qt::BDiagPattern` in device pixels, so the
  hatch would change density with the scale factor. Draw it as diagonal lines inside a clip.
- **No image blur.** Qt 5 has no public blur for a `QImage` (`QGraphicsBlurEffect` blurs
  whole widgets). Limit glow: 3-pass box blur of the over-limit mask at tile resolution
  (cheap: a few hundred cells a side), scaled up smoothly and drawn with
  `CompositionMode_Plus`. Canvas `shadowBlur` on handles and the limit contour becomes the
  same path stroked 2–3 times underneath with wider, more transparent pens.
- **Style sheets have no `box-shadow` or transitions.** The mock-up's glows on selected
  chips and focused fields become a 1-px accent border. `QGraphicsDropShadowEffect` exists
  but forces offscreen rendering of the widget; avoid it in the dialog.
- **No colour-scheme API** (that is Qt 6.5+). The plot widgets paint their own fixed dark
  background whatever the host theme, because heat-map colours must be judged against a
  constant neutral surround; the rest of the dialog follows the host app's style.
- **Remote displays.** EDA GUIs often run over VNC or X forwarding. The pulse and hover
  crosshair repaint only their own rectangles, the pulse stays a user toggle, and the heat
  image is rebuilt only when data or mapping changes, never per animation frame.
- **Language.** The table fits C++ and PyQt5/PySide2 alike. In Python, the LUT build,
  histogram and value→colour mapping must be numpy-vectorised; per-instance Python loops
  will not keep up with millions of instances.

If the host app turns out to be Qt Quick 2, the custom-painted parts become
`QQuickPaintedItem`s and the controls Qt Quick Controls 2; the design does not change.

Keep `colour(v)` and the LUT build in one non-GUI module so the renderer and the
configurator cannot drift, and so it can be unit-tested against the mock-up's output.

## JSON scale definition (v1)

```json
{
  "format": "heatmap-color-scale/1",
  "scope": {"metric": "IR", "layer": "*", "view": "dvd_vectorless"},
  "units": "mV", "normalization": "absolute",
  "range": {"min": 0, "cool": 0, "hot": 62.6, "max": 85},
  "flip": false, "mode": "ramp",
  "transfer": {"type": "linear"},
  "palette": {"name": "Turbo", "interpolation": "oklab", "stops": [[0, "#30123b"], [1, "#7a0403"]]},
  "out_of_range": {"below": "clamp", "above": "#ff3df0"},
  "thresholds": [{"name": "Limit", "pct_vdd": 10, "color": "#ff4d6d", "enabled": true}],
  "legend": {"font_px": 11, "numbers": "eng"}
}
```

## Calls made in the mock-up (change freely)

- Default palette Turbo, `below = black` (mirrors the reference screenshot), `hot = P99.5`.
- Thresholds per analysis kind: static warn 3 % / limit 5 %; dynamic warn 8 % / limit 10 %.
- Thresholds are policy, shared everywhere; range is per scope key; style is per metric.
- Only the scale set is undoable and applied; view selection and preview toggles are not.
- Band edges stored as absolute values; regenerated if they collapse while in ramp mode.
- Classic Tcl export approximates a ramp with 6 equal bands and says so in comments.

## Open questions

1. Toolkit: **Qt 5** (settled 2026-11-21). Still assumed: 5.15, Widgets, and whether the
   host app is C++ or Python — none of these change the design, only the port notes.
2. Data in: close to the mock-up's tile data (user, 2026-11-21). Still to confirm: what a
   tile's value is (worst instance or mean) and whether per-tile instance counts exist.
3. Persistence (per design/run, per user, or both): deferred by the user.
4. Per-net colormaps: needed? (Unknown in the reference tool too.)
5. Still unknown from the prior art: log/sqrt scales, the rule for `hot`'s default, scripting
   access to the reference colormap.
