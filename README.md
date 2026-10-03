# Portable Chrome Updater

A small Windows program that downloads **Google Chrome** in the channels **Canary, Developer, Beta and Stable** (each x86 and x64) and installs it as a **portable version** – no installer, nothing written to `%LOCALAPPDATA%`, one self-contained folder per version.

![Main window](docs/screenshot.png)

## Download

- **[Latest release](https://github.com/naderi/portable-chrome-updater/releases/latest)** – download `ChromeUpdater.exe`, put it into an empty folder where the browsers should go (e.g. `D:\Apps\Chrome`) and run it. No installation needed.
- Or with [Scoop](https://scoop.sh):

  ```
  scoop bucket add naderi https://github.com/naderi/scoop-bucket
  scoop install naderi/portable-chrome-updater
  ```

  With Scoop, keep **Create a folder for each version** switched on (the default): the version folders, the settings and the shared profile are kept across `scoop update`.

**Requirements:** Windows 11 with .NET Framework 4.5 or later (preinstalled).

## How it works

1. Google's update service (`tools.google.com/service/update2`, Omaha protocol) is asked for the current **full installer** of each channel and architecture – with version, size and SHA-256. Channel and architecture are selected by app ID and `ap` value (e.g. Stable x64 = `x64-stable-multi-chrome`; Canary has its own app ID).
2. `<version>_chrome_installer_uncompressed.exe` is downloaded from `dl.google.com` (HTTPS, about 500 MB for x64 and 430 MB for x86). For current versions Google only offers this uncompressed variant.
3. The download is verified by **SHA-256** against the value of the update service.
4. The installer is **not run**. The updater reads the resource `CHROME.7Z` from the EXE and extracts it with the `tar.exe` that comes with Windows: `CHROME.7Z` → `Chrome-bin\chrome.exe` + `Chrome-bin\<version>\…`. Older compressed installers (`CHROME.PACKED.7Z`) are understood as well.
5. `Chrome-bin\` becomes `App\`.
6. `App\` gets read/execute permissions for **ALL APPLICATION PACKAGES** and **ALL RESTRICTED APPLICATION PACKAGES** (see below).

> **Requirement:** Windows 11. The `tar.exe` of Windows 10 cannot read LZMA-compressed 7z archives; the updater then reports *"Unpacking failed (tar.exe needs 7z support, Windows 11)"*.
>
> The updater always reports Windows 10 to the update service. Otherwise .NET reports Windows as version 6.2 (Windows 8), and Google only offers Chrome 109, the last version for Windows 7/8.

> **Why the permissions?** Google Chrome runs its renderers in an AppContainer sandbox, which may only read folders that are shared with these two groups – as `C:\Program Files` is. Other drives or folders (e.g. `D:\Apps`) usually lack them, and then no page loads and the browser stops responding. The updater sets the permissions on every installation and also checks existing installations at startup. To do it by hand:
> `icacls "<folder>\App" /grant *S-1-15-2-1:(OI)(CI)(RX) *S-1-15-2-2:(OI)(CI)(RX)`

## Using it

| Element | What it does |
|---|---|
| **x86 / x64** per channel | Installs or updates this version. **Green** with a check mark = up to date, **orange** = update available, neutral with a download arrow = not installed. The tooltip shows the folder and the installed version. |
| **Install all: x86 / x64** + **Install all / Update all** | Installs or updates all channels of the selected architectures. Versions that are up to date are skipped. |
| **Create a shortcut on the desktop** | Creates a desktop shortcut after installing (pointing to `ChromePortable.exe`). |
| **Add to start menu** | Creates a start menu entry after every installation or update – in the start menu folder **"Chrome Portable"** (with *Create a folder for each version*) or as the entry "Chrome Portable". Single entries can be removed again via *Extras → Add to start menu*. |
| **Create a folder for each version** | On: every channel gets its own folder (`Chrome Stable x64` …). Off: installs directly into the updater's folder; *Install all* is disabled then. |
| **Ignore version check** | Downloads and installs again even if the version is already up to date. |
| **Language (--lang)** | UI language of Google Chrome. Written as `--lang=…` into `Flags=` of every `ChromePortable.ini` – immediately for all installed versions and on every further installation. An existing `--lang=…` is replaced; *(no flag – system language)* removes it. |
| **Quit** | Closes the program. During a download it becomes **Cancel**. |
| **ⓘ** (bottom left) | About window with version, developer, GitHub link and program updates. |

**Extras menu**

- *Check versions again* (F5)
- *Open install folder*
- *Create shortcuts on the desktop now* – for all installed versions
- *Add to start menu* – with **Create a folder for each version** a submenu: *All installed versions*, every installed version on its own (check mark = in the start menu, a click adds or removes it) and *Remove all from start menu*. The entries are in the start menu folder **"Chrome Portable"**. Without that option it is a single item that adds or removes the entry "Chrome Portable".
- *Portable profile for each version* (`.\User Data`) / *One portable profile for all versions* (`..\User Data`) – applied to all installed versions right away
- *Keep downloaded installers* / *Delete downloaded installers* – the installers are kept in `Update\`; an existing installer with the right size and checksum is reused

**Version Info menu:** links to the Chrome Releases blog, the release overview (chromiumdash) and the about window.

If a Google Chrome instance from the target folder is still running, the updater asks you to close it instead of overwriting files in use. The program appears in German when Windows is set to German, otherwise in English. It follows the light or dark Windows theme and the accent color.

## Folder structure

With **Create a folder for each version**:

```
<updater folder>\
├─ ChromeUpdater.exe
├─ ChromeUpdater.ini              settings of the updater
├─ User Data\                     only with "one profile for all versions"
├─ Chrome Stable x64\
│  ├─ ChromePortable.exe        ← start Google Chrome with this
│  ├─ ChromePortable.ini        settings of the launcher
│  ├─ User Data\                  portable profile (with "profile for each version")
│  └─ App\
│     ├─ chrome.exe …
│     └─ updates\Version.log
└─ …
```

Without that option, `ChromePortable.exe`, `ChromePortable.ini`, `App\` and `User Data\` are placed directly in the updater's folder.

## The launcher `ChromePortable.exe`

Google Chrome has no built-in portable mode. The launcher takes care of it:

- starts `App\chrome.exe` with `--user-data-dir=<portable profile>`, so nothing ends up in `%LOCALAPPDATA%`,
- appends the switches from `ChromePortable.ini` and its own command line arguments (e.g. a URL),
- carries the icon of its channel.

> The portable Chrome does **not update itself** (there is no Google Update service) – that is what this updater is for. Features that need an installed Chrome (e.g. "make default browser") are not available.

### ChromePortable.ini

Next to every `ChromePortable.exe`. Lines starting with `;` or `#` are comments.

```ini
UserDataDir=User Data
Flags=--no-default-browser-check --no-first-run --lang=en-US
```

**`UserDataDir`** – folder of the profile (bookmarks, extensions, history, cache …), relative to the launcher's folder or absolute. `User Data` = own profile for this version (default), `..\User Data` = one profile in the updater's folder for all versions.

> The updater rewrites `UserDataDir` on every installation and when you switch it in the *Extras* menu, so better change the profile there. Only the `Flags=` line is kept on updates. A **shared profile** should not be used by two versions at the same time, and a profile of a newer version (e.g. Canary) often cannot be opened by an older one (e.g. Stable) any more.

**`Flags`** – additional command line switches, all **on one line**, separated by spaces; put paths with spaces in quotes. Examples:

| Switch | Effect |
|---|---|
| `--no-default-browser-check` | no question about the default browser |
| `--no-first-run` | skips the welcome page on the first start |
| `--lang=de` | UI language – managed by the updater's **Language** selection |
| `--start-maximized` | starts maximized |
| `--incognito` | starts in incognito / private mode |
| `--disk-cache-dir="R:\Cache"` | puts the cache elsewhere (e.g. a RAM disk); use an absolute path |
| `--proxy-server="socks5://127.0.0.1:1080"` | uses a proxy |
| `--disable-extensions` | disables all extensions |
| `--remote-debugging-port=9222` | remote debugging (e.g. for Puppeteer/Playwright) |

An (unofficial) list of all Chromium switches: <https://peter.sh/experiments/chromium-command-line-switches/>. Arguments passed to `ChromePortable.exe` are forwarded too, e.g. `ChromePortable.exe https://example.com`.

## Updates of the updater

The updater updates itself from the [releases of this repository](https://github.com/naderi/portable-chrome-updater/releases):

- Once a day it looks for a new version in the background (can be switched off in the about window: *Check automatically (once a day)*). If there is one, the ⓘ button gets an orange dot.
- In the about window: *Check for updates* → *Download update* → *Restart & update*.
- A download is only used if its signature (`.sig`) matches the key built into the program; anything else is rejected. The running EXE is renamed to `.old`, the new one takes its name, and the program restarts. `ChromeUpdater.ini` and all data next to it stay untouched.
- Installed with Scoop, the program only points to `scoop update portable-chrome-updater`.

## Disclaimer

This is an independent project. It is not affiliated with, endorsed or sponsored by Google. Google Chrome and its logo are trademarks of their respective owners. The browsers downloaded by this program are subject to their own license terms.

## License

Freeware – free to use, but **not for sale**. See [LICENSE](LICENSE).

© 2026 [Ali Naderi](https://github.com/naderi)
