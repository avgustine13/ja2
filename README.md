# Jagged Alliance 2 Source Overview

This repository contains a legacy Jagged Alliance 2 source tree plus the shared `Standard Gaming Platform` runtime layer it depends on.

The codebase is organized as a Microsoft Visual C++ 6.0 workspace centered on `ja2/Build/JA2.dsw`. The original packaging notes remain in `readme.txt`; this README is a modern orientation document for navigating the source.

## Top-Level Structure

- `ja2/Build`
  - Main game code and VC6 workspace/projects.
- `Standard Gaming Platform`
  - Shared engine/platform code used by the game.
- `readme.txt`
  - Original historical build instructions.
- `SFI Source Code license agreement.txt`
  - Included license text.
- `JA2_AI_CONTEXT.md`
  - Ongoing working notes and architectural findings for this repository.

## How The Runtime Is Structured

The executable starts in `Standard Gaming Platform/sgp.c`, which provides `WinMain`, Windows message handling, and shared runtime bootstrap. From there, JA2-specific initialization happens in `ja2/Build/Init.c`.

The game itself is organized around a screen/state model:

- `ja2/Build/gameloop.c`
  - Main frame loop and screen transition handling.
- `ja2/Build/SCREENS.C`
  - Table of screen initialize/handle/shutdown callbacks.

This is a typical late-1990s architecture: subsystems are broad, stateful, and coordinated through globals plus central managers.

## Major Subsystems

### Standard Gaming Platform

Low-level runtime support:

- Windows bootstrap
- input and mouse systems
- video surfaces and blitters
- fonts and text rendering
- file/database helpers
- sound wrappers
- memory/debug helpers

### Tactical

In-sector gameplay and combat logic:

- soldiers
- weapons and items
- turn-based interactions
- LOS/FOV/pathing
- UI panels/cursors
- doors, corpses, militia, vehicles

### TacticalAI

Combat and behavior logic for AI-controlled actors:

- target/action selection
- movement
- attacks
- knowledge and reaction logic
- medical and panic behavior

### Strategic

Campaign/overworld simulation:

- map screen
- assignments and contracts
- quests
- queen AI
- strategic movement
- town loyalty and mines
- autoresolve and pre-battle systems

### TileEngine

World/map representation and rendering support:

- world data
- structures
- lighting
- fog of war
- tile caches and animation
- exit grids
- radar and overhead map

### Laptop

The in-game laptop and campaign service UI:

- AIM
- MERC
- Bobby Ray
- IMP character creation
- finances, email, files, personnel
- florist, funeral, insurance

### Editor

Integrated editing tools:

- terrain/building placement
- item editing
- smoothing tools
- sector/map summary
- editor UI/callbacks/undo support

### Utils

Shared helpers and support code:

- text utilities
- message boxes
- music/sound control
- event handling
- encrypted file support
- multilingual text tables

## Source Characteristics

- Mostly C, with a small amount of C++.
- Large use of global state.
- Strong coupling between engine, UI, and gameplay systems.
- Legacy file naming conventions, including spaces in filenames.
- Original build assumptions target VC6-era Windows and libraries.

## Build Notes

This is not a modernized build system. The original instructions expect the source tree to be copied into fixed `C:\` locations and built with Visual C++ 6.0.

If the goal is to work with this code today, likely next steps are:

1. Document the actual dependency set and path assumptions from the `.dsp` files.
2. Decide whether to preserve VC6 compatibility or migrate to a modern toolchain.
3. Isolate third-party/Windows-specific dependencies from gameplay code.

## VS2022 Build

A first-pass Visual Studio 2022 solution now lives under `vs2022/`.

- Solution: `vs2022/JA2.sln`
- Project: `vs2022/JA2.vcxproj`

Current design choices:

- Builds as a single `Win32` application target instead of recreating the old VC6 multi-project workspace.
- Uses the existing source tree directly with MSBuild file globs.
- Links against the bundled legacy libraries in `Standard Gaming Platform`.
- Preserves several legacy compatibility assumptions via project defines rather than rewriting broad areas of source.

Build result in the current environment:

- `Debug|Win32` builds successfully with VS2022 Build Tools.
- Output executable: `vs2022/bin/Debug/ja2.exe`

Command used:

```bat
call "C:\Program Files (x86)\Microsoft Visual Studio\2022\BuildTools\Common7\Tools\VsDevCmd.bat" -arch=x86
msbuild G:\_PROJECTS\_ja2\ja2\vs2022\JA2.sln /t:Build /p:Configuration=Debug /p:Platform=Win32 /m
```

This is a modernization scaffold, not a guarantee of a clean build on the first try. The next step after opening or building the solution is to fix compiler and linker incompatibilities exposed by MSVC 2022.

## Runtime Asset Requirements

The source tree alone is not enough to boot the game. JA2 expects the original game assets to be present at runtime under a top-level `Data` directory beside the repository root:

- expected path: `G:\_PROJECTS\_ja2\ja2\Data`

At minimum, this directory needs the original JA2 `.slf` archives referenced by the file database layer, including:

- `Data.slf`
- `Ambient.slf`
- `Anims.slf`
- `BattleSnds.slf`
- `BigItems.slf`
- `BinaryData.slf`
- `Cursors.slf`
- `Faces.slf`
- `Fonts.slf`
- `Interface.slf`
- `Laptop.slf`
- `Maps.slf`
- `MercEdt.slf`
- `Music.slf`
- `Npc_Speech.slf`
- `NpcData.slf`
- `RadarMaps.slf`
- `Sounds.slf`
- `Speech.slf`
- `TileSets.slf`
- `LoadScreens.slf`
- `Intro.slf`

These archives are listed in `Standard Gaming Platform/Ja2 Libs.c` and are mounted during startup from `ja2/Build/JA2 Splash.c`.

If `Data` is missing, startup will fail later with asset errors such as:

- `Cannot init FONT file FONTS\LARGEFONT1.sti`

In practice, copy the original JA2 installation `Data` directory into the repository root before running `vs2022/bin/Debug/ja2.exe`.

## Current VS2022 Runtime Status

The current bring-up work has resolved several modern Windows/VS2022 blockers:

- windowed DirectDraw startup now works under the debugger
- modern 32-bit desktop pixel format no longer breaks JA2's old 16-bit RGB assumptions
- missing `smackw32.dll` no longer blocks startup intros
- incompatible modern `mss32.dll` startup now falls back to "sound disabled" instead of crashing

The current remaining runtime dependency is the original JA2 asset set under `Data`.

## Working Notes

Use `JA2_AI_CONTEXT.md` as the running project notebook for discoveries, architectural mapping, and refactor/build planning.
