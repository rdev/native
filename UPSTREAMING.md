# ProUI fork changes and upstreaming notes

This fork carries the Native SDK changes the ProUI port needs. Everything
generic is split into PR branches cut from upstream `main`; what stays here
is ProUI-specific policy that should not become an SDK default.

## Branches

| Branch | What it is |
| --- | --- |
| `proui-runtime` | What ProUI builds against today (v0.10.1 base, extraction history). |
| `proui-runtime-0.10.1` | Backup of `proui-runtime` before the rebase. |
| `proui-runtime-next` | The whole series rebased onto upstream `main` `96943d4a`, plus the fork-only changes below. This file lives here. |
| PR branches | Ten themed PRs cut from upstream `main` `96943d4a`; only GPU scenes stacks on another. Each keeps one commit per change so it reviews commit by commit. |

The PR branches contain no ProUI code: no kit controls, painters, tokens,
catalog data, or ProUI names. Where the fork encoded a ProUI need through a
sentinel or a sidecar field, the PR branch replaces it with a typed API (for
example `SliderRange` instead of slider metadata packed into unrelated
`Widget` fields).

## PR branches

| # | Branch | Needs | Commits | Contents |
| --- | --- | --- | --- | --- |
| 1 | `fix/canvas-layout-and-render` | | 5 | Wrapped paragraphs measure at their capped width; overlay surfaces reserve wrapped child height; `style.border` paints on layout containers; an open accordion around a closed nested one settles open (it stayed inert); render animations compose in view space instead of through a command's local transform. |
| 2 | `feat/canvas-typography-and-rounded-clips` | | 7 | Letter spacing (`text_tracking`) through measurement, layout, paint, and every backend; per-widget `text_size` and paragraph `line_height`; buttons measure at the authored size; rounded clips on every backend (planner, reference rasterizer, AppKit, Direct2D), including cover images and revealing accordions. One wire format bump, v6, covers both tracking and clip radii. |
| 3 | `feat/canvas-custom-control-seams` | | 8 | A widget can delegate its chrome to a vector painter while the SDK keeps layout, input, focus, and semantics; nested emission restores tree state; custom-painted controls keep layout semantics; themes can turn off container feedback; builder-owned gradient stops; per-widget cursor override; `widget-action ... hover` for headless hover tests. |
| 4 | `feat/canvas-sliders-and-steppers` | | 3 | App-level pointer, keyboard, and focus-loss hooks; `SliderRange` (source range, step, orientation, direction, reset value, commit edge) through input, semantics, and AppKit accessibility; `WidgetLayoutStyle.vertical` stacks a flush button group. |
| 5 | `feat/canvas-overlays-and-menus` | | 6 | Left/right anchors with flipping and edge-docked modals; a dialog pinned at a fraction of the window; light dismiss and Escape for point-anchored menus and root-relative modals; hover holds inside the pointer's safe triangle to an open menu; tooltips composed from child elements; checked native context menu items. |
| 6 | `perf/runtime-layout-and-motion` | | 8 | Measurement memoized per layout pass; larger per-view budgets on native desktop targets; virtual lists reuse mounted rows while scrolling; animation frames invalidate only animated commands; runtime control motion survives model rebuilds; model motion starts at input time; boolean controls follow model flips; public span run command ids. |
| 7 | `feat/effects-native-apps` | | 6 | Open- and save-file-path host requests for Zig apps behind the `dialog` permission; app-owned frame sinks and image decoding; a real-time synthesized audio output stream. |
| 8 | `fix/macos-canvas-rendering` | | 5 | Drop shadows at full strength and the right vertical direction; sub-point and zero stroke widths; nil-font and non-finite size guards; settable-attribute accessibility queries. |
| 9 | `perf/macos-scroll-and-compositing` | | 7 | Native scroller and content stay in step; scroll and frame delivery stay live during AppKit tracking; registered image pixels hash once; frame timers follow the display refresh rate (120 Hz); scrolling retargets cached rasters, and cache misses build concurrently; overlapping runs composite through one layer; images and simple shapes composite as GPU quads. Adds `zig build test-macos-scroll-host`, which `zig build test` runs on macOS. |
| 10 | `feat/macos-gpu-scenes` | 9 | 1 | Retained GPU 3D scenes (meshes, textures, text, shaders) rendered into registered images. |

PR 10 builds on PR 9: open it once 9 merges, or open it earlier with a
note that only its last commit is new.

Open an issue before 3, 4, 5, 9, and 10, and before the audio part of 7,
as `CONTRIBUTING.md` asks for larger changes. PR 3 is the largest; if review
stalls on the painter API, its cursor and hover commits stand on their own
and can be split back out.

The 31 single-change branches these were assembled from are kept as refs
under `refs/backup/split/*` (not pushed) in case a PR needs splitting again.

## Landing order and known conflicts

Every conflict between the ten branches is mechanical (neighboring lines,
not competing behavior):

- **Widget footprint test.** Each PR that adds a `Widget` field sets the
  reviewed size in `widgets.zig`: 2 at 792, 3 at 816, 4 at 808, 5 at 784;
  upstream is 776. Whichever lands later rebases and
  re-measures.
- **Packet wire format.** 2 moves it to v6 (serializer, both host decoder
  gates, their prose, and the `build.zig` checks, which now pin the gates
  too). If another change bumps the version first, renumber all of them.
- **Shared test anchors.** 1, 2, and 5 insert tests at the same spot in
  `widget_builtin_tests.zig`, and so do 3 and 4.
- **Widget fields.** 3 and 4 add neighboring fields to `Widget` and
  neighboring comparisons to `widget_invalidation.zig`.
- **`canvas_limits.zig`.** 3 (painter gradient stops) and 6 (desktop
  budgets) edit neighboring lines.
- **Effects exports.** 7 and 10 add neighboring exports to `src/root.zig`
  and neighboring methods to `effects.zig`.

## Fork-only changes

These stay on `proui-runtime-next` and are not proposed upstream as they
stand:

- **SF system fonts.** AppKit maps the low font ids to San Francisco (and SF
  Mono) so ProUI matches its SwiftUI reference. Upstream would need an
  explicit system-font selection API, not a changed default.
- **Compact controls hug columns.** `DesignTokens.controls.compact_controls_hug_columns`
  keeps buttons, toggles, and tab strips at their intrinsic width in a column.
  It is token-gated and off by default, so it could be offered upstream, but
  it is a design-system policy ProUI's theme turns on.
- **Switch timing and custom thumb.** The switch thumb uses the `.normal`
  duration to match SwiftUI, and a painter-drawn thumb animates through
  `paint_data[7] == 1`.
- **Native-select menus and search field sizing.** A primary click opens a
  native menu for widgets marked `paint_data[7] == 2`, and `paint_data[6]`
  sizes the search field icon and its text inset. These should become typed
  fields before they could go upstream.
- **Skeleton sweep.** Skeletons animate a travelling highlight instead of an
  opacity pulse, matching ProUI's reference.
- **Composed disclosures.** An accordion can drop its header
  (`paint_data[6] == -1`) and take a reveal duration from `paint_data[7]`.
  Needs a typed disclosure configuration and a rule for simultaneous
  disclosures with different durations before it is proposed. Its two
  general fixes are already extracted: the nested-disclosure settle fix in
  `fix/canvas-layout-and-render` and the revealing-accordion clip in
  `feat/canvas-typography-and-rounded-clips`.
- **Test and Wasm path budgets.** The fork keeps 8192 chart path elements for
  Wasm and test builds; upstream (and `perf/runtime-layout-and-motion`)
  keep 2048.

`feat/canvas-custom-control-seams` restores `tree_visible_bounds` after
nested emission as well as the scrim viewport; the fork still restores only
the scrim viewport, which leaves culling off for the rest of the outer tree
after a nested call.

## Verification

- Upstream `main` `96943d4a` passes `zig build test` on this machine (macOS,
  Zig 0.16.0), which is the baseline every branch is compared against.
- Every PR branch passes `zig build test` at its current tip; on 9 and 10
  that includes the AppKit scroll host harness. Every commit in 2 and 4
  (where the combined `Widget` footprint was re-measured) passes the canvas
  suite on its own.
- New regression tests were run against the unfixed code and fail there:
  overlay dismissal, the scroll host harness, nested disclosure settling,
  rounded image and accordion clips, animation transform order, the
  painter's nested-emission scrim, the menu safe triangle, and composed
  tooltips.
- Not yet run: `scripts/gate.sh fast` per branch, and ProUI against
  `proui-runtime-next`.

## Before opening the PRs

- Sign the commits. `CONTRIBUTING.md` requires cryptographic signatures and
  none of these are signed; once a key is configured, re-sign a branch with
  `git rebase --exec 'git commit --amend --no-edit -S' upstream/main` (use
  the base branch for stacked ones).
- Five commits carry a `Co-Authored-By: Claude Opus 5` trailer from the
  fork's history and the rest carry none; make that consistent.
- Run `scripts/gate.sh fast` on each branch.
- Leave `CHANGELOG.md` alone; the release agent writes it.

## Keeping this current

Make SDK changes on the fork, commit them with a scoped message, and when a
change is generic, add it to the PR it fits (or cut a new branch from
upstream `main` if none does) with a regression test that fails without it.
Update the table above, and keep anything ProUI-specific in the fork-only
list with the reason it stays.
