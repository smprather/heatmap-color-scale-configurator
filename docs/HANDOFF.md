# Handoff

Updated 2026-11-21. Read this first, then
`docs/design/ir-drop-color-scale-configurator-detailed.md` (normative) or the overview next to
it.

## Rules from the user

- **No vendor or product names anywhere in the repository** (requested 2026-11-21). The prior
  art is described as "an established EMIR tool", its older generation as "classic" and its
  newer colormap dialog as "the reference dialog". Keep commit messages clean too.
- Qt 5 is the target toolkit.

## State

- Repo: `github.com/smprather/heatmap-color-scale-configurator` (private), branch `main`.
- `mockup/index.html` — single-file, offline mock-up. Synthetic data shaped like what the real
  GUI will read (user: "pretty similar"): a tile raster per metric × view × layer with value
  and supply domain, per-tile instance counts, bumps, floorplan blocks. All 18 deviations
  found in the first mock-up are fixed (detailed design §18).
- `docs/design/ir-drop-color-scale-configurator-detailed.md` — full design: data model,
  mapping rule with golden colour vectors (§4.10), every control and string, presets,
  undo/apply, render budgets, fixture with reference numbers (§16), Qt 5 implementation guide,
  acceptance checklist (§19). Numbers there were produced by running the mock-up headless.
- `docs/design/ir-drop-color-scale-configurator.md` — overview.
- `docs/research/prior-art-color-scales.md` — what is known about the established tool's
  colour scales. Further web research is not worthwhile (documentation is login-gated). If
  fidelity on an unknown matters, ask the user for a screenshot.
- `Screenshot_20261121_113444.png` — the reference colormap dialog; leave it in place.

## Decided

- User: Qt 5; real data close to the mock-up's tile data; saved-scale persistence deferred.
- Claude (open to change): Qt 5.15 + Widgets; range per scope key, style per metric,
  thresholds per analysis kind; only the scale set is undone/applied (view state is not);
  statistics instance-weighted from per-tile counts; classic Tcl export refused for metrics
  without a known classic map id.

## Open

- What a tile's value is (worst instance or mean) and whether per-tile instance counts exist.
- C++ or Python host; JSON versioning policy; per-net colormaps.

## Next

Before building the native app: turn the detailed design into an implementation plan
(spec → plan gate). No native code exists yet.

## Environment notes

- `uv`, `uvx`, `ffmpeg`, `tesseract`, headless Chromium are available. `pip` is absent from
  `/usr/bin/python3`; use `uv`.
- The mock-up exposes `window.__app` (state, render, actions, exports, import parser) for
  headless checks over the Chrome DevTools Protocol.
