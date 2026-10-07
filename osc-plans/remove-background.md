---
generated_by: osc-newfeature
type: feat
repo: storytold/photocraft
bulk_run: true
feature_complexity: S
dogfooded_general: false
stem: remove-background
---

# Remove Background: one-click subject cutout

## Problem

The Properties panel already lists **Remove Background** as a Quick Action on pixel layers (`crates/ui-egui/src/props_layout.rs`), and the locale catalogs already translate that label. There is no engine command behind it. `visible_quick_actions` filters the button out because `menus::is_live("layer.removeBackground")` is false. The test in `props_layout_tests.rs` calls this out as the one listed id that is not a command.

Isolating a subject today is several commands and several history states: `select.subject`, then `select.refineEdge` (output `layerMask` or `newLayerWithMask`), then optionally a fill underneath. Agents and the CLI cannot do it in one shot. Portrait, product, and listing photos all need a cutout that leaves the original pixels recoverable.

## User-facing behavior

- On a pixel layer, Properties › Quick Actions shows **Remove Background**. Clicking it runs the new command (no extra dialog).
- **Layer › Remove Background** is enabled for the same layers.
- One history step named "Remove Background". Undo restores pixels, mask, layer name, and locks.
- Default result: a layer mask that reveals the subject and hides the rest. Original RGB is untouched, so painting or disabling the mask brings the backdrop back.
- Optional `"output": "transparency"` bakes the mask into pixel alpha instead (and drops any mask).
- Optional `"fill": "#rrggbb"` inserts a solid Color Fill layer immediately **below** the subject in that same history step, so a new backdrop is already there. The subject stays the active layer.
- If the active layer is the locked Background (name `"Background"` + transparency and position locks), promote it first the same way the Eraser does (`name = "Layer 0"`, unlock transparency and position) so a mask is legal.
- If no subject is found, return `Err` and leave the document unchanged (no empty mask, no stray history step).
- Pixel-locked / all-locked layers, type / shape / smart / fill / adjustment / group layers, and "no document" disable the command (and `execute` returns `Err`).
- Respects `"layer": id` when given; otherwise the active layer.
- Long work runs through `jobs::edit_job` so the desktop app can cancel it; tests and the CLI still see a synchronous `execute`.

## Engine command API

```
id:       layer.removeBackground
label:    Remove Background
menu:     ["Layer"]
shortcut: none
params:   {"layer":id?,"sampleAllLayers":bool=true,"output":"mask|transparency"="mask","refine":bool=true,"radius":px=2,"fill":"#rrggbb"?}
enabled:  active (or `layer`) is a raster layer, not pixel-locked / all-locked
journal:  true
```

`run` (one `edit` / `edit_job` closure):

1. Resolve the target raster layer. Promote Background if needed.
2. Build a sampler: active-layer pixels, or the visible composite when `sampleAllLayers` is true (same split as `select.subject`).
3. `photocraft_algo::segment::subject::select_subject(sampler, canvas)` → `Option<Region>`. `None` → `Err("no subject found")`.
4. If `refine` (default true): `photocraft_algo::matting::refine_mask` with `RefineParams { radius, smart_radius: true, smooth: 10, feather: 0.5, contrast: 10, shift_edge: 0 }` (radius clamped 0..=64).
5. Convert the region to a gray mask surface via `matting::region_surface`.
6. `output == "mask"`: set `layer.mask = Some(LayerMask { surface, ..LayerMask::reveal_all() })`, clear the document selection.
7. `output == "transparency"`: multiply pixel alpha by mask coverage, delete any layer mask, clear selection.
8. If `fill` is a colour string: parse with the existing colour helper (invalid → `BadParams` **before** any edit). Insert a `LayerContent::Fill(Fill::Solid(...))` sibling **below** the subject (`Document` stores the top layer last: insert at the subject's sibling index). Do **not** change `active` away from the subject.
9. Result JSON: `{"layer":id,"output":"mask"|"transparency","fillLayer":id?,"bounds":[x,y,w,h]}`.

Do not call `select.subject` / `select.refineEdge` as nested `execute`s: that would record extra history steps. Call the algo functions directly.

## Files to touch

| File | Change |
|---|---|
| `crates/engine/src/cutout_cmds.rs` | **New** module: `specs()`, `run`, tests. Parallel-agent rule: new commands go in a new `*_cmds.rs`, not a shared file. |
| `crates/engine/src/lib.rs` | `pub mod cutout_cmds;` |
| `crates/engine/src/commands.rs` | One line: `v.extend(crate::cutout_cmds::specs());` Re-read immediately before editing. |
| `crates/ui-egui/src/menus.rs` | `PLACE_AFTER`: `("layer.removeBackground", "layer.layerMask.fromTransparency")` so the item sits with the mask commands. |
| `crates/ui-egui/src/props_layout_tests.rs` | Drop the `\|\| *id == "layer.removeBackground"` exception; the id must be live. Add a harness test that a pixel layer shows a "Remove Background" Quick Action button. |
| `crates/ui-egui/src/i18n/{ja,zh-hans,zh-hant,es,ru,cs}.tsv` | Label **Remove Background** is already present. No new rows unless the menu label is worded differently from that string. `ja` / `zh-hant` / `es` / `ru` / `cs` require every command label with a non-empty menu path (`complete_menus`). |
| `docs/parity.md` | Regenerated by `cargo xtask parity` (catalog live-count should be unchanged; this command is extra, not a catalog id). |

No `menu_catalog.rs` row: that file is the fixed menu tree. Extra engine commands with a `menu` path are merged in `menus.rs`.

Do **not** add `unsafe`, `unwrap`/`expect`/`panic` in non-test code. Validate `output`, colour, `radius`, and `layer` before indexing. Cap working-resolution work by using the existing subject / matting helpers (they already budget pixels).

## Tests

All in `cutout_cmds.rs` `#[cfg(test)]` (engine tests may `unwrap`). Reuse the disc-on-contrasting-background construction from `smartselect_cmds.rs` tests.

- **Happy path, 8/16/32-bit:** disc IoU of the mask vs the true disc ≥ ~0.75; a background corner pixel has mask coverage < 0.2; a disc-centre pixel ≥ 0.8. Composite of a corner is transparent (mask) or alpha ≈ 0 (transparency output).
- **Undo / redo:** mask gone after undo, restored after redo; fill layer gone after undo when `fill` was used.
- **Background promotion:** a new document's Background can run the command; afterwards the layer is not Background (name ≠ `"Background"` or locks cleared) and has a mask.
- **Fill:** `"fill":"#00aa44"` creates exactly one new fill layer below the subject; subject remains active; fill pixels are that colour where the mask is 0.
- **No subject:** a flat 32×32 gray field → `Err`, revision unchanged, no mask.
- **Disabled / Err:** no document; type layer; shape layer; `locks.pixels`; unknown `"output"`; `"fill":"nope"`; `"radius": -1` or non-finite; `"layer": 999999`.
- **sampleAllLayers:** subject on a hidden lower layer is ignored when `sampleAllLayers` is false and a different raster layer is active.
- **Idempotent enough:** running twice on the same cutout does not panic and still leaves one mask.
- **`props_layout_tests`:** pixel Quick Actions contain `layer.removeBackground` and `is_live` is true.

`panic_hunt` picks up the new id automatically; keep it green (bad params must `Err`, never panic).

## Verification commands

```sh
CARGO_TARGET_DIR=target/agent-remove-bg cargo test -p photocraft-engine --lib cutout_cmds
CARGO_TARGET_DIR=target/agent-remove-bg cargo test -p photocraft-ui-egui --lib props_layout_tests
CARGO_TARGET_DIR=target/agent-remove-bg cargo clippy -p photocraft-engine -p photocraft-ui-egui --all-targets -- -D warnings
CARGO_TARGET_DIR=target/agent-remove-bg cargo xtask layers
CARGO_TARGET_DIR=target/agent-remove-bg cargo xtask wasm
CARGO_TARGET_DIR=target/agent-remove-bg cargo xtask parity
CARGO_TARGET_DIR=target/agent-remove-bg cargo test -p photocraft-engine --test panic_hunt -- --ignored
```

Visual (public-domain still life or a synthetic disc PNG, never a personal photo):

```sh
# Before: Properties on a pixel layer — no Remove Background button (or document the current stub).
cargo run --release -p photocraft-ui-egui --example snapshot -- \
  --out /tmp/remove-bg-before.png --size 1440x900 --open <public-domain.jpg> \
  --script '[["ui.set", {"dock": {"collapsed": ["color","navigator","history"]}}]]'

# After: same UI with the Quick Action visible; then execute and capture the cutout.
cargo run --release -p photocraft-ui-egui --example snapshot -- \
  --out /tmp/remove-bg-after.png --size 1440x900 --open <public-domain.jpg> \
  --script '[["engine.execute", {"command":"layer.removeBackground","params":{}}]]'
```

Attach both PNGs to the PR. User-Agent when fetching demo art: `Photocraft-dev`.

## Demo storyboard (30–45 s)

1. **0–5 s.** Open a product photo on a plain backdrop (public-domain bottle, fruit, or toy). Checkerboard is not visible; Properties shows the pixel layer.
2. **5–12 s.** Point at Properties › Quick Actions › **Remove Background**. Click. Status may show a short job. The backdrop becomes checkerboard; the subject stays.
3. **12–20 s.** Toggle the layer-mask thumbnail off and on in Layers to show pixels were not destroyed.
4. **20–30 s.** Repeat on a copy with `fill` `#1a1a1a` (or click again after undo, this time via Layer › Remove Background plus a documented fill param from the CLI/control channel). Subject on a new solid ground.
5. **30–40 s.** Undo once: original photo. File › Export a PNG of the masked layer if time remains, showing transparency.
6. **40–45 s.** Optional: run the same `layer.removeBackground` from the CLI on the file to show agents share the command.

## Duplicate-check evidence

Checked **2026-10-07** against `storytold/photocraft` (114 open issues, 49 open PRs) and this tree.

| Query | Result |
|---|---|
| Code `layer.removeBackground` in `storytold/photocraft` | Only `props_layout.rs` (button id) and `props_layout_tests.rs` ("not implemented yet"). **No command.** |
| Open issues whose titles mention background / cutout / matte / subject removal | None. Closest: #653 (keep RGB of fully transparent pixels on export — complementary, not this feature). |
| Open PRs (titles + `gh search prs "remove background"`) | None implementing the command. #192 (merged 2026-10-05) added the Quick Action **stub**. |
| Closed issues/PRs `remove background` | Hits #192 / eraser-on-Background bugs (#15, #76), not a cutout command. |
| `select.subject` / `select.refineEdge` | Already live. This PR **composes** them into one command; it does not replace them. |
| Scorecard TOOL-214-1 Select and Mask | Partial workspace UI. Different surface. |
| #41 generative editing | Deferred learned/generative tools. This path is the existing classical saliency + GrabCut + guided-filter refine. |

CLEAN for a focused PR: implement the already-named command id, wire the dead button, tests + snapshots.
