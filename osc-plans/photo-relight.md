---
generated_by: osc-newfeature
type: feat
repo: storytold/photocraft
bulk_run: true
feature_complexity: M
dogfooded_general: false
stem: photo-relight
---

# Relight: redirect a photo's lighting

## Problem

Changing where the light appears to come from in a finished photo is a stack of gradient fills, overlays, and masks. PhotoCraft already has **Filter › Lighting Effects**, but that filter shades the image as a bumped 3D *surface* (Blinn–Phong on a height map). It is the wrong tool for a photograph: it adds fake relief instead of moving the existing illumination.

Editors who shoot or retouch portraits, interiors, and product shots need a single, undoable control: pick a direction, strength, and warmth, and the subject looks as if the lamp moved.

## User-facing behavior

- **Filter › Relight…** opens the standard live-preview filter dialog (schema-driven, same as other `filter.*` commands with parameters).
- Sliders: **Angle** (−180…180°, 0° = light from the right, 90° from above in image space), **Elevation** (0…90°, 0 = grazing, 90 = overhead), **Intensity** (0…100, default 40), **Ambient** (0…100, default 55), **Warmth** (−100…100, 0 = white light, positive = warmer, negative = cooler), **Softness** (1…100, default 25; how broad the estimated shading is).
- **Intensity = 0** is a true no-op: the pixels are unchanged (identity). Useful as a dialog reset and as a test.
- Honours the current selection (feathered). Outside the selection, pixels stay put.
- Pixel layers: destructive, one history step "Relight".
- Smart objects: recorded as a smart filter and re-rendered from the source (existing `filters.rs` path). `.pcraft` keeps it. PSD export of this PhotoCraft-only filter is omitted from `filterFX` with the same warning already used for other unmapped filters; the document is not corrupted.
- RGB at 8 / 16 / 32-bit float. Gray: relight luminance only. CMYK / Lab: convert through the existing filter `Ctx` RGBA helpers (same as Lighting Effects), write back.
- Pixel-locked layers and "no document" disable it. Invalid / non-finite params → `Err`, no edit.
- Desktop: runs as a background job (`jobs::edit_job`) with per-tile cancel, like other filters. Wasm stays inline.

## Engine command API

```
id:       filter.render.relight
label:    Relight…
menu:     ["Filter", "Render"]
shortcut: none
params:   {"angle":-180..180=45,"elevation":0..90=40,"intensity":0..100=40,"ambient":0..100=55,"warmth":-100..100=0,"softness":1..100=25}
enabled:  has_pixel_or_channel (same predicate as other filters)
journal:  true
```

Register in `filters_ext::specs()` / `params_for` so `run_filter` and `filter.lastFilter` pick it up. Do **not** add a `menu_catalog.rs` row (that list is the frozen menu tree). Extra commands with a `menu` path are merged in `menus.rs`. Add:

```
PLACE_AFTER: ("filter.render.relight", "filter.render.lightingEffects")
```

so it sits next to Lighting Effects under Filter › Render.

### Algorithm (`photocraft_algo::relight`)

Classical intrinsic-image relight, public ingredients only (Retinex shading + Lambertian relight). Work **per tile** with a halo of `~3 * sigma` pixels, `sigma = (softness/100) * 0.08 * min(bounds.w, bounds.h)`, clamped to `[2, 96]`. Use `get()` for every input-derived index. Check `Interrupt` once per tile (existing `apply_in_with` hook). No `unsafe`.

For each output pixel in linear-ish display RGB (the filter `Ctx` RGBA path):

1. Luminance `Y = 0.2126 R + 0.7152 G + 0.0722 B` (alpha-weighted; skip or copy pixels with α = 0).
2. Shading `S` = guided or box/Gaussian blur of `Y` with radius `sigma` (guided filter already lives in `matting.rs`; a separable Gaussian is enough for v1). `S = S.max(0.04)`.
3. Albedo `A = rgb / S`.
4. Surface normal from shading gradients: `nx = (S(x-1) - S(x+1)) * scale`, `ny = (S(y-1) - S(y+1)) * scale`, `nz = 1`, then normalize. `scale` is a small constant (~2) so a smooth photo does not explode.
5. Light vector from `angle` / `elevation` (same spherical mapping Lighting Effects uses for infinite lights): `L = normalize(cos(θ) cos(e), −sin(θ) cos(e), sin(e))`.
6. New shading `S' = (ambient/100) + (intensity/100) * sat(dot(N, L))`.
7. Warmth: multiply `S'` by a light colour `mix((1,1,1), (1.15, 0.95, 0.75), warmth/100)` for warmth > 0, and `mix((1,1,1), (0.80, 0.90, 1.15), −warmth/100)` for warmth < 0.
8. `rgb' = A * S'`. Preserve hue slightly better by optionally applying the ratio `S'/S` to luminance only (`mode` not in v1; always RGB scale).
9. Alpha unchanged. Clamp finite; if any channel is NaN/inf, keep the source pixel.

`FilterParams::halo` for Relight: `Halo::Radius((3.0 * sigma).ceil() as i32 + 1)` with sigma derived from softness and bounds. If computing sigma needs bounds, use `Halo::Bounds` (simpler, still correct; slower). Prefer a conservative radius of 97 so tiling stays parallel.

Identity: `intensity == 0` → copy source (do not run the blur).

## Files to touch

| File | Change |
|---|---|
| `crates/algo/src/relight.rs` | **New**: `relight(...)` kernel + unit tests. |
| `crates/algo/src/lib.rs` | `mod relight;`, `FilterParams::Relight { angle, elevation, intensity, ambient, warmth, softness }`, `label()`, `halo_ext`, `apply` match arm. |
| `crates/algo/src/tests_ext.rs` | Directional test (see below) if not kept in `relight.rs`. |
| `crates/engine/src/filters_ext.rs` | `params_for("filter.render.relight" => ...)`, `cmd!(...)` in `specs()`, clamp every numeric field. |
| `crates/engine/src/filters_ext/tests.rs` | Include `"filter.render.relight"` in the command-id list that must run / reject bad params. |
| `crates/ui-egui/src/menus.rs` | `PLACE_AFTER` entry above. |
| `crates/ui-egui/src/filter_dialog.rs` | Add `"filter.render.relight"` to the `has_dialog` assertion list. Dialog is automatic from the params spec (`angle` already gets a `°` suffix). |
| `crates/ui-egui/src/i18n/{ja,zh-hant,es,ru,cs}.tsv` | **Required** (`complete_menus`): row for source `Relight…`. Optional: `zh-hans`. Clean-room translations from the English word. |
| `docs/parity.md` | Regenerated by `cargo xtask parity` (live catalog count unchanged). |

No `photocraft-format` / `Adjustment` enum change in this PR: it is a filter, so smart-object non-destructiveness is free. A later PR can promote the same kernel to an adjustment layer if Properties sliders are wanted without a smart object.

GPU compositor: no change. Filters run on the CPU surface before compose.

## Tests

**Algo (`relight.rs`):**

- **Identity:** `intensity: 0` on 8/16/32-bit RGB and Gray → bit-identical (or ≤ 1 ULP on f32) to the source, including alpha.
- **Direction:** synthetic shaded sphere (or a left-to-right luminance ramp) on mid-gray albedo. Light from the left (`angle = 180`) makes the left half brighter than the right; light from the right (`angle = 0`) flips that. Δ mean luminance ≥ 0.05.
- **Warmth:** `warmth: 80` raises R relative to B on a neutral gray field with intensity > 0; `warmth: -80` the opposite.
- **Ambient:** high ambient + modest intensity does not clip a white pixel to non-finite.
- **Alpha:** transparent pixels stay transparent; opaque stay opaque.
- **Tiles:** `apply` on the full bounds vs 32×32 tiles with the declared halo — max channel delta ≤ 1e-4 (no seams).
- **Hostile sizes:** 1×1, 2×2, empty `Rect` — no panic, `Err` or identity.
- **NaN in / inf in:** those pixels pass through; neighbours that are finite still relight.

**Engine:**

- Command exists, menu path `Filter / Render`, enabled with a raster layer, disabled with no document / type layer.
- `execute` with defaults changes a shaded disc; undo restores.
- Smart object: after convert-to-smart + Relight, `smart_filters` contains this id; disabling the smart filter restores the un-relit pixels.
- Bad params: `intensity: 999`, `elevation: -3`, `angle: "east"`, `softness: 0`, `null` body → `Err`.
- Selection: only the selected rectangle moves; a corner outside it stays equal to the source.

## Verification commands

```sh
CARGO_TARGET_DIR=target/agent-relight cargo test -p photocraft-algo --lib relight
CARGO_TARGET_DIR=target/agent-relight cargo test -p photocraft-engine --lib filters_ext
CARGO_TARGET_DIR=target/agent-relight cargo clippy -p photocraft-algo -p photocraft-engine -p photocraft-ui-egui --all-targets -- -D warnings
CARGO_TARGET_DIR=target/agent-relight cargo xtask layers
CARGO_TARGET_DIR=target/agent-relight cargo xtask wasm
CARGO_TARGET_DIR=target/agent-relight cargo xtask parity
CARGO_TARGET_DIR=target/agent-relight cargo test -p photocraft-engine --test panic_hunt -- --ignored
```

Visual (public-domain portrait or interior; User-Agent `Photocraft-dev`):

```sh
# Before: photo as opened.
cargo run --release -p photocraft-ui-egui --example snapshot -- \
  --out /tmp/relight-before.png --size 1440x900 --open <public-domain.jpg>

# Dialog + a strong left light.
cargo run --release -p photocraft-ui-egui --example snapshot -- \
  --out /tmp/relight-dialog.png --size 1440x900 --open <public-domain.jpg> \
  --script '[["ui.menu.invoke", {"id":"filter.render.relight"}]]'

# Applied: angle 180, intensity 70, warmth 25.
cargo run --release -p photocraft-ui-egui --example snapshot -- \
  --out /tmp/relight-after.png --size 1440x900 --open <public-domain.jpg> \
  --script '[["engine.execute", {"command":"filter.render.relight","params":{"angle":180,"intensity":70,"warmth":25}}]]'
```

Attach before / dialog / after to the PR.

## Demo storyboard (30–45 s)

1. **0–6 s.** Open a public-domain head-and-shoulders or still-life with a clear light side. Say the light is "stuck" on the left.
2. **6–12 s.** Filter › Relight. The live-preview dialog opens on a proxy.
3. **12–22 s.** Drag **Angle** from ~45° through 180°. The bright side travels around the subject; no embossed-metal look (that would mean the wrong filter).
4. **22–30 s.** Raise **Warmth**, then **Intensity**; drop **Ambient** so the fill light recedes. Pause on a dramatic frame.
5. **30–38 s.** OK. Layers / History show one "Relight" step. Undo / Redo.
6. **38–45 s.** Optional: Convert to Smart Object first, Relight again, toggle the smart-filter eye to show it is non-destructive.

## Duplicate-check evidence

Checked **2026-10-07** against `storytold/photocraft` (114 open issues, 49 open PRs) and this tree.

| Query | Result |
|---|---|
| Code search `relight` / `Relight` in `storytold/photocraft` | **Zero hits.** |
| Open issues: lighting / relight / neural | None about photographic relight. #650 / #634 are 3D *model* import, out of scope. |
| Open PRs (keyword scan of 49 titles + `gh search prs relight`) | None. |
| Existing `filter.render.lightingEffects` | Live. Different kernel (bumped Blinn–Phong). Relight must not reuse that look; tests should fail if output merely embosses. |
| Camera Raw `dehaze` / colour grade | Live inside `lens_cmds`. Global grade, not a directional lamp. |
| #41 generative editing | Deferred. This kernel is Retinex + Lambertian, no model file. |

CLEAN for a focused PR: new `FilterParams` variant, one command, algo tests that prove directionality, dialog via existing schema, i18n label, snapshots.
