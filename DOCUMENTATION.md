# Dynamic Island for Windows (Fork) Documentation

Complete functional reference for `mod.wh.cpp`. Purpose: if the mod ever breaks (for example after a Windows update), this document describes **how it is supposed to behave**, **where the code lives**, and **which Windows APIs it uses**, so a fix can restore exactly the same functionality.

- **Code version when this document was written:** `1.7.7` (header `@version`), about 16,500 lines.
- **Line numbers** below apply to this version and will drift as the code changes. **Always search for the function name** (e.g. `grep -n "DrawMinimizedLine" mod.wh.cpp`). Function names are the primary reference; line numbers are only a starting hint.
- General project conventions (tool-mod process, threading, localization) are also summarized in `CLAUDE.md`.

---

## Table of contents

1. [Overview & building](#1-overview--building)
2. [Architecture](#2-architecture)
3. [Render loop (the heart of the mod)](#3-render-loop-the-heart-of-the-mod)
4. [Feature: Media (music pill & dashboard)](#4-feature-media)
5. [Feature: Lyrics (LRCLIB)](#5-feature-lyrics-lrclib)
6. [Feature: Album art & iTunes fallback](#6-feature-album-art--itunes-fallback)
7. [Feature: Minimize (thin line)](#7-feature-minimize-thin-line)
8. [Feature: Transient & inline notifications](#8-feature-transient--inline-notifications)
9. [Feature: Volume (instant + Windows OSD hidden)](#9-feature-volume)
   - [Brightness (since 1.7.6)](#91-brightness-since-176)
10. [Feature: Dashboard tabs (Date & Weather, Hardware, File Tray, Activity)](#10-feature-dashboard-tabs)
11. [Feature: Idle strip, Privacy, Timer](#11-feature-idle-strip-privacy-timer)
12. [Feature: Position, theme, backdrop, auto-hide, hotkey, menu](#12-feature-position-theme-backdrop-auto-hide-hotkey-menu)
13. [Settings (Mod Settings) & stored values](#13-settings--stored-values)
14. [External services (internet)](#14-external-services)
15. [Repair guide after Windows updates](#15-repair-guide-after-windows-updates)
16. [Fork changelog](#16-fork-changelog)

---

## 1. Overview & building

**What it is:** a single-file [Windhawk](https://windhawk.net) mod that shows an iPhone-style "Dynamic Island" overlay on Windows. The overlay is a layered window (per-pixel alpha) drawn with **Direct2D + DirectWrite**. It reacts to playing media, lyrics, notifications, volume, clipboard, battery, devices, and more.

**File structure (top to bottom):**

| Section | Lines (±) | Contents |
|---|---|---|
| `==WindhawkMod==` | 1–11 | id, version, `@include windhawk.exe`, `@compilerOptions` (linked libraries: `-lole32 -ld2d1 -ldwrite -lwinhttp -lpdh ...`) |
| `==WindhawkModReadme==` | 13–90 | readme shown in Windhawk |
| `==WindhawkModSettings==` | 92–470 | Mod Settings YAML schema (see §13) |
| Includes + WinRT guards | 470–537 | `DYNAMIC_ISLAND_HAS_USER_NOTIFICATION_LISTENER`, `DYNAMIC_ISLAND_HAS_BLUETOOTH_WATCHER` |
| `namespace {` … | 538–end | all of the logic |
| Windhawk entry points | 14716–end | `WhTool_*` and `Wh_*` |

**Building:**
- Compiled **by Windhawk** (mod editor → Compile). There is no local build, test suite, or linter.
- New libraries must be added to the `@compilerOptions` line.
- A local syntax check (without producing a binary) can use Windhawk's bundled clang:

```bash
"/c/Program Files/Windhawk/Compiler/bin/clang++.exe" -fsyntax-only -Wall -std=c++23 \
  -target x86_64-w64-mingw32 -DWH_MOD -DUNICODE -D_UNICODE \
  -DWINVER=0x0A00 -D_WIN32_WINNT=0x0A00 \
  '-DWH_MOD_ID=L"dynamic-island-for-windows-fork"' '-DWH_MOD_VERSION=L"1.7.15"' \
  -include windhawk_api.h -x c++ mod.wh.cpp
```

- **Logs:** `Wh_Log(...)` shows up in the Windhawk editor's log pane. This is the only runtime debugging tool.

---

## 2. Architecture

### 2.1 Process model (tool-mod)
Windhawk injects the mod into `windhawk.exe`, but the island must not run inside that process.

- `Wh_ModInit` (±14765): if this is the launcher process, set `g_isToolModProcessLauncher`. If this is the `-tool-mod <id>` process, create the single-instance mutex, call `WhTool_ModInit`, then hook the entry point.
- `Wh_ModAfterInit` (±14844): the launcher re-launches `windhawk.exe -tool-mod "<WH_MOD_ID>"` via `CreateProcessInternalW`.
- `WhTool_ModInit` (±14716): `LoadSettings()`, read stored values (`LyricsEnabled`, `IslandMinimized`, `HardwareDiskLetter`), then `StartThreads()`.
- `WhTool_ModUninit` (±14735): send `WM_CLOSE`, `StopThreads()`. Afterwards the process calls `ExitProcess(0)`.
- **Rule:** overlay logic belongs in `WhTool_*`, not in `Wh_*`.

### 2.2 Threads
All threads are created in `StartThreads()` (±14636) and stopped by `StopThreads()` (±14666) via `g_stopEvent` (manual-reset), with a join of at most 3 seconds per thread.

| Thread | Function (±line) | Job |
|---|---|---|
| Render | `RenderThreadProc` (13616) | creates the overlay window, owns the `Renderer`, runs the frame loop, and pumps window messages |
| Media | `MediaThreadProc` (4208) | polls SMTC every 1.5 seconds (or immediately via `g_mediaRefreshEvent` when the player is switched): title, artist, album, cover, position, list of all sessions |
| Audio | `AudioThreadProc` (5853) | WASAPI loopback for waveform data (only while `g_audioCaptureNeeded`) |
| Weather | `WeatherThreadProc` (5337) | WinHTTP to wttr.in, `Geolocator` (optional) |
| Lyrics | `LyricsThreadProc` (5573) | LRCLIB for timestamped lyrics |
| Cover fallback | `ArtworkThreadProc` (5759) | iTunes Search when SMTC provides no cover |
| Volume watcher | `VolumeWatcherThreadProc` (6344) | volume callbacks from Windows (event-driven, not polled) |
| Brightness | `BrightnessThreadProc`, `BrightnessOsdThreadProc` | WMI `WmiMonitorBrightnessEvent` events; `EVENT_OBJECT_SHOW` hook on explorer to hide the brightness OSD (§9.1) |
| Notifications | `NotificationThreadProc` (4237) | WinRT `UserNotificationListener` |
| Bluetooth | `BluetoothThreadProc` (4780 / stub 4865) | WinRT `DeviceWatcher` for connected BT devices |
| Keyboard | `KeyboardThreadProc` (12832) | `WH_KEYBOARD_LL` hook: Caps/Num Lock, volume keys, and recording keys without a VK (brightness) |
| Mouse | `MouseThreadProc` (12928) | `WH_MOUSE_LL` hook, only while the island is parked, for unhide-on-hover |

DND does not use its own thread but a WNF subscription (`SubscribeDndNotification`, ±4199).

### 2.3 Shared state
- **`SharedState g_state`** (struct ±1236) is guarded by **`g_stateMutex`**. It contains: `media`, `lyrics`, `clipboard`, `notification`, `volume`, `capsLock`, `timer`, `device`, `bluetoothDevice`, `doNotDisturb`, `battery`, `progress`, `system` (CPU/RAM/GPU/disk/net/volume/privacy; since 1.7.7 also `hwDiskLetter` / `hwDiskFreePercent` / `fixedDriveCount` for the Hardware tab's DISK card), `weather`, `waveform[48]` (ring buffer, written via `waveformWrite`), `fileTrayItems`, and (since 1.7.4) `playingMediaSources`, `mediaSessionSources` (AUMIDs of all SMTC sessions, deduplicated) and `pinnedMediaSource` (the player chosen via the arrows; empty = automatic).
- **`Settings g_settings`** (struct ±945) is guarded by `g_settingsMutex`. The render loop uses `GetSettingsCopy()` (±1303) once per frame.
- **Important atomic globals** (±1290–1340): `g_layoutDirty` (request repaint/reposition), `g_idleTab` (active dashboard tab), `g_hoveredMediaButton` / `g_pressedMediaButton`, `g_scrubbing`, `g_manuallyHidden`, `g_autoHiddenParked`, `g_lyricsEnabled`, `g_lyricsDelayMs`, `g_islandMinimized`, `g_minimizedEdgeFraction`, `g_activityTabVisible`, `g_audioCaptureNeeded`, `g_mediaSessionCount` (number of players, used to show the player-switch arrows), `g_scrubInsetLeft` / `g_scrubInsetRight` (where the scrubber ends are actually drawn), `g_hwDiskLetter` (drive on the Hardware tab's DISK card, since 1.7.7) and `g_hoveredHwDiskArrow` (cursor over the drive-switch arrow).

### 2.4 Window messages (`OverlayWndProc`, ±12972)

| Message | Purpose |
|---|---|
| `WM_CREATE` | `AddClipboardFormatListener`, `DragAcceptFiles` |
| `WM_DROPFILES` | File Tray: store the dropped file paths |
| `WM_APP_APPLY_HOTKEY` / `WM_APP_APPLY_BACKDROP` | apply the hotkey/backdrop on the window-owning thread (posted from `LoadSettings`) |
| `WM_APP_MOUSE_WAKE` | wake from park because of hover |
| `WM_APP_VOLUME_KEY` | apply a volume key swallowed by the hook (`ApplyVolumeKey`) and write a diagnostic log |
| `WM_APP_CAPSLOCK` | record Caps/Num Lock state (checks the `capsLock` setting) |
| `WM_DEVICECHANGE` | USB/COM notification (`DBT_DEVICEARRIVAL` / `DBT_DEVICEREMOVECOMPLETE`) |
| `WM_POWERBROADCAST` | after resume from sleep: refresh weather, then nudge |
| `WM_CLIPBOARDUPDATE` | `CaptureClipboard` (±6889) |
| `WM_SETCURSOR` / `WM_MOUSEMOVE` / `WM_LBUTTONDOWN` / `WM_LBUTTONUP` | hit-testing for media buttons, scrubber, File Tray, weather chips (hover explanation), tab swipe, click to expand |
| `WM_MOUSEWHEEL` | switch dashboard tab (150ms debounce) |
| `WM_MBUTTONUP` | toggle mute (`ToggleEndpointMute`) |
| `WM_LBUTTONDBLCLK` | toggle "Pin expanded" |
| `WM_RBUTTONUP` | right-click menu (`ShowContextMenu`, ±7310) |
| `WM_HOTKEY` | hide/show hotkey (default Ctrl+Alt+D) |

`WM_APP` IDs in use: `+0x442` LAYOUT_CHANGED, `+0x443` NEW_EVENT, `+0x444` CAPSLOCK, `+0x446` MOUSE_WAKE, `+0x447` APPLY_HOTKEY, `+0x448` APPLY_BACKDROP, `+0x449` VOLUME_KEY.

### 2.5 Layout ↔ hit-test coupling
Drawing and mouse hit-testing use **the same constants** in the namespaces `MediaLayout` (±559), `FileTrayLayout` (±644), `WeatherLayout` (right column and chips of the Date & Weather tab, ±685), `IdleStripLayout` (±681), and `HardwareLayout` (the Hardware tab's 2×2 grid and the drive-switch arrow on the DISK card, since 1.7.7; hit-tested via `HardwareDiskArrowAtContentPoint`). Media hit-testing uses `MediaContentFromClient`, which reads the last frame's geometry through a seqlock, then `MediaTransportHitTest` (±1725), `MediaScrubFractionFromContent` (±1767), and `MediaArtHitTest`.

**Exception (1.7.4):** while the player-switch arrows are shown, the scrubber ends follow the measured width of the time text, so they cannot be constants. `DrawMedia` writes their positions to `g_scrubInsetLeft` / `g_scrubInsetRight`, and the scrubber hit-test reads those atomics.

**Rule:** when moving an element, change its constant, not loose numbers in `Draw*`, so click areas don't drift away from the visuals.

---

## 3. Render loop (the heart of the mod)

`RenderThreadProc` (±13616). Order of each iteration:

1. **Process messages** (`PeekMessage`/`DispatchMessage`). `WM_APP_NEW_EVENT` triggers the "nudge" animation (the pill bounces slightly).
2. **Park:**
   - If `g_manuallyHidden`: the window is hidden, then `MsgWaitForMultipleObjects(INFINITE)`, so CPU is about 0%.
   - If `g_autoHiddenParked` (idle auto-hide): same. (Since 1.6.2 fullscreen video no longer parks the island but turns it into a line.)
3. **Snapshot state** (copy `g_state` under the lock), then clear expired notifications.
4. **Choose activities:**
   - `ChooseActivities` (±12699) returns a `primary` and optionally a `secondary` (two pills side by side).
   - `ActivityForKind` (±12595) returns the base size per kind.
   - `ChooseInlineAlert` (±12676) picks the notification that is drawn **inside** the music pill (§8).
   - `g_activityTabVisible` = timer or progress active (§10).
5. **Hover:** the pill rect is computed from `GetWindowRect`, plus padding when it is a minimize line (§7), whether manual minimize or the fullscreen-video line. `isHoverExpanded` follows the Expand-on-hover setting (or click), and is forced to `hover` while minimized or `videoSuppressed`.
6. **Auto-hide / fullscreen video:** `IsForegroundFullscreenVideo` (±3659) is checked every 0.5 seconds **before** `ChooseInlineAlert` (±14183), producing `videoSuppressed` (±14178). During video, the island becomes the minimize line. After that come `isHidden` and `EnsureTopmost` (every 1 second). Details: §12 "Fullscreen video".
7. **System poll every 1 second:** `UpdateSystemSnapshot` (±6124) reads CPU, RAM, disk (the DISK card's selected drive via `SampleHardwareDisk`), and volume level. GPU is only read when the Hardware tab is visible; CPU clock (`GetCpuClockGHz`) and GPU temperature (`GetGpuTemperatureC`) only while the Hardware tab is visible (parameter `includeHwDetails`). Since 1.7.7 the network is not read at all (`needNetStats = false`) because nothing displays it anymore; `GetNetworkUsage` remains in the code.
8. **Target size:**
   - Idle: `MeasureIdleStrip`.
   - Media: 400×184 when expanded (380 before 1.7.4), or the inline notification width.
   - Minimize: 160×8 line.
   - Hidden: 0×0.
9. **Springs:** `widthSpring`, `heightSpring`, `nudgeSpring`, and `edgeSpring` (for minimize). Stiffness and damping follow `AnimationStyle` (Smooth/Default/Bouncy/Snappy) and `AnimationSpeed`.
10. **Need to render?** A frame is rendered if any spring is moving, `g_layoutDirty` is set, hover changed, a continuous animation is running (media playing, marquee, notification), privacy dots are shown, or data changed (art generation, title, volume, battery, etc.). A once-per-second repaint happens for the timer and for clocks showing seconds.
11. **`renderer.Render(...)`** (±7687): create a backing bitmap the size of the pill + padding (`kRenderPadX` 28, `kRenderPadY` 22), `PositionOverlayWindow`, draw, then `UpdateLayeredWindow`.
12. **Pacing:** target FPS from settings (0 = monitor refresh rate). If nothing changed, the loop polls every 16ms.

**Renderer** (`class Renderer`, ±7651): `Initialize`, `Render`, `DrawPill` (±8334, switch per `IslandKind`), `DrawPillSurface`, `DrawSoftShadow`, `DrawPrivacyDots`, `EnsureBrushes` (±8202, theme color tokens), and `EnsureTextFormats` (±7999, fonts and text scale).

---

## 4. Feature: Media

**Data source:** Windows **SMTC** (`GlobalSystemMediaTransportControlsSessionManager`) in `MediaThreadProc` (±3889), polled every 1.5 seconds.
- `SelectActiveMediaSession` (±4143) picks the session the island follows:
  1. the player the user picked via the arrows (`pinnedMediaSource`), as long as its session still exists; if one AUMID has several sessions (browsers), the one that is Playing wins;
  2. `GetCurrentSession()` if it is Playing;
  3. the first session that is Playing;
  4. `GetCurrentSession()`.

  If the selected player is closed, `pinnedMediaSource` is cleared and selection goes back to automatic.
- Data read: `title`, `artist`, `albumTitle`, `sourceAppUserModelId`, `PlaybackStatus`, timeline (`positionTicks`, `endTicks`, `lastUpdatedTicks`), and the thumbnail (cover).
- **VLC fallback:** if VLC reports an empty title, the title is taken from the VLC window caption.
- **Source icon:** `FindMediaSourceIcon` (±3681), with a text badge (e.g. "SP") as a fallback.

**Collapsed pill** (`DrawMedia`, ±10352, "Collapsed UI" branch):
- Left: round album art. Right: waveform (`DrawWaveform`, ±11542) while playing, or static dots while paused.
- Middle: **inline notification** (§8) if there is one; otherwise, for **4.5 seconds after the track changes** (`TrackPeekActive`, `kTrackPeekSeconds`), the **track peek**: title (`boldTextFormat_`) over artist (`smallTextFormat_`, muted), centred, with 0.25 s fade-in and 0.35 s fade-out (`DrawTrackPeek`, since 1.7.14). It applies to every track, with or without lyrics, needs the track to be playing, and widens the pill to `kCollapsedLyricsWidth` through `ActivityForKind`. After the peek come the **lyrics** (§5), which fade in (`kTrackPeekShownKey` marks the hand-over), or, for a track without lyrics, the **beamed note** (§5). With `MediaAutoExpand` on, the 5-second auto-expanded card covers the peek.
- Base width `MediaLayout::kCollapsedWidth` = 150. With lyrics: `kCollapsedLyricsWidth` = 320.

**Waveform (since 1.7.10):** the waveform shows **perceived loudness**, influenced by frequency content and the number of "instruments". It is not a per-bar spectrum.
- **Source:** `AudioThreadProc` opens WASAPI loopback on the default output device, only while `g_audioCaptureNeeded` (Media pill visible and playing). All audio going to the speakers counts, not just the player. `g_waveformAnalyzer.Reset(sampleRate)` is called every time a capture session starts.
- **Analysis** (`WaveformAnalyzer`, audio thread only, no mutex):
  - `PushAudioChunks` mixes all channels down to mono and feeds them into **6 band-pass filters** (RBJ, two stages each) at 80 / 250 / 700 / 2000 / 5000 / 11000 Hz.
  - Bands above 0.45 × sample rate are disabled.
  - Each band's energy is accumulated per **~10 ms** (`sampleRate/100` frames); chunks may span packets.
- **Per-chunk features** (`FinishChunk`):
  - **Loudness:** sum of band energies weighted `0.55, 0.85, 1.0, 1.0, 0.9, 0.7`, roughly matching ear sensitivity, so deep bass and hiss don't dominate. Result in dBFS.
  - **Density:** number of active bands, i.e. within 15 dB (`kWaveformDensitySpanDb`) of the strongest band and above −65 dB. Result is `(active−1)/(bands−1)`, smoothed with an EMA of 0.92. A single tone ≈ 0.2, full orchestra ≈ 1.
- **Mapping** (`Map`):
  - The peak reference `refDb` jumps up to a new peak immediately, then decays 0.03 dB per sample (~3 dB/second).
  - Boost = `clamp(−14 − refDb, 0, 8)` dB. Quietly mastered songs are raised by **at most 8 dB**, so soft passages stay smaller than forte.
  - Level = `((dB + boost) − (−60)) / 50`, raised to the power 1.3, then multiplied by `0.6 + 0.4 × density`.
  - Below −70 dB is treated as silence (0).
  - All `kWaveform*` constants sit right above `WaveformAnalyzer`.
- **Envelope:** `PushWaveformSample(value 0..1)` applies attack 0.7 / release 0.85, then writes to the ring buffer `g_state.waveform[48]` (~480 ms of history).
- **Zero values:** sent for `AUDCLNT_BUFFERFLAGS_SILENT` packets, or when no packets arrive at all for more than 50 ms.
- **Drawing:** `DrawWaveform` puts the **newest sample in the center bar**. Older samples spread symmetrically left and right, 4 slots (~40 ms) per bar, and the edge bars are 20% shorter. This matches `DrawMinimizedLine`, which uses the whole ring buffer.
- **Simulation results:**

  | Signal | Average |
  |---|---|
  | Solo flute −38 dB | ≈ 0.33 |
  | Three soft notes −38 dB | ≈ 0.36 |
  | Loud tutti −14 dB | ≈ 0.84 |
  | −75 dB | 0 |

- **History:**
  - 1.7.9: dB scale with full auto-gain (30 dB window below the reference, reference down to −45 dB). Soft passages looked as big as forte.
  - Before 1.7.9: linear `RMS × 4` scale with ~1.3 ms chunks. Music below about −27 dBFS looked silent.

**Expanded card (hover or pin):** 400×184 (`kExpandedWidth`/`kExpandedHeight`). Since 1.7.4 it is 400 wide (previously 380) so the player-switch arrows aren't crammed against the edge; other tabs follow at 400 because they are drawn relative to the card rect.
- 64px art, title/artist/album with marquee, waveform, scrubber (draggable to seek via `SeekMediaToTicks`), elapsed/remaining.
- Scrubber distance from the edge: `kScrubMargin` 24 + `kScrubBarLeftInset`/`kScrubBarRightInset` 56 (48 before 1.7.4).
- Control row (y = `kControlsY` 148, x relative to center):

| X offset | Button | Hit-test command |
|---|---|---|
| −118 | **Minimize** toggle (`DrawMinimizeToggleButton`, ±11130) | 4 |
| −64 | Previous | 0 |
| 0 | Play/Pause (radius 22) | 1 |
| +64 | Next | 2 |
| +118 | **Lyrics** toggle (`DrawLyricsToggleButton`, ±11173) | 3 |

- Commands 0–2 are sent via `SendMediaTransportCommand` (±7738, SMTC `TrySkipPrevious`/`TryTogglePlayPause`/`TrySkipNext`) to the session the island is showing (`ResolveMediaSession`, matched by AUMID). Commands 3–6 are handled locally in `WM_LBUTTONUP`.
- Press animation: `mediaBtnPress_[7]` and `UpdateMediaButtonAnimations`.

**Player switching (1.7.4):** when more than one app has an SMTC session (e.g. Spotify and YouTube in a browser), `‹` / `›` arrows appear in the scrubber row.

| Part | Details |
|---|---|
| Position | `kSessionArrowInsetX` 20 from the card's left/right edge, y = `kSessionArrowY` (= `kScrubberY` 114). Hit box ±`kSessionArrowHitHalf` 12 |
| Command | 5 = previous player, 6 = next player (`MediaTransportHitTest`, only when `g_mediaSessionCount > 1`) |
| Drawing | `DrawSessionArrow` (±11815): chevron, round background on hover/press |
| Time text | while the arrows are shown, elapsed is left-aligned and remaining right-aligned at `kSessionTimeInsetX` 32, so the arrow–time gap is the same on both sides. Scrubber end = time text (measured with `MeasureTextWidth`) + 8px, so the time–bar gap is also equal. Without arrows, the time text goes back to hugging the bar with a fixed inset |
| Click | `SwitchMediaSession` (±7710) rotates the index in `mediaSessionSources` starting from the currently shown session, stores the AUMID in `pinnedMediaSource`, then `SetEvent(g_mediaRefreshEvent)` so `MediaThreadProc` re-polls immediately |
| Limitations | the choice is per session (not persisted, reset on restart). Multiple tabs from the same browser share one AUMID, so they count as one player |
- Clicking an empty area of the card opens the media app (`OpenRelevantApp`, ±7219).

**Related settings:**
- `Media` and `MediaAutoExpand`: auto-expand for 5 seconds when the track changes.
- `MediaExpandBlocklist`: apps that don't trigger auto-expand.

**If it breaks:** symptoms are the music pill not appearing or an empty title. The cause is usually the SMTC API changing or failing; check the log for `"WinRT media session unavailable"`.

---

## 5. Feature: Lyrics (LRCLIB)

**Expected behavior:**
- Timestamped (synced) lyrics are looked up on **lrclib.net** when the track changes. Lyrics without timestamps are treated as missing.
- The line currently being sung is shown in the middle of the collapsed pill, between the art and the waveform, with a 180ms crossfade between lines. Long lines scroll slowly while they are shown.
- **Dynamic width:** the pill only widens to 320px when a line is about to be or is being sung.
  - It starts widening `kGrowLeadSeconds` = **0.35 seconds** before the line starts.
  - Intros, outros, and instrumental breaks **≥ 4 seconds** (`kShortGapSeconds`): back to 150px with a small ♪ icon.
  - Gaps under 4 seconds between lines: stays wide.
  - The line being sung is never cut off even if a gap is close.
- **Tracks without lyrics (since 1.7.14):** when a lookup finishes without synced lines, the 150px pill shows **two eighth notes joined by a single beam, in the accent** (`DrawInstrumentalNote`), fading in as the track peek (§4) ends. Deliberately different from the single ♪ text glyph of a sung song's break.
  - **Instrumental:** when LRCLIB answers with `"instrumental": true` (`FirstJsonBoolFieldTrue`, which also stops the looser searches), the status is `Instrumental` (cached like other results). The log line reads `0 synced lines, instrumental`.
  - **Not found:** a finished lookup with no synced lyrics (`NotFound`: not on LRCLIB, or plain lyrics only) gets the same note (`LyricsShowInstrumentalNote`), so the middle is never left empty for a track that will not get lyrics. A lookup LRCLIB was too busy to answer (`retryLater`) shows nothing until its retry answers, because the thread clears `trackKey` for the retry.
  - **Drawing:** vectors (`DrawBeamedEighthNotes`), because Segoe UI renders U+266B with a double beam that reads as sixteenths. The drawing is as tall as the ink of the typed U+266B at `smallTextFormat_` size × 1.3 (`kNoteScale`), measured with `GetOverhangMetrics` in `BeamedNoteGlyphInkHeight` and cached per size.
- **Toggle** via the button to the right of Next. State is stored in `Wh_SetIntValue("LyricsEnabled")`.
- **`LyricsDelayMs` setting** (−10000..10000): positive values make lyrics appear later.

**Code:**

| Function | ±Line | Role |
|---|---|---|
| `LyricsTrackKey` | 5268 | track key = `title + U+001F + artist` (used by the thread, width, and drawing) |
| `LyricsShowable` | 5276 | lyrics enabled, status Found, and key matches the current track |
| `LyricsShowInstrumentalNote` | ±6167 | lyrics enabled, status Instrumental or NotFound, and key matches the current track |
| `MediaPlayheadSeconds` / `LyricsPlayheadSeconds` | 5291 / 5300 | track position (`positionTicks` + time since `lastUpdatedTicks`), minus the delay |
| `LyricLineIndexAt`, `IsLyricGap` | 5284 / 5305 | active line; empty lines or lines with only ♪/♫ count as gaps |
| `LyricsWantWidePill` | 5322 | dynamic-width decision |
| `NextJsonStringField` | 5393 | minimal JSON string parser (unescapes `\n \" \\ \/ \uXXXX` + surrogates) |
| `ParseLrc` | 5470 | `[mm:ss.xx] text`, including multiple tags per line; metadata tags are ignored |
| `SimplifyTrackTitle` | 5514 | strips " - Remastered", " (feat.", " [" for looser searching |
| `FetchLyricsFromLrclib` | ±6460 | `/api/get` (title, artist, album, duration), then `/api/search` (full title, then simplified title). Since 1.7.11 only status **200/404** counts as an answer; 503/429 is retried once after 1.5 seconds; anything else (timeout, 5xx) sets `retryLater`. `trace` records each request's status for the log |
| `LyricsThreadProc` | ±6530 | 500ms loop, waits for duration ≤ 3 seconds, caches 30 tracks. Empty results with `retryLater` are **not cached** and are retried after 2 → 2 → 2 → 5 → 10 seconds, then every 20 seconds (`kRetryDelays`, `failedAttempts`). Delays are measured from when the lookup finished. While the track is paused, the retry that is running or pending (n) and the one after it (n+1) still run; from n+2 on they are held until the track plays again (`pausedRetries`, reset while playing). Pausing mid-lookup counts that lookup as n. Retries stop when the track changes |
| `DrawCollapsedLyrics` / `DrawLyricLine` | 10985 / 11048 | draw the line, crossfade, scroll, ♪ icon at normal width |
| `ActivityForKind` (Media case) | 12595 | uses `LyricsWantWidePill` for the width |

**If it breaks:** lyrics never appear. Check whether the LRCLIB API changed (the `syncedLyrics` field). Per-track logs (since 1.7.11):
- `Lyrics: "<title>" - <artist>: N synced lines (get 404, search 200)`: final cached result. N = 0 means LRCLIB really has no synced lyrics for that query.
- `Lyrics: LRCLIB busy or unreachable for "<title>" (search 503, search 503), retrying in 2s`: server busy or timed out, will retry. A trailing ` after playback resumes` means the next retry is held because the track is paused.

**Cause of "lyrics sometimes don't show" before 1.7.11:** LRCLIB frequently replies with **503 `ServerOverloaded`** (measured at about 1 in 3 `/api/get` requests for tracks it doesn't have, sometimes also `/api/search`). The old mod treated any body as an answer, so a 503 was cached as "no lyrics" until the mod restarted. Real example: "スウェイ・イット・フラ・ガール" (Lisa Ono) failed due to `search 503`, even though the lyrics exist.

---

## 6. Feature: Album art & iTunes fallback

**Behavior:** the cover comes from the SMTC thumbnail. If a track still has no cover after **4 seconds**, the cover is fetched from the **iTunes Search API** (600×600). A cover from SMTC always takes priority. If everything fails, an **accent gradient placeholder + music note icon** is shown (since 1.7.8), and a cover that arrives later fades in over 0.25 seconds.

| Function | ±Line | Role |
|---|---|---|
| `ReadWinRtStreamBytes` | 3777 | reads the thumbnail stream **incrementally** (`InputStreamOptions::Partial`, 64KB chunks, until exhausted, max 8MB), wrapped in try/catch |
| `DecodeImageBytesToPixels` | 3061 | WIC decode to premultiplied BGRA, fills `generation` and `sampledAccent` (UI accent color) |
| Cover "settle" logic | in `MediaThreadProc` | for 3 seconds after a track change, the cover keeps being re-fetched (browsers often swap the cover late); the old cover is kept if a momentary fetch fails |
| `ArtistNamesMatch` | 5686 | loose artist matching (contains, first artist before `,`/`&`) |
| `SearchItunesArtwork` | ±6520 | one request to `itunes.apple.com/search?...&limit=15&term=...&country=..`; results are split per `"wrapperType"`; priority: artist+title match → exact title match (`NormalizeTitleForMatch`, for classical/compilations that list the performer, not the composer) → artist only (not for browser sources). A body without `"resultCount"` (rate limit/403) = network failure |
| `FetchArtworkFromItunes` | ±6590 | order: user's Windows store region (`UserStoreCountryCode`) → US store → title only; replaces `100x100bb` with `600x600bb`; failed download/decode = network failure (retried, not cached) |
| `ArtworkThreadProc` | ±6670 | checks every 1 second, 4-second grace, cache of 30, 30-second retry on network failure, logs `"Album art: ..."`; if a cached cover disappears again (pause/seek) it is re-applied immediately |
| `HttpGet` | ±5650 | WinHTTP GET (binary-safe, timeouts 5/5/8/8 seconds); optional `status` parameter returns the HTTP code (0 = no response), used by lyrics since 1.7.11 |
| `DrawAlbumArt` / `DrawAlbumArtPlaceholder` | ±13075 | draws the cover (fade-in `MediaLayout::kArtFadeSeconds`); if empty: diagonal gradient from the accent color + music note glyph `` (glyph color adapts to accent lightness), app icon badge in the corner when expanded |

**If it breaks:** the note placeholder shows even though the track has a cover. Check the `Album art:` logs (DbgViewMini): "matched by ..." / "found no match" / "unreachable or refused". If SMTC itself fails, check the stream reading (Storage.Streams API) and the WIC decode.

---

## 7. Feature: Minimize (thin line)

**Expected behavior:**
- The button to the left of Previous enables minimize mode (stored in `Wh_SetIntValue("IslandMinimized")`).
- **Without music (since 1.7.13)** the same toggle is drawn in the top-right corner of the idle card's Time tab, on the greeting's line (`TimeLayout::MinimizeButtonX` / `kMinimizeButtonY`, shared by `DrawTimeDashboard` and `IdleMinimizeButtonHitTest`). It reuses command 4, `g_hoveredMediaButton` / `g_pressedMediaButton` and `ToggleIslandMinimized`; the wndproc only accepts it while `IdleTimeTabShown()` (no music pill, Idle chosen, tab 0). Minimize mode itself and the fullscreen-video line never depended on the pill kind, so the idle island already rests as the line (static dots instead of a waveform) and hover opens the idle card.
- In minimize mode the island becomes a **160×8 glass-style line** sitting **3px from the top edge of the screen** (or the bottom edge for the Bottom position). The line shows a thin accent-colored waveform radiating from the center with fading ends, or static dots while paused. Privacy dots appear at the right end.
  - The line's waveform uses the same `g_state.waveform` ring buffer as the pill/card, so its height follows the analyzer in §4 (frequency-weighted loudness, density, auto-gain max +8 dB). Audio is still captured because the pill kind is still Media.
  - The line shows the **entire buffer** from center to edge. Since 1.7.9 one sample ≈ 10 ms, so center to edge ≈ 480 ms (previously ≈ 64 ms), and the pulse spreads more slowly than on the pill (~120 ms). Edge bars fade via transparency (`edgeFade`); their height is not reduced.
- **Hover:** hitbox = line + 4px to the sides + 4px below + up to the screen edge, **with no delay**. Hover always opens the full card, even when Expand-on-hover is off. The card opens from the top edge and closes again when the mouse leaves the card.
- The island stays at the edge while minimized, including while open, to avoid an open/close loop.
- Transient notifications still bring up the pill, then it returns to a line.
- The line is not affected by idle auto-hide. During fullscreen video the island also automatically becomes this line (§12), and since 1.7.4 hover opens it as well.

**Code:**

| Part | ±Line | Details |
|---|---|---|
| `MediaLayout` constants | 590–612 | `kMinimizeButtonOffsetX` −118, `kMinimizedLineWidth` 160, `kMinimizedLineHeight` 8, `kMinimizedHoverPadX` 4, `kMinimizedHoverPadBelow` 4, `kMinimizedLineEdgeGap` 3 |
| `g_islandMinimized`, `g_minimizedEdgeFraction` | 1332 / 1336 | state and the animation fraction toward the edge (0..1) |
| `PositionOverlayWindow` | 2779 | shifts the window `y` by `shift × edgeFraction` so the line sits 3px from the edge |
| Render loop: hover (`lineResting`) | ±14780 | hitbox is enlarged while `heightSpring < 30`, for both manual minimize and `videoSuppressed` |
| Render loop: `minimizedMode` | 14047 | `isHoverExpanded = hover`, `isHidden = false` |
| Render loop: `minimizedLine` | 14175 | overrides the size to the line, `renderer.SetMinimizedLine` |
| Render loop: `edgeSpring` | 14266 | critically damped spring, sets `g_layoutDirty` while moving |
| `Renderer::Render` | 7687 | draws the line (`DrawMinimizedLine`) when `minimizedLine_` is active or height < 24 in minimize mode |
| `DrawMinimizedLine` | 11575 | pill fill alpha 0.5, white gradient sheen 0.16→0, 1.6px waveform bars, newest sample in the center, sample age = distance from center × whole buffer (48 slots ≈ 480 ms) |
| `WM_LBUTTONUP` cmd 4 | 13481 | toggle and persist (`ToggleIslandMinimized`, shared with the idle Time tab button) |
| `expanded` on click | `WM_LBUTTONUP` | minimize and the fullscreen-video line count as expanded so the first click isn't "eaten" |

---

## 8. Feature: Transient & inline notifications

### 8.1 Notification kinds (triggers & durations)

| Kind (`IslandKind`) | Trigger (code) | Duration | Setting |
|---|---|---|---|
| Clipboard | `WM_CLIPBOARDUPDATE` → `CaptureClipboard` (±6889): text (`CF_UNICODETEXT`), image (`CF_DIB`), owner app icon | 2.5 seconds | `Clipboard` |
| CapsLock (Caps/Num) | keyboard hook → `WM_APP_CAPSLOCK` | 2.5 seconds | `CapsLock` |
| Volume | `VolumeChangeCallback` (§9) | 1.8 seconds | `Volume` |
| Brightness | `ReportBrightnessChange` (§9.1) | 1.8 seconds | `Brightness` |
| Notification | `NotificationThreadProc` (±4237): `UserNotificationListener.GetNotificationsAsync(Toast)` polled every 1 second, already-seen IDs skipped, 32px app icon | 3 seconds | `NotificationRespectDnD` (silent during DND) |
| BatteryLow | `UpdateBatterySnapshot` (±5985): `GetSystemPowerStatus`; charger plugged/unplugged, or dropping to 20%/10% | 3 seconds | `Battery` |
| Device | `WM_DEVICECHANGE`: USB Drive (`DBT_DEVTYP_VOLUME`), COM, others | 3 seconds | — |
| Bluetooth | `BluetoothThreadProc` (±4780): `DeviceWatcher` with a Connected selector (Classic + LE), battery via GATT | 3 seconds | `BluetoothIndicator`, `BluetoothShowBattery` |
| DoNotDisturb | WNF `kWnfQuietHoursActiveProfileChanged` (`WnfDndCallback` ±4166) | 3 seconds | `DoNotDisturbIndicator` |
| Timer finished | render loop (`finishedExpiresAt`) | 3 seconds | `TimerModule` |

Full cards per kind (used when **no music is playing**): `DrawClipboard` (±11685), `DrawNotification` (11755), `DrawVolume` (11829) and `DrawBrightness` (both via `DrawLevelPill`), `DrawTimer` (11988), `DrawCapsLock` (12078), `DrawDevice` (12142), `DrawDoNotDisturb` (12210), `DrawBluetoothDevice` (12309), `DrawBattery` (12409). The `StatusCountdownProgress` setting adds a countdown line, with a `ttl` constant in each function that must match the durations above.

### 8.2 Inline notifications (while music is playing)
**Behavior:**
- While the music pill is shown, **there is no second pill**. `ChooseActivities` only returns `{Media}`.
- Transient notifications are drawn on a single row in the middle of the pill (icon, primary text, optional bar, caption), **temporarily replacing the lyrics**.
- The pill width follows the notification content, minimum 150 and maximum **460** (`kInlineAlertMaxWidth`). Text that is too long is truncated with "…".
- The notification content fades in as the pill widens.
- On hover, the expanded card is shown without the notification.
- **Without music:** notifications still appear as their own pill, as in the original version.

| Function | ±Line | Role |
|---|---|---|
| `ChooseInlineAlert` | 12676 | the first active notification by priority (same order as `ChooseActivities`) |
| `MediaPillShown` / `ChooseActivities` | 12695 / 12699 | media present → only `{Media}` |
| Render loop (`inlineAlert`) | 13936 | `inlineAlert`, `hasInlineAlert` feeds `isTransientAlert`, `renderer.SetInlineAlert`, width from `MeasureInlineAlert` |
| `InlineAlertContent` (struct) | 7608 | model of the single-row content |
| `BuildInlineAlertContent` | 10680 | **the single source of content**, used by both the measurer and the drawer |
| `MeasureInlineAlert` | 7789 | width = left reserve 48 + right 52 (+22 privacy) + padding 2×6 + row |
| `InlineAlertRowWidth`, `CollapsedMediaReserve`, `DrawTrimmedText` | 10659 / 10637 / 10806 | measuring, reserves, text with ellipsis |
| `DrawInlineAlert`, `DrawInlineAlertIcon` | 10909 / 10830 | draws the row (alpha layer), icon (bitmap, glyph, keycap, battery, ring) |
| `DrawMedia` collapsed | 10620 | notification or lyrics in the middle area |

---

## 9. Feature: Volume

**Expected behavior:**
- The volume notification appears **instantly** (≤ 1 frame), and the number updates on each press or while held.
- While the mod is active and the `Volume` setting is on, **the Windows volume OSD does not appear**. The island handles the volume keys.
- The Windows OSD returns when the mod is disabled, the `Volume` setting is turned off, or the island is manually hidden.

**How it works:**
1. **Event-based detection:**
   - `VolumeChangeCallback` (±6259, `IAudioEndpointVolumeCallback::OnNotify`) receives `fMasterVolume` and `bMuted` from Windows for every change (keys, slider, apps), then calls `ReportVolumeChange` (±6238), which fills `g_state.volume` and `system.volumePercent`, then `TriggerNudge`.
   - `DefaultDeviceWatcher` (±6305, `IMMNotificationClient`) re-registers the callback when the default output device changes, and bumps `g_defaultRenderDeviceGeneration` so every cached endpoint is re-fetched (since 1.7.15).
   - Both are run by `VolumeWatcherThreadProc` (±6344).
2. **`UpdateSystemSnapshot`** (±6124) only reads the level for the dashboard and **does not trigger notifications**. It used to, which was the source of a 0–1 second delay. Its cached endpoint is dropped when `g_defaultRenderDeviceGeneration` changes, so the dashboard shows the level of the device that is actually playing.
3. **Suppressing the OSD:** the Windows OSD appears because the shell handles the volume keys, not because the volume changes.
   - `LowLevelKeyboardProc` (±12785) swallows `VK_VOLUME_UP`/`VK_VOLUME_DOWN`/`VK_VOLUME_MUTE` (`return 1`), including synthetic keys (`LLKHF_INJECTED`, e.g. from OEM software for Fn + F), then posts `WM_APP_VOLUME_KEY` on key-down (including auto-repeat).
   - `ApplyVolumeKey` (±6412) runs `VolumeStepUp`/`VolumeStepDown`/`SetMute` on the window thread, with a cached endpoint and one retry. Since 1.7.15 the cache is also dropped whenever `g_defaultRenderDeviceGeneration` changes. A speaker that stops being the default stays a valid endpoint, so the retry (which only fires on a failed call) never noticed a switch to Bluetooth/headphones, and the keys kept stepping the old speaker.
   - `KeyboardHookWanted` (±12821) installs the hook if `capsLock || volume`.
   - `LoadSettings` calls `NotifyKeyboardThreadSettingChanged` when either changes.
4. **Diagnostic log:** `Volume key: vk=0x.. injected=..` (at most 1 per 2 seconds per combination), to confirm that Fn + F or a keyboard knob is being caught.

**Limitations:** volume changed by OEM software directly (without a key), or by Bluetooth headset buttons, doesn't go through the hook, so its OSD may still appear. The island notification is still instant via the callback.

### 9.1 Brightness (since 1.7.6)

**Expected behavior:** when the laptop screen brightness changes (keys, Settings slider), the island shows a brightness pill identical to the volume pill (sun icon ``, label "Brightness", percent, accent bar) for 1.8 seconds, and the Windows brightness OSD is not visible. Setting `Brightness` (on by default). While music is playing, the pill appears inline in the media pill; over fullscreen video, brightness passes through just like volume.

**Why it can't work like volume:** laptop brightness keys are firmware hotkeys. By the time the event reaches Windows, the brightness has already changed, and the shell shows the OSD because of the *change*, not because of the key. Tested on an HP Victus 15 (Win11 25H2): the keys send a keyboard event `vk=0xFF scan=0x2B` (extended, identical for up and down); swallowing it prevents neither the brightness change nor the OSD. The OSD is a small `XamlExplorerHostIslandWindow` owned by explorer.exe at the bottom center of the screen, visible for about 2 seconds.

**How it works:**
1. **Detection:** `BrightnessThreadProc` subscribes to `SELECT * FROM WmiMonitorBrightnessEvent` in `ROOT\WMI` (semisync, `Next` with a 1-second timeout only to check the stop event). On each event: `g_lastBrightness`, `g_lastBrightnessEventMs`, then `ReportBrightnessChange` fills `g_state.brightness` and calls `TriggerNudge`.
2. **Hiding the OSD:** `BrightnessOsdThreadProc` installs `SetWinEventHook(EVENT_OBJECT_SHOW)` for the explorer process only (pid from `Shell_TrayWnd`, re-checked every 5 seconds for explorer restarts or setting toggles). `BrightnessOsdShown` filters with `LooksLikeShellOsd` (class `XamlExplorerHostIslandWindow`, at most 1/3 of the monitor width and 1/5 of its height, so Alt+Tab isn't affected), then hides it with `ShowWindowAsync(SW_HIDE)` if:
   - there was a WMI event < 1.5 seconds earlier; or
   - the learned brightness key was just pressed (< 0.8 seconds); or
   - a direct query of `WmiMonitorBrightness.CurrentBrightness` differs from `g_lastBrightness` (the WMI event hasn't arrived yet).
   The window is **always recorded first** in `g_pendingBrightnessOsd` before the checks above, and the WMI thread **records the event time first** (`g_lastBrightnessEventMs`) before taking the pending window. With this ordering (seq_cst atomics on both sides), at least one side is guaranteed to see the other. The initial 1.7.6 version checked first and recorded afterwards, so if the WMI event arrived right in between, the OSD on the first press wasn't hidden (only the second press hid it). The pending-time difference is computed with a fresh tick and guarded against unsigned wrap-around.
3. **At 0% / 100%:** pressing the key changes nothing so there is no WMI event, yet the OSD still appears. The keyboard hook records every `vk 0xFF` key (`NoteUnmappedKey`, just an atomic store, the key isn't swallowed). If a WMI event arrives < 0.4 seconds after that key, its scan code is learned as the brightness key (`g_brightnessKeyScan`, log `Brightness: learned brightness key scan=0x..`). After that, the same key at 0%/100% still hides the OSD and shows the pill. This learning is lost when the mod restarts (one real brightness change is needed first).
4. `KeyboardHookWanted` is now `capsLock || volume || brightness`.

**Logs:** `Brightness: shell OSD shown, sinceEvent=..ms key=.. query=.. changed=.. -> hide/left pending` (every explorer OSD that passes `LooksLikeShellOsd`), `Brightness: hid the Windows brightness flyout (recent change / brightness key / level changed / after change)`, `Brightness: learned brightness key scan=0x..`, `Brightness: no built-in panel reports brightness; module idle`.

**Limitations:**
- Built-in panels (laptops) only. External monitors don't report to WMI, so both threads stop and the Windows OSD is left alone.
- The OSD may be visible briefly (a few milliseconds) before being hidden, since explorer shows it first and only then is it hidden.
- Automatic brightness changes (adaptive brightness) also bring up the pill.

---

## 10. Feature: Dashboard tabs

While expanded (hover or pin), tabs are switched with the mouse wheel or a horizontal swipe (> 40px within < 500ms). Vertical pagination dots are on the right.

**Tab order** (must match `ActiveTabCount` ±1420, and the two `activeTabs` lists in `DrawIdleDashboard` ±10212 and `DrawMedia` ±10641):
1. Main: Media (if music is playing) or Time (`DrawTimeDashboard`). Since 1.7.13 the Time tab matches the other cards: a square `DrawCard` on the left (geometry in `TimeLayout`: `left+28`, band `top+30 … bottom−30`, so a 124px square; wider insets than other tabs on purpose, to leave empty space) holding an analog clock (12 hour marks, hour/minute hands in the text colour, accent second hand and centre pin; `ClockAccentGlow` adds an accent radial glow inside the card), and a column on the right (`card.right+26 … right−30`) with an accent greeting for the time of day (`boldTextFormat_`; "Good morning" 05–11, "Good afternoon" 12–17, "Good evening" otherwise, 12 languages via `Loc`), then the digital clock at `timeDashboardFormat_` × `kClockScale` 1.2 (≈67px, shrunk to fit the column; AM/PM at 40% size and muted via `StyleClockDayPeriod`) over a 34px week strip drawn as plain text like the calendar grid (Sunday first, initials from `LOCALE_SSHORTESTDAYNAME*`, today's initial in the accent and its date on an accent disc). The date is not repeated here (it lives on the Date & Weather tab); only `DateFirst` brings it back, in the greeting's place. While the mic/camera is in use, a privacy chip replaces the week strip. The render loop repaints once a second while this tab is open so the second hand ticks.
2. **Date & Weather** (since 1.7.0, replacing the separate Calendar and Weather tabs): `DrawDateWeatherDashboard` (±9605).
   - **Left:** date hero from `DrawCalendarHero` (±9130), containing the month (accent), year, large day number (`calDayLargeFormat_` 63px), and weekday name. This helper is also used by the calendar tab. The `heroWidth` parameter is 93 on the calendar tab and 112 on this tab (since 1.7.3), because there is no month grid here.
   - **Card corners** use `InsetCardRadius` (±9121): the island radius (`ContentIslandRadius`) minus 16px, clamped to 6–30. Result: 28 for pill, 6 for notch and W11. This keeps card corners concentric with the island so they don't look boxy. Text inside the hero is shifted a few px away from the corner.
   - **Divider:** hairline at `left+146`.
   - **Right** (`left+160 … right−24`):
     - city name (`state.weather.city` or "Locating...");
     - 60px vector weather icon (`DrawWeatherIcon`) and temperature in the same 63px face as the day number (`calDayLargeFormat_`, alignment saved and restored);
     - description (`weatherDescFormat_`, at most 2 lines at `top+99…123`, adaptive font size 11.5 → 9.5);
     - **3 chips** (height 28, bottom-aligned with the hero, width weights 1.3 : 1 : 1). Each contains a `DrawMetricGlyph` glyph 0/1/2 (wind, thermometer, droplet) plus a number in `boldTextFormat_`, without a label. Since 1.7.2 the compass, thermometer, and ring visuals from 1.7.1 were removed as too heavy.
     - **Hover explanation (since 1.7.5):** hovering a chip shows a small card above the chip row, as wide as the right column: the chip name (bold) followed by one sentence describing it (wind = current wind speed, feels like = how warm or cold it actually feels, humidity = water vapor content in the air). The hovered chip is also drawn `raisedStrong`. `WM_MOUSEMOVE` computes the chip via `WeatherChipAtContentPoint` (only when Weather is enabled and tab index is 1) into `g_hoveredWeatherChip`; reset on `WM_MOUSELEAVE` or when the cursor leaves the island. Chip geometry comes from `WeatherLayout::ChipSpan`, shared by drawing and hit-testing. The text is localized for 12 languages.
   - If the **Weather module is off**, this tab falls back to `DrawCalendarDashboard`: hero plus a month grid, with today highlighted in the accent color.
   - The old `DrawWeatherDashboard` has been removed.
3. Hardware Monitor (optional): `DrawHardwareMonitorDashboard` (±10955). Since 1.7.7 it contains **4 cards in a 2×2 grid**: CPU, RAM, GPU (PDH `\GPU Engine(*)\Utilization Percentage`), and DISK. The NET UP / NET DOWN cards were removed.
   - **Geometry** in `HardwareLayout`: 8px gap between cards in both directions, card height about 59px, corners follow the island shape (`InsetCardRadius`, nearly a capsule for the pill shape).
   - **Card contents:** a 21px `DrawMetricGlyph` glyph inside a 34px circle (well) with an 18% tint; bold 12px label at 62% white; value in `textFormat_` at 97%; a 3px bar that stops 22px from the right edge. Tint and bar color: accent, yellow above 75%, red above 90% (`LoadStateColor`).
   - **Muted detail text** (small font, `mutedBrush_`) next to the main value, via the `detail` field in `HwMetric`; hidden when its source has no value:
     - **CPU:** current clock, e.g. `3.8 GHz`. `GetCpuClockGHz` uses PDH `\Processor Information(_Total)\Processor Frequency` × `% Processor Performance` ÷ 100 (same as "Speed" in Task Manager; can exceed the base frequency under turbo, averaged across all cores). The performance counter is a rate, so the first sample only primes it (the detail appears the next second). Cost about 0.1 ms per sample.
     - **RAM:** main value is a percentage (`memoryPercent`, same as Task Manager), followed by `used/total GB` (total rounded to whole GB).
     - **GPU:** temperature, e.g. `62°C`. `GetGpuTemperatureC` uses D3DKMT (`D3DKMTEnumAdapters2` / `D3DKMTQueryAdapterInfo` type 62 `KMTQAITYPE_ADAPTERPERFDATA`, field `Temperature` in deci-Celsius), the same source as Task Manager's GPU temperature. The functions are resolved dynamically from `gdi32.dll` and the structs are declared locally in the `Kmt` namespace (WDK headers aren't available in the Windhawk toolchain). Adapter handles are opened once and kept; the highest temperature is used. Usually only dedicated GPUs report it (iGPUs report 0); software adapters (Basic Render) always reject the query. If *all* adapters reject it, the handles are closed and reopened no sooner than 30 seconds later. Cost about 1 µs per sample.
   - **DISK:** percent used of the selected drive, label `DISK C:`. If there is more than one **fixed** drive (`GetDriveTypeW == DRIVE_FIXED`, excluding USB/optical/network), a `›` arrow appears at the card's vertical center, aligned with the end of the bar. Its hitbox is the card's whole right strip (22px left of the arrow to the edge, full card height); hover just brightens the arrow. Clicking calls `CycleHardwareDisk` (C: → D: → … → C:), which immediately updates `g_state.system` and saves the choice to `Wh_SetIntValue("HardwareDiskLetter")`. A drive that disappears falls back to the first fixed drive.
4. File Tray (optional): `DrawFileTrayDashboard` (±9478). Files dropped onto the island (`WM_DROPFILES`), only paths are stored, click to open (`ShellExecuteExW`), capacity via `FileTrayMaxItems`.
5. **Activity** (automatic, always last): `DrawActivityDashboard` (±11903). Appears while the Focus Timer is running or progress is active.
   - Timer: Focus/Break label (+Paused), large `mm:ss` countdown, orange (focus) or green (break) bar, and the hint "Right-click the island to pause or stop".
   - Progress: label and percent bar.
   - `ActivityTabIndex` (±1442) is always the last index, so other tabs don't shift.

**Indices:** `NormalizedTabIndex` (±1428, modulo the tab count), `HardwareMonitorTabIndex` (±1441, always `2` since 1.7.0, used to decide whether the GPU needs polling, and for hover/click on the DISK drive arrow), `FileTrayTabIndex` (±1448, `2 + hardware`). The Weather module no longer affects the tab count.

**Weather:** `WeatherThreadProc` (±5337): `wttr.in/<location>?format=j1` (HTTPS, HTTP fallback), reads `current_condition` and `areaName` (label). Refreshes every 15 minutes. On failure there is a 1/2/5/15-minute backoff. It also refreshes when the location setting changes (`cityChanged` covers `WeatherUseLocation`) or on resume from sleep. Settings: `Weather`, `WeatherUseLocation`, `WeatherCity`, `WeatherFahrenheit`.
- **Location priority (1.6.3):**
  1. **Windows location** if `WeatherUseLocation` is on. `QueryWindowsLocation` (±5293) calls `Geolocator::RequestAccessAsync`, then `GetGeopositionAsync(maximumAge 10 minutes, timeout 10 seconds)` with Default accuracy (Wi-Fi, no GPS). The result is sent to wttr.in as `lat,lon` with 4 decimals.
  2. `WeatherCity`, encoded with `PercentEncodeUtf8`.
  3. No path (`/?format=j1`): wttr.in guesses the location from the IP. The area name from this guess can be odd, e.g. a village name.
- **Windows location requirement:** in Windows Settings → Privacy & security → Location, "Let desktop apps access your location" must be on. If denied, the log shows `Weather: Windows location not allowed (status N)` (only once). On failure, the log shows `Weather: Windows location failed (0x...)`. In both cases the mod falls back to step 2 or 3.
- **Technical requirement:** the weather thread calls `winrt::init_apartment(MTA)` so it can wait on WinRT async operations. The code is guarded by `DYNAMIC_ISLAND_HAS_GEOLOCATION` (`__has_include(<winrt/Windows.Devices.Geolocation.h>)`); without that header, the feature compiles to a stub.
- **Label:** still `areaName` from wttr.in, i.e. the nearest area name to the data point. It may be a district or village name rather than a city.

---

## 11. Feature: Idle strip, Privacy, Timer

- **Idle strip** (no music and no notifications): clock and weather. Width is measured from the text via `MeasureIdleStrip` (±7860) and drawn by `DrawIdleDashboard` (±9924). Since 1.7.13 the weather slot is the accent vector icon from `DrawWeatherIcon` (`IdleStripLayout::WeatherIconSize`, 18px × Text size) + temperature instead of a colour emoji, and the clock's AM/PM marker is muted (`StyleClockDayPeriod`, same size so the measured width holds). Settings: `ClockFormat`, `ShowSeconds`, `DateFormat`, `DateFirst`.
- **Privacy:**
  - `UpdatePrivacyIndicators` (±6607) and `IsDeviceActiveViaRegistry` (±6527) read the registry `HKCU`/`HKLM\...\CapabilityAccessManager\ConsentStore\{microphone|webcam}`: an app currently using the device has `LastUsedTimeStop = 0`.
  - Orange dot for the microphone and green for the camera, drawn by `DrawPrivacyDots` (±8597), or at the end of the minimize line.
  - Settings: `PrivacyDots`, `PrivacyDotsMic`/`Cam`, and their colors.
- **Focus Timer:** from the right-click menu, `StartFocusTimer` (±6976; 25/50-minute focus, 5-minute break), `ToggleTimerPause` (±6990), `StopFocusTimer` (±7008).
  - With music playing: only in the Activity tab.
  - Without music: also shown as a timer pill.
  - Finished: 3-second notification.
- **Progress:** `UpdateProgressSnapshot` (±6600) only reads `Wh_GetIntValue("ProgressPercent")`. **Nothing in the mod writes this value** (it's only reset to −1), so in practice it is never active. This is not a new bug.

---

## 12. Feature: Position, theme, backdrop, auto-hide, hotkey, menu

- **Position:** `GetAnchorWorkRect` (±2721; target monitor: primary, 1–5, or follow mouse) and `PositionOverlayWindow` (±2779; Position, OffsetX/Y, `ResolveOffsetY` ±2759 for a separate offset when expanded, and the shift to the edge when minimized). `GetIslandDockRect` (±3007) is the hover area for a parked island.
- **Shape:** Pill (default), Windows 11 flyout (`w11Style`), macOS notch (`notchStyle`, attached to the top), `BorderMergedMode`.
- **Theme:** `ThemePreset` (Obsidian, Graphite, Slate, Nord, Evergreen, Espresso, Plum, Porcelain, Custom), `ThemePalette` (±872), color tokens in `EnsureBrushes`. Hex colors accept `#RRGGBBAA`. The accent color is taken from the album cover (`sampledAccent`) with a smooth transition.
- **Backdrop:** `Backdrop` namespace (±2598): `SetWindowCompositionAttribute` (undocumented) for blur or acrylic, with the region clipped to the pill shape (`ApplyBackdropRegion` ±2860). Applied via `WM_APP_APPLY_BACKDROP` (`ApplyBackdropMaterial` ±2641).
- **Auto-hide:** `AutoHideIdleSeconds` (−1 = hidden except for notifications, 0 = never, >0 = idle seconds), `AutoHideFullscreen` (video only, see below), `UnhideOnHover` (mouse hook while parked). Parking means `ShowWindow(SW_HIDE)` and about 0% CPU.
- **Fullscreen video (1.6.2):** during fullscreen **video**, the island **shrinks into the minimize line** at the top edge of the screen. Games, browser F11 without video, and other fullscreen apps keep showing the normal island.
  - **Detection:**
    - `MediaThreadProc` stores the AUMIDs of **all** `Playing` SMTC sessions in `g_state.playingMediaSources` every 1.5 seconds, not just the session the pill follows.
    - `IsForegroundFullscreenVideo` is true if `IsForegroundFullscreen` is true (the foreground window's client rect covers its monitor) **and** that window belongs to one of those sources. The result is sticky per HWND, so a paused video is still treated as video until it leaves fullscreen or the foreground changes.
    - **Since 1.7.4:** if no matching Playing session is found, all SMTC sessions (`mediaSessionSources`, including Paused ones) are checked too. This covers paused videos and browsers like Zen/Firefox whose status lags when entering fullscreen. Fullscreen apps with no media session at all (games, browser F11 that never played media) still show the normal island.
  - **Matching** in `WindowBelongsToMediaSource`, in order:
    - (a) `WindowAppId`: the window's `PKEY_AppUserModel_ID` property via `SHGetPropertyStoreForWindow`. Chromium browsers use the same id as SMTC.
    - (b) `PackagedAppIdForPid` (`GetApplicationUserModelId`) for UWP apps. The `ApplicationFrameHost.exe` frame is replaced by its child process via `FindFrameContentProc`, which must be a `CALLBACK` function, not a lambda, because x86 builds need stdcall.
    - (c) The exe name without `.exe` appearing in the AUMID, plus aliases for chrome, edge, and vlc.
    - (d) Firefox: the AUMID is a hex hash (`LooksLikeFirefoxAumid`).
  - **Diagnostic logs:**
    - `Full-screen video detected (source "...")` appears on detection (plus `, not playing` if it matched via a session that isn't Playing).
    - `Full screen, not video: exe=... windowAppId=... packagedId=... sessions=[...]` appears once per window and session set when there is **no** match; `sessions` lists all SMTC sessions. Use this log to add matching rules.
    - `Video line: fullscreenVideo=.. suppressed=.. line=.. hover=.. armed=.. alert=.. privacy=.. override=..` appears whenever any of those values changes during (or right after) fullscreen video. Use it to see whether detection failed (`suppressed=0`) or something is keeping the island large.
    - Real example: Zen (`zen.exe`) uses a hex AUMID identical to its `windowAppId` (e.g. `F81484BD793E1A42`), so it matches via step (a).
  - **Behavior while `videoSuppressed`** (render loop):
    - `videoLine` forces `minimizedLine`.
    - `edgeSpring` moves the island to the edge like minimize mode.
    - `renderer.SetLineMode` ensures transitions are still drawn as a line.
    - **Hover opens the card (since 1.7.4)**, same as manual minimize: `isHoverExpanded = hover`, then `videoHoverOpen` allows the expanded size even though `isFullscreen`. Hover only counts once the island has actually become a line (`videoLineArmed`, set when `heightSpring < 30` and reset on leaving fullscreen). Without this, a cursor that happened to be over the island when the video went fullscreen would keep the island from ever shrinking. Pin is still ignored.
    - `recentTrackChange` is disabled, and `inlineAlert` / `isTransientAlert` apply to Volume only.
    - Volume briefly appears as a pill, then returns to the line.
    - **Privacy dots no longer hold the pill open (since 1.7.4).** The dots are drawn at the end of the line (`DrawMinimizedLine`). Previously a continuously used mic (e.g. Discord) kept the island from ever becoming a line over video.
    - Since 1.7.4 this line isn't affected by idle auto-hide (`isHidden = false`), same as manual minimize, so it can always be hovered.
    - `EnsureTopmost` keeps running so the line stays above the player.
  - The hide/show hotkey forces the normal island (`g_fullscreenOverrideVisible`). This is reset on leaving fullscreen.
  - `g_isFullscreen` / `isFullscreen` means "fullscreen video". Other gates that use it (the HW monitor) switch off only during fullscreen video, not in fullscreen games. `parkedForFullscreen` is now always false (the island is no longer parked during video).
- **Topmost:** `EnsureTopmost` (±2952) reclaims its z-order position every 1 second. When `AlwaysOnTop` is off, the parent is set to Progman.
- **Click-through:** `SetClickThrough` (±6949) for idle without hover, or temporarily on Ctrl+hover (the island becomes see-through).
- **Hide/show hotkey:** `ApplyHideShowHotkey` (±2687) calls `RegisterHotKey` on the window thread. Default Ctrl+Alt+D. State is stored in `ManuallyHidden`.
- **Right-click menu** (`ShowContextMenu` ±7310): Hide/Show, Dismiss, Focus Timer (25/50 minutes, 5-minute break, pause, stop), privacy status, Pin expanded, W11/Pill style, Notch style, Expand on Hover/Click, Transparency 100/85/70/55/reset, Theme, and Open Windhawk settings. Choices in this menu are **per session** (reset from settings on restart), except those stored in `Wh_*IntValue` values (§13).
- **Other gestures:** middle-click to mute, double-click to pin expanded.
- **Localization:** `Loc()` (±1554) translates strings into 12 languages (`UiLanguage` ±1454). Strings missing from the table are shown in English. Some of this fork's new strings ("Image copied", "Removed", "Focus Complete", "Right-click the island…") are still English-only.

---

## 13. Settings & stored values

**Locations:** the YAML schema (file header), the `Settings` struct (±945), and the reads in `LoadSettings()` (±2081). **Adding a setting means changing all three.**

**Mod Settings groups:**
- **Appearance:** Position, TargetMonitor, OffsetX/Y, SeparateExpandedOffsetY, OffsetYExpanded, BorderMergedMode, ShapeStyle, SizeScale, AutoDpiScale, AlwaysOnTop.
- **Behavior:** ExpandOnHover, AutoHideIdleSeconds, AutoHideFullscreen, UnhideOnHover.
- **Animations:** TargetFPS, AnimationStyle, AnimationSpeed.
- **Themes:** ThemePreset, PillOpacity, TintIntensity, BackdropMaterial/FillOpacity/Tint, MaterialDepth, DropShadow, AccentBloom, TextScale, AccentColorMode, CustomAccentHex, CalendarAccent, ClockAccentGlow, FontFamily, ContourBorderMode/Hex, PillBgColor, TextPrimary/SecondaryColor.
- **Indicators:** PrivacyDots, PrivacyDotsMic/Cam (+colors).
- **Modules:** Media, MediaAutoExpand, **LyricsDelayMs**, Volume, **Brightness**, CapsLock, Battery, BluetoothIndicator, BluetoothShowBattery, Progress, TimerModule, Clipboard, StatusCountdownProgress, DoNotDisturbIndicator, NotificationRespectDnD, HardwareMonitorModule, Weather, **WeatherUseLocation**, WeatherCity, WeatherFahrenheit, Language, ClockFormat, ShowSeconds, DateFormat, DateFirst, FileTrayModule, FileTrayMaxItems, MediaExpandBlocklist.
- **Shortcuts:** HideShowHotkeyEnabled, HideShowModifiers, HideShowKey.

**Stored values (`Wh_GetIntValue` / `Wh_SetIntValue`):**

| Key | Meaning | Survives restart? |
|---|---|---|
| `LyricsEnabled` | lyrics toggle | yes |
| `IslandMinimized` | minimize mode | yes |
| `HardwareDiskLetter` | drive on the Hardware tab's DISK card (letter code, default `C`) | yes |
| `ManuallyHidden` | hidden via hotkey/menu | yes (with reset rules in `LoadSettings`) |
| `PinnedExpanded` | pin expanded | no (reset in `LoadSettings`) |
| `PillOpacityOverride` | transparency from the menu | yes |
| `W11StyleOverride`, `NotchStyleOverride`, `ExpandOnHoverOverride` | menu overrides | reset from settings |
| `ColorThemeV2` (`kThemeValueName`) | active theme | synced with settings |
| `ProgressPercent` | progress value (nothing writes it) | — |

---

## 14. External services

| Service | Used for | Endpoint | If it changes or goes down |
|---|---|---|---|
| wttr.in | weather | `https://wttr.in/<city or lat,lon>?format=j1` | falls back to Open-Meteo (below) |
| api.open-meteo.com | backup weather (1.7.2) | `/v1/forecast?latitude=..&longitude=..&current=...` | right side of the Date & Weather tab shows "Locating..." / "Loading..." |
| geocoding-api.open-meteo.com | city → coordinates for the fallback | `/v1/search?count=1&name=<city>` (cached per city) | fallback doesn't work if relying on IP only |
| lrclib.net | lyrics | `/api/get`, `/api/search` (field `syncedLyrics`) | lyrics don't appear; pill stays 150px |
| itunes.apple.com | backup cover | `/search?media=music&entity=song` (fields `artistName`, `artworkUrl100`) | source badge is shown if SMTC also provides no cover |

All go through `HttpGet` (WinHTTP, user agent `DynamicIsland/1.0`). The HTTP status is generally not checked, since error bodies fail to parse on their own. Exception: LRCLIB lyrics read the status (§5), because 503s from LRCLIB are common and must not be treated as "no lyrics".

---

## 15. Repair guide after Windows updates

Use this table to map symptoms to code areas. Every feature above is **target behavior**: a fix must restore that behavior, not remove the feature.

| Symptom | Area to check | Windows dependency |
|---|---|---|
| Island doesn't hide during a screenshot (Win+Shift+S / PrtSc) | `IsScreenSnipActive` (foreground owned by `SnippingTool.exe`/`ScreenClippingHost.exe` and covering the monitor), the snip block at the start of the `RenderThreadProc` frame. Log: `Screen snip active: island hidden until it closes.` | Snipping Tool |
| Volume pill doesn't appear at 100%/0% | `ApplyVolumeKey` (since 1.7.3 calls `ReportVolumeChange` directly after each key) | `IAudioEndpointVolume` |
| Island doesn't appear at all | `Wh_ModInit`/`Wh_ModAfterInit` (tool-mod model), `RenderThreadProc` (window creation), `Renderer::Initialize` | Windhawk tool-mod, `CreateProcessInternalW`, D2D/DWrite, `UpdateLayeredWindow` |
| Music pill/title empty | `MediaThreadProc`, `SelectActiveMediaSession` | WinRT `Windows.Media.Control` (SMTC) |
| Player-switch arrows don't appear / don't switch | `mediaSessionSources` and `g_mediaSessionCount` in `MediaThreadProc`, `SwitchMediaSession`, `pinnedMediaSource` in `SelectActiveMediaSession` | SMTC `GetSessions`, `SourceAppUserModelId` |
| Scrubber clicks miss the bar | `g_scrubInsetLeft` / `g_scrubInsetRight` (written by `DrawMedia`, read by `MediaScrubFractionFromContent` / `MediaScrubFractionUnbounded`) | — |
| Cover is always a badge | `ReadWinRtStreamBytes`, `DecodeImageBytesToPixels`, `ArtworkThreadProc` | `Windows.Storage.Streams`, WIC |
| Prev/play/next buttons don't work | `SendMediaTransportCommand`, `ResolveMediaSession` | SMTC `Try*Async` |
| Waveform is static | `AudioThreadProc`, `g_audioCaptureNeeded`; if it's just not sensitive enough, the `kWaveform*` constants above `WaveformAnalyzer` (§4) | WASAPI loopback |
| Windows notifications don't appear | `NotificationThreadProc` (check the log for access Denied) | `UserNotificationListener`, Privacy → Notifications permission |
| Bluetooth not detected | `BluetoothThreadProc` | `DeviceInformation.CreateWatcher`, `BluetoothDevice`/`BluetoothLEDevice` selectors |
| Wrong DND status | `SubscribeDndNotification`, `WnfDndCallback` | WNF state `0x0D83063EA3BF1C75` (**undocumented**, very likely to change) |
| Wrong privacy dots | `IsDeviceActiveViaRegistry` | registry `CapabilityAccessManager\ConsentStore` |
| Volume delayed or not appearing | `VolumeWatcherThreadProc`, `VolumeChangeCallback` | `IAudioEndpointVolume`, `IMMNotificationClient` |
| Windows brightness OSD still appears | `BrightnessOsdShown`, `LooksLikeShellOsd` (check the `Brightness: hid ...` log); at 0%/100% check the `learned brightness key` log | `SetWinEventHook`, WMI `WmiMonitorBrightnessEvent` |
| Windows volume OSD appears again | `LowLevelKeyboardProc`, `KeyboardHookWanted`, `ApplyVolumeKey` (check the `Volume key:` log) | `WH_KEYBOARD_LL`, `VK_VOLUME_*` |
| Volume keys dead entirely | the hook swallows the key but `ApplyVolumeKey` fails: check the endpoint in `ApplyVolumeKey` | Core Audio |
| Volume keys change the wrong device after switching output (e.g. Bluetooth); the pill moves but the sound doesn't | `ApplyVolumeKey` / `UpdateSystemSnapshot` endpoint caches, `g_defaultRenderDeviceGeneration` bumped in `DefaultDeviceWatcher::OnDefaultDeviceChanged` | `IMMNotificationClient`, `IAudioEndpointVolume` |
| Clipboard doesn't appear | `WM_CLIPBOARDUPDATE`, `CaptureClipboard` | `AddClipboardFormatListener` |
| Blur/acrylic missing or boxy | `Backdrop` namespace, `ApplyBackdropRegion` | `SetWindowCompositionAttribute` (**undocumented**) |
| Island covered by other windows | `EnsureTopmost`, `PositionOverlayWindow` | `HWND_TOPMOST` z-order |
| Wrong fullscreen auto-hide (fullscreen detection) | `IsForegroundFullscreen` | monitor rect comparison and window class |
| Fullscreen video doesn't hide the island / games get hidden too | `IsForegroundFullscreenVideo`, `WindowBelongsToMediaSource`, `playingMediaSources` / `mediaSessionSources` in `MediaThreadProc`; `Video line:` log | per-browser/app SMTC AUMID format, `GetApplicationUserModelId`, `ApplicationFrameHost` |
| Line doesn't open on hover over fullscreen video / island doesn't become a line | `videoSuppressed`, `videoLineArmed`, `videoHoverOpen`, `videoLine`, `edgeSpring` in the render loop | — |
| A particular video isn't detected | read the `Full screen, not video:` log, then add a rule in `WindowBelongsToMediaSource` | SMTC AUMID vs `PKEY_AppUserModel_ID` |
| Weather doesn't follow the laptop's location | `QueryWindowsLocation` (check the `Weather: Windows location ...` log), fallback order in `WeatherThreadProc` | `Windows.Devices.Geolocation`, desktop-app location permission |
| Hardware monitor 0/empty | `GetGpuUsage`, `SampleHardwareDisk` / `DriveFreePercent` (DISK), `GetCpuClockGHz` (GHz), `GetGpuTemperatureC` (GPU temperature; check whether Task Manager shows a GPU temperature) | English PDH counters (`PdhAddEnglishCounterW`) |
| Wrong position on multi-monitor | `GetAnchorWorkRect`, `MonitorEnumProc` | `EnumDisplayMonitors`, DPI |

**Principles when fixing:**
- Don't block global input in hooks (LL hooks must be lightweight: only post messages, no locks, no COM).
- Window operations (hotkey, backdrop) must run on the window-owning thread (post `WM_APP_*`).
- Preserve the layout constant ↔ hit-test coupling (§2.5).
- Preserve the CPU-saving behavior: park when hidden, expensive polls (GPU/net) only when visible, audio capture only while the waveform is visible.
- Always run the syntax check (§1), then test in Windhawk.

---

## 16. Fork changelog

| Version | Changes |
|---|---|
| 1.4.0 | **LRCLIB lyrics** in the collapsed pill (+ toggle, cache, retry); incremental cover reading; **iTunes backup cover**; binary-safe `HttpGet` + timeouts; dynamic pill width following lyrics (0.35-second lead, gaps ≥ 4 seconds return to normal); **Lyrics delay (ms)** setting |
| 1.5.0 | **Minimize**: glass line + waveform, sits 3px from the screen edge, small hitbox with no delay, hover opens the card |
| 1.6.0 | **Inline notifications** in the middle of the music pill (no second pill, width follows the notification, max 460); **Activity tab** for Timer/Progress; transient notification duration capped at 3 seconds |
| 1.7.14 | **Track peek**: for 4.5 s after every track change (lyrics or not) the collapsed media pill widens and shows the title over the artist (`TrackPeekActive`, `DrawTrackPeek`), then the lyrics fade in. **Tracks without lyrics** show accent beamed eighth notes in the middle of the pill: LRCLIB's `instrumental` flag is now read (new `Instrumental` lyrics status, `FirstJsonBoolFieldTrue`), and a finished lookup with no synced lyrics gets the same note (`LyricsShowInstrumentalNote`); the note is drawn as vectors with a single beam (`DrawBeamedEighthNotes`), sized from the ink of the U+266B glyph |
| 1.7.15 | **Volume keys follow the active output**: after the default output changed (e.g. connecting Bluetooth headphones), Fn + F volume keys and the volume pill kept controlling the old speaker, because `ApplyVolumeKey` cached an endpoint that stays valid and only re-fetched it on a failed call. `DefaultDeviceWatcher` now bumps `g_defaultRenderDeviceGeneration`, and `ApplyVolumeKey` and the dashboard volume read in `UpdateSystemSnapshot` re-fetch their endpoint whenever it changes |
| 1.7.13 | **Redesigned idle clock**: Time tab rebuilt from the shared card parts (analog clock card with ticking accent second hand, accent greeting over a large digital clock with muted AM/PM, plain-text week strip with today on an accent disc, no duplicate date unless `DateFirst`; **minimize button on the Time tab** so minimize mode can be switched without music; privacy chip replaces the strip while mic/camera is in use); collapsed idle strip uses the accent vector weather icon instead of an emoji and mutes AM/PM; weekday name on the date hero (Date & Weather and Calendar tabs) in the text colour instead of grey |
| 1.7.11 | **Lyrics no longer go missing when LRCLIB is busy**: `HttpGet` returns the HTTP status; LRCLIB lookups only accept 200/404, retry 503/429 once after 1.5 seconds, and uncertain empty results aren't cached (retry 2/2/2/5/10 seconds then every 20 seconds; while paused only retries n and n+1 still run, the rest wait for playback to resume); per-track log with each request's status |
| 1.7.10 | **Balanced waveform**: `WaveformAnalyzer` with 6 band-pass filters (frequency-weighted loudness + density of active bands as an estimate of instrument count); auto-gain capped at +8 dB, absolute −60…−10 dBFS scale with a 1.3 curve, so soft music moves but stays small; `DrawWaveform` now radiates from the center to the left and right like the minimize line |
| 1.7.9 | **More accurate waveform for soft music**: dB scale with auto-gain (`g_waveformRefDb`, 30 dB window, reference floor −45 dB, −70 dB gate) replacing linear `RMS × 4`; one sample per ~10 ms so release and bar spacing match real time; no more false zeros between packets |
| 1.7.7 | **Simplified Hardware tab**: 4 cards (CPU, RAM, GPU, DISK) in a 2×2 grid with geometry in `HardwareLayout`; NET UP/DOWN removed and network counters no longer polled; 21px icons in round wells, card corners follow the island shape, brighter labels; RAM shows percent + `used/total GB`; **CPU shows GHz** (PDH `% Processor Performance`) and **GPU shows temperature** (D3DKMT adapter perf data), both read only while the Hardware tab is visible; **drive-switch arrow** on the DISK card (`CycleHardwareDisk`, stored in `HardwareDiskLetter`) |
| 1.7.6 | **Brightness**: brightness pill like volume (WMI `WmiMonitorBrightnessEvent`), Windows brightness OSD hidden (WinEvent hook on explorer), brightness key learned for the 0%/100% case; `Brightness` setting; `DrawVolume` and `DrawBrightness` share `DrawLevelPill`; links `-lwbemuuid` |
| 1.7.5 | **Weather chip hover explanations**: wind, feels-like, and humidity on the Date & Weather tab show a short explanation card (12 languages); chip geometry moved to `WeatherLayout` |
| 1.7.4 | **Player switching**: arrows in the scrubber row to choose which SMTC session the island follows (`pinnedMediaSource`, `SwitchMediaSession`, commands 5/6), immediate media poll via `g_mediaRefreshEvent`; 400px expanded card; time text and scrubber ends follow the arrows with equal spacing on both sides. **The fullscreen-video line can be hovered** to open the card (once the island has become a line), and is no longer affected by idle auto-hide. Privacy dots (mic/camera) no longer hold the pill open over video; video detection also accepts sessions that aren't Playing (Zen/Firefox, paused video); `Video line:` diagnostic log |
| 1.7.3 | 12px clipboard notification text (`clipboardTextFormat_`, also in the inline pill via `InlineAlertContent::primaryFormat`); **island hides immediately during the screenshot overlay** (`IsScreenSnipActive`, no topmost/alerts until it closes; not excluded from capture); 112px date box on the Date & Weather tab; volume keys at 100%/0% still show the pill |
| 1.7.2 | Weather chips simplified again (glyph + number), 60px weather icon. **Open-Meteo fallback** when wttr.in fails/times out (`FetchOpenMeteo`, `GeocodeCityOpenMeteo`, `WmoToWwoCode`); wttr.in is skipped for 1 hour after a failure. Logs: `Weather: Requesting fallback from api.open-meteo.com`, `Weather parsed success (Open-Meteo)` |
| 1.7.1 | Date & Weather tab revision: card corners concentric with the island (`InsetCardRadius`); 52px weather icon and 63px temperature; wind compass / feels-like thermometer / humidity ring tiles replacing plain chips |
| 1.7.0 | **Date & Weather tab**: calendar and weather tabs merged (date hero on the left, weather conditions on the right + 3 wind/feels-like/humidity chips); falls back to the old calendar if Weather is off; `DrawCalendarHero` extracted, `DrawWeatherDashboard` removed |
| 1.6.3 | **Weather follows Windows location** (optional `WeatherUseLocation`, every 15-minute refresh, falls back to city then IP) |
| 1.6.2 | **Fullscreen video becomes the minimize line** (instead of disappearing); games and other fullscreen apps unaffected; hover/pin don't open it; only volume and privacy appear; paused video stays a line (sticky); detection via window AUMID + diagnostic logs |
| 1.6.1 | **Instant volume** (Core Audio callback instead of a 1-second poll); **Windows volume OSD hidden** while the mod is active (hook swallows `VK_VOLUME_*`) + diagnostic log |

Backup files from each stage were stored temporarily in the Claude Code session scratchpad folder (`mod.wh.cpp.bak`, `.before-minimize.bak`, `.before-inline-alerts.bak`, `.before-volume.bak`). That folder is not a permanent location, so use git for long-term history.
