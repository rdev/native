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

## Proposed upstream PR slices

1. **Kit-owned widget painting API.** Submit the painter contract, invalidation behavior, rounded clipping, editable alignment/inset, and footprint coverage together. Decide with maintainers whether the fixed `[8]f32` payload is acceptable or whether a typed/tagged extension payload is preferable.
2. **Complete slider value model.** Submit explicit slider range metadata plus shared pointer, keyboard, reset, commit, blur, semantics, and accessibility behavior. Replace the field-reuse sidecar before review and include platform-neutral tests for every orientation and direction.
3. **macOS retained-scroll compositor.** Submit translation-aware cache reuse, fractional texture sampling, alpha/transform fixes, and cache-miss concurrency with performance traces and raster-equivalence tests.
4. **Display refresh scheduling.** Submit refresh-rate-derived timers separately so multi-display movement, screen changes, and non-ProMotion behavior can be reviewed without the compositor diff.
5. **Vector path capacity.** Prefer a configurable or measured capacity policy over an unconditional 8,192-element global budget unless profiling shows the memory tradeoff is acceptable for all consumers.
6. **Custom control layout semantics.** Submit authored button insets, custom-painted group segmentation, and custom input-group focus-within as focused compatibility fixes. These behaviors are independent of ProUI's painters and keep extension seams consistent with their built-in counterparts.

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
