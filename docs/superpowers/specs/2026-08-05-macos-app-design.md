# macOS App for Path of Building — Design

**Goal:** A native macOS app (`Path of Building.app`) on Apple Silicon, built via Makefile targets in this repo, using the upstream SimpleGraphic host.

**Status of upstream (verified 2026-08-05, SimpleGraphic `master` @ 3b1a346):** The
[PathOfBuilding-SimpleGraphic](https://github.com/PathOfBuildingCommunity/PathOfBuilding-SimpleGraphic)
repo already contains most of the macOS port: `if (APPLE)` CMake handling,
`engine/system/win/sys_macos.mm` (LaunchServices URL opening),
`sys_console_unix.cpp`, ObjC++ enabled, `__APPLE__` branches throughout
`sys_main.cpp`, GCC-visibility-correct export of `RunLuaFileAsWin(argc, argv)`,
and cross-platform vcpkg dependencies (ANGLE, GLFW, LuaJIT, curl, fmt, re2,
zstd, zlib, libwebp, sol2, gli, ms-gsl). Its CMake also builds all four native
Lua modules PoB needs: `lcurl`, `lua-utf8`, `socket` (luasocket), `lzip`.
What does not exist: any macOS launcher executable (the counterpart of
`Path of Building.exe`), macOS CI/artifacts, and an untested build (known snag:
`libs/luasocket/src/wsocket.c` is listed unconditionally in CMake; POSIX needs
`usock.c`).

## Decisions

1. **Personal app first.** Ad-hoc codesigned, no Apple Developer account.
   Signing/notarization is a follow-on.
2. **Auto-updater disabled in v1 — by running in dev mode.** The updater
   sha1-compares every managed file (Lua *and* native runtime) against the
   published branch manifest, and deletes local files absent from the remote
   manifest. The published manifest has no macOS runtime section yet, so an
   update-enabled macOS install would delete its own native runtime. Running
   from a git checkout keeps `launch.devMode = true` (local `manifest.xml` has
   no `platform`/`branch` attributes), which disables updates entirely;
   updates happen via `git pull` + `make`. Updater parity is a follow-on that
   depends on upstream manifest hosting (see `docs/crossPlatform.md`).
3. **arm64 only.** vcpkg triplet `arm64-osx`; Intel/universal is a follow-on.
4. **SimpleGraphic lives in a sibling clone** at
   `~/dev/thirdparty/PathOfBuilding-SimpleGraphic`, branch `feat/macos-build`.
   The PoB Makefile references it via an overridable `SG_DIR` variable.
5. **The app is a thin shim over the checkout.** It bundles the native
   runtime but executes `src/Launch.lua` from the PoB checkout (absolute path
   baked at build time, overridable via the `POB_SCRIPT_PATH` environment
   variable). Consequences: dev mode stays on (safe, per decision 2), user
   data lives in the checkout (`src/`), and `git pull` updates the app's
   logic instantly. A self-contained bundle is deferred until updater support
   exists.

## Architecture

Two workstreams:

### Workstream 1: SimpleGraphic macOS build + launcher (sibling repo)

- Branch `feat/macos-build` in the sibling clone; submodules initialized.
- Build with CMake + vcpkg manifest mode, triplet `arm64-osx` (release).
  Expected outputs (install tree): `libSimpleGraphic.dylib`, `lcurl.so`,
  `lua-utf8.so`, `socket.so`, `lzip.so` (Lua modules are loaded by
  `require`, which on macOS expects `.so`), plus dependent dylibs
  (ANGLE `libEGL`/`libGLESv2`, libcurl, etc.).
- Fix build breaks as the compiler surfaces them; the known one is a platform
  conditional for luasocket (`wsocket.c` → `usock.c`/`unixtcp` set on POSIX).
  All fixes stay upstreamable: guarded, minimal, no macOS-only forks of
  shared logic.
- New `pob` launcher target (`macos/main.cpp`, ~50 lines): resolves the Lua
  entry script (first CLI arg, else `POB_SCRIPT_PATH` env var, else a
  compile-time default), `chdir`s to the script's directory, builds
  `argv[0] = <script path>` plus passthrough args, links `SimpleGraphic`, and
  calls `RunLuaFileAsWin`. Errors (missing script, missing dylib) print to
  stderr and exit non-zero.

### Workstream 2: PoB Makefile + .app assembly (this repo)

New root `Makefile` with targets:

- `test` — full busted suite via the Docker harness (platform-pinned
  `docker run`, matching `docs`/CI usage).
- `test-python` — pytest over `tests/`.
- `manifest` — `python3 update_manifest.py --in-place`.
- `macos-runtime` — configure + build + install the SimpleGraphic install
  tree from `$(SG_DIR)` into `build/macos/runtime-native/`. Fails with a
  clear message if `$(SG_DIR)` is missing.
- `macos-app` — assembles `build/macos/Path of Building.app`:
  - `Contents/MacOS/Path of Building` — the `pob` launcher.
  - `Contents/Frameworks/` — `libSimpleGraphic.dylib` + dependent dylibs
    (install_name fixed up via `@rpath`; launcher gets an rpath entry).
  - `Contents/Resources/runtime/` — mirrors the Windows `runtime/` layout:
    native Lua modules, `lua/` (pure-Lua modules copied from `runtime/lua/`),
    `SimpleGraphic/Fonts/` (copied from `runtime/SimpleGraphic/`).
  - `Contents/Info.plist` — bundle metadata + `CFBundleURLTypes` registering
    the `pob:` URL scheme.
  - Icon: generated `.icns` if a suitable source image is available in the
    repo; otherwise omitted (system default icon) — not a blocker.
  - Ad-hoc codesign (`codesign --force --deep -s -`).
  - Launch.lua path default baked as this checkout's `src/Launch.lua`.
- `run-macos` — builds `macos-app` if needed and `open`s it.

The exact module-search expectations of the host (how `require` finds native
modules relative to the dylib/script) are verified during implementation and
the bundle layout adjusted if the host expects modules elsewhere; the
Windows-runtime-mirroring layout is the starting point.

## Error handling

- Launcher: missing script/dylib → stderr message + non-zero exit.
- Make targets: missing `SG_DIR`, missing build outputs, missing tools
  (cmake, ninja/xcodebuild) → fail fast with actionable messages.

## Testing

- Existing busted suite unaffected (never loads the native host); `make test`
  must stay green.
- SG build: smoke = launcher runs `src/Launch.lua` and the UI renders.
- Acceptance: app launches from Finder, renders the passive tree, loads a
  build, `open "pob://..."` routes into the app, user data persists in the
  checkout.

## Out of scope (follow-ons)

Auto-update parity (needs upstream manifest hosting + basic Update host),
signing/notarization, universal/Intel binary, upstream macOS CI artifacts,
self-contained bundle.

## As built (2026-08-05)

Implementation deviations from the design above, all sanctioned during
execution: the bundle uses a flat `Contents/MacOS/` layout mirroring the
Windows `runtime/` directory (the host resolves fonts, Lua modules, and
native modules relative to the executable's directory — the
Frameworks/Resources split described earlier does not match the host's
contract); the SimpleGraphic install tree lives at `$(SG_DIR)/build/dist`;
the vcpkg triplet is `arm64-osx` with an overlay making only ANGLE dynamic
(GLFW dlopens `libEGL.dylib` at runtime); `CFBundleExecutable` is a wrapper
script baking `POB_SCRIPT_PATH`, keeping PoB-specific paths out of the
upstreamable SimpleGraphic repo. `pob://` URL delivery into Lua works for
CLI invocation; Apple-Event delivery at app launch is best-effort and
delivery to a running instance is not implemented.
