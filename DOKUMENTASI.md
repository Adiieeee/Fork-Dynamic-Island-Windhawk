# Dokumentasi Dynamic Island for Windows (Fork)

Acuan fungsional lengkap untuk `mod.wh.cpp`. Tujuannya: kalau suatu saat mod rusak (misalnya setelah update Windows), dokumen ini menjelaskan **perilaku yang seharusnya**, **di mana kodenya**, dan **API Windows apa yang dipakai**, supaya perbaikan bisa mengembalikan fungsi yang sama persis.

- **Versi kode saat dokumen ditulis:** `1.6.2` (header `@version`), sekitar 14.900 baris.
- **Nomor baris** di bawah berlaku untuk versi ini dan akan bergeser saat kode berubah. **Selalu cari nama fungsinya** (misalnya `grep -n "DrawMinimizedLine" mod.wh.cpp`). Nama fungsi adalah acuan utama; nomor baris hanya petunjuk awal.
- Konvensi umum proyek (proses tool-mod, threading, lokalisasi) juga diringkas di `CLAUDE.md`.

---

## Daftar isi

1. [Gambaran umum & cara build](#1-gambaran-umum--cara-build)
2. [Arsitektur](#2-arsitektur)
3. [Render loop (jantung mod)](#3-render-loop-jantung-mod)
4. [Fitur: Media (pill musik & dashboard)](#4-fitur-media)
5. [Fitur: Lirik (LRCLIB)](#5-fitur-lirik-lrclib)
6. [Fitur: Cover album & fallback iTunes](#6-fitur-cover-album--fallback-itunes)
7. [Fitur: Minimize (garis tipis)](#7-fitur-minimize-garis-tipis)
8. [Fitur: Notifikasi sesaat & notifikasi inline](#8-fitur-notifikasi-sesaat--notifikasi-inline)
9. [Fitur: Volume (instan + OSD Windows disembunyikan)](#9-fitur-volume)
10. [Fitur: Tab dashboard (Kalender, Cuaca, Hardware, File Tray, Aktivitas)](#10-fitur-tab-dashboard)
11. [Fitur: Idle strip, Game overlay, Privasi, Timer](#11-fitur-idle-strip-game-overlay-privasi-timer)
12. [Fitur: Posisi, tema, backdrop, auto-hide, hotkey, menu](#12-fitur-posisi-tema-backdrop-auto-hide-hotkey-menu)
13. [Settings (Mod Settings) & nilai tersimpan](#13-settings--nilai-tersimpan)
14. [Layanan eksternal (internet)](#14-layanan-eksternal)
15. [Panduan perbaikan setelah update Windows](#15-panduan-perbaikan-setelah-update-windows)
16. [Riwayat perubahan fork ini](#16-riwayat-perubahan-fork-ini)

---

## 1. Gambaran umum & cara build

**Apa ini:** mod [Windhawk](https://windhawk.net) satu file yang menampilkan overlay "Dynamic Island" ala iPhone di Windows. Overlay ini berupa window layered (per-pixel alpha) yang digambar dengan **Direct2D + DirectWrite**. Mod ini bereaksi terhadap media yang diputar, lirik, notifikasi, volume, clipboard, baterai, perangkat, dan lain-lain.

**Struktur file (atas ke bawah):**

| Bagian | Baris (±) | Isi |
|---|---|---|
| `==WindhawkMod==` | 1–11 | id, versi, `@include windhawk.exe`, `@compilerOptions` (library yang di-link: `-lole32 -ld2d1 -ldwrite -lwinhttp -lpdh ...`) |
| `==WindhawkModReadme==` | 13–90 | readme yang tampil di Windhawk |
| `==WindhawkModSettings==` | 92–470 | schema YAML Mod Settings (lihat §13) |
| Include + guard WinRT | 470–537 | `DYNAMIC_ISLAND_HAS_USER_NOTIFICATION_LISTENER`, `DYNAMIC_ISLAND_HAS_BLUETOOTH_WATCHER` |
| `namespace {` … | 538–akhir | seluruh logika |
| Entry point Windhawk | 14716–akhir | `WhTool_*` dan `Wh_*` |

**Build:**
- Dikompilasi **oleh Windhawk** (editor mod → Compile). Tidak ada build lokal, test, maupun linter.
- Library baru harus ditambahkan di baris `@compilerOptions`.
- Syntax check lokal (tanpa menghasilkan binary) bisa memakai clang bawaan Windhawk:

```bash
"/c/Program Files/Windhawk/Compiler/bin/clang++.exe" -fsyntax-only -Wall -std=c++23 \
  -target x86_64-w64-mingw32 -DWH_MOD -DUNICODE -D_UNICODE \
  -DWINVER=0x0A00 -D_WIN32_WINNT=0x0A00 \
  '-DWH_MOD_ID=L"dynamic-island-for-windows-fork"' '-DWH_MOD_VERSION=L"1.6.2"' \
  -include windhawk_api.h -x c++ mod.wh.cpp
```

- **Log:** `Wh_Log(...)` tampil di panel log editor Windhawk. Ini satu-satunya alat debug runtime.

---

## 2. Arsitektur

### 2.1 Model proses (tool-mod)
Windhawk menyuntikkan mod ke `windhawk.exe`, tapi island tidak boleh berjalan di dalam proses itu.

- `Wh_ModInit` (±14765): kalau ini proses launcher, set `g_isToolModProcessLauncher`. Kalau ini proses `-tool-mod <id>`, buat mutex single-instance, panggil `WhTool_ModInit`, lalu hook entry point.
- `Wh_ModAfterInit` (±14844): launcher menjalankan ulang `windhawk.exe -tool-mod "<WH_MOD_ID>"` lewat `CreateProcessInternalW`.
- `WhTool_ModInit` (±14716): `LoadSettings()`, baca nilai tersimpan (`LyricsEnabled`, `IslandMinimized`), lalu `StartThreads()`.
- `WhTool_ModUninit` (±14735): kirim `WM_CLOSE`, `StopThreads()`. Setelah itu proses `ExitProcess(0)`.
- **Aturan:** logika overlay ada di `WhTool_*`, bukan di `Wh_*`.

### 2.2 Thread
Semua thread dibuat di `StartThreads()` (±14636) dan dihentikan oleh `StopThreads()` (±14666) lewat `g_stopEvent` (manual-reset), dengan join maksimal 3 detik per thread.

| Thread | Fungsi (±baris) | Tugas |
|---|---|---|
| Render | `RenderThreadProc` (13616) | membuat window overlay, memiliki `Renderer`, menjalankan frame loop, dan memproses pesan window |
| Media | `MediaThreadProc` (3889) | poll SMTC tiap 1,5 detik: judul, artis, album, cover, posisi |
| Audio | `AudioThreadProc` (5853) | WASAPI loopback untuk data waveform (hanya saat `g_audioCaptureNeeded`) |
| Cuaca | `WeatherThreadProc` (5062) | WinHTTP ke wttr.in |
| Lirik | `LyricsThreadProc` (5573) | LRCLIB untuk lirik ber-timestamp |
| Cover fallback | `ArtworkThreadProc` (5759) | iTunes Search kalau SMTC tidak memberi cover |
| Volume watcher | `VolumeWatcherThreadProc` (6344) | callback volume dari Windows (event, bukan poll) |
| Notifikasi | `NotificationThreadProc` (4237) | `UserNotificationListener` WinRT |
| Bluetooth | `BluetoothThreadProc` (4780 / stub 4865) | `DeviceWatcher` WinRT untuk perangkat BT terhubung |
| Keyboard | `KeyboardThreadProc` (12832) | hook `WH_KEYBOARD_LL`: Caps/Num Lock dan tombol volume |
| Mouse | `MouseThreadProc` (12928) | hook `WH_MOUSE_LL`, hanya saat island di-park, untuk unhide-on-hover |

DND tidak memakai thread sendiri, tapi subscription WNF (`SubscribeDndNotification`, ±4199).

### 2.3 State bersama
- **`SharedState g_state`** (struct ±1236) dijaga **`g_stateMutex`**. Isinya: `media`, `lyrics`, `clipboard`, `notification`, `volume`, `capsLock`, `timer`, `device`, `bluetoothDevice`, `doNotDisturb`, `battery`, `progress`, `system` (CPU/RAM/GPU/disk/net/volume/privasi/renderFps), `weather`, `waveform[48]` (ring buffer, ditulis lewat `waveformWrite`), dan `fileTrayItems`.
- **`Settings g_settings`** (struct ±945) dijaga `g_settingsMutex`. Render loop memakai `GetSettingsCopy()` (±1303) sekali per frame.
- **Atomic global penting** (±1290–1340): `g_layoutDirty` (minta repaint/reposisi), `g_idleTab` (tab dashboard aktif), `g_hoveredMediaButton` / `g_pressedMediaButton`, `g_scrubbing`, `g_manuallyHidden`, `g_autoHiddenParked`, `g_lyricsEnabled`, `g_lyricsDelayMs`, `g_islandMinimized`, `g_minimizedEdgeFraction`, `g_activityTabVisible`, `g_audioCaptureNeeded`.

### 2.4 Pesan window (`OverlayWndProc`, ±12972)

| Pesan | Fungsi |
|---|---|
| `WM_CREATE` | `AddClipboardFormatListener`, `DragAcceptFiles` |
| `WM_DROPFILES` | File Tray: simpan path file yang di-drop |
| `WM_APP_APPLY_HOTKEY` / `WM_APP_APPLY_BACKDROP` | terapkan hotkey/backdrop di thread pemilik window (dikirim dari `LoadSettings`) |
| `WM_APP_MOUSE_WAKE` | bangun dari park karena hover |
| `WM_APP_VOLUME_KEY` | terapkan tombol volume yang ditahan hook (`ApplyVolumeKey`) dan tulis log diagnostik |
| `WM_APP_CAPSLOCK` | catat state Caps/Num Lock (cek setting `capsLock`) |
| `WM_DEVICECHANGE` | notifikasi USB/COM (`DBT_DEVICEARRIVAL` / `DBT_DEVICEREMOVECOMPLETE`) |
| `WM_POWERBROADCAST` | setelah resume dari sleep: refresh cuaca, lalu nudge |
| `WM_CLIPBOARDUPDATE` | `CaptureClipboard` (±6889) |
| `WM_SETCURSOR` / `WM_MOUSEMOVE` / `WM_LBUTTONDOWN` / `WM_LBUTTONUP` | hit-test tombol media, scrubber, File Tray, swipe tab, klik untuk expand |
| `WM_MOUSEWHEEL` | ganti tab dashboard (debounce 150ms) |
| `WM_MBUTTONUP` | toggle mute (`ToggleEndpointMute`) |
| `WM_LBUTTONDBLCLK` | toggle "Pin expanded" |
| `WM_RBUTTONUP` | menu klik kanan (`ShowContextMenu`, ±7310) |
| `WM_HOTKEY` | hotkey hide/show (default Ctrl+Alt+D) |

ID `WM_APP` yang terpakai: `+0x442` LAYOUT_CHANGED, `+0x443` NEW_EVENT, `+0x444` CAPSLOCK, `+0x446` MOUSE_WAKE, `+0x447` APPLY_HOTKEY, `+0x448` APPLY_BACKDROP, `+0x449` VOLUME_KEY.

### 2.5 Kopling layout ↔ hit-test
Gambar dan hit-test mouse memakai **konstanta yang sama** di namespace `MediaLayout` (±559), `FileTrayLayout` (±644), `IdleStripLayout` (±681), dan `GameOverlayLayout` (±715). Hit-test media memakai `MediaContentFromClient` (±1615), yang membaca geometri frame terakhir lewat seqlock, lalu `MediaTransportHitTest` (±1674), `MediaScrubFractionFromContent` (±1706), dan `MediaArtHitTest`.

**Aturan:** kalau mengubah posisi elemen, ubah konstantanya, jangan angka lepas di `Draw*`, supaya area klik tidak bergeser dari gambarnya.

---

## 3. Render loop (jantung mod)

`RenderThreadProc` (±13616). Urutan tiap iterasi:

1. **Proses pesan** (`PeekMessage`/`DispatchMessage`). `WM_APP_NEW_EVENT` memicu animasi "nudge" (pill sedikit memantul).
2. **Park:**
   - Kalau `g_manuallyHidden`: window disembunyikan, lalu `MsgWaitForMultipleObjects(INFINITE)`, jadi CPU sekitar 0%.
   - Kalau `g_autoHiddenParked` (auto-hide idle atau fullscreen video): sama, dengan cek ulang `IsForegroundFullscreenVideo` tiap 1,5 detik.
3. **Snapshot state** (copy `g_state` di bawah lock), lalu bersihkan notifikasi yang sudah kedaluwarsa.
4. **Pilih aktivitas:**
   - `ChooseActivities` (±12699) memberi `primary` dan, opsional, `secondary` (dua pill berdampingan).
   - `ActivityForKind` (±12595) memberi ukuran dasar per jenis.
   - `ChooseInlineAlert` (±12676) memilih notifikasi yang digambar **di dalam** pill musik (§8).
   - `g_activityTabVisible` = timer atau progress aktif (§10).
5. **Hover:** rect pill dihitung dari `GetWindowRect`, ditambah padding saat mode minimize (§7). `isHoverExpanded` mengikuti setting Expand-on-hover (atau klik), dan dipaksa `hover` saat minimize.
6. **Auto-hide / fullscreen video:** `IsForegroundFullscreenVideo` (±3625) dicek tiap 0,5 detik **sebelum** `ChooseInlineAlert` (±14117), lalu menghasilkan `videoSuppressed` (±14112). Setelah itu `isHidden` dan `EnsureTopmost` (tiap 1 detik). Detail: §12 "Fullscreen video".
7. **Poll sistem tiap 1 detik:** `UpdateSystemSnapshot` (±6124) membaca CPU, RAM, disk, dan level volume. GPU dan jaringan hanya dibaca kalau tab Hardware atau game overlay tampil.
8. **Ukuran target:**
   - Idle: `MeasureIdleStrip`.
   - Game overlay: `GameOverlayLayout`.
   - Media: 380×184 saat expanded, atau lebar notifikasi inline.
   - Minimize: garis 160×8.
   - Hidden: 0×0.
9. **Spring:** `widthSpring`, `heightSpring`, `nudgeSpring`, dan `edgeSpring` (untuk minimize). Kekakuan dan redaman mengikuti `AnimationStyle` (Smooth/Default/Bouncy/Snappy) dan `AnimationSpeed`.
10. **Perlu render?** Render dilakukan kalau ada spring yang bergerak, `g_layoutDirty`, perubahan hover, animasi kontinu (media diputar, marquee, notifikasi), titik privasi, atau data berubah (art generation, judul, volume, baterai, dll.). Repaint per detik dilakukan untuk timer dan jam yang menampilkan detik.
11. **`renderer.Render(...)`** (±7687): buat backing bitmap seukuran pill + padding (`kRenderPadX` 28, `kRenderPadY` 22), `PositionOverlayWindow`, gambar, lalu `UpdateLayeredWindow`.
12. **Pacing:** target FPS dari setting (0 = refresh rate monitor). Kalau tidak ada yang berubah, loop poll tiap 16ms.

**Renderer** (`class Renderer`, ±7651): `Initialize`, `Render`, `DrawPill` (±8334, switch per `IslandKind`), `DrawPillSurface`, `DrawSoftShadow`, `DrawPrivacyDots`, `EnsureBrushes` (±8202, token warna tema), dan `EnsureTextFormats` (±7999, font dan skala teks).

---

## 4. Fitur: Media

**Sumber data:** Windows **SMTC** (`GlobalSystemMediaTransportControlsSessionManager`) di `MediaThreadProc` (±3889), dengan poll tiap 1,5 detik.
- `SelectActiveMediaSession` (±3851) memilih sesi yang sedang aktif.
- Data yang dibaca: `title`, `artist`, `albumTitle`, `sourceAppUserModelId`, `PlaybackStatus`, timeline (`positionTicks`, `endTicks`, `lastUpdatedTicks`), dan thumbnail (cover).
- **Fallback VLC:** kalau judul kosong dari VLC, judul diambil dari caption window VLC.
- **Ikon sumber:** `FindMediaSourceIcon` (±3681), dengan badge teks (misalnya "SP") sebagai cadangan.

**Pill collapsed** (`DrawMedia`, ±10352, cabang "Collapsed UI"):
- Kiri: album art bulat. Kanan: waveform (`DrawWaveform`, ±11542) saat playing, atau titik-titik diam saat pause.
- Tengah: **notifikasi inline** (§8) kalau ada; selain itu **lirik** (§5).
- Lebar dasar `MediaLayout::kCollapsedWidth` = 150. Dengan lirik: `kCollapsedLyricsWidth` = 320.

**Card expanded (hover atau pin):** 380×184 (`kExpandedWidth`/`kExpandedHeight`).
- Art 64px, judul/artis/album dengan marquee, waveform, scrubber (bisa di-drag untuk seek lewat `SeekMediaToTicks`, ±7042), elapsed/remaining.
- Baris kontrol (y = `kControlsY` 148, x relatif ke tengah):

| Offset X | Tombol | Command hit-test |
|---|---|---|
| −118 | **Minimize** toggle (`DrawMinimizeToggleButton`, ±11130) | 4 |
| −64 | Previous | 0 |
| 0 | Play/Pause (radius 22) | 1 |
| +64 | Next | 2 |
| +118 | **Lirik** toggle (`DrawLyricsToggleButton`, ±11173) | 3 |

- Command 0–2 dikirim lewat `SendMediaTransportCommand` (±7108, SMTC `TrySkipPrevious`/`TryTogglePlayPause`/`TrySkipNext`). Command 3 dan 4 ditangani lokal di `WM_LBUTTONUP`.
- Animasi tekan: `mediaBtnPress_[5]` dan `UpdateMediaButtonAnimations` (±11088).
- Klik area kosong card membuka aplikasi media (`OpenRelevantApp`, ±7219).

**Pengaturan terkait:**
- `Media` dan `MediaAutoExpand`: expand otomatis selama 5 detik saat lagu berganti.
- `MediaExpandBlocklist`: daftar app yang tidak memicu auto-expand.

**Kalau rusak:** gejalanya pill musik tidak muncul atau judul kosong. Penyebabnya biasanya API SMTC berubah atau gagal; cek log `"WinRT media session unavailable"`.

---

## 5. Fitur: Lirik (LRCLIB)

**Perilaku yang seharusnya:**
- Lirik ber-timestamp (synced) dicari di **lrclib.net** saat lagu berganti. Lirik tanpa timestamp dianggap tidak ada.
- Baris yang sedang dinyanyikan tampil di tengah pill collapsed, di antara art dan waveform, dengan crossfade 180ms antar baris. Baris yang panjang digeser pelan selama baris itu tampil.
- **Lebar dinamis:** pill hanya memanjang ke 320px saat ada baris yang akan atau sedang dinyanyikan.
  - Mulai memanjang `kGrowLeadSeconds` = **0,35 detik** sebelum baris masuk.
  - Intro, outro, dan jeda instrumental **≥ 4 detik** (`kShortGapSeconds`): kembali 150px dengan ikon ♪ kecil.
  - Jeda di bawah 4 detik di antara baris: tetap lebar.
  - Baris yang sedang dinyanyikan tidak dipotong walaupun jeda sudah dekat.
- **Toggle** lewat tombol di kanan Next. Status disimpan di `Wh_SetIntValue("LyricsEnabled")`.
- **Setting `LyricsDelayMs`** (−10000..10000): nilai positif membuat lirik muncul lebih lambat.

**Kode:**

| Fungsi | ±Baris | Peran |
|---|---|---|
| `LyricsTrackKey` | 5268 | kunci lagu = `title + U+001F + artist` (dipakai thread, lebar, dan gambar) |
| `LyricsShowable` | 5276 | lirik aktif, status Found, dan kunci cocok dengan lagu sekarang |
| `MediaPlayheadSeconds` / `LyricsPlayheadSeconds` | 5291 / 5300 | posisi lagu (`positionTicks` + waktu sejak `lastUpdatedTicks`), dikurangi delay |
| `LyricLineIndexAt`, `IsLyricGap` | 5284 / 5305 | baris aktif; baris kosong atau hanya ♪/♫ dianggap jeda |
| `LyricsWantWidePill` | 5322 | keputusan lebar dinamis |
| `NextJsonStringField` | 5393 | parser string JSON minimal (unescape `\n \" \\ \/ \uXXXX` + surrogate) |
| `ParseLrc` | 5470 | `[mm:ss.xx] teks`, termasuk beberapa tag per baris; tag metadata diabaikan |
| `SimplifyTrackTitle` | 5514 | buang " - Remastered", " (feat.", " [" untuk pencarian longgar |
| `FetchLyricsFromLrclib` | 5530 | `/api/get` (judul, artis, album, durasi) lalu `/api/search` (judul penuh, lalu judul yang disederhanakan) |
| `LyricsThreadProc` | 5573 | loop 500ms, tunggu durasi ≤ 3 detik, cache 30 lagu, retry 60 detik kalau jaringan gagal |
| `DrawCollapsedLyrics` / `DrawLyricLine` | 10985 / 11048 | gambar baris, crossfade, scroll, ikon ♪ saat lebar normal |
| `ActivityForKind` (case Media) | 12595 | memakai `LyricsWantWidePill` untuk lebar |

**Kalau rusak:** lirik tidak pernah muncul. Cek apakah API LRCLIB berubah (field `syncedLyrics`), atau ada log `"Lyrics: LRCLIB unreachable"`.

---

## 6. Fitur: Cover album & fallback iTunes

**Perilaku:** cover diambil dari thumbnail SMTC. Kalau setelah **4 detik** sebuah lagu masih tanpa cover, cover diambil dari **iTunes Search API** (600×600, dicocokkan dengan nama artis). Cover dari SMTC selalu diutamakan.

| Fungsi | ±Baris | Peran |
|---|---|---|
| `ReadWinRtStreamBytes` | 3777 | baca stream thumbnail **bertahap** (`InputStreamOptions::Partial`, chunk 64KB, sampai habis, maks 8MB), dibungkus try/catch |
| `DecodeImageBytesToPixels` | 3061 | WIC decode ke BGRA premultiplied, mengisi `generation` dan `sampledAccent` (warna aksen UI) |
| Logika "settle" cover | di `MediaThreadProc` | selama 3 detik setelah ganti lagu, cover terus diambil ulang (browser sering terlambat mengganti cover); cover lama dipertahankan kalau fetch sesaat gagal |
| `ArtistNamesMatch` | 5686 | pencocokan artis longgar (contains, artis pertama sebelum `,`/`&`) |
| `FetchArtworkFromItunes` | 5703 | `itunes.apple.com/search?media=music&entity=song&limit=10&term=...`, ganti `100x100bb` jadi `600x600bb` |
| `ArtworkThreadProc` | 5759 | cek tiap 1 detik, grace 4 detik, cache 30, retry 30 detik saat jaringan gagal, log `"Album art: ..."` |
| `HttpGet` | 4969 | WinHTTP GET (aman untuk biner, timeout 5/5/8/8 detik) |
| `DrawAlbumArt` | 11301 | gambar cover; kalau kosong: ikon sumber atau badge teks |

**Kalau rusak:** yang tampil badge "SP" padahal lagu punya cover. Cek pembacaan stream (API Storage.Streams) dan decode WIC.

---

## 7. Fitur: Minimize (garis tipis)

**Perilaku yang seharusnya:**
- Tombol di kiri Previous mengaktifkan mode minimize (disimpan di `Wh_SetIntValue("IslandMinimized")`).
- Dalam mode minimize, island menjadi **garis 160×8** bergaya glass yang menempel **3px dari tepi atas layar** (atau tepi bawah untuk posisi Bottom). Di garis tampil waveform tipis berwarna aksen yang memancar dari tengah dengan ujung memudar, atau titik diam saat pause. Titik privasi tampil di ujung kanan.
- **Hover:** hitbox = garis + 4px ke samping + 4px ke bawah + sampai tepi layar, **tanpa jeda**. Hover selalu membuka card penuh, walaupun Expand-on-hover mati. Card terbuka dari tepi atas, dan menutup lagi saat mouse keluar card.
- Island tetap di tepi selama minimize, juga saat terbuka, supaya tidak terjadi loop buka-tutup.
- Notifikasi sesaat tetap memunculkan pill, lalu kembali jadi garis.
- Garis tidak ikut auto-hide idle, tapi ikut tersembunyi saat fullscreen video (hover pun tidak membukanya).
- Tidak berlaku saat game overlay aktif.

**Kode:**

| Bagian | ±Baris | Keterangan |
|---|---|---|
| Konstanta `MediaLayout` | 590–612 | `kMinimizeButtonOffsetX` −118, `kMinimizedLineWidth` 160, `kMinimizedLineHeight` 8, `kMinimizedHoverPadX` 4, `kMinimizedHoverPadBelow` 4, `kMinimizedLineEdgeGap` 3 |
| `g_islandMinimized`, `g_minimizedEdgeFraction` | 1332 / 1336 | state dan fraksi animasi ke tepi (0..1) |
| `PositionOverlayWindow` | 2779 | menggeser `y` window sebesar `shift × edgeFraction` agar garis 3px dari tepi |
| Render loop: hover (`lineResting`) | 13976 | hitbox diperlebar saat `heightSpring < 30` |
| Render loop: `minimizedMode` | 14047 | `isHoverExpanded = hover`, `isHidden = false` |
| Render loop: `minimizedLine` | 14175 | override ukuran ke garis, `renderer.SetMinimizedLine` |
| Render loop: `edgeSpring` | 14266 | spring critically damped, set `g_layoutDirty` saat bergerak |
| `Renderer::Render` | 7687 | menggambar garis (`DrawMinimizedLine`) saat `minimizedLine_` aktif atau tinggi < 24 dalam mode minimize |
| `DrawMinimizedLine` | 11575 | isi pill alpha 0.5, kilau gradient putih 0.16→0, batang waveform 1.6px |
| `WM_LBUTTONUP` cmd 4 | 13481 | toggle dan simpan |
| `expanded` saat klik | 13430 | minimize dianggap expanded supaya klik pertama tidak "dimakan" |

---

## 8. Fitur: Notifikasi sesaat & notifikasi inline

### 8.1 Jenis notifikasi (pemicu & durasi)

| Jenis (`IslandKind`) | Pemicu (kode) | Durasi | Setting |
|---|---|---|---|
| Clipboard | `WM_CLIPBOARDUPDATE` → `CaptureClipboard` (±6889): teks (`CF_UNICODETEXT`), gambar (`CF_DIB`), ikon app pemilik | 2,5 detik | `Clipboard` |
| CapsLock (Caps/Num) | hook keyboard → `WM_APP_CAPSLOCK` | 2,5 detik | `CapsLock` |
| Volume | `VolumeChangeCallback` (§9) | 1,8 detik | `Volume` |
| Notification | `NotificationThreadProc` (±4237): `UserNotificationListener.GetNotificationsAsync(Toast)` dipoll tiap 1 detik, ID yang sudah dilihat dilewati, ikon app 32px | 3 detik | `NotificationRespectDnD` (diam saat DND) |
| BatteryLow | `UpdateBatterySnapshot` (±5985): `GetSystemPowerStatus`; charger dicolok/dicabut, atau turun ke 20%/10% | 3 detik | `Battery` |
| Device | `WM_DEVICECHANGE`: USB Drive (`DBT_DEVTYP_VOLUME`), COM, lainnya | 3 detik | — |
| Bluetooth | `BluetoothThreadProc` (±4780): `DeviceWatcher` dengan selector Connected (Classic + LE), baterai lewat GATT | 3 detik | `BluetoothIndicator`, `BluetoothShowBattery` |
| DoNotDisturb | WNF `kWnfQuietHoursActiveProfileChanged` (`WnfDndCallback` ±4166) | 3 detik | `DoNotDisturbIndicator` |
| Timer selesai | render loop (`finishedExpiresAt`) | 3 detik | `TimerModule` |

Kartu penuh per jenis (dipakai saat **tidak ada musik**): `DrawClipboard` (±11685), `DrawNotification` (11755), `DrawVolume` (11829), `DrawTimer` (11988), `DrawCapsLock` (12078), `DrawDevice` (12142), `DrawDoNotDisturb` (12210), `DrawBluetoothDevice` (12309), `DrawBattery` (12409). Setting `StatusCountdownProgress` menambah garis hitung mundur, dengan konstanta `ttl` di masing-masing fungsi yang harus sama dengan durasi di atas.

### 8.2 Notifikasi inline (saat musik diputar)
**Perilaku:**
- Selama ada pill musik, **tidak ada pill kedua**. `ChooseActivities` hanya mengembalikan `{Media}`.
- Notifikasi sesaat digambar dalam satu baris di tengah pill (ikon, teks utama, bar opsional, keterangan), **menggantikan lirik sementara**.
- Lebar pill mengikuti isi notifikasi, minimal 150 dan maksimal **460** (`kInlineAlertMaxWidth`). Teks yang terlalu panjang dipotong dengan "…".
- Isi notifikasi muncul perlahan mengikuti pill yang memanjang.
- Saat hover, card expanded tampil tanpa notifikasi.
- **Tanpa musik:** notifikasi tetap tampil sebagai pill sendiri seperti versi asli.

| Fungsi | ±Baris | Peran |
|---|---|---|
| `ChooseInlineAlert` | 12676 | notifikasi prioritas pertama yang aktif (urutan sama dengan `ChooseActivities`) |
| `MediaPillShown` / `ChooseActivities` | 12695 / 12699 | ada media → hanya `{Media}` |
| Render loop (`inlineAlert`) | 13936 | `inlineAlert`, `hasInlineAlert` masuk `isTransientAlert`, `renderer.SetInlineAlert`, lebar dari `MeasureInlineAlert` |
| `InlineAlertContent` (struct) | 7608 | model isi satu baris |
| `BuildInlineAlertContent` | 10680 | **satu-satunya sumber isi**, dipakai pengukur dan penggambar |
| `MeasureInlineAlert` | 7789 | lebar = reserve kiri 48 + kanan 52 (+22 privasi) + padding 2×6 + baris |
| `InlineAlertRowWidth`, `CollapsedMediaReserve`, `DrawTrimmedText` | 10659 / 10637 / 10806 | ukur, reserve, teks dengan ellipsis |
| `DrawInlineAlert`, `DrawInlineAlertIcon` | 10909 / 10830 | gambar baris (layer alpha), ikon (bitmap, glyph, keycap, baterai, ring) |
| `DrawMedia` collapsed | 10620 | notifikasi atau lirik di area tengah |

---

## 9. Fitur: Volume

**Perilaku yang seharusnya:**
- Notifikasi volume muncul **seketika** (≤ 1 frame), dan angka berganti setiap tekan atau saat ditahan.
- Selama mod aktif dan setting `Volume` nyala, **OSD volume Windows tidak muncul**. Island yang menangani tombol volume.
- OSD Windows kembali saat mod dimatikan, setting `Volume` dimatikan, atau island disembunyikan manual.

**Cara kerja:**
1. **Deteksi berbasis event:**
   - `VolumeChangeCallback` (±6259, `IAudioEndpointVolumeCallback::OnNotify`) menerima `fMasterVolume` dan `bMuted` dari Windows untuk setiap perubahan (tombol, slider, aplikasi), lalu memanggil `ReportVolumeChange` (±6238), yang mengisi `g_state.volume` dan `system.volumePercent`, lalu `TriggerNudge`.
   - `DefaultDeviceWatcher` (±6305, `IMMNotificationClient`) mendaftarkan ulang callback saat perangkat output default berganti.
   - Keduanya dijalankan oleh `VolumeWatcherThreadProc` (±6344).
2. **`UpdateSystemSnapshot`** (±6124) hanya membaca level untuk dashboard dan **tidak memicu notifikasi**. Dulu fungsi ini memicu notifikasi, yang menjadi sumber delay 0–1 detik.
3. **Menahan OSD:** OSD Windows muncul karena shell menangani tombol volume, bukan karena volumenya berubah.
   - `LowLevelKeyboardProc` (±12785) menelan `VK_VOLUME_UP`/`VK_VOLUME_DOWN`/`VK_VOLUME_MUTE` (`return 1`), termasuk tombol tiruan (`LLKHF_INJECTED`, misalnya dari software OEM untuk Fn + F), lalu memposting `WM_APP_VOLUME_KEY` saat key-down (termasuk auto-repeat).
   - `ApplyVolumeKey` (±6412) menjalankan `VolumeStepUp`/`VolumeStepDown`/`SetMute` di thread window, dengan endpoint di-cache dan retry sekali.
   - `KeyboardHookWanted` (±12821) memasang hook kalau `capsLock || volume`.
   - `LoadSettings` memanggil `NotifyKeyboardThreadSettingChanged` saat salah satunya berubah.
4. **Log diagnostik:** `Volume key: vk=0x.. injected=..` (maksimal 1 per 2 detik per kombinasi), untuk memastikan Fn + F atau knob keyboard tertangkap.

**Batasan:** volume dari software OEM yang mengubah volume langsung (tanpa tombol), atau dari tombol headset Bluetooth, tidak lewat hook, jadi OSD-nya mungkin tetap muncul. Notifikasi island tetap seketika lewat callback.

---

## 10. Fitur: Tab dashboard

Saat expanded (hover atau pin), tab diganti dengan scroll roda mouse atau swipe horizontal (> 40px dalam < 500ms). Titik paginasi vertikal ada di kanan.

**Urutan tab** (harus sama dengan `ActiveTabCount` ±1406, dan dua daftar `activeTabs` di `DrawIdleDashboard` ±9924 dan `DrawMedia` ±10352):
1. Utama: Media (kalau ada musik) atau Jam (`DrawTimeDashboard` ±9774)
2. Kalender: `DrawCalendarDashboard` (±8821), grid bulan dengan hari ini ditandai warna aksen
3. Cuaca (opsional): `DrawWeatherDashboard` (±9283)
4. Hardware Monitor (opsional): `DrawHardwareMonitorDashboard` (±9634). CPU, RAM, GPU (PDH `\GPU Engine(*)\Utilization Percentage`), disk, jaringan (PDH `\Network Interface(*)\Bytes Sent/Received/sec`). Bar menjadi kuning di atas 75% dan merah di atas 90%.
5. File Tray (opsional): `DrawFileTrayDashboard` (±9478). File di-drop ke island (`WM_DROPFILES`), hanya path yang disimpan, klik untuk membuka (`ShellExecuteExW`), kapasitas lewat `FileTrayMaxItems`.
6. **Aktivitas** (otomatis, paling akhir): `DrawActivityDashboard` (±11903). Muncul saat Focus Timer berjalan atau progress aktif.
   - Timer: label Focus/Break (+Paused), countdown `mm:ss` besar, bar oranye (fokus) atau hijau (break), dan hint "Right-click the island to pause or stop".
   - Progress: label dan bar persen.
   - `ActivityTabIndex` (±1442) selalu indeks terakhir, jadi tab lain tidak bergeser.

**Indeks:** `NormalizedTabIndex` (±1415, modulo jumlah tab), `HardwareMonitorTabIndex` (±1426, dipakai untuk memutuskan apakah GPU/net perlu dipoll), `FileTrayTabIndex` (±1433).

**Cuaca:** `WeatherThreadProc` (±5062): `wttr.in/<kota>?format=j1` (HTTPS, fallback HTTP), membaca `current_condition`. Backoff kalau gagal; refresh saat kota berubah atau resume dari sleep. Setting: `Weather`, `WeatherCity` (kosong = lokasi dari IP), `WeatherFahrenheit`.

---

## 11. Fitur: Idle strip, Game overlay, Privasi, Timer

- **Idle strip** (tidak ada musik maupun notifikasi): jam dan cuaca. Lebar diukur dari teks lewat `MeasureIdleStrip` (±7860), digambar oleh `DrawIdleDashboard` (±9924). Setting: `ClockFormat`, `ShowSeconds`, `DateFormat`, `DateFirst`.
- **Game overlay:** `DrawGameOverlay` (±10134) mengganti idle strip dengan kartu FPS, CPU, GPU, RAM, dan disk.
  - **Catatan:** FPS yang ditampilkan adalah **FPS render island sendiri** (`system.renderFps`), bukan FPS game.
  - Setting: `GameOverlay` dan `GameOverlayShow*`/`Compact`. Bisa juga di-pin dari menu (`GameOverlayPinned`).
- **Privasi:**
  - `UpdatePrivacyIndicators` (±6607) dan `IsDeviceActiveViaRegistry` (±6527) membaca registry `HKCU`/`HKLM\...\CapabilityAccessManager\ConsentStore\{microphone|webcam}`: aplikasi yang sedang memakai perangkat punya `LastUsedTimeStop = 0`.
  - Titik oranye untuk mikrofon dan hijau untuk kamera, digambar oleh `DrawPrivacyDots` (±8597), atau di ujung garis minimize.
  - Setting: `PrivacyDots`, `PrivacyDotsMic`/`Cam`, dan warnanya.
- **Focus Timer:** dari menu klik kanan, `StartFocusTimer` (±6976; 25/50 menit fokus, 5 menit break), `ToggleTimerPause` (±6990), `StopFocusTimer` (±7008).
  - Saat ada musik: hanya di tab Aktivitas.
  - Tanpa musik: tampil sebagai pill timer juga.
  - Selesai: notifikasi 3 detik.
- **Progress:** `UpdateProgressSnapshot` (±6600) hanya membaca `Wh_GetIntValue("ProgressPercent")`. **Di dalam mod tidak ada yang mengisi nilai ini** (hanya direset ke −1), jadi praktis tidak pernah aktif. Ini bukan bug baru.

---

## 12. Fitur: Posisi, tema, backdrop, auto-hide, hotkey, menu

- **Posisi:** `GetAnchorWorkRect` (±2721; monitor target: primary, 1–5, atau follow mouse) dan `PositionOverlayWindow` (±2779; Position, OffsetX/Y, `ResolveOffsetY` ±2759 untuk offset terpisah saat expanded, dan geser ke tepi saat minimize). `GetIslandDockRect` (±3007) adalah area hover untuk island yang di-park.
- **Bentuk:** Pill (default), Windows 11 flyout (`w11Style`), macOS notch (`notchStyle`, menempel ke atas), `BorderMergedMode`.
- **Tema:** `ThemePreset` (Obsidian, Graphite, Slate, Nord, Evergreen, Espresso, Plum, Porcelain, Custom), `ThemePalette` (±872), token warna di `EnsureBrushes`. Warna hex menerima `#RRGGBBAA`. Warna aksen diambil dari cover album (`sampledAccent`) dengan transisi halus.
- **Backdrop:** namespace `Backdrop` (±2598): `SetWindowCompositionAttribute` (undocumented) untuk blur atau acrylic, dengan region dibatasi ke bentuk pill (`ApplyBackdropRegion` ±2860). Diterapkan lewat `WM_APP_APPLY_BACKDROP` (`ApplyBackdropMaterial` ±2641).
- **Auto-hide:** `AutoHideIdleSeconds` (−1 = sembunyi kecuali ada notifikasi, 0 = tidak pernah, >0 = detik idle), `AutoHideFullscreen` (khusus video, lihat di bawah), `UnhideOnHover` (hook mouse saat di-park). Park berarti `ShowWindow(SW_HIDE)` dan CPU sekitar 0%.
- **Fullscreen video (1.6.2):** island **hanya** hilang saat fullscreen **video**. Game, browser F11 tanpa video, dan aplikasi fullscreen lain tetap menampilkan island seperti biasa.
  - `MediaThreadProc` (±4256) menyimpan AUMID **semua** sesi SMTC yang `Playing` ke `g_state.playingMediaSources` tiap 1,5 detik, bukan hanya sesi yang diikuti pill.
  - `IsForegroundFullscreenVideo` (±3625) bernilai true kalau `IsForegroundFullscreen` (±1984: client rect jendela foreground menutupi monitornya) **dan** jendela itu milik salah satu sumber tersebut. Hasilnya sticky per HWND, jadi video yang di-pause tetap dianggap video sampai keluar fullscreen atau jendela foreground berganti.
  - `WindowBelongsToMediaSource` (±3562) mencocokkan jendela dengan AUMID lewat tiga cara:
    - (a) `GetApplicationUserModelId` (`PackagedAppIdForPid`) untuk aplikasi UWP. Frame `ApplicationFrameHost.exe` diganti dengan proses child-nya.
    - (b) Nama exe tanpa `.exe` yang muncul di AUMID, plus alias chrome, edge, dan vlc.
    - (c) Firefox: `firefox.exe` dengan AUMID berupa hash hex (`LooksLikeFirefoxAumid`).
  - Saat `videoSuppressed`, island **benar-benar hilang**:
    - `hover` dipaksa false.
    - `recentTrackChange` dimatikan.
    - `isTransientAlert` hanya untuk Volume.
    - `inlineAlert` hanya Volume.
    - Supresi tidak bisa dilewati oleh pin, hover, atau mode minimize.
    - `LowLevelMouseProc` (±13047) tidak membangunkan island yang di-park.
  - Yang masih boleh muncul hanya **volume** (OSD Windows disembunyikan, jadi ini satu-satunya umpan balik) dan **titik privasi**.
  - Hotkey hide/show memaksa island tampil (`g_fullscreenOverrideVisible`). Status ini direset saat keluar dari fullscreen.
  - `g_isFullscreen` / `isFullscreen` sekarang berarti "fullscreen video". Gate lain yang memakainya (game overlay, HW monitor, `EnsureTopmost`) otomatis tidak aktif di game fullscreen.
- **Topmost:** `EnsureTopmost` (±2952) mengambil kembali posisi z-order tiap 1 detik. `AlwaysOnTop` mengatur parent ke Progman kalau mati.
- **Click-through:** `SetClickThrough` (±6949) untuk idle tanpa hover, atau sementara saat Ctrl+hover (island menjadi tembus pandang).
- **Hotkey hide/show:** `ApplyHideShowHotkey` (±2687) memanggil `RegisterHotKey` di thread window. Default Ctrl+Alt+D. Status disimpan di `ManuallyHidden`.
- **Menu klik kanan** (`ShowContextMenu` ±7310): Hide/Show, Dismiss, Focus Timer (25/50 menit, break 5 menit, pause, stop), status privasi, Pin expanded, Show/Hide game overlay, gaya W11/Pill, gaya Notch, Expand on Hover/Click, Transparency 100/85/70/55/reset, Theme, dan Open Windhawk settings. Pilihan di menu ini **per sesi** (di-reset dari settings saat restart), kecuali yang disimpan di nilai `Wh_*IntValue` (§13).
- **Gestur lain:** klik tengah untuk mute, double-click untuk pin expanded.
- **Lokalisasi:** `Loc()` (±1554) menerjemahkan string ke 12 bahasa (`UiLanguage` ±1454). String yang belum ada di tabel tampil dalam bahasa Inggris. Beberapa string baru fork ini ("Image copied", "Removed", "Focus Complete", "Right-click the island…") masih bahasa Inggris saja.

---

## 13. Settings & nilai tersimpan

**Lokasi:** schema YAML (header file), struct `Settings` (±945), dan pembacaan di `LoadSettings()` (±2081). **Menambah setting berarti mengubah ketiganya.**

**Grup Mod Settings:**
- **Appearance:** Position, TargetMonitor, OffsetX/Y, SeparateExpandedOffsetY, OffsetYExpanded, BorderMergedMode, ShapeStyle, SizeScale, AutoDpiScale, AlwaysOnTop.
- **Behavior:** ExpandOnHover, AutoHideIdleSeconds, AutoHideFullscreen, UnhideOnHover.
- **Animations:** TargetFPS, AnimationStyle, AnimationSpeed.
- **Themes:** ThemePreset, PillOpacity, TintIntensity, BackdropMaterial/FillOpacity/Tint, MaterialDepth, DropShadow, AccentBloom, TextScale, AccentColorMode, CustomAccentHex, CalendarAccent, ClockAccentGlow, FontFamily, ContourBorderMode/Hex, PillBgColor, TextPrimary/SecondaryColor.
- **Indicators:** PrivacyDots, PrivacyDotsMic/Cam (+warna).
- **Modules:** Media, MediaAutoExpand, **LyricsDelayMs**, Volume, CapsLock, Battery, BluetoothIndicator, BluetoothShowBattery, Progress, TimerModule, Clipboard, StatusCountdownProgress, DoNotDisturbIndicator, NotificationRespectDnD, HardwareMonitorModule, GameOverlay(+Show*/Compact), ShowMetricText, Weather, WeatherCity, WeatherFahrenheit, Language, ClockFormat, ShowSeconds, DateFormat, DateFirst, FileTrayModule, FileTrayMaxItems, MediaExpandBlocklist.
- **Shortcuts:** HideShowHotkeyEnabled, HideShowModifiers, HideShowKey.

**Nilai tersimpan (`Wh_GetIntValue` / `Wh_SetIntValue`):**

| Kunci | Arti | Bertahan setelah restart? |
|---|---|---|
| `LyricsEnabled` | toggle lirik | ya |
| `IslandMinimized` | mode minimize | ya |
| `ManuallyHidden` | disembunyikan lewat hotkey/menu | ya (dengan aturan reset di `LoadSettings`) |
| `PinnedExpanded` | pin expanded | tidak (di-reset di `LoadSettings`) |
| `GameOverlayPinned` | game overlay dari menu | ya |
| `PillOpacityOverride` | transparansi dari menu | ya |
| `W11StyleOverride`, `NotchStyleOverride`, `ExpandOnHoverOverride` | override dari menu | di-reset dari settings |
| `ColorThemeV2` (`kThemeValueName`) | tema aktif | disinkronkan dengan settings |
| `ProgressPercent` | nilai progress (tidak ada yang mengisi) | — |

---

## 14. Layanan eksternal

| Layanan | Untuk | Endpoint | Jika berubah atau mati |
|---|---|---|---|
| wttr.in | cuaca | `https://wttr.in/<kota>?format=j1` | tab cuaca menampilkan "Loading..." |
| lrclib.net | lirik | `/api/get`, `/api/search` (field `syncedLyrics`) | lirik tidak muncul; pill tetap 150px |
| itunes.apple.com | cover cadangan | `/search?media=music&entity=song` (field `artistName`, `artworkUrl100`) | badge sumber tampil kalau SMTC juga tidak memberi cover |

Semuanya lewat `HttpGet` (WinHTTP, user agent `DynamicIsland/1.0`, tanpa cek status HTTP; body error akan gagal diparsing dengan sendirinya).

---

## 15. Panduan perbaikan setelah update Windows

Gunakan tabel ini untuk memetakan gejala ke area kode. Semua fitur di atas adalah **perilaku target**: perbaikan harus mengembalikan perilaku tersebut, bukan menghapus fiturnya.

| Gejala | Area yang dicek | Dependensi Windows |
|---|---|---|
| Island tidak muncul sama sekali | `Wh_ModInit`/`Wh_ModAfterInit` (model tool-mod), `RenderThreadProc` (pembuatan window), `Renderer::Initialize` | Windhawk tool-mod, `CreateProcessInternalW`, D2D/DWrite, `UpdateLayeredWindow` |
| Pill musik/judul kosong | `MediaThreadProc`, `SelectActiveMediaSession` | WinRT `Windows.Media.Control` (SMTC) |
| Cover selalu badge | `ReadWinRtStreamBytes`, `DecodeImageBytesToPixels`, `ArtworkThreadProc` | `Windows.Storage.Streams`, WIC |
| Tombol prev/play/next tidak bekerja | `SendMediaTransportCommand`, `ResolveMediaSession` | SMTC `Try*Async` |
| Waveform diam | `AudioThreadProc`, `g_audioCaptureNeeded` | WASAPI loopback |
| Notifikasi Windows tidak muncul | `NotificationThreadProc` (cek log akses Denied) | `UserNotificationListener`, izin Privacy → Notifications |
| Bluetooth tidak terdeteksi | `BluetoothThreadProc` | `DeviceInformation.CreateWatcher`, selector `BluetoothDevice`/`BluetoothLEDevice` |
| Status DND salah | `SubscribeDndNotification`, `WnfDndCallback` | WNF state `0x0D83063EA3BF1C75` (**undocumented**, sangat mungkin berubah) |
| Titik privasi salah | `IsDeviceActiveViaRegistry` | registry `CapabilityAccessManager\ConsentStore` |
| Volume delay atau tidak muncul | `VolumeWatcherThreadProc`, `VolumeChangeCallback` | `IAudioEndpointVolume`, `IMMNotificationClient` |
| OSD volume Windows muncul lagi | `LowLevelKeyboardProc`, `KeyboardHookWanted`, `ApplyVolumeKey` (cek log `Volume key:`) | `WH_KEYBOARD_LL`, `VK_VOLUME_*` |
| Tombol volume mati total | hook menelan tombol tapi `ApplyVolumeKey` gagal: cek endpoint di `ApplyVolumeKey` | Core Audio |
| Clipboard tidak muncul | `WM_CLIPBOARDUPDATE`, `CaptureClipboard` | `AddClipboardFormatListener` |
| Blur/acrylic hilang atau kotak | `Backdrop` namespace, `ApplyBackdropRegion` | `SetWindowCompositionAttribute` (**undocumented**) |
| Island tertutup window lain | `EnsureTopmost`, `PositionOverlayWindow` | z-order `HWND_TOPMOST` |
| Fullscreen auto-hide salah (deteksi fullscreen) | `IsForegroundFullscreen` | perbandingan rect monitor dan class window |
| Video fullscreen tidak menyembunyikan island / game ikut tersembunyi | `IsForegroundFullscreenVideo`, `WindowBelongsToMediaSource`, `playingMediaSources` di `MediaThreadProc` | format AUMID SMTC per browser/aplikasi, `GetApplicationUserModelId`, `ApplicationFrameHost` |
| Island muncul saat hover di atas video fullscreen | `videoSuppressed` di render loop, `LowLevelMouseProc` | — |
| Hardware monitor 0/kosong | `GetGpuUsage`, `GetNetworkUsage` | PDH counter berbahasa Inggris (`PdhAddEnglishCounterW`) |
| Posisi salah di multi-monitor | `GetAnchorWorkRect`, `MonitorEnumProc` | `EnumDisplayMonitors`, DPI |

**Prinsip saat memperbaiki:**
- Jangan memblok input global di hook (hook LL harus ringan: hanya post message, tanpa lock maupun COM).
- Operasi window (hotkey, backdrop) harus di thread pemilik window (post `WM_APP_*`).
- Jaga kopling konstanta layout ↔ hit-test (§2.5).
- Jaga perilaku hemat CPU: park saat hidden, poll mahal (GPU/net) hanya saat tampil, audio capture hanya saat waveform terlihat.
- Selalu jalankan syntax check (§1) lalu uji di Windhawk.

---

## 16. Riwayat perubahan fork ini

| Versi | Perubahan |
|---|---|
| 1.4.0 | **Lirik LRCLIB** di pill collapsed (+ toggle, cache, retry); pembacaan cover bertahap; **cover cadangan iTunes**; `HttpGet` aman untuk biner + timeout; lebar pill dinamis mengikuti lirik (lead 0,35 detik, jeda ≥ 4 detik kembali normal); setting **Lyrics delay (ms)** |
| 1.5.0 | **Minimize**: garis glass + waveform, menempel 3px dari tepi layar, hitbox kecil tanpa jeda, hover membuka card |
| 1.6.0 | **Notifikasi inline** di tengah pill musik (tanpa pill kedua, lebar mengikuti notifikasi, maks 460); **tab Aktivitas** untuk Timer/Progress; durasi notifikasi sesaat maksimal 3 detik |
| 1.6.2 | **Auto-hide fullscreen khusus video** (game dan fullscreen lain tetap menampilkan island); saat video island benar-benar hilang (hover/pin/minimize tidak membukanya), hanya volume dan privasi yang muncul; pause tetap tersembunyi (sticky) |
| 1.6.1 | **Volume instan** (callback Core Audio, bukan poll 1 detik); **OSD volume Windows disembunyikan** selama mod aktif (hook menelan `VK_VOLUME_*`) + log diagnostik |

File backup tiap tahap disimpan sementara di folder scratchpad sesi Claude Code (`mod.wh.cpp.bak`, `.before-minimize.bak`, `.before-inline-alerts.bak`, `.before-volume.bak`). Folder itu bukan lokasi permanen, jadi gunakan git untuk riwayat jangka panjang.
