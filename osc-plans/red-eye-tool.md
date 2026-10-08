---
generated_by: osc-newfeature
bulk_run: true
type: feat
repo: storytold/photocraft
stem: red-eye-tool
feature_complexity: S
dogfooded_general: false
---

# Red Eye tool: click a pupil to remove the red reflection

## Problem

The J-group flyout currently has Spot Healing, Healing Brush, Patch, and Content-Aware Move. There is no Red Eye tool: no `Tool` variant, no engine command, and no algorithm. Portrait and event photos with flash still have to be fixed by hand (desaturate + burn, or a Hue/Saturation selection), which is slow and easy to overdo.

This is a click tool, not a full-image filter. Work stays in a window around the click.

## User-facing behavior

- New toolbar tool **Red Eye Tool** in the J flyout (with Spot Healing / Healing / Patch / Content-Aware Move). Shortcut **J** cycles onto it.
- Cursor: a circle whose diameter follows **Pupil Size**. Click on or near a red pupil.
- Options bar:
  - **Pupil Size** 1–100 (default 50). Scales the search window around the click. 50 ≈ 64 px diameter on a typical portrait; 100 ≈ 128 px.
  - **Darken Amount** 0–100 (default 50). How far the corrected pixels go toward a neutral, darker pupil.
- The click finds red-dominant pixels in that window (R clearly above G and B, not a specular highlight), replaces their red with a blend of G and B, then darkens by Darken Amount. Near-white catchlights stay put.
- One undo step per click (`Red Eye`). Locked pixels, no document, or a non-paintable target: the command returns an error; the menu/tool stays disabled.
- Works on the paint target (layer pixels, layer mask, alpha channel, Quick Mask), at 8 / 16 / 32-bit, RGB and Gray. CMYK/Lab: convert through the existing RGBA round-trip used by other retouch tools.
- Agents and the CLI run the same command with `{x, y}` (or `points`) — no UI required.
- Status bar on a miss: a clear error such as `no red-eye pixels near the click`, never a silent no-op.

PR title (plain feature): **Add the Red Eye tool (J-group)**

## Engine command API

`paint.redEye` — empty menu path (toolbar tool, like `paint.cloneStamp`).

```
{"x":px, "y":px} | {"points":[[x,y],…]}
,"pupilSize":1..100=50
,"darken":0..100=50
,"layer":id?=active
,"target":"pixels"|"mask"|"quickMask"|{"channel":i}
 → {"damage":[x,y,w,h], "pixels":n}
```

- `enabled`: paintable target (same predicate as other retouch commands).
- Each click is one history step. Several `points` in one call are still one step (batch for actions/agents).
- Reject, do not skip:
  - missing/non-finite `x,y` / empty `points` → `BadParams`
  - more than 64 points in one call → `BadParams`
  - pupil window that would allocate more than 512² px (hostile `pupilSize` on a huge coord) → `BadParams` with the cap in the message
  - locked layer / no document / not paintable → existing errors
  - zero red-eye pixels in the window → `Other("no red-eye pixels near the click")` so the user sees it
- Never scan the whole document. The read/write rect is the click ± `radius(pupilSize)` plus the edge halo below.

### Algorithm (`photocraft-algo`)

New module `crates/algo/src/redeye.rs` (or a section of `retouch.rs` if it stays under ~200 lines).

1. Window `W` = click ± `r`, `r = (4 + 1.20 * pupilSize).clamp(8, 128)` px.
2. Halo: a 1 px morphological shrink plus a small Gaussian (`σ = 0.8`) on the mask so the edge isn't a hard disc. **Halo radius = 1 + ceil(σ × 3) + 1 = 5 px.** Reads extend `W` inflated by that halo; if you change σ, recompute the halo so it covers the blur reach (do not hard-code a number smaller than the kernel).
3. Mask: pixels in `W` whose straight RGB satisfies `R > G + 0.15` and `R > B + 0.15` and luma `< 0.92` (drop catchlights) and chroma is not near-gray. Optional: keep the largest connected component that contains the click (or the nearest one within `r`).
4. Correct: `R' = mix(R, (G+B)/2, t)` then luma `*= (1 - 0.45 * darken/100)`, with `t` = mask × (darken/100). Hue of non-red iris stays. Alpha unchanged.
5. Depth-agnostic: operate in straight RGBA `f32`, write back through `from_rgba` / `to_rgba`.
6. Deterministic. No RNG. Tile-safe: result in a tile depends only on that tile plus the 5 px halo.

Public entry:

```text
redeye::apply(px: &mut [[f32; 4]], w, h, click: (i32, i32), origin: (i32, i32), pupil_size: f32, darken: f32) -> u32
```

`px` is the window including halo; `origin` is the document coord of `px[0]`. Returns how many pixels were changed.

## Files to touch

Implementation:

- `crates/algo/src/redeye.rs` — kernel + unit tests (synthetic red disc, catchlight kept, Gray/RGB, halo/seam).
- `crates/algo/src/lib.rs` — `pub mod redeye`.
- `crates/algo/examples/bench_redeye.rs` — 24 MP timing (see Verification).
- `crates/engine/src/retouch_cmds.rs` — `paint.redEye` spec + `run` (new `redeye` submodule if the file is already large: `crates/engine/src/retouch_cmds/redeye.rs`).
- `crates/engine/src/retouch_cmds/tests.rs` — command tests (or `redeye.rs` `#[cfg(test)]`).
- `crates/engine/src/commands.rs` — no change if `retouch_cmds::specs()` already feeds the registry.

UI:

- `crates/ui-egui/src/state.rs` — `Tool::RedEye`; `ALL` length 48 → 49; J-group `key()`; `is_brushlike` = false (click, not a stroke); `from_name`.
- `crates/ui-egui/src/panels.rs` — J flyout `&[Tool::SpotHealing, Tool::Healing, Tool::Patch, Tool::ContentAwareMove, Tool::RedEye]`; options bar Pupil Size + Darken Amount.
- `crates/ui-egui/src/state.rs` `ToolOptions` — `red_eye_pupil_size: f32` (50), `red_eye_darken: f32` (50).
- `crates/ui-egui/src/icons.rs` — `Tool::RedEye => "scan-eye"` (or another existing lucide name that is already in `assets/icons`; do not add a new file unless it is listed in `ATTRIBUTION.md`).
- `crates/ui-egui/src/canvas.rs` — click (not drag-stroke) → `paint.redEye`; circle cursor from pupil size.
- `crates/ui-egui/src/retouch_ui.rs` — only if click dispatch lives there; otherwise canvas is enough.
- `crates/ui-egui/src/i18n/*.tsv` — English source `Red Eye Tool` / command label `Red Eye` (engine labels are harvested). Add translations for every registered language, even if the first pass copies English; `cargo xtask i18n-coverage` must not drop.

Scorecard / log (implementation PR, not this planning commit):

- `scorecard/tools.toml` — new item, e.g. `TOOL-213-17` Red Eye tool, `status = "done"` once tests pass.
- `docs/scorecard.md` — regenerate with `cargo xtask scorecard`.
- `log/devlog.md` — terse entry with the 24 MP number.

No `menu_catalog.rs` row (toolbar-only, like Clone Stamp). `cargo xtask parity` should stay 626/626.

Do **not** add notice/canary files.

## Tests

Algo (`redeye.rs`):

- Synthetic 32×32: red disc (R=0.95, G=B=0.1) on a brown iris → after apply, disc R ≈ (G+B)/2 and luma dropped; iris RGB unchanged outside the mask.
- White catchlight in the disc centre (R=G=B=1) stays ≥ 0.9.
- Gray document path (via RGBA) does not panic; red-chroma mask is empty → 0 pixels (engine then errors).
- 8 / 16 / 32-bit surfaces through the engine command.
- **Tile seam:** 128×128 canvas, 64 px tiles, click on a tile boundary; the corrected disc has no 1 px jump (max neighbour delta across the boundary matches an un-tiled reference). Halo must be the blur reach, not a smaller constant.
- Hostile: `pupilSize` 0 / 101 clamped; NaN click rejected; window cap 512².

Engine:

- Happy path: file.new, paint a red disc, `paint.redEye` at its centre → `pixels > 0`, undo/redo restores.
- Disabled: no doc, locked pixels, type layer without a mask → `enabled` Err / `run` Err.
- Empty miss: click on a blue field → Err, not Ok with 0 pixels.
- Bad params: `{}`, `{x:null}`, 65 points, non-finite coords → `BadParams`.
- `panic_hunt` stays green (new command is in the registry).

UI:

- `Tool::from_name("redEye")` / `"Red Eye Tool"`.
- J flyout contains Red Eye; options bar shows the two sliders.
- Control-channel: `ui.set` tool `redEye`, `ui.pointer` click, document pixels change. Offscreen snapshot of the options bar (generated disc, not a personal photo).

## Verification commands

```sh
CARGO_TARGET_DIR=target/agent-red-eye cargo test -p photocraft-algo redeye -- --nocapture
CARGO_TARGET_DIR=target/agent-red-eye cargo test -p photocraft-engine retouch -- --nocapture
CARGO_TARGET_DIR=target/agent-red-eye cargo test -p photocraft-ui-egui --lib red_eye -- --nocapture
CARGO_TARGET_DIR=target/agent-red-eye cargo clippy -p photocraft-algo -p photocraft-engine -p photocraft-ui-egui --all-targets -- -D warnings
CARGO_TARGET_DIR=target/agent-red-eye cargo xtask layers
CARGO_TARGET_DIR=target/agent-red-eye cargo xtask wasm
CARGO_TARGET_DIR=target/agent-red-eye cargo test -p photocraft-engine --test panic_hunt -- --ignored
```

24 MP timing (windowed kernel — prove it does **not** scan the document):

```sh
CARGO_TARGET_DIR=target/agent-red-eye cargo run --release -p photocraft-algo --example bench_redeye
```

Example should build a 6016×4000 (~24.1 MP) RGBA8 surface, run `redeye::apply` at one interior click with pupilSize=50, and print elapsed ms plus the window size. Record the number in the PR and `log/devlog.md`. Fail the example (or the PR description) if the run reads more than the window+halo (a full-image scan is a bug).

Optional UI snapshot:

```sh
cargo run -p photocraft-ui-egui --example snapshot
```

## Demo storyboard (30–45 s)

Demo image: a **generated** 800×600 RGB portrait stand-in — two dark ellipses (eyes) with a bright red disc in each pupil and a white catchlight. No photographs of people.

1. (0–5 s) Open the generated file. Zoom 200% on the face.
2. (5–12 s) J-flyout → Red Eye Tool. Show Pupil Size 50, Darken 50. Circle cursor over the left pupil.
3. (12–20 s) Click left pupil. Red collapses; catchlight remains. One history step “Red Eye”.
4. (20–28 s) Click right pupil. Same.
5. (28–35 s) Undo twice, redo twice.
6. (35–45 s) Cut to the same file after a miss-click on the cheek: status error, pixels unchanged.

## Duplicate-check evidence

Checked 2026-10-08 against `storytold/photocraft` (open PRs, issues, merges since 2026-10-01):

- **Open PRs:** ~62. None titled or scoped as Red Eye / `paint.redEye` / pupil correction. Closest J-group work: Patch (merged #381), Content-Aware Move (merged #814), Magnetic Lasso (merged #864, L-group), Direct Selection (open #865, A-group). Hidden-layer paint refuse (open #872) is orthogonal.
- **Issues:** #213 lists Pencil/Mixer/Patch/CAM (those tools now exist). #794 asks for Remove + Patch (Patch landed; Remove is the content-aware Remove brush, not Red Eye). No Red Eye issue.
- **Recent merges:** #734 Remove Background and #735 Relight — different commands (`layer.removeBackground`, `filter.render.relight`). Do not extend Relight.
- **Code:** `rg` for `RedEye` / `red.?eye` / `pupil` in `crates/` is empty except this plan. `Tool` enum has no Red Eye. J flyout is four tools.
- **Authored open PRs** on upstream by this fork’s GitHub user: 0.

CLEAN: no open or last-week PR/issue owns this tool.
