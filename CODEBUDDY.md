# CODEBUDDY.md

This file provides guidance to CodeBuddy Code when working with code in this repository.

## Project Overview

Timelapse is a MapleStory v83 Trainer — a C++/CLI DLL that is injected into the MapleStory.exe process. It provides a WinForms GUI for memory manipulation, code cave hooks, packet sending, and botting features. The target is a 32-bit (x86) Windows process.

## Build

- **Solution**: `Timelapse.sln` — Visual Studio 2022 (Platform Toolset v143)
- **Project type**: C++/CLI (`/clr` enabled), outputs a **DLL** (Win32/Debug|Release) or EXE (x64 configs)
- **Target platform**: Win32 (x86) is the active configuration; x64 configs exist but are not the primary target
- **Language standard**: `stdcpp20` (C++/CLI does not support beyond C++20)
- **Windows SDK**: 10.0.22621.0
- **Dependencies**: Microsoft Detours (`detours.h` / `detours.lib` bundled in repo)
- **Build command (VS)**: Open `Timelapse.sln` in Visual Studio, select **Debug|Win32** or **Release|Win32**, and build.
- **Build command (CLI)**:
  ```
  MSBuild Timelapse.sln -p:Configuration=Debug -p:Platform=x86 -t:Build
  ```
  Output DLL goes to `Debug/Timelapse.dll` or `Release/Timelapse.dll`.

### Migrating from VS2017

The project was originally built with VS2017 (v141 / SDK 10.0.17763.0 / `stdcpplatest`). The following changes were made to support VS2022:
- `PlatformToolset` v141 → v143
- `WindowsTargetPlatformVersion` 10.0.17763.0 → 10.0.22621.0
- `LanguageStandard` stdcpplatest → stdcpp20
- `cliext::priority_queue` replaced with custom `ManagedPriorityQueue<T>` (managed binary heap) — `cliext` is incompatible with v143
- `cliext::vector` replaced with `System::Collections::Generic::List<T>` — same reason
- `Timelapse.rc` icon path fixed from hardcoded absolute path to relative `..\\Timelapse.ico`

## Architecture

### Injection & Entry Point

`DllMain` (in `MainForm.cpp`) handles `DLL_PROCESS_ATTACH` by spawning a thread that runs `Main()`, which creates the `MainForm` WinForms instance. The DLL is injected via a separate injector (e.g., Extreme Injector v3.7.2 with Stealth Inject).

### Managed/Unmanaged Boundary

The codebase mixes native C++ and C++/CLI extensively. Critical boundaries are marked with `#pragma managed` / `#pragma unmanaged`:
- **Unmanaged code**: Assembly code caves (`CodeCave` macro), `ReadMultiPointer*` functions, and low-level memory operations
- **Managed code**: WinForms UI, macro system, pointer display functions (`PointerFuncs` namespace)

### Core Modules

| File | Purpose |
|------|---------|
| `MainForm.h/cpp` | WinForms UI definition and all event handlers (~70% of the codebase). Very large single file. |
| `Addresses.h` | All hardcoded memory addresses for MapleStory v83 — hack addresses, codecave addresses, hook addresses, pointer bases and offsets |
| `Hooks.h` | Microsoft Detours-based function hooking (`SetHook`), assembly code caves (inline `__asm`), and the `Assembly` namespace for shared state between hooks and UI |
| `Memory.h` | Low-level memory utilities: `ReadPointer`, `WritePointer`, `WriteMemory`, `Jump`, `MakePageWritable`, ZtlSecureFuse readers |
| `MapleFunctions.h` | Game-specific helpers: `Teleport()`, `GetMSWindowHandle()`, `PointerFuncs` namespace (reads HP/MP/EXP/position/etc.), `HelperFuncs` namespace (game state checks) |
| `Macro.h` | Key macro system with priority queue — `Macro` class (timed key sends), `ManagedPriorityQueue<T>` (custom binary heap replacing `cliext`), `PriorityQueue` worker thread, `KeyMacro` struct (PostMessage-based input) |
| `Packet.h/cpp` | Packet send/recv functions (`SendPacket`, `RecvPacket`) and packet builder helpers (`writeByte`, `writeInt`, etc.) |
| `Keyboard.h/cpp` | Low-level keyboard input simulation via `SendInput` |
| `Mouse.h/cpp` | Low-level mouse input simulation via `SendInput` |
| `Structs.h` | Data structures: `COutPacket`, `CInPacket`, `SpawnControlData`, `PortalData`, `MapData`, `MapPath` |
| `Settings.h/cpp` | XML-based save/load of form control state |
| `Log.h/cpp` | Logging to file and console |
| `Inventory.h/cpp` | Inventory slot structures (partially implemented) |
| `GeneralFunctions.h` | String conversion utilities (`System::String^` <-> `std::string`, ASCII-to-hex) |
| `HelperFunctions.h` | Duplicate of some `HelperFuncs` from `MapleFunctions.h` (stale) |
| `Assembly.h` | Near-duplicate of `Hooks.h` (appears to be an older version) |
| `resource.h` | Resource IDs for icon and embedded text resources (ItemsList, MobsList, MapsList) |

### Key Patterns

**Hack toggling**: Each hack is a checkbox event handler that writes bytes to a hardcoded address — either `WriteMemory()` for simple byte patches (NOP, JMP, JNE flips) or `Jump()` for codecave hooks. The original bytes are hardcoded as the "off" state.

**Code caves**: Defined via the `CodeCave(name) { ... } EndCodeCave` macro which creates `__declspec(naked)` functions containing inline assembly. They intercept game functions to read/modify data (e.g., `StatHook` captures HP/MP/EXP values, `ItemVacHook` redirects item pickup coordinates).

**Pointer reading**: Multi-level pointer dereferencing via `ReadPointer(base, offset)` and `ReadMultiPointerSigned(base, level, ...)`. Game data is accessed through static base addresses + offsets defined in `Addresses.h`.

**Macro system**: `Macro` objects use `System::Threading::Timer` to push `KeyMacro` items into a `ManagedPriorityQueue` (managed binary heap, replacing `cliext::priority_queue`), processed by a dedicated worker thread that sends keystrokes via `PostMessage(WM_KEYDOWN)`.

**Global state**: `GlobalRefs` (managed) and `GlobalVars` (unmanaged) hold shared state. `MainForm::TheInstance` provides static access to the form from anywhere.

### Embedded Resources

Item IDs, mob IDs, and map data are stored as embedded TEXT resources (`ItemsList`, `MobsList`, `MapsList` in `Timelapse.rc`) and loaded at runtime via `FindResource`/`LoadResource`.

## Known Issues (from README)

- `HelperFunctions.h` includes a non-existent `"Functions.h"` and duplicates code from `MapleFunctions.h`
- `Assembly.h` is a near-duplicate of `Hooks.h`
- `cliext` (STL/CLR) has been replaced — do not reintroduce `#include <cliext/...>` as it is incompatible with v143 toolset
- Hotkey queue is unresponsive when attack/loot macros spam input
- FMA and Kami vacs are unstable on large maps or with flying mobs
- AutoCS doesn't work as intended (blue box popup issue)
- Several features are TODO/incomplete (Map Rusher, Auto Sell, various hacks)
