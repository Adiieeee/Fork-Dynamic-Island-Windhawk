# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-file [Windhawk](https://windhawk.net) mod, `mod.wh.cpp` (~16.9k lines): a Dynamic Island–style pill overlay for Windows, rendered with Direct2D/DirectWrite. It is a fork of [devcode90/Dynamic-Island-for-Windows](https://github.com/devcode90/Dynamic-Island-for-Windows).

Repository files:

| File | Purpose |
|---|---|
| `mod.wh.cpp` | the whole mod (code, in-file readme, settings schema) |
| `README.md` | GitHub readme: what the mod does, what this fork adds, install, credits |
| `DOCUMENTATION.md` | functional reference: expected behavior of every feature, where its code lives, Windows APIs used, repair guide, fork changelog |
| `Demo.gif` | demo shown at the top of `README.md` |
| `LICENSE` | MIT |

There is no build system, test suite, linter, or package manager. The project is tracked in git.

**Read `DOCUMENTATION.md` before changing a feature**: it describes the target behavior that a change must preserve, and is the place to look up function names for each feature.

## Building / running

- The file is compiled **by Windhawk**, not by a local toolchain. Open it in Windhawk's mod editor (Create new mod → paste/replace source) and use Compile / Enable. Compiler flags and link libs come from the `@compilerOptions` header line (`-ld2d1 -ldwrite -lwinhttp -lpdh -lwbemuuid ...`); add new libs there.
- Local syntax check (no binary) with Windhawk's bundled clang; run it after every change:

  ```bash
  "/c/Program Files/Windhawk/Compiler/bin/clang++.exe" -fsyntax-only -Wall -std=c++23 \
    -target x86_64-w64-mingw32 -DWH_MOD -DUNICODE -D_UNICODE \
    -DWINVER=0x0A00 -D_WIN32_WINNT=0x0A00 \
    '-DWH_MOD_ID=L"dynamic-island-for-windows-fork"' '-DWH_MOD_VERSION=L"1.7.14"' \
    -include windhawk_api.h -x c++ mod.wh.cpp
  ```

- Windhawk logs via `Wh_Log`; view them in the editor's log pane. There is no way to run the mod outside Windhawk.
- The file must stay one self-contained `.wh.cpp`: Windhawk mods are a single source file.

## File layout (top to bottom)

Line numbers drift; search for the names instead.

1. `==WindhawkMod==` header (id, version, `@include windhawk.exe`, compiler options), then the Markdown readme (`==WindhawkModReadme==`, includes the Credits list) and the YAML settings schema (`==WindhawkModSettings==`). **Adding a setting means editing three places:** the YAML schema, `struct Settings` (~line 1004), and `LoadSettings()` (~line 2215).
2. Includes, with `__has_include` guards that set `DYNAMIC_ISLAND_HAS_USER_NOTIFICATION_LISTENER` / `DYNAMIC_ISLAND_HAS_BLUETOOTH_WATCHER` / `DYNAMIC_ISLAND_HAS_GEOLOCATION`. Features depending on newer WinRT headers compile to stubs when the SDK lacks them, so keep both `#if` branches working.
3. Everything lives in one anonymous `namespace` (~line 528): layout namespaces (`MediaLayout`, `FileTrayLayout`, `WeatherLayout`, `HardwareLayout`, `IdleStripLayout`), enums/snapshot structs, `Settings`, `SharedState`, `SpringValue`, localization, helpers, worker threads, the `Renderer` class (~9281–14367), input hooks, `OverlayWndProc` (~14762), `RenderThreadProc` (~15452).
4. Tail: `WhTool_ModInit/Uninit/SettingsChanged` hold the real mod logic; `Wh_ModInit`, `Wh_ModAfterInit`, `Wh_ModSettingsChanged`, `Wh_ModUninit` are the Windhawk entry points.

## Architecture

**Tool-mod process model.** Windhawk injects into `windhawk.exe`, but the island must not run inside it. `Wh_ModInit` in the injected copy sets `g_isToolModProcessLauncher`, and `Wh_ModAfterInit` re-launches `windhawk.exe -tool-mod <WH_MOD_ID>` via `CreateProcessInternalW`. In that child (guarded by a named mutex for single instance) `Wh_ModInit` calls `WhTool_ModInit` and hooks the process entry point to `EntryPoint_Hook`. The launcher copy does nothing else; its `Wh_ModUninit` is a no-op for the launcher, and the tool process calls `ExitProcess(0)` after `WhTool_ModUninit`. Don't put overlay logic in the `Wh_*` functions directly; put it in `WhTool_*`.

**Threads** (all started in `StartThreads`, stopped via `g_stopEvent`):
- `RenderThreadProc`: creates the overlay window, owns the `Renderer`, runs the frame loop (target FPS from settings, event-wait sleeping, parks completely when hidden for ~0 CPU).
- Producers: `MediaThreadProc` (SMTC / WinRT media sessions), `AudioThreadProc` (WASAPI loopback → `WaveformAnalyzer`), `LyricsThreadProc` (LRCLIB), `ArtworkThreadProc` (iTunes cover fallback), `WeatherThreadProc` (WinHTTP → wttr.in, Open-Meteo fallback, optional Windows location), `VolumeWatcherThreadProc` (Core Audio volume callbacks), `BrightnessThreadProc` / `BrightnessOsdThreadProc` (WMI brightness events, hiding the Windows brightness flyout), `NotificationThreadProc`, `BluetoothThreadProc`, plus the DND WNF subscription, clipboard listener, battery, and PDH-based hardware counters.
- Input: `KeyboardThreadProc` / `MouseThreadProc` run `WH_KEYBOARD_LL` / `WH_MOUSE_LL` hooks (Caps/Num Lock, volume keys swallowed to suppress the Windows OSD, brightness-key learning, mouse wake via `WM_APP_MOUSE_WAKE`). Hook procs must stay light: post a message, no locks, no COM.

**Shared state.** Producers write snapshot structs into the global `g_state` (`SharedState`) under `g_stateMutex`; the renderer reads them. `g_settings` is guarded by `g_settingsMutex`: always read via `GetSettingsCopy()` once per frame (never hold the live struct). `g_settingsMutex` is a strict leaf lock: never acquire another mutex while holding it (see the comment above its definition).

**Cross-thread window operations.** `RegisterHotKey` and backdrop changes must run on the thread that owns the window. `LoadSettings` runs on Windhawk's settings thread, so it posts `WM_APP_APPLY_HOTKEY` / `WM_APP_APPLY_BACKDROP` / `WM_APP_LAYOUT_CHANGED` to the overlay window, handled in `OverlayWndProc`. Follow this pattern for any new window-affecting setting.

**Rendering.** `IslandKind` enumerates what the pill shows (Idle, Media, Clipboard, Notification, Volume, Brightness, …). While media is playing, other alerts are drawn inline inside the media pill (`ChooseInlineAlert`, `BuildInlineAlertContent`) instead of as a second pill. `Renderer` draws every surface with `Draw*` methods; sizes and transitions are animated by `SpringValue` springs (animation style/speed from settings). Minimize mode and fullscreen video both render the island as a thin line (`DrawMinimizedLine`, `edgeSpring`). `ThemePalette` / `ThemePreset` define themes; hex colors accept `#RRGGBBAA`. Backdrop blur/acrylic uses undocumented `SetWindowCompositionAttribute` (`Backdrop` namespace).

**Layout/hit-test coupling.** Drawing and mouse hit-testing share constants in the `*Layout` namespaces (e.g. `MediaLayout`, with `MediaContentFromClient` in the wndproc). When changing a dashboard's geometry, change the constants, not separate numbers in `Draw*` and `OverlayWndProc`, or clicks drift from visuals. The one exception is the scrubber ends while the player-switch arrows show, which `DrawMedia` publishes through `g_scrubInsetLeft` / `g_scrubInsetRight`.

**Localization.** `UiLanguage` has 12 languages (follows Windows display language or an explicit setting through `g_uiLanguage`); new user-visible strings need entries for every language.

## Conventions worth knowing

- Right-click menu choices (theme, shape, pin) are intentionally per-session and re-applied from settings on restart. Persistent configuration belongs in the settings schema; a few toggles (lyrics, minimize, DISK drive) persist via `Wh_SetIntValue`.
- Keep the CPU-saving behavior: park when hidden, sample expensive counters (GPU, CPU clock, GPU temperature) only while the Hardware tab is visible, capture audio only while the waveform is visible.
- Windhawk identifies the mod by `@id` and `@version`. For a user-visible change, bump `@version` in the header, the version in `README.md`, the version in the syntax-check command in `DOCUMENTATION.md` and above, and add a row to the changelog in `DOCUMENTATION.md` §16.
- When a feature's behavior or code location changes, update its section in `DOCUMENTATION.md` (and `README.md` if it is user-facing).
- Per the readme's contributing rules, don't remove other contributors' features without explaining why, and add yourself to the Credits section (in the in-file readme and `README.md`).
