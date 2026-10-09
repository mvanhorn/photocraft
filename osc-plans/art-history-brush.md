---
generated_by: osc-newfeature
type: feat
repo: storytold/photocraft
bulk_run: true
feature_complexity: M
deep_discovery_report: osc-plans/result.json
dogfooded_general: false
title: Art History Brush in the Y-group flyout
stem: art-history-brush
---

# Art History Brush in the Y-group flyout

Planning only. Implement as one focused PR. Do not name other products in PR copy.

## Problem

The Y-group today is a single tool: History Brush (`Tool::HistoryBrush`, `paint.historyBrush` in `crates/engine/src/retouch_cmds.rs`). That command copies pixels from a chosen history state through a normal brush stamp. There is no companion that lays down *stylized* dabs from the same source — oriented along the stroke, spaced by an Area control, with Tight / Loose / Dab / Curl styles and a Tolerance that skips pixels already close to the source.

The toolbar slot is `&[Tool::HistoryBrush]` (`panels.rs`). Shift+Y has nothing to cycle. The History Brush options bar already says it paints from the opening state; Art History Brush is the missing Y-group sibling, the same way Pattern Stamp sits next to Clone Stamp in the S-group (#922, still open — do not touch S-group files).

This is a standard retouching tool, not a new filter. Keep the kernel in `photocraft-algo` / `photocraft-paint` and expose one engine command.

## User-facing behavior

- **Toolbar.** Add `Tool::ArtHistoryBrush` to the Y slot: `&[Tool::HistoryBrush, Tool::ArtHistoryBrush]`. Shortcut **Y** shared (cycle with Shift+Y). Icon: existing Lucide `sparkles` (`assets/icons/sparkles.svg`); History Brush keeps `clock`.
- **Stroke.** Paint with the current brush (size, hardness, opacity, flow, spacing, blend mode, selection clip, transparency lock, mask/`target` like History Brush). Each dab samples a patch from a history state and stamps it with a style-dependent pose.
- **Options bar.**
  - **Style** (required): `tightShort`, `tightMedium`, `tightLong`, `looseMedium`, `looseLong`, `dab`, `tightCurl`, `looseCurl`. Default `tightMedium`.
  - **Area** 0–100, default 50. Larger Area → larger dab footprint and more spacing along the stroke.
  - **Tolerance** 0–100, default 50. Skip a dab when the mean absolute difference between the current pixels and the source in that footprint is below `tolerance/100` of the format range. Tolerance 0 stamps every dab; 100 stamps only where the layer has diverged a lot.
  - **History state**: same as History Brush (`state` index; default 0 = oldest held snapshot, usually Open). The History panel's "source" state is used when the UI has one; the command still takes an explicit index so CLI/MCP work.
- **Impressionist is not an option here** (that is Pattern Stamp). Do not invent extra modes.
- **Miss / empty.** A stroke whose coverage is empty, a style that is not in the list, a `state` that is not in `0..=past_len`, a pixel-locked layer, a type/shape/smart layer without rasterize, or no document → `Err` with a specific message. Never look like a successful no-op.
- **One history step** per mouse-up (coalesce the drag). Label: "Art History Brush".
- **Cursor.** Same brush cursor as History Brush (`is_brushlike` includes the new tool).

## Engine command API

New module `crates/engine/src/retouch_cmds/art_history.rs`, registered from `retouch_cmds::specs()` (do not grow the shared `retouch_cmds.rs` run closures; add one `CommandSpec` and `mod art_history`).

```text
id:       paint.artHistoryBrush
label:    Art History Brush
menu:     []          (toolbar tool; no menu item)
shortcut: None        (Y is the tool key in the UI)
params:   {
            "points": [[x,y,pressure?], ...],
            "size": px,
            "hardness": 0..100=100,
            "opacity": 0..100=100,
            "flow": 0..100=100,
            "spacing": 0..1000?,
            "mode": blend?,
            "layer": id?,
            "target": "pixels"|"mask"|channel?,
            "state": index=0,
            "style": "tightShort|tightMedium|tightLong|looseMedium|looseLong|dab|tightCurl|looseCurl"="tightMedium",
            "area": 0..100=50,
            "tolerance": 0..100=50
          }
enabled:  has_pixel_layer (same as paint.historyBrush: pixel layer, or mask/channel target)
run:      art_history
journal:  true
```

Validation (all `EngineError::BadParams` or the lock error History Brush already uses):

| Condition | Error (do not stamp) |
|---|---|
| params not an object | expected an object |
| `points` missing/empty | no stroke points |
| more than 4096 points | too many points |
| non-finite x/y/size/area/tolerance | invalid number |
| size ≤ 0 or coverage rect > 4096² | stroke too large |
| unknown `style` | unknown Art History style |
| `state` out of `0..=past_len` | no history state N (same wording as History Brush) |
| source layer/mask/channel missing in that state | the target did not exist in that history state |
| pixel lock / type layer | existing lock / kind messages |
| no document | no document |

Kernel (`crates/algo/src/art_history.rs` or `crates/paint/src/art_history.rs` — prefer **algo** for the pose/sample math, **paint** only if you reuse `apply_dab_stroke` in place):

Clean-room, from described behaviour (oriented dabs from a snapshot, Area, Tolerance, named styles). Do not copy proprietary code.

1. Build the stroke polyline from `points` (existing `Stroke` / `stroke_coverage` helpers).
2. Walk arc length. Emit a dab centre every `spacing` px, where  
   `spacing = size * style.spacing_mul * (0.25 + 0.02 * area)`  
   Style multipliers (start here; tune with tests, keep deterministic):

   | style | spacing_mul | scatter_mul | curl_deg | size_mul |
   |---|---:|---:|---:|---:|
   | tightShort | 0.35 | 0.00 | 0 | 0.7 |
   | tightMedium | 0.55 | 0.00 | 0 | 0.9 |
   | tightLong | 0.85 | 0.05 | 0 | 1.1 |
   | looseMedium | 0.70 | 0.45 | 0 | 1.0 |
   | looseLong | 1.10 | 0.70 | 0 | 1.2 |
   | dab | 1.60 | 0.10 | 0 | 1.0 |
   | tightCurl | 0.50 | 0.05 | 18 | 0.9 |
   | looseCurl | 0.80 | 0.40 | 28 | 1.1 |

3. For dab *i* at centre *c* with stroke tangent *t*:  
   `theta = atan2(t.y, t.x) + i * curl + scatter`  
   Scatter is a **seeded** RNG (`0xA7B1_57E0 ^ state_index` plus dab index) so the same stroke is replayable. Scatter offset = `scatter_mul * size * (u, v)` with `u,v ∈ [-1,1]`.
4. Read a square of side `ceil(size * size_mul) + 2` from the **history source** surface, rotate by `theta` about *c* (bilinear, clamp-to-edge; transparent outside the source bounds), and stamp through the current brush coverage.
5. **Tolerance:** mean of `|src − dst|` over the dab's covered pixels, divided by the format's peak (1.0 in working f32). If `mean < tolerance/100`, skip the dab.
6. **Halo:** every destination tile must read `halo = ceil(size * size_mul / 2 + scatter_mul * size + 2)` extra pixels from the source. Assert in tests that a dab whose centre is just inside a tile still matches an untiled reference at the seam (same pattern as Red Eye's 5 px halo covering erode+blur). If the halo is too small, **fail the test**; do not ship a seam.
7. Work in native channels at 8 / 16 / 32f. No `u8`-only path. Colour conversion, if any, goes through `photocraft-cms` — this kernel should not convert; it copies the source format (convert the history surface once if the current layer's format differs, same as History Brush).
8. Parallelise per destination tile with rayon; skip empty tiles. Selection coverage and transparency lock use the existing `apply_coverage` / `apply_dab_stroke` path.

Sequential dabs (`apply_dab_stroke`) are the right shape: dab *n* may overlap dab *n−1*.

## Files to touch

Keep this PR off Pattern Stamp / Clone Stamp / live-stroke PRs.

Implementation:

- `crates/algo/src/art_history.rs` — **new** kernel + halo constant + style table. Export from `crates/algo/src/lib.rs`.
- `crates/algo/src/art_history.rs` tests (or `crates/algo/src/art_history/tests.rs`) — seam, skip-on-tolerance, style spacing, seeded scatter.
- `crates/engine/src/retouch_cmds/art_history.rs` — **new** `run` + param parse.
- `crates/engine/src/retouch_cmds.rs` — `mod art_history;` one `CommandSpec` in `specs()`. Do not rewrite History Brush.
- `crates/engine/src/channel_cmds.rs` — if the paint-command id list must include `paint.artHistoryBrush` (History Brush is already listed ~line 554).
- `crates/engine/src/lib.rs` — only if a new top-level module is used (prefer a submodule of `retouch_cmds`, so **no** `commands.rs` change beyond the existing `v.extend(retouch_cmds::specs())`).
- `crates/engine/examples/bench_art_history.rs` — 6016×4000 stroke timing (copy `bench_pattern_stamp` / `bench_redeye` shape).
- `crates/ui-egui/src/state.rs` — `Tool::ArtHistoryBrush` in enum, `ALL`, `label`, `is_brushlike`, `key` (`'Y'` with HistoryBrush).
- `crates/ui-egui/src/panels.rs` — Y flyout `&[Tool::HistoryBrush, Tool::ArtHistoryBrush]`.
- `crates/ui-egui/src/icons.rs` — `sparkles`.
- `crates/ui-egui/src/retouch_ui.rs` — command id `paint.artHistoryBrush`, options bar (style combo, Area, Tolerance). Keep History Brush's hint intact.
- `crates/ui-egui/src/canvas.rs` — include the tool in the brushlike / freehand dispatch match (same arms as `Tool::HistoryBrush`).
- `crates/ui-egui/src/i18n/*.tsv` — new strings in every catalog.
- `scorecard/tools.toml` — `TOOL-213-18` or next free id: "Art History Brush: Y-group, styles, Area, Tolerance, paints from a history state". (Pattern Stamp's open PR claimed TOOL-213-18; **check the file on `upstream/main` at implement time** and take the next free TOOL-213-*.)
- `docs/roadmap.md` — remove Art History Brush from the missing-tools list.
- Regenerated `docs/scorecard.md`.

Tests:

- `crates/engine/src/retouch_cmds/art_history.rs` (cfg tests) or `crates/engine/src/retouch_cmds/art_history_tests.rs`

Avoid:

- `crates/engine/src/retouch_cmds/pattern_stamp.rs` and S-group UI (#922 open).
- Live-stroke rewrites in `canvas.rs` beyond adding the enum arm (#1040 open for History Brush live strokes). Adding `Tool::ArtHistoryBrush` to the same `is_brushlike` / HistoryBrush match is enough; do not refactor live-stroke plumbing.

## Tests

Algo:

- Halo ≥ read reach: stamp a dab whose centre is 1 px inside a 64-tile; tiled output equals a single-rect reference (max channel delta 0 at 8-bit, 1/32767 at 16-bit, 1e-5 at 32f).
- Tight Short spacing < Tight Long spacing for the same `size`/`area` (count dabs on a 100 px horizontal stroke).
- Tight Curl: successive dab angles increase; Loose Curl increases faster and has non-zero scatter vs Tight Curl with the same seed.
- Tolerance 100 on an unchanged layer: zero pixels written (source == dest).
- Tolerance 0: pixels change where the history state differs.
- Empty buffer / mismatched src-dst size: kernel returns without writing (and the command maps that to `Err`, not a success).
- Seeded scatter: two runs with the same points/style/state match bit-for-bit.

Engine:

- 8 / 16 / 32-bit RGB and 16-bit gray: a stroke from a filled Open state onto a painted layer restores source colour inside the dab, undo/redo.
- `state: 1` uses that snapshot, not 0.
- Bad params: `{}`, `{"points":[]}`, `{"points":[[0,0]],"style":"nope"}`, `{"points":[[0,0]],"state":99}`, `{"points":[[0,0]],"area":-1}`, NaN size, 10_000 points, 8000 px size → `Err`.
- Pixel-locked layer → `Err` (existing message).
- Type layer → `Err` (or the rasterize prompt path already used by brush tools — if you hit that UI, the command itself still errors when called headless).
- One mouse-up = one history entry even with many dabs.
- Grayscale and CMYK: no panic; if a path cannot run, `Err` with "unsupported", never a `u8` stub.

UI:

- Y-group order is History Brush then Art History Brush.
- Long-press on Y selects Art History Brush.
- A stroke with the tool executes `paint.artHistoryBrush` (control-channel / command journal).
- Options bar defaults: style tightMedium, area 50, tolerance 50.
- `Tool::from_name("Art History Brush")` / `"artHistoryBrush"`.
- `every_tl_literal_is_translated` stays green.

`panic_hunt`: new command is picked up automatically. Run it.

## Verification commands

```sh
CARGO_TARGET_DIR=target/agent-art-history cargo test -p photocraft-algo art_history
CARGO_TARGET_DIR=target/agent-art-history cargo test -p photocraft-engine art_history
CARGO_TARGET_DIR=target/agent-art-history cargo test -p photocraft-ui-egui art_history
CARGO_TARGET_DIR=target/agent-art-history cargo clippy -p photocraft-algo -p photocraft-engine -p photocraft-ui-egui --all-targets -- -D warnings
CARGO_TARGET_DIR=target/agent-art-history cargo xtask layers
CARGO_TARGET_DIR=target/agent-art-history cargo xtask wasm
CARGO_TARGET_DIR=target/agent-art-history cargo xtask parity
CARGO_TARGET_DIR=target/agent-art-history cargo test -p photocraft-engine --test panic_hunt -- --ignored
```

**24 MP timings (required, AGENTS rule 8).** Release:

```sh
CARGO_TARGET_DIR=target/agent-art-history cargo run -p photocraft-engine --release --example bench_art_history
```

Document: 6016×4000, RGB 8-bit. Stroke: 200 px, size 40, area 50, tolerance 0, style `tightMedium`, then `looseCurl`. Paste both milliseconds in the PR (Pattern Stamp's PR used 96 ms aligned / 202 ms Impressionist as the comparison shape — report raw numbers, do not cite that PR). If a style reads a halo that does not cover scatter+size, fix the halo before posting.

UI screenshots: `cargo run -p photocraft-ui-egui --example snapshot` — Y flyout before/after, options bar, a Tight Medium stroke on generated bars.

## 30–45s demo storyboard

Generated document: 1600×1000, a sharp black rectangle on a mid-grey field, then a Gaussian-blurred duplicate as the current pixels so the Open state is sharp. No photos.

| t | Shot |
|---|---|
| 0–5s | Y-group: only History Brush. Paint a History Brush stroke: sharp rectangle returns under a soft brush. Undo. |
| 5–10s | After: flyout is History Brush + Art History Brush. Select Art History. Options: Style Tight Medium, Area 50, Tolerance 50. |
| 10–20s | Stroke across the blur. Tight Medium dabs bring back sharp structure in short aligned stamps. |
| 20–28s | Switch Style to Loose Curl, same path. Dabs rotate and scatter; still from the Open state. |
| 28–34s | Raise Tolerance to 90; a second stroke on already-restored pixels skips (little change). Drop Tolerance to 0; a third stroke stamps again. |
| 34–40s | Undo twice: one step per stroke. History panel names "Art History Brush". |
| 40–45s | Hold on the Y flyout + Loose Curl result. End card: "Art History Brush — stylized dabs from a history state." |

## Duplicate-check evidence

Checked 2026-10-09 against `storytold/photocraft` (`upstream/main` = `a7f6c2b`).

| Query | Result |
|---|---|
| Open PRs "Art History" | **None.** |
| Merged since 2026-10-02 | History Brush live-stroke work is **open** as #1040 (Healing Brush / History Brush / dab tools), not Art History. Do not merge-conflict that plumbing. |
| Issues | No Art History Brush issue. coygeek `feat(ui)` list: Color Sampler #1046, Perspective Crop #1043, Type Mask #1055, Freeform/Curvature Pen #1054, Anchor tools #1044, Color Replacement #1045 — not this. Umbrella #213 names painting gaps; Art History is in the roadmap missing-tools sentence, not a dedicated issue. |
| Code on `upstream/main` | `paint.historyBrush` exists; no `paint.artHistoryBrush`. `Tool::HistoryBrush` is the whole Y slot. |
| Our PRs | #922 Pattern Stamp (S-group, open) and #1084 Red Eye (J-group, **merged** on upstream). Different flyouts. |

Re-run immediately before the implementation PR:

```sh
gh pr list -R storytold/photocraft --search 'Art History' --state open
gh issue list -R storytold/photocraft --search 'Art History Brush' --state open
```

## Implementation notes (reviewer expectations)

- Tile halo must cover the full read reach (rotated sample + scatter). Prove it with a seam test.
- Never silently no-op: empty stroke, unknown style, bad history index, lock → `Err`.
- Post 24 MP release timings for Tight Medium and Loose Curl.
- No `unwrap` / `expect` / `panic!` / `unsafe` outside tests.
- No `u8`-only public path; test 8/16/32f.
- i18n: append catalog rows; the 13 locale files conflict constantly.
- Rebase onto **current** `storytold/photocraft` `main` before push. Discovery found `mvanhorn/photocraft` `main` **211 commits behind** `upstream/main` (`6f39aba` vs `a7f6c2b`), including merged Red Eye (#1084) and Relight halo follow-up (#895).
