# ProUI fork changes and upstreaming notes

This branch carries the Native SDK changes needed by ProUI while the component library is being ported. It starts at Native SDK `v0.10.1`, commit `064ca9890cc0cf8adc198215bd0ddaeb586c220a`, and is consumed by the ProUI repository through `NATIVE_SDK_PATH`.

The current commits preserve the extraction history, but they are not intended to be submitted upstream unchanged. Some APIs and their consumers landed in adjacent commits, and several implementation choices need to be generalized before review. Use the proposed PR slices below when preparing upstream work.

## Current commit inventory

### `d27016a0` — canvas rendering extension seams

- Lets a retained `Widget` delegate its chrome to a kit-owned vector painter while Native SDK continues to own layout, children, input, focus, semantics, and invalidation.
- Includes custom paint identity and payload in retained-tree invalidation so visual state changes cannot reuse stale display lists.
- Uses an authored widget radius when clipping generic container children, which prevents square child content from leaking through rounded preview surfaces.
- Preserves text alignment in editable controls and exposes a painter-supplied text inset for readout/editor visual parity.
- Adds vertical slider geometry and state-aware slider rail painting.
- Raises the visible canvas and chart path arenas from 2,048 to 8,192 elements for detailed vector icon galleries.

The `paint` callback and its eight-float payload are declared in `src/primitives/canvas/widgets.zig`, which currently appears in `5a3a05c7`. Move those hunks and the `d8fc38ec` footprint assertion into this change before proposing it independently.

### `5a3a05c7` — source-range sliders and native editing hooks

- Adds slider minimum, maximum, step, orientation, reversed direction, and optional reset value while retaining a normalized internal fraction.
- Normalizes and snaps pointer, keyboard, and accessibility input through the same source-space conversion functions.
- Adds vertical and reversed keyboard behavior, Home/End handling, Shift acceleration, double-click reset, and one release-time commit after the final change event.
- Propagates slider metadata into accessibility snapshots and their retained hashes.
- Adds the runtime dispatch half of optional kit-level pointer and keyboard interception without replacing Native SDK's normal event ownership.
- Adds the runtime dispatch half of non-consuming focus-loss delivery so a blur can rebuild a controlled editor before the original click is resolved against the new tree.

The current implementation stores slider metadata in fields unused by slider widgets and encodes flags in `stream_size`. That avoided another hot-path size increase during the port, but an upstream proposal should use an explicit, reviewed representation or a tagged auxiliary payload. The public behavior is the part to preserve; the sidecar encoding is not an API recommendation.

### `4ff6fc98` — macOS retained rendering and frame pacing

- Retargets cached command rasters when a retained scroll update changes only their translation, avoiding CoreGraphics rerasterization on every wheel event.
- Samples translated textures through a linear Metal sampler so fractional scrolling remains smooth instead of snapping to integer texels.
- Builds independent raster-cache misses concurrently, publishes them in display order on the main thread, and raises the per-entry cache ceiling to 16 MiB.
- Handles translated flat-color quads, premultiplied alpha, command transforms, zero-width strokes, and compositor presenter initialization correctly.
- Derives retained canvas and window frame timers from the active screen refresh rate and assigns a quarter-frame timer tolerance, allowing 120 Hz presentation on ProMotion displays.
- Extends the AppKit accessibility ABI with slider range and orientation metadata.
- Completes the UI-tree and GPU-event plumbing for slider commits, outside-click blur, final drag samples, and change-before-commit ordering.
- Maps the low numbered font roles to AprpKit system fonts so the ProUI reference uses San Francisco metrics.

The SF font-role mapping is ProUI-specific and should not be proposed as a global default. Upstream it should either be omitted or replaced by an explicit system-font selection API. The cache size, concurrent raster creation, and refresh-rate scheduling also need focused memory, thread-safety, multi-display, and lifecycle tests before submission.

### `d8fc38ec` — retained widget footprint assertion

Updates the reviewed 64-bit `Widget` footprint from 776 to 816 bytes after adding the optional painter pointer and eight-float payload. This belongs with the final painter representation rather than as a separate upstream PR.

### `7d9a768c` — custom control layout semantics

- Treats explicitly authored horizontal button padding as the control's intrinsic-width contract, so a custom-painted compact button is not widened again by Native SDK's 44-point house minimum and token inset.
- Preserves button-group child segmentation in the static tree renderer when the group owns a custom painter; the retained layout renderer already applied the equivalent stamp.
- Resolves focus-within before invoking a custom input-group painter in both static and retained render paths, allowing a kit-owned field surface to use the same focused state as Native SDK's built-in input-group chrome.

The intrinsic-width change deliberately keys off `padding_is_kind_default`, so ordinary Native SDK buttons keep their current minimum size. Before upstreaming, add focused coverage for explicit zero and asymmetric padding, icon-plus-label measurement, custom-painted grouped buttons in both render walks, and custom input-group focus driven by baked and runtime state.

### Pending — macOS packet-text font resolution guard

- Sanitizes non-finite packet font sizes before asking AppKit to resolve a face.
- Falls back to the system font when the requested face cannot be resolved, and rejects the draw command if AppKit cannot provide that fallback.
- Prevents a nil `NSFont` from entering the attributed-string attributes dictionary, where Objective-C raises `NSInvalidArgumentException` on a raster worker and terminates the app.

This is an SDK reliability fix independent of ProUI. Upstream it with a malformed-font-packet regression test and coverage for concurrent raster-cache construction, since the observed crash occurred while packet text was being rendered on a cache worker.

### Pending — authored control text size

- Adds `WidgetStyle.text_size` as an optional exact point size. `widgetTypographySize` uses finite positive values ahead of the theme's small/default/large steps, so input layout, caret/selection metrics, and paint share the same font size. Invalid values fall back to the theme step, and retained invalidation compares the override so editing it repaints.
- ProUI currently uses it for the catalog's 13pt filter field without changing the kit's 11pt default body token or every other input in the app.

Before upstreaming, cover layout/render agreement for edited fields and decide whether invalid values should be rejected at the authoring boundary rather than falling back. The catalog's 13pt choice stays in ProUI; only the generic override belongs upstream.

### Pending — virtual-list scroll rebuild cadence

- Reuses the retained, already-translated window for fixed-height virtual lists while its visible range remains within the mounted rows, rebuilding only as coverage nears an edge. This removes a full app view/layout pass from most native scroll-driver reports.
- Adds an opt-in `exact_extents` contract for variable-height lists whose extent callback returns each row's actual laid-out height. Those lists get the same reuse; inexact estimates keep per-scroll rebuilding so skipped short rows are still measured and the scrollbar's total converges.
- ProUI's sidebar declares exact 25/47/30.5-point row heights and widens overscan to amortize re-windowing without mounting the whole catalog.

Before upstreaming, profile sustained 120 Hz wheel input and fast flings on macOS, verify other platform drivers, and consider whether exactness should be validated or represented as a separate extent mode rather than an author promise.

### Pending — macOS app-test updater linkage

- Links the updater C API exports into the dedicated app-test module on macOS. The production app root already exports these symbols, but the test root linked `appkit_host.m` without those exports, leaving `native_sdk_update_verify_feed` and `native_sdk_update_verify_archive` unresolved at the final link.
- Keeps this object test-only so a production binary does not get duplicate updater exports.

Before upstreaming, verify both Debug and ReleaseFast app tests link on macOS and retain the updater's existing signature-verification tests.

### Pending — native file picker request for Zig apps

- Routes `native-sdk.dialog.openFilePath` through the existing keyed `Effects.hostRequest` channel, returning the selected path (or an empty result on cancel) to a Zig model without requiring a JavaScript bridge.
- Binds the platform's open dialog through `UiApp` and checks the app's `dialog` permission before presenting it. The ProUI catalog consumes the result to show the actual selected filename in its file-input preview.

Before upstreaming, settle the command's name and result encoding against the existing JSON-based `native-sdk.dialog.openFile` bridge, and add permission, cancellation, fake-executor, and platform-service tests. The raw-path command is a fork-local integration seam for now.

### `4e938b07` — side-anchored popups and edge-docked modal panels

- Adds left/right anchor placement and center alignment for popovers, menus, and tooltips. Layout flips a side popup when it would leave the viewport, then clamps its final frame; ProUI uses this for submenu and side-placement variants.
- Adds explicit top/right/bottom/left modal edges and a fractional height for horizontal panels. ProUI uses those for drawer and sheet variants while keeping the content and visual styling in the kit.
- Includes the new layout fields in retained invalidation and accepts the new anchor strings in markup. No ProUI colors, painters, or catalog data enter the SDK.

Before upstreaming, cover side placement near every viewport edge, RTL submenu direction, modal edge resizing, fractional-height bounds, and keyboard/focus dismissal. ProUI's visual batch is awaiting user approval, so its component snapshot tests are intentionally deferred.

### `b81d9079` — context-press coordinates for point-anchored menus

- Carries the surface-local secondary-click point through `CanvasWidgetContextPressEvent` and exposes an optional `UiApp.on_widget_context_press` callback before the existing `on_hold` fallback. Apps can place their own context surface at the actual click position; apps without a callback retain their previous behavior.
- ProUI uses the already-existing point-anchor layout in the kit, so no menu-specific painter or placement policy is added to the SDK.

Before upstreaming, add event-routing coverage for callback ownership, fallback `on_hold`, and point coordinates on each platform. ProUI's visual batch is awaiting user approval, so those tests remain deferred here.

### `ebc4a807` — non-consuming pointer observation

- Adds `UiApp.on_widget_pointer_observe` after the existing consuming pointer hook. It can update app state from a primary down without bypassing normal focus, press, or long-press routing.
- ProUI records the primary press position for its long-press context menu through this hook; the regular right-click location comes from `b81d9079`.

Before upstreaming, cover an observed down followed by hold and release, including a model rebuild between down and timer firing.

### Pending — composed accordion disclosure motion

- Allows an accordion to use a zero-height, chrome-free header when an application composes its own trigger and surface. The retained accordion still owns closed/open clipping, hit testing, accessibility concealment, and frame-clock animation; ProUI uses it for accordion and collapsible variants rather than mounting/unmounting their content.
- Accepts a bounded per-disclosure duration in the existing paint payload and clips an authored-radius accordion to that radius. ProUI requests 75 ms to match its SwiftUI transition, with reduced motion still snapping.
- During a composed reveal, moves and fades the full-size content under the animated clip. A nested disclosure contributes only its own animated frame to its parent's settled-open test, so hidden descendants cannot leave the parent permanently non-interactive.

Before upstreaming, replace the paint-payload sentinels with a typed disclosure configuration, specify how simultaneous disclosures with different durations arbitrate the view-level tween, and add layout/render, nested-interaction, reduced-motion, and frame-cadence coverage after ProUI's visual review.

## Proposed upstream PR slices

1. **Kit-owned widget painting API.** Submit the painter contract, invalidation behavior, rounded clipping, editable alignment/inset, and footprint coverage together. Decide with maintainers whether the fixed `[8]f32` payload is acceptable or whether a typed/tagged extension payload is preferable.
2. **Complete slider value model.** Submit explicit slider range metadata plus shared pointer, keyboard, reset, commit, blur, semantics, and accessibility behavior. Replace the field-reuse sidecar before review and include platform-neutral tests for every orientation and direction.
3. **macOS retained-scroll compositor.** Submit translation-aware cache reuse, fractional texture sampling, alpha/transform fixes, and cache-miss concurrency with performance traces and raster-equivalence tests.
4. **Display refresh scheduling.** Submit refresh-rate-derived timers separately so multi-display movement, screen changes, and non-ProMotion behavior can be reviewed without the compositor diff.
5. **Vector path capacity.** Prefer a configurable or measured capacity policy over an unconditional 8,192-element global budget unless profiling shows the memory tradeoff is acceptable for all consumers.
6. **Custom control layout semantics.** Submit authored button insets, custom-painted group segmentation, and custom input-group focus-within as focused compatibility fixes. These behaviors are independent of ProUI's painters and keep extension seams consistent with their built-in counterparts.
7. **macOS packet-text validation.** Submit finite font-size validation and nil-font fallback as a focused crash fix, with a malformed packet fixture that proves a failed font resolution cannot terminate a raster worker or the host process.
8. **Authored control typography.** Submit an explicit per-widget text-size override with finite-value validation and input measurement/paint parity tests; leave ProUI's chosen point sizes out of the SDK.
9. **Virtual-list scroll cadence.** Submit retained-window reuse and the exact-extent opt-in with fixed and variable list regression tests; keep the ProUI sidebar sizing out of the SDK.
10. **Overlay positioning.** Submit side-anchor flipping and explicit modal docking with viewport-edge, focus, and dismissal coverage; keep ProUI's overlay recipes and catalog fixtures out of the SDK.

Do not upstream ProUI painters, design tokens, component metrics, catalog code, or the SF font-role override as a Native SDK default.

## Verification recorded during extraction

- All 21 transformed SDK source files matched the pinned post-patch SHA-256 digests before the first fork commits were created.
- ProUI's checkout resolver and bridge tests passed: 4 tests, 0 failures.
- `native check` passed for the ProUI catalog with `--strict`.
- The ProUI catalog compiled successfully as `ReleaseFast` against this checkout and launched from the forked runtime.
- The first full SDK gate identified the intentional `Widget` footprint change; `d8fc38ec` updates that assertion from 776 to 816 bytes.

The full Native SDK gate is not green yet. With Zig 0.16.0 and Node 24.21.0 selected, `zig build test` still exits while compiling the TypeScript conformance fixture at `tests/ts-core/fixture.ts:547` after diagnostic `NS1420`; the interrupted run also emitted diagnostics from several environment-sensitive fixture paths without reaching a final result for them. Establish a clean `v0.10.1` baseline for the same host, isolate any fork-caused failures, and record a complete successful `scripts/gate.sh fast` run before opening upstream PRs.

## Before opening upstream PRs

- Fetch `upstream/main`, rebase deliberately, and resolve each change against the current APIs rather than replaying the 0.10.1 text transformations.
- Turn the slices above into independently compiling, independently tested commits; the extraction commits are provenance, not cherry-pick units.
- Add focused correctness and performance tests before changing behavior further. The ProUI visual port intentionally deferred component snapshot tests until design approval, but upstream runtime changes need their own coverage.
- Run `scripts/gate.sh fast` with Zig 0.16.0 and Node 24+, then run the relevant macOS compositor and accessibility checks.
- Sign the rewritten commits cryptographically as required by `CONTRIBUTING.md`; the current extraction commits were created as local preservation commits.
- Keep `CHANGELOG.md` unchanged. Native SDK maintainers generate release notes from accepted history.
