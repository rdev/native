# ProUI fork changes and upstreaming notes

This fork carries the Native SDK changes the ProUI port needs. Everything
generic is split into PR branches cut from upstream `main`; what stays here
is ProUI-specific policy that should not become an SDK default.

## Releases

ProUI ships against a tagged release of this fork, not a branch. Each
release is an upstream release tag with the fork's changes applied, named
`<upstream tag>-proui.<n>`, for example `v0.11.0-proui.1`. Apps depend on
the tag's source archive by URL and Zig package hash:

```sh
zig fetch --save=native_sdk https://github.com/rdev/native/archive/refs/tags/v0.11.0-proui.1.tar.gz
```

ProUI records the release it uses in `packages/native-sdk/native-sdk.json`
(version, URL, and hash). Its `npm run native-sdk:sync` fetches that release
into Zig's package cache, and its customer archives write the same URL and
hash into each example app's `build.zig.zon`.

To cut a release:

1. Rebase the release branch onto the new upstream tag (or add commits for a
   new `-proui.<n>` on the same base), and run `zig build test-canvas` and
   the ProUI app builds against it with `NATIVE_SDK_PATH` set to this checkout.
2. Tag the branch tip `<upstream tag>-proui.<n>` and push the branch and tag
   to `rdev`.
3. Run `zig fetch <archive URL>` and put the printed hash, with the version
   and URL, into ProUI's `native-sdk.json`.

Once upstream merges everything the kit needs, ProUI points
`native-sdk.json` and the customer docs at the upstream release archive
instead, and the fork stops cutting releases.

## Branches

| Branch | What it is |
| --- | --- |
| `proui-0.11` | The release branch: upstream `v0.11.0` plus the whole series and the fork-only changes below. `v0.11.0-proui.1` tags its tip. This file lives here. |
| `proui-runtime-next` | The series on upstream `main` `96943d4a`, before the `v0.11.0` rebase. |
| `proui-runtime` | The v0.10.1-based branch ProUI built against before releases (extraction history). |
| `proui-runtime-0.10.1` | Backup of `proui-runtime` before the first rebase. |
| PR branches | Ten themed PRs cut from upstream `main` `96943d4a`; only GPU scenes stacks on another. Each keeps one commit per change so it reviews commit by commit. |

The PR branches contain no ProUI code: no kit controls, painters, tokens,
catalog data, or ProUI names. Where the fork encoded a ProUI need through a
sentinel or a sidecar field, the PR branch replaces it with a typed API (for
example `SliderRange` instead of slider metadata packed into unrelated
`Widget` fields).

## PR branches

Each branch is open upstream as a pull request on `vercel-labs/native`,
#471 through #480 in the order below.

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

PR 10 builds on PR 9, so only its last commit is new until 9 merges. PR 3
is the largest; if review stalls on the painter API, its cursor and hover
commits stand on their own and can be split back out.

The PR branches are still based on `96943d4a`. Against `v0.11.0`, branches
1, 4, and 7 no longer merge cleanly, and the others merge without textual
conflicts but have not been rebuilt there. Upstream `v0.11.0` also routes
layout, measurement, rendering, and widget construction through optional
compiled-TypeScript policies, so changes that add widget fields or enum
cases must now also widen the policy wire guards. The `v0.11.0` adaptation
commits on `proui-0.11` show what each needed; fold them into the PR
branches when rebasing them.

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

These stay on the release branch and are not proposed upstream as they
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

Not yet proposed: `fix(macos): let open panels choose package documents`
lets a file panel's extension filter accept package documents such as
`.rtfd`. It is generic and fits PR 7.

## v0.11.0 adaptation

Three commits on `proui-0.11` adapt the series to `v0.11.0`. Zig apps leave
upstream's compiled-TypeScript policies unset, so the fork's Zig paths stay
authoritative; these keep both sides compiling and the policy wires stable:

- The anchor alignment's `center` case is appended after upstream's cases so
  compiled construction keeps its ordinals, and the construction wire guards
  name the fork's placements and alignments.
- The compiled render planner's rectangular clips lift into `RenderClip`.
- Painters run before upstream's render coordination policy, and nested
  emission saves and restores the text run policy.
- Tooltips lay out as stacks in the layout coordination policy, so composed
  tooltip content measures.
- Canvas view labels go through upstream's shell admission policy.
- The AppKit host defers a scroll driver's drift report to the next main
  queue turn, so it never re-enters a scroll driver update.
- The fork's desktop budgets allow more widgets per view than the compiled
  policy transports carry. Those transports keep their 1024-node wire
  (`max_compiled_policy_widget_nodes_per_view`), provenance records are
  capped to it, and widget motion skips views larger than it.
- The retained `Widget` footprint is 864 bytes.

## Verification

On `proui-0.11` (`v0.11.0` base), macOS, Zig 0.16.0:

- `zig build test-canvas` passes (1149 tests).
- All eight ProUI apps (catalog, minimal, and the six example apps) build
  in ReleaseFast against it.
- The desktop runtime and shell test suites crash with a stack overflow in
  `validateShellViewCreatePlan`. Not yet investigated; the fork's raised
  test-build budgets are the likely cause.

On the PR branches (`96943d4a` base):

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
- Not yet run: `scripts/gate.sh fast` per branch.

## Keeping this current

Make SDK changes on the fork, commit them with a scoped message, and when a
change is generic, add it to the PR it fits (or cut a new branch from
upstream `main` if none does) with a regression test that fails without it.
Update the table above, and keep anything ProUI-specific in the fork-only
list with the reason it stays.
