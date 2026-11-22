# Heatmap colour-scale configurator

Design and clickable mock-up of a colour-scale editor for IR-drop heat maps. It builds on how
established EMIR tools configure their maps — a four-number ramp dialog (`min / cool / hot /
max`, `flip`, `cool_is_black`, histogram on the bar's axis) and the classic band table — and
adds signoff thresholds, per-domain `% VDD`, histogram ↔ map brushing, a palette lab and live
scale insights.

The end product is a native Linux GUI on **Qt 5**. Nothing here is that product yet.

## Status

Mock-up stage. `mockup/index.html` is a single offline HTML file driven by synthetic SoC
data shaped like what the real GUI will read (a tile raster with values, supply domains and
instance counts); it exists to settle the design, not to be ported line by line.

## Try the mock-up

```sh
xdg-open mockup/index.html
```

Best at 1600 px wide or more. Press `?` inside it for gestures and shortcuts.

## Layout

| path | what |
|---|---|
| `mockup/index.html` | the mock-up (no build, no network) |
| `docs/design/ir-drop-color-scale-configurator.md` | design overview: model, mapping rule, interactions, Qt 5 port notes, JSON format, open questions |
| `docs/design/ir-drop-color-scale-configurator-detailed.md` | the full, normative design: every control, formula, default, string and gesture; golden colour vectors; fixture numbers; Qt 5 implementation guide; acceptance checklist |
| `docs/research/prior-art-color-scales.md` | how an established EMIR tool configures its colour scales |
| `docs/HANDOFF.md` | session handoff notes |
| `Screenshot_20261121_113444.png` | the reference colormap dialog the design starts from |
