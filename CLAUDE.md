# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Build / Develop Commands

```sh
# Quick start: download prebuilt librime (skips building librime from source)
bash ./action-install.sh

# Build Squirrel.app (Release)
make

# Build Squirrel.app (Debug)
make debug

# Build and install to /Library/Input Methods/Squirrel.app
make install          # Release
make install-debug    # Debug

# Build distributable .pkg
make package

# Clean build artifacts (not dependencies)
make clean
# Clean everything including librime, plum, Sparkle
make clean-deps
# Clean package outputs only
make clean-package

# Lint (runs via swiftlint, also in CI)
swiftlint

# Check for unused code (runs via periphery in CI)
periphery scan --relative-results --skip-build --index-store-path build/Index.noindex/DataStore
```

Build environment variables (set before `make`):
- `BOOST_ROOT` — required when building librime from source
- `ARCHS='arm64 x86_64'` — build universal binary
- `MACOSX_DEPLOYMENT_TARGET='13.0'` — minimum macOS version
- `DEV_ID='Your Apple ID name'` — enable code signing and notarization
- `PLUM_TAG=':preset'` — bundle specific plum formulae

## Architecture Overview

Squirrel is the macOS frontend for the **Rime input method engine (librime)**. It uses InputMethodKit (IMK) to register as a macOS text input source. All source code lives in `sources/` (11 files, ~100KB total, mixed Swift + one ObjC bridging header).

### Three-layer interaction with librime

Squirrel calls librime entirely through its **C API** (`rime_api.h` / `rime_api_stdbool.h`), accessed via the `RimeApi_stdbool` function pointer struct obtained from `rime_get_api_stdbool()`. The struct uses a self-versioning scheme: every struct has `data_size` as its first field, so librime can detect which fields are available at runtime.

1. **Lifecycle & config** — `SquirrelApplicationDelegate.swift` handles `setup()`, `initialize()`, `start_maintenance()`, `deploy()`, `sync_user_data()`, `finalize()`, and `set_notification_handler()` for schema/option/deploy notifications.
2. **Key handling** — `SquirrelInputController.swift` creates Rime sessions, converts macOS NSEvent → Rime keycodes via `MacOSKeyCodes.swift`, calls `process_key()`, then reads results via `get_commit()`, `get_context()`, `get_status()` to populate marked text and candidate panels.
3. **Config reading** — `SquirrelConfig.swift` wraps `config_open()` / `schema_open()` and `config_get_bool/cstring/double` to read `squirrel.yaml` and per-schema style settings.

### C bridging

The bridging header (`Squirrel-Bridging-Header.h`) imports `<rime_api_stdbool.h>` and `<rime/key_table.h>`, exposing all librime types to Swift. `BridgingFunctions.swift` extends structs (`RimeContext_stdbool`, `RimeTraits`, `RimeCommit`, `RimeStatus_stdbool`) with:
- `rimeStructInit()` — zero-initializes and sets `data_size` for versioned struct compatibility
- `setCString()` — copies Swift strings to persistent C string pointers in struct fields

### Key source files

| File | Role |
|------|------|
| `Main.swift` | `@main` entry point, CLI flag handling (`--build`, `--reload`, `--sync`, etc.) |
| `SquirrelApplicationDelegate.swift` | App lifecycle, librime init/shutdown, notification handler, Sparkle updates |
| `SquirrelInputController.swift` | IMKInputController subclass, the core keyboard→Rime bridge, candidate selection, chord-typing, vim mode |
| `SquirrelConfig.swift` | Reads squirrel.yaml and schema configs via `RimeConfig` API |
| `MacOSKeyCodes.swift` | macOS keyCode + modifiers → Rime XK_ keycode + modifier mask conversion |
| `SquirrelPanel.swift` | NSPanel for candidate window, position tracking |
| `SquirrelView.swift` | Custom NSView rendering of candidates/preedit |
| `SquirrelTheme.swift` | Theme/color parsing from Rime config |
| `InputSource.swift` | macOS TIS input source registration |
| `BridgingFunctions.swift` | C struct init helpers, operators (`?=`, `+=`, etc.) |

### Git submodules

- `librime/` — Rime core engine (C++), built separately then copied into `lib/` and `bin/`
- `plum/` — Rime package manager, data output goes to `data/plum/`
- `Sparkle/` — Auto-update framework

### Librime build integration

`Makefile` builds librime via `make -C librime release install`, then copies `librime.1.dylib` → `lib/`, plugins → `lib/rime-plugins/`, and `rime_deployer`/`rime_dict_manager` → `bin/`. These are bundled into the app bundle by Xcode build phases. The CI `action-install.sh` can download prebuilt librime instead of compiling from source.

### Testing

There is no dedicated test suite for Squirrel itself. CI runs `swiftlint` for linting and `periphery` for unused code detection. The Sparkle submodule has its own test suite.

### Code style

- SwiftLint configured with line length 200, function body 200 lines, file length 800 warning / 1200 error
- Disabled rules: `force_cast`, `force_try`, `todo`
- Comments are mostly in Chinese, inline design-decision notes use Chinese prose
- Naming: `Squirrel` prefix for app-level types, `rime` prefix for C API calls via the struct
