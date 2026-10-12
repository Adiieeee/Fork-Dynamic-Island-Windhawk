# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-file [Windhawk](https://windhawk.net) mod, `mod.wh.cpp` (~13k lines): a Dynamic Island–style pill overlay for Windows, rendered with Direct2D/DirectWrite. There is no repo tooling: no git, no build system, no tests, no linter, no package manager.

## Building / running

- The file is compiled **by Windhawk**, not by a local toolchain. Open it in Windhawk's mod editor (Create new mod → paste/replace source) and use Compile / Enable. Compiler flags and link libs come from the `@compilerOptions` header line (`-ld2d1 -ldwrite -lwinhttp -lpdh ...`); add new libs there.
- Windhawk logs via `Wh_Log`; view them in the editor's log pane. There is no way to unit-test or run it outside Windhawk.
- The file must stay one self-contained `.wh.cpp`: Windhawk mods are a single source file.

## File layout (top to bottom)

1. `==WindhawkMod==` header (id, version, `@include windhawk.exe`, compiler options), then the Markdown readme (`==WindhawkModReadme==`, includes the Credits list that contributors are asked to update) and the YAML settings schema (`==WindhawkModSettings==`). **Adding a setting means editing three places:** the YAML schema, the `Settings` struct (~line 897), and `LoadSettings()` (~line 1985).
2. Includes, with `__has_include` guards that set `DYNAMIC_ISLAND_HAS_USER_NOTIFICATION_LISTENER` / `DYNAMIC_ISLAND_HAS_BLUETOOTH_WATCHER`. Features depending on newer WinRT headers compile to stubs when the SDK lacks them, so keep both `#if` branches working.
3. Everything lives in one anonymous `namespace`: layout namespaces (`MediaLayout`, `FileTrayLayout`, `WeatherLayout`, `HardwareLayout`, `IdleStripLayout`, `GameOverlayLayout`), enums/snapshot structs, `Settings`, `SharedState`, `SpringValue`, localization, helpers, worker threads, the `Renderer` class (~6685–10970), input hooks, `OverlayWndProc`, `RenderThreadProc`.
4. Tail: `WhTool_ModInit/Uninit/SettingsChanged` hold the real mod logic; `Wh_ModInit`, `Wh_ModAfterInit`, `Wh_ModSettingsChanged`, `Wh_ModUninit` are the Windhawk entry points.

## Architecture

**Tool-mod process model.** Windhawk injects into `windhawk.exe`, but the island must not run inside it. `Wh_ModInit` in the injected copy sets `g_isToolModProcessLauncher`, and `Wh_ModAfterInit` re-launches `windhawk.exe -tool-mod <WH_MOD_ID>` via `CreateProcessInternalW`. In that child (guarded by a named mutex for single instance) `Wh_ModInit` calls `WhTool_ModInit` and hooks the process entry point to `EntryPoint_Hook`. The launcher copy does nothing else; its `Wh_ModUninit` is a no-op for the launcher, and the tool process calls `ExitProcess(0)` after `WhTool_ModUninit`. Don't put overlay logic in the `Wh_*` functions directly; put it in `WhTool_*`.

**Threads** (all started in `StartThreads`, stopped via `g_stopEvent`):
- `RenderThreadProc`: creates the overlay window, owns the `Renderer`, runs the frame loop (target FPS from settings, event-wait sleeping, parks completely when hidden for ~0 CPU).
- Producers: `MediaThreadProc` (SMTC / WinRT media sessions), `AudioThreadProc` (WASAPI volume, waveform, mic/cam privacy), `WeatherThreadProc` (WinHTTP → wttr.in), `NotificationThreadProc`, `BluetoothThreadProc`, plus DND subscription, clipboard, battery, PDH-based hardware counters.
- Input: `KeyboardThreadProc` / `MouseThreadProc` run `WH_KEYBOARD_LL` / `WH_MOUSE_LL` hooks (hotkey, scroll wheel, mouse wake via `WM_APP_MOUSE_WAKE`).

**Shared state.** Producers write snapshot structs into the global `g_state` (`SharedState`) under `g_stateMutex`; the renderer reads them. `g_settings` is guarded by `g_settingsMutex`: always read via `GetSettingsCopy()` once per frame (never hold the live struct), and take `g_settingsMutex` and `g_stateMutex` consistently (see the comment near line 1232 about deadlock-freedom).

**Cross-thread window operations.** `RegisterHotKey` and backdrop changes must run on the thread that owns the window. `LoadSettings` runs on Windhawk's settings thread, so it posts `WM_APP_APPLY_HOTKEY` / `WM_APP_APPLY_BACKDROP` / `WM_APP_LAYOUT_CHANGED` to the overlay window, handled in `OverlayWndProc`. Follow this pattern for any new window-affecting setting.

**Rendering.** `IslandKind` enumerates what the pill shows (Idle, Media, Progress, Clipboard, Notification, Volume, …, Split). `Renderer` draws every surface with `Draw*` methods; sizes and transitions are animated by `SpringValue` springs (animation style/speed from settings). `ThemePalette` / `ThemePreset` define themes; hex colors accept `#RRGGBBAA`. Backdrop blur/acrylic uses undocumented `SetWindowCompositionAttribute` (`Backdrop` namespace).

**Layout/hit-test coupling.** Drawing and mouse hit-testing share constants in the `*Layout` namespaces (e.g. `MediaLayout`, with `MediaContentFromClient` in the wndproc). When changing a dashboard's geometry, change the constants, not separate numbers in `Draw*` and `OverlayWndProc`, or clicks drift from visuals.

**Localization.** `UiLanguage` has 12 languages (follows Windows display language or an explicit setting through `g_uiLanguage`); new user-visible strings need entries for every language.

## Conventions worth knowing

- Right-click menu choices (theme, shape, pin) are intentionally per-session and re-applied from settings on restart. Persistent configuration belongs in the settings schema.
- Windhawk bumps/identifies the mod by `@id` and `@version`; bump `@version` when making a user-visible change.
- Per the readme's contributing rules, don't remove other contributors' features and add yourself to the Credits section (in the in-file readme and the GitHub `README.md`, which is not in this folder).
