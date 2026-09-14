# WinBoat lean Electron fork

WinBoat uses a custom Electron 43.2.0 build for Linux x86-64 packages.
The fork removes browser features that WinBoat does not use.
It retains the trusted renderer, Node integration, DevTools, storage, networking, and accelerated rendering.

This document records the maintained configuration, final measurements, validation, and release process.
The tagged source remains the authority for implementation details.

## Published build

| Item                   | Value                                                                                         |
| ---------------------- | --------------------------------------------------------------------------------------------- |
| Repository             | [winboat-org/electron](https://github.com/winboat-org/electron)                               |
| Branch                 | `winboat-43.2.0`                                                                              |
| Immutable tag          | [`winboat-v43.2.0-1`](https://github.com/winboat-org/electron/releases/tag/winboat-v43.2.0-1) |
| Fork commit            | `c328b030fd3357139f66c8a3a84a4384c0136eed`                                                    |
| Upstream Electron base | `9b58e96340a34cccaccc08e410e76838b50b0cb2`                                                    |
| Chromium               | 150.0.7871.129                                                                                |
| Node.js                | 24.18.0                                                                                       |
| Build type             | Official release build with ThinLTO, PGO, and size optimization                               |

The release contains `electron-winboat-v43.2.0-1-linux-x64.zip`.

| Artifact            |              Size | SHA-256                                                            |
| ------------------- | ----------------: | ------------------------------------------------------------------ |
| Electron ZIP        | 104,350,649 bytes | `9163cb462555e606b025522a5f38e75cf78240435e00d3724742575147fd6245` |
| Electron executable | 165,784,824 bytes | `2aa1117c49640454a3f12792e977ee2149ead5a68eb9dabeff481f1734864d42` |

The senior review approved the corrected source and published artifact.

## Scope and constraints

- Optimize whole-process-tree PSS, not one process or summed RSS alone.
- Keep Node and native addon access in the trusted renderer.
- Keep the USB addon, Guest Server, Helios assets, and FreeRDP workflow.
- Keep DevTools, WebAssembly, HTTP/HTTPS, `fetch`, file URLs, WebSockets, and WebGL.
- Keep hardware GPU compositing, GPU rasterization, and a working SwiftShader fallback.
- Keep Chromium storage for `localStorage`, IndexedDB, cookies, and cache.
- Keep process sandboxing, the zygote model, HSTS, Safe Browsing, and BackupRefPtr.

## Fork changes

The complete GN configuration is
[`build/args/winboat.gn`](https://github.com/winboat-org/electron/blob/winboat-v43.2.0-1/build/args/winboat.gn).
The tagged repository also contains the Electron, Chromium, and Node patch series.

### Removed or reduced features

| Area             | Changes                                                                                                |
| ---------------- | ------------------------------------------------------------------------------------------------------ |
| Build output     | Optimize for size, use relative vtables, and omit local debug symbols.                                 |
| Browser UI       | Remove PDF, printing, spellcheck, Electron extensions, plugins, and background mode.                   |
| Device and AI    | Remove JXL, browser speech, XR, BlueZ, WebNN backends, and the on-device-model service.                |
| Graphics         | Remove Dawn, WebGPU, and Chromium Vulkan. Keep ANGLE Vulkan only for SwiftShader.                      |
| Media            | Remove FFmpeg, browser audio, media remoting, HLS, hardware video, AV1, VPx, and OpenH264.             |
| WebRTC           | Remove bundled codecs, audio processing, SCTP, and native desktop-capture integrations.                |
| Network services | Remove Kerberos, mDNS, reporting, device-bound sessions, and the SQL disk-cache backend.               |
| Desktop services | Remove notifications, captive portal detection, translation, Compose, Lens, remoting, CUPS, and MPRIS. |
| Node.js          | Remove Amaro, `node:sqlite`, and Node experimental Web Storage.                                        |
| V8               | Remove Temporal support and reduce Maglev. Sparkplug, TurboFan, and WebAssembly remain.                |

Chromium SQLite remains enabled.
The `node_use_sqlite` option does not control renderer `localStorage` or IndexedDB.

### Required source work

Some disabled combinations needed source guards because upstream does not test them regularly.

- Electron now guards BlueZ setup, extension state, background mode, and FFmpeg packaging.
- Chromium now guards the optional on-device-model utility service and its Dawn dependency.
- Chromium supports Linux GPU builds without Chromium Vulkan or Dawn.
- The GPU patch preserves GPU rasterization through ANGLE/OpenGL.
- ANGLE retains its Vulkan backend only for the SwiftShader software path.
- Linux capture retains one small GPU-channel host required by unconditional references.
- Node removes disabled SQLite inspector code and Amaro metadata.
- Node constructs selected embedded metadata only when a process requests it.
- Node configuration queries the filtered Electron GN graph instead of unrelated Chromium targets.
- Desktop translation WebUI code keeps its operating-system and feature guards.

The GPU rasterization guard is important.
Removing Dawn initially made normal 2D rendering use software rasterization.
That regression added approximately 90 MiB of renderer-private memory on the Home screen.

### Deliberately retained

- DevTools frontend, inspector protocol, tracing, and required workers
- V8, Node.js, libuv, N-API, native modules, and ASAR support
- HTTP, HTTPS, DNS, TLS, `fetch`, WebSockets, and file URLs
- Chromium storage services and SQLite
- Skia, ANGLE/OpenGL, Ozone, Aura, X11, and Wayland
- GPU process isolation, hardware compositing, and GPU rasterization
- SwiftShader and the ANGLE backend that it requires
- Native dialogs, shell integration, clipboard, text shaping, and accessibility
- PNG, WebP, SVG, WOFF2, and other formats required by the interface
- HSTS, Safe Browsing, BackupRefPtr, and normal Chromium sandboxing

## Measured results

Linux measurements use `/proc/<pid>/smaps_rollup` for the browser and all descendants.
Each build uses the same application payload, screen, installation state, and settle time.
Runs use fresh Chromium profiles and alternate build order.

PSS is the primary physical-memory result.
Summed RSS counts shared mappings more than one time.
USS excludes shared pages and reports private memory.

### Stock Electron compared with the published fork

Both runtimes used the same WinBoat 1.0.6 application archive.
Its SHA-256 value was `d57f9bae4b07bc8591be60a15b29fa812aaaa6775e0f8a5c6494c01c2749220a`.

| Metric      | Stock Electron 43.2.0 | Published fork |                     Saving |
| ----------- | --------------------: | -------------: | -------------------------: |
| RSS, summed |             710.3 MiB |      603.2 MiB | 107.1 MiB, or 15.1 percent |
| PSS         |             357.2 MiB |      312.6 MiB |  44.6 MiB, or 12.5 percent |
| USS/private |             230.0 MiB |      211.0 MiB |   19.0 MiB, or 8.2 percent |

Mapping analysis attributes most PSS savings to executable and shared-library pages.
These pages consume physical memory but remain clean and reclaimable under pressure.
The private-memory reduction is smaller and is shown by USS.

### Electron disk size

The table compares the official and custom Electron 43.2.0 Linux x64 distributions.

| Scope            |             Stock |    Published fork |                            Saving |
| ---------------- | ----------------: | ----------------: | --------------------------------: |
| Distribution ZIP | 124,904,101 bytes | 104,350,649 bytes | 20,553,452 bytes, or 16.5 percent |
| Extracted files  | 326,181,148 bytes | 265,642,764 bytes | 60,538,384 bytes, or 18.6 percent |
| Main executable  | 219,917,560 bytes | 165,784,824 bytes | 54,132,736 bytes, or 24.6 percent |

### WinBoat package trimming

The published Electron ZIP remains a complete redistribution archive.
WinBoat removes more files when electron-builder creates the application package.

| Package change                                 |  Unpacked saving |
| ---------------------------------------------- | ---------------: |
| Keep only `en-US.pak`                          | 46,924,178 bytes |
| Remove the unused Crashpad handler             |  1,701,456 bytes |
| Compress Chromium notices with Brotli level 11 | 19,786,551 bytes |
| Total                                          | 68,412,185 bytes |

The package retains Chromium notices as `LICENSES.chromium.html.br`.
It also includes offline decompression instructions.

## Related WinBoat changes

Application work contributes additional RAM, startup, and disk savings.
These changes are separate from the matched Electron comparison above.

| Commit    | Change                                           | Primary effect                                                           |
| --------- | ------------------------------------------------ | ------------------------------------------------------------------------ |
| `811d314` | Allowlist compiled application files.            | Reduced accidental package content and the application archive.          |
| `48a0945` | Load non-initial Vue routes dynamically.         | Defers route parsing and initialization.                                 |
| `e8a0f5a` | Replace ApexCharts with a small SVG gauge.       | Removes a large renderer runtime and reactive chart work.                |
| `1dff62c` | Remove `@electron/remote` and `electron-store`.  | Earlier combined tests measured approximately 20 MiB less memory.        |
| `3b4c1ea` | Clean up Apps-view subscriptions.                | Prevents stale listeners and component references.                       |
| `81316ef` | Prevent overlapping setup refreshes.             | Bounds concurrent polling work.                                          |
| `78b512b` | Run Chromium's network service in-process.       | Removes one utility process but moves its work into the browser process. |
| `39f232b` | Remove unused VueUse Motion and keep one locale. | Reduces renderer and package size.                                       |

The production `app.asar` decreased from approximately 454 MB to 51.6 MB.
Most of this reduction is a disk and startup improvement.

## Validation

The final tagged artifact passed these checks:

- Home and Setup rendered correctly.
- Home detected the existing WinBoat installation.
- The Guest API and container were online.
- Electron, Chromium, and Node reported the expected versions.
- The USB native addon loaded and enumerated devices.
- Guest Server, Helios, and data payloads remained present and unchanged.
- `localStorage`, IndexedDB, WebAssembly, and normal renderer DevTools worked.
- The embedded DevTools frontend loaded successfully.
- Hardware tests used ANGLE OpenGL and Skia GaneshGL.
- GPU compositing and GPU rasterization remained enabled.
- A forced SwiftShader launch used `ANGLE_SWIFTSHADER` successfully.
- The GPU process remained isolated.
- `ldd` reported no missing shared libraries.
- The package contained one locale, compressed notices, and no Crashpad or FFmpeg file.
- The application remained stable during the settled Home-screen test.

## WinBoat package integration

Linux x86-64 package builds use the published fork by default.

1. `scripts/prepare-electron.mjs` downloads or copies the pinned archive.
2. The script verifies the archive size and SHA-256 value.
3. `electron-builder.json` passes the verified ZIP through `electronDist`.
4. `scripts/after-pack.mjs` rejects other platforms and architectures.
5. The hook verifies the packaged executable against the published SHA-256 value.

The cache location is `.cache/electron`.
Set `WINBOAT_ELECTRON_ZIP` to use a verified local copy for offline builds.

Run the normal package command:

```bash
bun run build:linux-gs
```

Both GitHub Actions package jobs use this command.
Local and CI package builds therefore use the same verified Electron input.

Development mode keeps the stock Electron package.
The dependency remains pinned to 43.2.0 for API, type, Chromium, and Node ABI parity.

## Rebuild and release procedure

Use the tagged source and the
[fork build guide](https://github.com/winboat-org/electron/blob/winboat-v43.2.0-1/docs/development/winboat-lean-build.md).
The guide follows Electron's
[manual GN build instructions](https://www.electronjs.org/docs/latest/development/build-instructions-gn).

From the synchronized Chromium source directory:

```bash
gn gen out/Lean \
  --args='import("//electron/build/args/winboat.gn")' \
  --root-target=//electron:electron_dist_zip

autoninja -C out/Lean electron:electron_dist_zip
```

For each new Electron version:

1. Rebase the fork onto the exact upstream release.
2. Refresh Electron, Chromium, and Node dependency patches.
3. Generate the filtered GN graph from `winboat.gn`.
4. Build `electron:electron_dist_zip` with release PGO and ThinLTO.
5. Test Home, Setup, storage, USB, DevTools, hardware GPU, and SwiftShader.
6. Compare matched whole-tree PSS against stock Electron.
7. Publish an immutable tag and release asset.
8. Update the pinned URL, size, and hashes in WinBoat.
9. Build WinBoat and verify its packaged executable hash.

## Future optimization experiments

The following items are unverified candidates, not release results.
An exploratory Home-screen inspection measured approximately 266.8 MiB PSS after a long settle:
81.4 MiB in the browser, 100.1 MiB in the renderer, 67.0 MiB in the GPU process,
17.6 MiB in the zygotes, and 0.7 MiB in the AppImage helper.
The state and profile differed from the published benchmark, so these values are not comparable with it.

### First priority: ANGLE Vulkan

The current ANGLE/OpenGL path maps LLVM and Mesa Gallium code.
These mappings used approximately 12.4 MiB and 4.3 MiB PSS during the inspection.
The build keeps the ANGLE Vulkan backend for SwiftShader, so the same binary can test native Vulkan with
`--use-angle=vulkan` while Chromium Vulkan and Dawn remain disabled.

This test has the clearest chance of a material RAM saving without another build.
The initial estimate is 8 to 15 MiB PSS, but the result can be smaller or negative.
Test hardware rendering, WebGL, DevTools, resizing, X11, Wayland, and SwiftShader fallback before adoption.

### Smaller RAM experiments

- Cap Chromium browser and renderer worker pools at four to six threads. The inspection found seven browser workers and eleven renderer workers idle after startup. This change can save thread stacks and allocator caches, but it can slow startup or parallel rendering.
- Test `--num-raster-threads=1`. The current renderer uses four raster threads. Check resizing and animation performance.
- Test `--disable-features=PartitionAllocLargeThreadCacheSize`. The renderer held approximately 2.2 MiB in PartitionAlloc thread caches.
- Reduce the Skia glyph texture cache from 8 MiB to 2 MiB. Check text quality, scrolling, and redraw performance.
- Train a PGO profile with WinBoat startup and normal routes. The current profile represents general Chromium use. A WinBoat profile can improve executable layout without removing APIs.
- Test early `madvise(MADV_RANDOM)` on executable mappings. This can reduce Linux file-mapping readahead, but it can increase page faults and startup time. Treat this as research, not a planned change.

The WinBoat Home screen also uses two large 200-pixel CSS blurs and a backdrop-blurred titlebar in `App.vue`.
A controlled test should replace them with pre-rendered gradients.
This is an application change, but it can reduce full-window GPU surfaces with little visual difference.

### Disk-focused experiments

- Create a WinBoat Blink and content feature profile. Candidate removals include browser WebRTC and media APIs, browser device APIs, service and background APIs, Protected Audience, shared storage, attribution, WebAudio, payments, credentials, WebNN, WebXR, WebGPU bindings, WebTransport, and direct sockets.
- Keep WebGL, WebSockets, storage, DevTools workers, normal HTTP, and `fetch`.
- Test ThinLTO optimization level 0 and `-Oz`. These options can reduce the executable but can also reduce performance.
- Test a conservative English ICU data filter. `icudtl.dat` is 10.9 MB, but it contributes little live PSS.
- Consider omitting frame pointers only after the loss of native profiling and stack-unwind quality is accepted.

### Low-value or unsuitable changes

- The renderer V8 heap was approximately 13.1 MiB. Disabling Sparkplug or making broad V8 changes has little RAM potential and can reduce JavaScript performance.
- Maglev execution is disabled. V8 still compiles some Maglev graph-builder code because Turboshaft and TurboFan use it.
- The general Skia cache limit is 256 MiB, but actual use was approximately 9 MiB. Reducing the limit above actual use has no effect.
- Zero-copy switches do not affect the active GPU-raster path.
- Zygote PSS is mostly shared executable code. Removing zygotes does not recover the displayed total and can reduce startup performance or security.

## Current limits

- The published fork supports Linux x86-64 only.
- Disabled feature combinations have less upstream build coverage.
- Every change requires compile, runtime, GPU, native-addon, and memory tests.
- BackupRefPtr remains enabled because removing it has a memory-safety cost.
- Accessibility remains enabled because no measured saving justifies its removal.

Do not report an estimate, executable-size change, or process-count change as a RAM saving.
Use matched whole-tree PSS measurements for every future optimization claim.
