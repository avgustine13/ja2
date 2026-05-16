# JA2 AI Context

This file tracks working context, architectural notes, and repository findings gathered while exploring this Jagged Alliance 2 source tree.

## Repository Identity

- This repository is a legacy Jagged Alliance 2 source drop arranged for Microsoft Visual C++ 6.0.
- The original packaging notes are in `readme.txt`.
- The code appears to be the JA2 source bundle distributed with JA2: Wildfire, plus helper files to make the original build easier.

## Top-Level Layout

- `ja2/Build`
  - Main game source tree and VC6 workspace/projects.
- `Standard Gaming Platform`
  - Shared engine/platform layer used by JA2.
- `readme.txt`
  - Original build notes from the source bundle.
- `SFI Source Code license agreement.txt`
  - License text included with the bundle.

## Build System Notes

- Primary workspace: `ja2/Build/JA2.dsw`
- Primary application project: `ja2/Build/JA2.dsp`
- The workspace contains multiple VC6 projects:
  - `ja2`
  - `Editor`
  - `Laptop`
  - `Strategic`
  - `Tactical`
  - `TacticalAI`
  - `TileEngine`
  - `Utils`
  - `Standard Gaming Platform`
- The original readme expects the source to live under:
  - `C:\ja2`
  - `C:\Standard Gaming Platform`
- The VC6 project files contain hardcoded include/output assumptions and old third-party libraries such as `smackw32.lib` and `mss32.lib`.

## Entry Points And Runtime Flow

- Process entry point: `Standard Gaming Platform/sgp.c`
  - Defines `WinMain`.
  - Owns shared runtime initialization and Windows message handling.
- JA2 initialization: `ja2/Build/Init.c`
  - Initializes JA2-specific systems such as text, sound, strategic, tactical, world, lighting, and event management.
- High-level game loop: `ja2/Build/gameloop.c`
  - Runs screen transitions and global per-frame orchestration.
- Screen registry: `ja2/Build/SCREENS.C`
  - Central table mapping screen IDs to initialize/handle/shutdown callbacks.

## Major Subsystems

### Standard Gaming Platform

Core runtime layer with services such as:

- Windows application bootstrap
- Input and mouse handling
- Video/surfaces/blitters
- Sound wrappers
- File/database helpers
- Fonts and text drawing
- Memory and debug utilities

Representative files:

- `Standard Gaming Platform/sgp.c`
- `Standard Gaming Platform/video.c`
- `Standard Gaming Platform/vobject_blitters.c`
- `Standard Gaming Platform/soundman.c`
- `Standard Gaming Platform/FileMan.c`

### Tactical

Turn-based combat and in-sector gameplay:

- Soldier control
- Weapons and items
- UI panels and cursors
- LOS/FOV/pathing
- militia, vehicles, doors, corpses, interaction systems

Representative directory:

- `ja2/Build/Tactical`

### TacticalAI

AI decision-making for combat and NPC behaviors:

- action selection
- attacks
- movement
- medical behavior
- knowledge modeling
- panic buttons and scripted logic

Representative directory:

- `ja2/Build/TacticalAI`

### Strategic

Overworld systems and campaign layer:

- assignments
- contracts
- quests
- queen AI
- strategic movement
- town loyalty and mines
- pre-battle and autoresolve
- map screen logic

Representative directory:

- `ja2/Build/Strategic`

### TileEngine

Map/world representation and rendering support:

- world data
- structures
- lighting
- fog of war
- exit grids
- tile animation and caches
- overhead map and radar rendering

Representative directory:

- `ja2/Build/TileEngine`

### Laptop

The in-game laptop/web UI and related campaign services:

- AIM
- MERC
- Bobby Ray
- IMP creation flow
- email, files, finances, personnel, insurance, florist, funeral

Representative directory:

- `ja2/Build/Laptop`

### Editor

Integrated map/editor tooling:

- terrain editing
- item editing
- map summary
- smoothing
- sector info
- undo/taskbar/editor callbacks

Representative directory:

- `ja2/Build/Editor`

### Utils

Cross-cutting helpers used by the rest of the game:

- text helpers
- message boxes
- sound/music control
- event pumping
- encrypted files
- multilingual text data
- progress bars and UI helpers

Representative directory:

- `ja2/Build/Utils`

## Architectural Characteristics

- Predominantly C with a small amount of C++.
- Heavy use of globals and central managers.
- Screen/state-machine-driven front-end flow.
- Tight coupling between gameplay systems and engine services.
- Naming is inconsistent by modern standards and often reflects original asset/system terminology.
- Many filenames contain spaces and legacy capitalization patterns.

## Repository Scale Snapshot

- Approximate tracked source/build files discovered during initial scan: `724`
- Extension breakdown from the scan:
  - `370` headers
  - `332` C source files
  - `3` C++ source files
  - `9` VC6 project files
  - `1` VC6 workspace file

## Risks / Friction Areas

- Original toolchain assumptions are obsolete.
- Hardcoded paths and legacy library expectations will complicate modern builds.
- Legacy DirectX and Windows APIs are likely to need compatibility work.
- Global-state-heavy architecture raises regression risk during refactors.
- The project predates modern warnings, sanitizers, and build hygiene.

## Progress Log

- 2026-05-17:
  - Investigated a VS2022 runtime issue where `ja2.exe` started under the debugger but showed no visible graphics and stopped logging after `Initializing Video Manager`.
  - Confirmed the process was getting through early SGP startup and stalling in or around `InitializeVideoManager`.
  - Hardened `Standard Gaming Platform/video.c` for modern debugger launches:
    - windowed startup now creates a normal visible overlapped window instead of a bare popup
    - `ShowWindow()` now falls back to `SW_SHOWNORMAL` if the launcher passes `0`
    - added extra `FastDebugMsg()` checkpoints through early DirectDraw setup
  - Hardened `ja2/Build/Intro.c`:
    - intro auto-skips if `smackw32.dll` is missing
    - `NoIntro.txt` is accepted from either the current working directory or the old parent-directory location
  - Updated `vs2022/JA2.vcxproj` so VS2022 launches the debugger with the repository root as the working directory. This is required for runtime access to `Data\\...` assets.
  - Rebuild was blocked once by a locked output binary: `vs2022/bin/Debug/ja2.exe` was still running during link.
  - Instrumented `InitializeVideoManager()` in `Standard Gaming Platform/video.c` to localize startup hangs through:
    - DirectDraw object creation
    - surface creation and locking
    - mouse surface setup
    - RGB distribution setup
  - Fixed a modern DirectDraw/windowed-mode compatibility issue in `GetRGBDistribution()`:
    - modern Windows reported a `32`-bit `R=00ff0000 G=0000ff00 B=000000ff` layout
    - legacy JA2 code assumed a 16-bit primary format and would die in old mask logic
    - the VS2022 path now falls back to JA2-friendly `RGB 565` defaults when the surface format is not 16-bit-compatible
  - Confirmed startup then progressed through:
    - `Initializing Video Object Manager`
    - `Initializing Video Surface Manager`
    - `Initializing the Font Manager`
    - `Initializing Sound Manager`
  - Identified `mss32.dll` incompatibility:
    - the Steam-provided `mss32.dll` loads, but is missing old Miles procedures expected by this source build
    - this raised a delay-load exception `0xC06D007F` during sound startup
  - Hardened `Standard Gaming Platform/soundman.c` so Miles startup is non-fatal:
    - startup exceptions are caught
    - the game continues with sound disabled instead of aborting
  - Confirmed startup then reached `Initializing Game Manager`.
  - Identified the current remaining runtime blocker as missing game assets:
    - this repository contains source only
    - there is no top-level `Data` directory and no `.slf` archives in the repo
    - runtime failed with `Cannot init FONT file FONTS\\LARGEFONT1.sti`
  - Confirmed the asset expectation path:
    - `ja2/Build/JA2 Splash.c` changes to `<repo root>\\Data`
    - `InitializeFileDatabase()` mounts the archives listed in `Standard Gaming Platform/Ja2 Libs.c`
    - without the original JA2 `Data` folder, archive-backed assets like fonts cannot load

### 2026-05-16

- Scanned the repository root and identified the two primary source trees.
- Confirmed the project is organized as a Visual C++ 6.0 workspace.
- Confirmed runtime entry begins in `Standard Gaming Platform/sgp.c`.
- Confirmed JA2 startup flow continues through `ja2/Build/Init.c`.
- Confirmed screen/state dispatch is centered in `ja2/Build/gameloop.c` and `ja2/Build/SCREENS.C`.
- Mapped the major subsystem directories: Tactical, TacticalAI, Strategic, TileEngine, Laptop, Editor, Utils.
- Added this context file to preserve findings for future work.

### 2026-05-17

- Inspected the legacy VC6 project settings in `ja2/Build/JA2.dsp` and `Standard Gaming Platform/Standard Gaming Platform.dsp`.
- Confirmed the environment has Visual Studio 2022 Build Tools installed.
- Confirmed `cmake` is not present in the current environment, so a native MSBuild/VS2022 solution is the more direct modernization path here.
- Added a first-pass VS2022 solution and project under `vs2022/`.
- The VS2022 project intentionally builds the codebase as a single Win32 application target using modern MSBuild globs rather than mirroring the original VC6 project split.
- Fixed several source/toolchain compatibility issues required by MSVC 2022:
  - corrected old absolute include paths in `Standard Gaming Platform`
  - removed obsolete `iostream.h` includes
  - fixed `screenids.h` enum typedef for modern C/C++
  - fixed `Quantize` C++ declarations/scope issues
  - changed the resource script to use `winres.h`
- Added compatibility defines for legacy CRT and Windows SDK behavior:
  - `_CRT_NON_CONFORMING_SWPRINTFS`
  - `_CRT_NON_CONFORMING_WCSTOK`
  - `WINDOWS_IGNORE_PACKING_MISMATCH`
- Excluded a few legacy translation units from the VS2022 target where the original codebase already treated them as embedded/sidecar implementation rather than independent objects:
  - `Standard Gaming Platform/Compression.c`
  - `Standard Gaming Platform/Ja2 Libs.c`
  - `ja2/Build/Utils/Win Util.c`
- Verified that `vs2022/JA2.sln` builds successfully as `Debug|Win32`.
- Verified output executable path: `vs2022/bin/Debug/ja2.exe`

## Suggested Next Exploration

- Trace the exact startup path from `WinMain` to first playable tactical/map screen.
- Identify data file formats and where assets are loaded.
- Evaluate what would be required to build under a modern Visual Studio toolchain.
- Map save/load boundaries and core global state structures.
