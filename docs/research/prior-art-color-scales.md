# Prior art: colour scales in an established EMIR tool

Status: complete for what was learned. Written 2026-11-21; vendor and product names removed
on the user's request (keep it that way).

Purpose: establish how an established EMIR signoff tool configures heat-map colour scales, so
the configurator is modelled on practice rather than invented. Two generations of the tool
were studied: an older GUI with banded scales (called **classic** here) and a newer one with
a continuous ramp set by four numbers (called the **reference dialog** here).

## Sources

- The classic generation's user manual and the newer generation's training material. Neither
  is kept in this repository; the vendor documentation is login-gated, so further web
  research is not worthwhile.
- A screenshot of the running reference dialog, supplied by the user on 2026-11-21:
  `Screenshot_20261121_113444.png` at the repository root (leave it in place).

## Classic generation — banded colour scale

One colour dialog per metric: voltage drop (IR), electromigration (EM), current, power
density, instance power, clock instance power, leakage power, instance frequency,
decap/device capacitance density, thermal profile. Reached from a "Set Color Range" button
or a per-metric map-configuration menu entry.

Anatomy:

- A **table of ranges**. Each row has an enable checkbox, a colour swatch (opens a colour
  picker) and the range's value.
- Default: **six equal steps** between the dialog's maximum and minimum.
- Ranges can be switched on and off independently; "Default" restores the original display.
- Voltage maps choose **percent of VDD** or **absolute volts**; a VDD-domain dropdown picks
  the reference, and the mV column recalculates against it (the colour map itself still
  reflects the worst domain — the manual calls this out as a trap in multi-VDD designs).
- IR maps choose wires or instances.
- Density maps use a **tile grid, 30 µm squares by default**, editable as grid count or size.
- A layer dialog has a "reassign colours" option that re-runs the layer palette.
- Legend text size 7–32 px, default 10.
- Palette direction: highest values red, lowest blue/magenta.

Scripting equivalent, the machine-readable version of the same model:

```
config colormap -map <ID> -min <v> -max <v> [-wire|-instance] [-percent|-absolute]
                [-enable all] [-values {<list>}] [-disable {<list>}]
                [-gridsize {x y}] [-gridnum {x y}] [-refvdd <v>] [-refmax <v>]
                [-gndref on|off]
```

- `-values {…}` lists explicit band boundaries; `-disable {…}` hides bands by index. The
  manual's example `config colormap -map EM -values {0 0 0 75 80 85 90 95 100}
  -disable {0 1 2 3}` gives an EM map showing only ratios ≥ 80 %.
- `-refmax` rescales the reference maximum (capacitance for density maps, power for power
  maps).
- `dump gif -map <ID> -o <file> [-nolegend]` writes the map; a saved GUI configuration can be
  re-imported. Automated sweeps re-run `config colormap -percent -min 0 -max $i` in a Tcl loop
  and dump one image per range.

## Reference dialog — continuous ramp, four numbers, two flags

Data model: heat maps are first-class objects (per instance or per geometry), queryable in
bulk; graphics are one rendering of them. Each has per-layer statistics over the full
dataset, and the chip is partitioned into adjoining sections so every object belongs to one
partition. Heat maps are generated per view (each analysis result has its own).

GUI: the layout window lists views with pre-generated heat maps; nets and layers carry a map
selector to enable a heat map on that layer; a legend on/off button sits with the navigation
controls, plus "clear highlights" and "highlight regions with high value in heatmap".

### The colormap dialog (from the screenshot and training text)

```
+------------------------------------------------------------------+
| colormap CURR::M1...av                                    _ [] X |
|   [count axis]   histogram curve of the full-chip distribution   |
|   +---------------------------+--------------------------------+ |
|   |  gradient blue->green->red | o handle o                    | |
|   min 5.8539e-08  cool 5.8539e-08  hot 0.0018428  max 0.002134   |
|    [ ] flip                                                      |
|    [x] cool_is_black                                             |
+------------------------------------------------------------------+
```

| field | meaning |
|---|---|
| `min` | lower bound of histogram/domain; left end of the track |
| `cool` | value painted the cold end of the ramp; left handle; defaults to `min` |
| `hot` | value painted the hot end; right handle; sits below `max` |
| `max` | upper bound of the domain |
| `flip` | reverse ramp direction |
| `cool_is_black` | the below-`cool` clamp region is painted black rather than the cold colour |

Semantics: the grey track spans `min…max`; the gradient is drawn only between the handles
(`cool…hot`); values outside clamp to the end colour. The histogram shares the bar's x-axis,
which is exactly why the dialog is useful — the training text says it is "used as a guide
when changing the value range to display in the heatmap". Default `cool = min` and
`hot < max` means the saturated ends sit inside the data range, so a long tail does not
flatten the map.

`cool_is_black` cannot be recolouring the ramp's cold end — the gradient is still blue at the
left handle in the screenshot — so it suppresses the clamped sub-`cool` fill (background /
negligible values). This reading is inferred from pixels, not text.

Title format `colormap <METRIC>::<LAYER>...<view>`: one colormap instance per
metric × layer × view. Metric ids match the classic family (CUR, IR, EM, PD, …).

## Classic vs reference — the change that matters

| | classic | reference dialog |
|---|---|---|
| scale type | discrete bands, each enable/disable + colour | continuous ramp between two handles |
| parameters | list of band boundaries | `min, cool, hot, max` |
| out of range | bands exist as rows, can be hidden | clamped at both ends; cold end optionally black |
| direction | fixed (red = worst) | `flip` |
| guidance | none in the dialog | full-chip histogram behind the bar |
| scope | per metric, global view config | per metric × layer × view |

## Still unknown

- Whether the reference dialog offers log/sqrt or other non-linear scales.
- Named palettes/presets beyond the rainbow ramp in the screenshot.
- Whether ranges are saved with the view/database or per user, and whether they can be set
  from scripts (a `config colormap` equivalent).
- The rule for `hot`'s default value (percentile? fixed fraction of `max`?).
- Whether per-net (as opposed to per-layer) colormaps exist.

## Implications for a configurator (proposal, not vendor fact)

Model the scale as: domain (data extent), `cool`/`hot` handles inside it, ramp stops, `flip`,
`cool_is_black`, plus clamp semantics at both ends; show the value histogram on the same
x-axis as the bar. Keep the classic band editor as an alternative display mode for metrics
judged on thresholds (EM ratio ≥ 100 %), because bands and thresholds are how signoff is
argued, while a continuous ramp is how hotspots are found.
