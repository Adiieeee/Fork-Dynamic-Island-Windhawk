# Dynamic Island for Windows (Fork)

<p align="center">
  <img src="Demo.gif" alt="Dynamic Island for Windows demo" width="800">
</p>

---

A [Windhawk](https://windhawk.net) mod that adds an iPhone-style **Dynamic Island** to the Windows desktop: a floating pill at the top of the screen that reacts to what you're doing. It shows the music you're playing, notifications, volume, clipboard, battery, devices, weather, and more.

This is a fork of [devcode90/Dynamic-Island-for-Windows](https://github.com/devcode90/Dynamic-Island-for-Windows) by Himanshu. It adds synced lyrics, a minimize mode, inline alerts, instant volume/brightness pills that replace the Windows OSD, player switching, a redesigned Date & Weather tab, a simpler Hardware tab, and many fixes.

- **Mod id:** `dynamic-island-for-windows-fork`
- **Version:** 1.7.13
- **License:** [MIT](LICENSE)
- **Technical reference:** [DOCUMENTATION.md](DOCUMENTATION.md) explains how each feature should behave, where its code lives, and which Windows APIs it uses. Use it when something breaks after a Windows update.

---

## What it does

The island is a layered, per-pixel-alpha window drawn with Direct2D and DirectWrite. It runs in its own process (Windhawk's tool-mod model), animates with springs at your monitor's refresh rate, and parks its render thread entirely when hidden (~0% CPU).

Depending on what's happening, the pill turns into:

| Surface | What it shows |
|---|---|
| **Idle** | Clock, date, weather, and battery |
| **Media** | Album art, live waveform, and synced lyrics; expands into a full player card with controls and a seek bar |
| **Alerts** | Volume, brightness, clipboard, Caps/Num Lock, Windows notifications, battery, USB, Bluetooth, Do Not Disturb, focus timer |
| **Dashboards** | Scroll or swipe between tabs while the island is expanded: Media/Clock, Date & Weather, Hardware Monitor, File Tray, Activity |
| **Privacy dots** | Orange while the microphone is in use, green while the camera is in use |

### Features from the original mod

- Media player via Windows SMTC (Spotify, browsers, VLC, UWP apps…) with album art and controls
- Calendar, weather (wttr.in), hardware monitor, and idle views
- Clipboard previews (text and images), Windows notifications, Bluetooth device battery, Do Not Disturb status
- Focus timer from the right-click menu
- File Tray: drop files on the island and click to open them later
- 8 themes (Obsidian, Graphite, Slate, Nord, Evergreen, Espresso, Plum, Porcelain) plus custom colors, including `#RRGGBBAA` transparency
- Real blur/acrylic backdrop, pill / notch / Windows 11 shapes
- 12 UI languages, custom fonts, clock and date formats
- Hide/show hotkey (default **Ctrl+Alt+D**), multi-monitor and follow-mouse placement

---

## What this fork adds

### Media & music
- **Synced lyrics (LRCLIB).** The current line scrolls in the collapsed pill with a crossfade between lines. The pill only widens while there is a line to sing and shrinks back during intros, outros and instrumental breaks. Toggle lyrics with the button to the right of Next; adjust timing with the **Lyrics delay (ms)** setting.
- **Reliable lyric lookups.** When LRCLIB is overloaded (HTTP 503/429), the lookup is retried instead of being cached as "no lyrics". Retries pause while the track is paused.
- **Album art fallback.** If a player doesn't provide a cover within 4 seconds, the cover is fetched from the iTunes Search API (600×600). If nothing is found, an accent-colored placeholder with a music note is shown, and a late cover fades in.
- **Player switching.** When several apps have a media session (e.g. Spotify and a YouTube tab), `‹` / `›` arrows appear on the media card to choose which player the island shows and controls.
- **Better waveform.** The waveform follows perceived loudness across 6 frequency bands and how "full" the music is, so quiet songs still move but stay smaller than loud ones. It radiates from the center outward.

### Minimize mode
- A button left of **Previous** turns the island into a thin 160×8 glass line 3px from the screen edge, with a faint live waveform and privacy dots. Without music, the same button sits in the top-right corner of the Time tab (the card you get when hovering the idle island).
- Hovering the line opens the full card instantly; moving away tucks it back. Alerts still pop out briefly.
- The choice is remembered across restarts.

### Alerts
- **Inline alerts while music plays.** Instead of a second pill, alerts are drawn inside the music pill (temporarily replacing the lyric line), and the pill stretches to fit them, up to 460px.
- **Activity tab.** A running focus timer or progress task gets its own tab at the end of the dashboard instead of taking over the pill.
- Transient alerts last at most 3 seconds.

### Volume & brightness
- **Instant volume pill.** Driven by Core Audio callbacks instead of a 1-second poll, so it updates on every key press.
- **Windows volume OSD hidden.** While the mod is running with the Volume module on, the island handles the volume keys and the Windows flyout doesn't appear. It comes back when the mod is disabled or the island is hidden.
- **Brightness pill** (laptop panels). Shows brightness changes like volume and hides the Windows brightness flyout, including at 0% and 100%.

### Fullscreen video
- During fullscreen **video** (YouTube, Netflix, VLC, …), the island shrinks into the minimize line instead of covering the video. Games and other fullscreen apps keep the normal island.
- Hover the line to open the card. Volume and brightness still pop up briefly.
- Works with Chromium browsers, Firefox, Zen, VLC, and UWP apps, including paused video.

### Dashboards
- **Redesigned clock.** The Time tab (first tab when you hover the idle island) is now an analog clock card with a ticking accent second hand, next to a greeting for the time of day over a large digital time (AM/PM smaller and muted) and this week's dates with today circled in the accent. The weekday on the Date & Weather tab is easier to read. The collapsed idle strip uses the same accent vector weather icons instead of emoji.
- **Date & Weather tab.** The calendar and weather tabs are merged: a large date on the left, and city, weather icon, temperature, description, and wind / feels-like / humidity chips on the right. Hover a chip for a short explanation (12 languages). Falls back to the monthly calendar when Weather is off.
- **Weather follows your location.** Optionally uses Windows location (`WeatherUseLocation`), then the configured city, then IP. Falls back to **Open-Meteo** when wttr.in is down.
- **Simpler Hardware tab.** Four cards in a 2×2 grid: CPU with its current clock (GHz), RAM with used/total GB, GPU with its temperature (where the GPU reports one), and DISK with an arrow to switch between fixed drives (remembered across restarts). Network cards were removed, and the extra readings are only collected while the tab is visible.

### Other fixes
- The island hides immediately while the screenshot overlay (Win+Shift+S / PrtSc) is open.
- Clicks no longer get "eaten" when opening the card from the minimize line.
- Weather refreshes correctly after the location setting changes or the PC resumes from sleep.

See the [changelog in DOCUMENTATION.md](DOCUMENTATION.md#16-fork-changelog) for per-version details.

---

## Installation

1. Install [Windhawk](https://windhawk.net).
2. In Windhawk, choose **Create a new mod**, replace the template with the contents of [`mod.wh.cpp`](mod.wh.cpp), then click **Compile** and **Enable**.
3. Configure it in the mod's **Settings** tab.

### Windows permissions
- **Notifications:** Settings → Privacy & security → Notifications → turn on *Let apps access your notifications*.
- **Weather by location** (optional): Settings → Privacy & security → Location → turn on *Let desktop apps access your location*, then enable `WeatherUseLocation`.

---

## Usage

| Action | Result |
|---|---|
| Hover the island | Expand it (or click, depending on the Expand-on-hover setting) |
| Scroll wheel / horizontal swipe | Switch dashboard tabs |
| Double-click | Pin the island open |
| Middle-click | Toggle mute |
| Right-click | Menu: hide, focus timer, theme, shape, transparency, pin |
| **Ctrl+Alt+D** | Hide/show the island (configurable under Shortcuts) |
| Drop files on the island | Add them to the File Tray |

Right-click menu choices (theme, shape, pin) last for the current session only; they are re-applied from Mod Settings on restart. Put anything you want to keep in the **Settings** tab.

---

## Development

- Everything is a single file, `mod.wh.cpp`, compiled by Windhawk. There is no local build system or tests.
- You can run a syntax check locally with Windhawk's bundled clang (command in [DOCUMENTATION.md §1](DOCUMENTATION.md#1-overview--building)).
- Logs (`Wh_Log`) appear in the Windhawk editor's log pane.
- Bump `@version` in the header for user-visible changes.

---

## Credits

- **[Himanshu (devcode90)](https://github.com/devcode90)**: original Dynamic Island for Windows.
- **[Sarthak Singh (sarthakaksh)](https://github.com/sarthakaksh)**: focus timer, hover clock, media controls, instant hide shortcut, Bluetooth battery, image clipboard thumbnails, fullscreen auto-hide fixes.
- **[ciizerr](https://github.com/ciizerr)**: layout alignment, dashboard scaling, calendar and weather integration.
- **[ChrisSch-dev](https://github.com/ChrisSch-dev)**: album title, weather word wrapping, sleep-resume fixes, performance and stability.
- **[thevioletto](https://github.com/thevioletto)**: custom fonts, Do Not Disturb integration, album-art color sampling, settings reorganization.
- **[Adiieeee](https://github.com/Adiieeee)**: this fork: synced lyrics, album-art fallback, minimize mode, inline alerts, instant volume/brightness with Windows OSD suppression, fullscreen-video line, player switching, Date & Weather tab, Hardware tab redesign, waveform rework.

Bug reports and feature requests: [devcode90/dynamic-island-for-windows issues](https://github.com/devcode90/dynamic-island-for-windows/issues) or [Windhawk mods issues](https://github.com/ramensoftware/windhawk-mods/issues).
