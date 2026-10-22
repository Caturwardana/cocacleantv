# Coocaa / Skyworth SWTV — debloat + cleanyourtv (weather fork)

This repo is a personal snapshot: upstream [Gathin23/cleanyourtv](https://github.com/Gathin23/cleanyourtv)
(`main`, 22dc48f) with the weather/wallpaper branch of
[manas-04/cleanyourtv](https://github.com/manas-04/cleanyourtv/tree/feat/wallpapers-weather) (`ca9ad37`)
merged on top. The merge was resolved for the newer upstream sources, so `weather`,
`builtin_wallpaper_minutes` and the 14 built-in wallpapers work with today's launcher.

Last updated: 22 October 2026.

## TV

| | |
|---|---|
| Model | `SWTV` |
| Device | `shibuya` |
| Android | 11 (SDK 30), build `RTO9.240708.002` |
| Chipset | MediaTek (`com.mediatek.*`) |
| Inputs | HDMI 1 = `HW5`, HDMI 2 = `HW6`, Composite = `HW2`, tuner `HW0`/`HW1` |
| Live TV app | `com.mediatek.wwtv.tvcenter/.nav.TurnkeyUiMainActivity` (tile labelled **TV Digital**) |
| Stock launcher | `com.google.android.tvlauncher/.MainActivity` (disabled) |
| Home today | `com.gathin.tvhome/.MainActivity` |

## What is installed

- Launcher `com.gathin.tvhome`, built from this repo (~1.4 MB release APK, wallpapers included).
  - cinematic shelf on, weather for **Bantul** (`-7.8881`, `110.3286`), Celsius, Open-Meteo, hourly cache
  - built-in wallpapers rotating every 60 minutes
  - 4 columns; dock = Netflix, Prime Video, YouTube, Disney+ Hotstar (the rest of `dock_apps`
    — Vidio, Google Play Movies, Bstation — fall into the grid because the dock has 4 slots)
  - "TV Digital" tile for the tuner app, banner `banner_tv`
- Optional weather add-on app `io.github.sumyabazar1.weather` (module-card tile + full-screen page).
  Not needed once the shelf weather is on; safe to uninstall.
- Debloated: 12 packages disabled, plus the stock launcher. See
  [`examples/cooca-swty-shbuya/debloat.txt`](../examples/cooca-swty-shbuya/debloat.txt).

Kept on purpose: all streaming apps, Google Assistant, Chromecast, phone remote, Play Store/Services,
Gboard, Bluetooth, `com.google.android.tungsten.setupwraith` (holds the saved Wi-Fi network),
`com.mediatek.tvinput`, `com.mediatek.tv.service`, `com.mediatek.wwtv.mediaplayer`, `com.coocaa.factory`.

## Build and install

JDK 17 and Android SDK platform 34 with build-tools 34.0.0. `launcher/local.properties` needs
`sdk.dir=...`.

```powershell
$env:JAVA_HOME="C:\Program Files\Microsoft\jdk-17.0.20.101-hotspot"
$env:ANDROID_HOME="$env:LOCALAPPDATA\Android\Sdk"

cd launcher
.\gradlew.bat assembleRelease -q --console=plain

# push and install (upstream tools, from the repo root)
$env:TV_HOST="<tv ip>"
$py="tools\venv\Scripts\python.exe"
& $py tools\b64push.py launcher\app\build\outputs\apk\release\app-release.apk /data/local/tmp/tvhome.apk
& $py tools\tv.py "pm install -r /data/local/tmp/tvhome.apk"
& $py tools\tv.py "rm -f /data/local/tmp/tvhome.apk"
& $py tools\tv.py "am force-stop com.gathin.tvhome"
& $py tools\tv.py "am start -n com.gathin.tvhome/.MainActivity"
```

`am start` sometimes times out inside `build-and-install.ps1` even though the install and
`set-home-activity` succeeded; the manual sequence above is the reliable path.

Python tools need `tools/venv` (`pip install -r tools/requirements.txt`).

## Configuration

Everything lives in `launcher/app/src/main/res/values/config.xml`.

| Key | Value here | Meaning |
|---|---|---|
| `columns` | 4 | tiles per row = dock slots |
| `dock_apps` | 7 packages | dock order; only the first `columns` show in the dock |
| `cinematic_home` | true | artwork shelf + big app name |
| `weather` | true | weather block opposite the shelf |
| `weather_place` / `weather_latitude` / `weather_longitude` | Bantul / -7.8881 / 110.3286 | Open-Meteo lookup, sends only coordinates |
| `builtin_wallpaper_minutes` | 60 | rotate the 14 bundled wallpapers; 0 = off |
| `rows_below_dock` | 0 | grid rows visible under the dock |
| `hidden_apps` | `com.mitv.gallery` | apps with no tile |
| `app_labels` / `app_banners` | `com.mediatek.wwtv.tvcenter=TV Digital` / `=banner_tv` | tile name and artwork |
| `stock_home_redirect` | `com.google.android.tvlauncher` | bounce back here if the stock launcher reappears |

On-TV choices (Launcher settings: columns, rows, dock opacity, wallpaper) override `config.xml`;
use "follow the theme" there to fall back to the file.

Other keys worth knowing: `dock_row`, `dock_title`, `dock_glass`, `ambient_glow`, `app_glow`,
`glow_style`, `focus_overshoot`, `tile_menu`, `app_library`, `sky_intensity`, `wallpaper_dim`,
`wallpaper_order`, `show_network_pill`, `show_inputs_pill`. Themes can override many of them from
`config.json` (`tools/theme.sh`, `themes/README.md`).

## Back to stock

Order matters: enable the stock pieces first, then remove the custom launcher.

```powershell
cd <repo>
$env:TV_HOST="<tv ip>"
$py="tools\venv\Scripts\python.exe"

# 1) re-enable the debloated packages from the recorded state
tools\restore.ps1 -TvHost $env:TV_HOST examples\cooca-swty-shbuya\debloat.txt

# 2) re-enable the stock launcher and make it Home
& $py tools\tv.py "pm enable com.google.android.tvlauncher"
& $py tools\tv.py "cmd package set-home-activity --user 0 com.google.android.tvlauncher/.MainActivity"

# 3) verify Home resolves to the stock launcher
& $py tools\tv.py "cmd package resolve-activity --brief -a android.intent.action.MAIN -c android.intent.category.HOME"

# 4) remove the custom launcher and the optional weather add-on
& $py tools\tv.py "pm uninstall --user 0 com.gathin.tvhome"
& $py tools\tv.py "pm uninstall --user 0 io.github.sumyabazar1.weather"

# 5) reboot and check Home, HDMI, Live TV and Play Store
& $py tools\tv.py "reboot"
```

Undo records live in `~/.cleanyourtv/` on the machine that ran `disable.ps1`; keep them — they are
what `restore.ps1` reads. Backups (package list, `getprop`, settings) belong in `backup/`, which this
repo ignores.

If `pm uninstall` is refused, reinstalling the stock launcher and re-running
`set-home-activity` is enough to get the old Home back.

## Notes and quirks

- A fully black screenshot means the TV is asleep: send `input keyevent KEYCODE_WAKEUP` first.
- Screenshots taken mid-scroll can show focus in the wrong place; wait 2–3 s after a key.
- Live TV, HDMI and Netflix return 0-byte screenshots (protected surfaces), not damage.
- `uiautomator dump` fails on this firmware (`null root node`); check
  `dumpsys activity activities | grep mResumedActivity` instead.
- `tools/check.py` on the debloat list should say "nothing drifted" with Home =
  `com.gathin.tvhome/.MainActivity`.
- Close ADB when done: turn off USB/Network debugging, revoke authorisations, reboot, then
  `tools/adb-closed.py <tv ip>` must report port 5555 closed.
