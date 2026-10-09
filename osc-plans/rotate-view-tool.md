---
generated_by: osc-newfeature
type: feat
repo: storytold/photocraft
bulk_run: true
feature_complexity: M
deep_discovery_report: osc-plans/result.json
dogfooded_general: false
title: Rotate View tool in the Hand flyout
stem: rotate-view-tool
---

# Rotate View tool in the Hand flyout

Planning only. Implement as one focused PR. Do not name other products in PR copy.

## Problem

The canvas camera today is pan + zoom + a left-right flip. `View` (`crates/ui-egui/src/state.rs`) stores `zoom`, `center`, `fit_pending`, and `doc_size`. `ViewXform` maps document pixels to the screen with zoom, centre, and `flip`; there is no rotation field.

That shows up in three unfinished places:

- Window › Arrange › Match Rotation is wired (`window.arrange.matchRotation` in `view_cmds.rs`) but always reports `"rotation": 0`, with the comment that views never rotate.
- Preferences › Enhanced Controls › Rotate View with Trackpad (`enhanced_controls.rotate_view_with_trackpad`) is listed among unread settings in `docs/scorecard.md`.
- The Hand slot on the toolbar is a single tool. There is no way to turn the view around a point so a stroke can follow the subject, then put the view back without rotating pixels.

Rotate View is view state, like Zoom and Hand. It does not rewrite the document. Agents already drive zoom through `ui.set` / `ui.inspect` `views[]`; rotation belongs on the same struct.

## User-facing behavior

- **Toolbar.** Add `Tool::RotateView` to the Hand slot: `&[Tool::Hand, Tool::RotateView]` in `TOOL_SECTIONS` (`panels.rs`). Icon: existing Lucide `compass` (`assets/icons/compass.svg`). Hand keeps **H**. Rotate View uses **R** (unused in `Tool::key` today). Shift+H still cycles the slot when "Use Shift Key for Tool Switch" is on.
- **Drag.** Press and drag around the view centre (the document point currently at the canvas centre). Clockwise drag increases the angle. The document, checker, pixel grid, selection outline, guides, and brush cursor all turn together. Pixels in the file do not change. History is not recorded.
- **Shift.** Constrain to 15° steps while dragging.
- **Reset.** Options bar: numeric Angle (degrees, −180…180, wrap) and a **Reset View** button (angle 0, keep zoom and centre). Escape, or a double-click on the canvas with this tool, also resets. `view.resetView` does the same.
- **Compass.** While the angle is not 0, draw a small compass in the canvas corner (north = document up). Clicking it resets.
- **Match Rotation.** `window.arrange.matchRotation` / `matchAll` copy `views[src].rotation` onto the other documents and floating windows. Stop hard-coding 0.
- **Trackpad.** When `enhanced_controls.rotate_view_with_trackpad` is true, a trackpad rotate gesture (egui rotation delta, if present; otherwise ignore cleanly) adds to the angle. Default stays false. Wiring this pref drops it from the unread-settings list.
- **No document.** The tool can be selected. Drags and commands report that no document is open and change nothing. Never a silent no-op that looks like success.
- **Flip.** Rotation composes with View › Flip Horizontal: map in `ViewXform` (rotate around centre, then optional flip).
- **GPU.** Today a flipped view takes the CPU blit because the GPU canvas shader has no mirroring (`canvas.rs` around the `!flip` guard). Rotation must stay on the GPU: painting while the view is turned is the reason for the tool. Add `rotation` (radians) to `gpu_canvas::ViewParams`, expand the view uniform (keep 16-byte alignment; `VIEW_UNIFORM_SIZE` / `VIEW_FLOATS` today are 112 / 28), and rotate document coordinates around `center` in the WGSL before sampling. CPU `ViewXform::{to_screen,to_doc}` must use the same convention so outlines and tools stay under the pixels.

## Engine / command API

View state lives in the shell (`AGENTS.md`: zoom, panels, screen mode). Do **not** add a document-mutating engine command.

| Id | Where | Params | Result |
|---|---|---|---|
| `view.rotateView` | `UI_COMMANDS` + `view_cmds.rs` | `{"angle": deg}?` absolute, or `{"delta": deg}?` add, optional `"reset": true`. Reject non-finite values. | `{"rotation": deg}` |
| `view.resetView` | same | `{}` | `{"rotation": 0, "zoom": …, "center": …}` — angle 0 only; do not change zoom/centre (those have Fit Screen / 100%). |

Also:

- `ui.set` / `ui.inspect`: `views[i].rotation` is an `f32` degrees, default 0, serde `#[serde(default)]`. `ui.set {"tool": "RotateView"}` must parse via `Tool::from_name`.
- `ui.menu.invoke` with `view.rotateView` / `view.resetView` / `window.arrange.matchRotation`.
- Selecting the tool is not a menu item (Hand is not either). Shortcut **R** is the tool key, rebindable like other tool keys.

`enabled`: `view.rotateView` / `view.resetView` need a document (`active_index`). With none, return `Err("no document open")`.

## Files to touch

Implementation:

- `crates/ui-egui/src/state.rs` — `Tool::RotateView` in the enum, `ALL`, `label`, `key` (`'R'`), `glyph`; `View.rotation: f32` with `#[serde(default)]`.
- `crates/ui-egui/src/panels.rs` — Hand flyout `&[Tool::Hand, Tool::RotateView]`; options bar (angle field, Reset View).
- `crates/ui-egui/src/icons.rs` — `Tool::RotateView => "compass"`.
- `crates/ui-egui/src/canvas.rs` — `ViewXform.rotation`; `to_screen` / `to_doc`; pass rotation into GPU `ViewParams`; drag handling next to the Hand pan block (new `rotate_view.rs` keeps `canvas.rs` small).
- `crates/ui-egui/src/rotate_view.rs` — **new**: drag, 15° constrain, reset, compass hit-test. Mirror `zoom_tool.rs`.
- `crates/ui-egui/src/lib.rs` — `pub mod rotate_view`.
- `crates/ui-egui/src/gpu_canvas.rs` — `ViewParams.rotation`; uniform pack; WGSL rotate around centre. Update `VIEW_FLOATS` / `VIEW_UNIFORM_SIZE`.
- `crates/ui-egui/src/view_cmds.rs` — handle `view.rotateView` / `view.resetView`; Match Rotation copies the angle; `handles()` / `is_enabled()`.
- `crates/ui-egui/src/menus.rs` — add the two ids to `UI_COMMANDS` (empty menu path, or View menu if a label is needed; Zoom lives there). Prefer View menu next to Flip Horizontal: `view.resetView` as "Reset View".
- `crates/ui-egui/src/wheel_nav.rs` — optional trackpad rotate when the pref is on; ignore non-finite deltas.
- `crates/ui-egui/src/control.rs` — `ui.inspect` already dumps `views`; confirm rotation is included. `ui.set` must reject NaN/Inf rotation the same way it rejects bad zoom (see #820: validate every field before applying the first).
- `crates/ui-egui/src/i18n/*.tsv` — every new `tl!` string in all locale catalogs (cs, de, es, fr, id, it, ja, ko, pl, pt-br, ru, zh-hans, zh-hant). Clean-room translations from the English meaning. Do not hand-merge: add rows, let tests enforce coverage (`every_tl_literal_is_translated`).
- `crates/engine/src/prefs.rs` — no schema change; the pref already exists. Only the UI starts reading it.
- `scorecard/tools.toml` — new done row, e.g. `TOOL-214-17` "Rotate View tool: drag to turn the view, Reset View, Match Rotation copies the angle".
- `docs/roadmap.md` — drop Rotate View from the missing-tools sentence (dated, measured).
- Regenerated `docs/scorecard.md` (`cargo xtask scorecard`) and `docs/parity.md` if a View menu id is new (`cargo xtask parity`).

Call sites that construct `ViewXform` by hand must pass `rotation` (tests and helpers today set `zoom/center/flip` only):

- `crates/ui-egui/src/canvas.rs` (`ViewXform::active`)
- `crates/ui-egui/src/preset_panels.rs`
- `crates/ui-egui/examples/layout_bench.rs`
- `crates/ui-egui/tests/layout_perf.rs`
- `crates/ui-egui/tests/drag_preview_canvas.rs`
- `crates/ui-egui/src/transform_undo_tests.rs`
- `crates/ui-egui/src/hold_keys/tests.rs`
- `crates/ui-egui/src/type_tool_tests.rs`
- `crates/ui-egui/src/polygon_lasso_tests.rs`
- `crates/ui-egui/src/magnetic_lasso_ui/tests.rs`
- `crates/ui-egui/src/marquee_tests.rs`

Add `rotation: 0.0` (or a helper `ViewXform { .. Default }` once Default includes it) so the struct update compiles.

Tests:

- `crates/ui-egui/src/rotate_view.rs` (unit tests in the module) and/or `crates/ui-egui/src/rotate_view_tests.rs`
- GPU uniform size test if `gpu_canvas.rs` already asserts `VIEW_UNIFORM_SIZE`

Do **not** edit `crates/engine/src/commands.rs` for this feature.

## Tests

`ViewXform` (no window):

- Identity: rotation 0 matches today's mapping for several zoom/centre/flip combinations.
- Inverse: `to_doc(to_screen(p))` is within 1e-3 px at 15°, 90°, −45°, and 180°.
- 90°: a point on +X in document space lands on +Y or −Y on screen (pick one convention and stick to it in GPU and CPU).
- Flip then rotate: order is documented and tested.

Commands / control channel:

- `ui.set {"tool":"RotateView"}` then `ui.inspect` reports that tool.
- `view.rotateView {"angle": 30}` sets 30; `{"delta": 15}` becomes 45; `{"reset": true}` returns 0.
- NaN / Inf / non-object params → `Err`, view unchanged.
- No document → `Err("no document open")`.
- `window.arrange.matchRotation` with two documents copies rotation; with one document the command stays disabled.
- Shift-constrained drag from a synthetic pointer path lands on a multiple of 15°.
- Double-click / Escape / Reset View zero the angle and do not add History entries (`history.entries` length unchanged).
- A 30° view + a `select.rect` via the engine still uses document coordinates (the selection is axis-aligned in the file; the outline is drawn rotated).

GPU:

- Uniform buffer size matches `VIEW_UNIFORM_SIZE` after the extra floats.
- With rotation 0 the GPU path is still used (`!flip` remains the only CPU fallback for mirroring, unless you also add flip to the shader — out of scope).
- With rotation 45° `on_gpu` stays true (assert via the existing perf/gpu flag or a small harness).

Prefs:

- With `rotateViewWithTrackpad: false`, a rotation-delta event does not change the angle.
- With true, a finite delta does.

Graceful failure (Rule 9): every `view.rotateView` path returns `Err`, never panics, for empty params, strings, huge angles (wrap, don't overflow), and no document.

## Verification commands

```sh
CARGO_TARGET_DIR=target/agent-rotate-view cargo test -p photocraft-ui-egui rotate_view -- --test-threads=1
CARGO_TARGET_DIR=target/agent-rotate-view cargo test -p photocraft-ui-egui --lib view_cmds
CARGO_TARGET_DIR=target/agent-rotate-view cargo clippy -p photocraft-ui-egui --all-targets -- -D warnings
CARGO_TARGET_DIR=target/agent-rotate-view cargo xtask layers
CARGO_TARGET_DIR=target/agent-rotate-view cargo xtask wasm
CARGO_TARGET_DIR=target/agent-rotate-view cargo xtask parity
CARGO_TARGET_DIR=target/agent-rotate-view cargo xtask scorecard
```

Rotate View is a camera transform, not a pixel kernel. Still time a 24 MP (6016×4000) GPU canvas frame with rotation 0 vs 37° (release, one document, no filters) and paste both numbers in the PR, next to the unrotated baseline. If the rotated frame is worse than ~1.5× the unrotated frame on the same machine, stop and check the shader (likely sampling the whole pasteboard). Offscreen: `cargo run -p photocraft-ui-egui --example snapshot` with the tool selected, before/after PNGs of the Hand flyout and a rotated checkerboard.

`panic_hunt` does not need a new case unless an engine command is added (it should not be).

## 30–45s demo storyboard

Generated document: 1600×1000, two wide colour bars (public-domain geometry, no photos).

| t | Shot |
|---|---|
| 0–4s | Toolbar: Hand slot, single button. Open the flyout: only Hand. |
| 4–8s | After: flyout shows Hand and Rotate View (compass). Select Rotate View. Options bar: Angle 0°, Reset View. |
| 8–18s | Drag clockwise. Bars turn; status/angle field tracks degrees. Shift: snap to 15°. |
| 18–26s | Paint a short brush stroke while the view is at 45°. Stroke follows the pointer on screen; undo/History shows a normal brush step, not a rotate. |
| 26–32s | Click Reset View (or the compass). Bars sit upright; the stroke stays where it was in the file. |
| 32–40s | Two documents. Rotate A to 30°. Window › Arrange › Match Rotation; B turns to 30°. |
| 40–45s | Hold on the flyout + a 30° canvas. End card: "Rotate View — turns the view, not the pixels." |

Screenshots for the PR: Hand flyout before (Hand only) / after (Hand + Rotate View); canvas at 0° / 45°; options bar. Demo images generated, not third-party art.

## Duplicate-check evidence

Checked 2026-10-09 against `storytold/photocraft` (fetched `upstream/main` = `a7f6c2b`).

| Query | Result |
|---|---|
| Open PRs whose title/body is Rotate View the tool | None. Hits were image-rotation previews: #1474, #1459 (arbitrary Image Rotation), not the view camera. |
| Merged since 2026-10-02 | #1363 Hand tool Fit Screen / 100% / Fill Screen — same flyout neighbour; rebase onto it, do not restyle the Hand options bar. #1480 Zoom Clicked Point to Center. |
| Issues | coygeek's `feat(ui)` series covers Color Sampler (#1046), Perspective Crop (#1043), Type Mask (#1055), Freeform/Curvature Pen (#1054), Anchor Point tools (#1044), Color Replacement (#1045). **No Rotate View issue.** |
| Code | `Tool` enum has Hand and Zoom only in that section. `View` has no rotation. `view_cmds.rs`: "Views never rotate in Photocraft, so Match Rotation has nothing to align." Pref `rotate_view_with_trackpad` unread. |
| Our earlier PRs | #734, #735/#895, #922, #1084 — none are this tool. |

Re-run `gh pr list -R storytold/photocraft --search 'Rotate View' --state open` immediately before opening the implementation PR; the queue moves fast.

## Implementation notes (reviewer expectations)

- Never silently no-op: bad params and "no document" are errors.
- No `unwrap` / `expect` / `panic!` / `unsafe` in non-test code.
- Tile halos do not apply (no pixel kernel). GPU sampling must still use the rotated mapping only; do not read off-document tiles as document pixels.
- i18n catalogs conflict often: add rows, do not rewrite the files.
- `docs/scorecard.md` / `docs/roadmap.md` / locale TSVs: regenerate or append; rebase onto `upstream/main` right before push. This fork's `origin/main` was **211 commits behind** `upstream/main` at discovery time.
