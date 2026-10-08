---
generated_by: osc-newfeature
bulk_run: true
type: feat
repo: storytold/photocraft
stem: pattern-stamp-tool
feature_complexity: M
dogfooded_general: false
---

# Pattern Stamp tool: paint with the current pattern

## Problem

The S-group flyout is only Clone Stamp. There is no Pattern Stamp: no `Tool` variant, no `paint.patternStamp`, and no stroke that samples the pattern library. Fill, pattern fill layers, and brush textures already tile a pattern; painting that same tile with a brush (Aligned / Impressionist) is missing. Texture, fabric, and repeating-detail work currently means Fill or a fill layer, not a brush.

This is a stroke tool. It must not silently skip large documents.

## User-facing behavior

- New toolbar tool **Pattern Stamp Tool** in the S flyout next to Clone Stamp. Shortcut **S** cycles between Clone Stamp and Pattern Stamp.
- Paints with the **current pattern** from the Patterns panel (the same selection Paint Bucket uses when “Pattern” is on). Options bar also exposes a pattern chip that opens the existing picker.
- Brush: size, hardness, opacity, flow, blend mode, and the session brush — same as Clone Stamp.
- Options bar:
  - **Aligned** (default on). On: pattern phase is locked in document space across strokes. Off: each stroke restarts phase at its first point (the click is the tile origin for that stroke).
  - **Impressionist** (default off). On: each dab jitters the pattern phase with a seeded RNG so the stamp looks broken-up, not a perfect wrap.
  - **Pattern** chip + scale % (default 100), angle ° (default 0).
- One undo step per stroke (`Pattern Stamp`), coalesced while the pointer is down (same as Clone Stamp).
- Paint target: layer pixels, layer mask, alpha, Quick Mask. Locked pixels → error. No pattern selected / empty tile → error (`no pattern selected`), never a transparent no-op.
- 8 / 16 / 32-bit, any colour mode (pattern sampled to straight RGBA, written back with `from_rgba`).
- Agents: `paint.patternStamp` with `points` and `pattern` id or name.

PR title (plain feature): **Add the Pattern Stamp tool (S-group)**

## Engine command API

`paint.patternStamp` — empty menu path.

```
{"points":[[x,y,pressure?],…]
,"size":1..MAX_BRUSH_SIZE px=session
,"hardness":0..100
,"opacity":1..100=100
,"flow":1..100=100
,"spacing":1..1000 (% of size)=25
,"pattern":id|name?   // default: session / Patterns panel current
,"scale":1..1000 %=100
,"angle":deg=0
,"aligned":bool=true
,"impressionist":bool=false
,"mode":"normal|multiply|…"="normal"
,"phase":[px,py]?     // document-space origin; aligned strokes pass the previous origin
,"layer":id?=active
,"target":"pixels"|"mask"|"quickMask"|{"channel":i}
} → {"damage":[x,y,w,h], "phase":[px,py], "aligned":bool, "pattern":id}
```

- `enabled`: paintable + a resolvable pattern (library or document). No pattern → disabled with `no pattern selected`.
- Reuse `parse_brush` / `validate_brush_size` / `run_stroke` / `stroke_coverage` / `apply_coverage` from `retouch_cmds.rs` (the Clone Stamp path).
- Paint image: `photocraft_compose::pattern::{Tile, Placement, render}` over the coverage bounds — the same tiling Fill already uses (`fill_cmds.rs` pattern contents). Convert that RGBA buffer into a native-channel `Region` and composite through coverage at the stroke opacity/mode.
- **Aligned:** `phase` is document-constant. The first point of the first stroke may set it; later strokes pass the returned `phase`. **Unaligned:** `phase` = first point of *this* stroke; result `phase` is ignored by the UI for the next stroke.
- **Impressionist:** `apply_dab_stroke` instead of one coverage map. Each dab’s `Placement` phase += seeded jitter (`photocraft_paint::rng`, seed from points so replay is identical). No extra blur. Halo = dab footprint only (0 extra px) unless you add a blur — if you do, halo must be `ceil(σ × 3) + 1`.
- Copy the pattern into `doc.patterns` in the same history step (`pattern_cmds::ensure_in_doc`), so the document still renders after the library changes.

Reject, do not skip:

- missing/empty `points`, non-finite coords, coords outside `±1e6` → `BadParams` (reuse `brush_cmds::parse_points` if Pattern Stamp switches to it; today’s `parse_brush` should grow the same cap).
- unknown `pattern` id/name → `BadParams("no pattern …")`
- empty tile (`width` or `height` 0) → `BadParams`
- coverage rect `width*height` > 16_777_216 (4096²) for a single stroke → `BadParams("stroke too large …")` rather than allocating or returning Ok with no paint
- locked / no document / not paintable → existing errors

Do **not** add a silent megapixel skip. A 24 MP document with a 200 px brush is in-bounds; only the stroke footprint is painted.

### Tile seams

`Placement` is in document space, so a stamp that crosses a 64 px tile boundary must match an untilled reference. Test that. If Impressionist or scale uses a kernel with reach, set `Halo::Radius` to that reach (same rule as the Relight review: halo must cover blur reach). Pattern bilinear wrap already lives in `Tile::sample`; do not sample with a halo of 0 if you blur.

## Files to touch

Implementation:

- `crates/engine/src/retouch_cmds.rs` — `pattern_stamp` run + `CommandSpec` next to `paint.cloneStamp`. Prefer `crates/engine/src/retouch_cmds/pattern_stamp.rs` if the parent is already large.
- `crates/engine/src/retouch_cmds/tests.rs` — command tests.
- `crates/engine/src/pattern_cmds.rs` — reuse `resolve_param`, `placement`, `ensure_in_doc` (export `ensure_in_doc` as `pub(crate)` if it isn’t).
- `crates/compose/src/pattern.rs` — no change if `Tile::new` / `Placement::new` / `render` stay public; only touch if you need a native-format write helper.
- `crates/algo/examples/bench_pattern_stamp.rs` **or** `crates/engine/examples/bench_pattern_stamp.rs` — 24 MP stroke timing.

UI:

- `crates/ui-egui/src/state.rs` — `Tool::PatternStamp`; `ALL` 48 → 49; `key()` shares `'S'` with Clone Stamp; `is_brushlike` = true; `from_name`.
- `crates/ui-egui/src/state.rs` `ToolOptions` — `pattern_stamp_aligned: bool` (true), `pattern_stamp_impressionist: bool` (false), `pattern_stamp_scale: f32` (100), `pattern_stamp_angle: f32` (0). Reuse `clone_aligned` only if you are sure the two tools should share it; separate fields are safer.
- `crates/ui-egui/src/panels.rs` — S flyout `&[Tool::CloneStamp, Tool::PatternStamp]`; options bar Aligned, Impressionist, pattern chip, scale.
- `crates/ui-egui/src/icons.rs` — `Tool::PatternStamp => "grid-3x3"` (or another icon already in `assets/icons`).
- `crates/ui-egui/src/retouch_ui.rs` — `finish_stroke` arm for `Tool::PatternStamp` → `paint.patternStamp`; keep returned `phase` on `UiState` for aligned strokes (`pattern_stamp_phase: Option<[f32; 2]>`).
- `crates/ui-egui/src/canvas.rs` — brush cursor (already `is_brushlike`).
- `crates/ui-egui/src/preset_panels.rs` — if the pattern chip should highlight the current pattern (read-only hook; do not rebuild the Patterns panel).
- `crates/ui-egui/src/i18n/*.tsv` — `Pattern Stamp Tool` / command label `Pattern Stamp`, every registered language.

Scorecard / log (implementation PR):

- `scorecard/tools.toml` — new `TOOL-213-18` Pattern Stamp, `status = "done"`.
- `docs/scorecard.md` via `cargo xtask scorecard`.
- `log/devlog.md` — 24 MP stroke time.

No menu catalog row. Parity stays 626/626.

Do **not** add notice/canary files.

## Tests

Algo / compose (if you add a helper; otherwise engine-level):

- Integer 100% scale, phase (0,0): a 16×16 checkerboard stamped with a hard 16 px brush reproduces the tile exactly in the dab.
- Scale 50% and 200%: no gaps; bilinear wrap at tile edges.
- Aligned two strokes: phase continuous (second stroke continues the wrap).
- Unaligned two strokes: each stroke’s first point is a new origin.

Engine:

- `file.new` + `paint.patternStamp` with `"pattern": "Checkerboard"` (built-in) and a short stroke → pixels change, undo/redo.
- 8 / 16 / 32-bit RGB; one Gray document.
- Missing pattern / empty points / size above `MAX_BRUSH_SIZE` / 4096²+ coverage → `Err`.
- Locked layer → `Err`.
- Impressionist on vs off: seeded replay matches; two runs with the same points match.
- **Tile seam:** 96×96 layer (tile 64), hard brush centred on x=64, pattern phase 0; stamped column equals a single-buffer reference (max abs error 0 at 100% integer scale).
- `panic_hunt` green.

UI:

- `from_name("patternStamp")`; S cycles Clone Stamp ↔ Pattern Stamp (`shortcuts.rs` already groups by `key()`).
- Flyout has two rows; options bar shows Aligned + Impressionist.
- Control-channel stroke changes pixels. Snapshot: generated checkerboard, not a personal photo.

## Verification commands

```sh
CARGO_TARGET_DIR=target/agent-pattern-stamp cargo test -p photocraft-engine pattern_stamp -- --nocapture
CARGO_TARGET_DIR=target/agent-pattern-stamp cargo test -p photocraft-ui-egui --lib pattern_stamp -- --nocapture
CARGO_TARGET_DIR=target/agent-pattern-stamp cargo clippy -p photocraft-engine -p photocraft-ui-egui -p photocraft-compose --all-targets -- -D warnings
CARGO_TARGET_DIR=target/agent-pattern-stamp cargo xtask layers
CARGO_TARGET_DIR=target/agent-pattern-stamp cargo xtask wasm
CARGO_TARGET_DIR=target/agent-pattern-stamp cargo test -p photocraft-engine --test panic_hunt -- --ignored
```

24 MP timing (heavy paint kernel):

```sh
CARGO_TARGET_DIR=target/agent-pattern-stamp cargo run --release -p photocraft-engine --example bench_pattern_stamp
```

Build a 6016×4000 RGBA8 document, resolve built-in `Checkerboard`, run `paint.patternStamp` with a 200 px, hardness 100 stroke along y=2000 from x=100 to x=5900, aligned, impressionist off. Print elapsed ms in release. Record it in the PR and `log/devlog.md`. A second run with impressionist on is optional. Do not skip the document for being “too big”; the command must paint the footprint or return `BadParams` if the *stroke* coverage exceeds the 4096² cap (a 200 px brush does not).

## Demo storyboard (30–45 s)

Demo image: a generated 1024×768 gray field (solid 0.75) plus the built-in Bricks or Checkerboard pattern. No personal photos.

1. (0–6 s) New file. Window › Patterns. Select **Bricks**.
2. (6–12 s) S-flyout → Pattern Stamp. Size ~80, Aligned on, Impressionist off. Stamp a short stroke; bricks wrap in document space.
3. (12–22 s) Turn Aligned off. New stroke: bricks restart at the new click. Contrast with the first stroke.
4. (22–32 s) Impressionist on, one stroke: broken-up brick noise, still one history step.
5. (32–40 s) Undo through the three strokes; redo the first.
6. (40–45 s) Switch to Clone Stamp with S, back to Pattern Stamp — flyout last-used is Pattern Stamp.

## Duplicate-check evidence

Checked 2026-10-08 against `storytold/photocraft`:

- **Open PRs:** none for Pattern Stamp / `paint.patternStamp` / Impressionist stamp. Clone Stamp already ships (`paint.cloneStamp`). Pattern *fill* (`layer.newFillLayer.pattern`, Edit › Fill contents `pattern`) is a different command.
- **Issues:** #213 does not list Pattern Stamp. No issue titled Pattern Stamp.
- **Recent merges:** Pencil (#279), Mixer Brush, Patch (#381), Content-Aware Move (#814), Magnetic Lasso (#864). None stamp a pattern with a brush.
- **Code:** `Tool::CloneStamp` is a singleton flyout in `panels.rs`. `rg` PatternStamp / `pattern.?stamp` / Impressionist-as-tool is empty. `photocraft_compose::pattern` and `pattern_cmds` are the reuse surface, not an existing stamp command.
- **Round 1:** #734 Remove Background, #735 Relight — unrelated.
- **Authored open PRs** on upstream: 0.

CLEAN: no open or last-week PR/issue owns this tool.
