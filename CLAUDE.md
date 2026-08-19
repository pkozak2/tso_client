# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

An unofficial portable client for *The Settlers Online*. Two cooperating pieces:

1. **Launcher** (`client/`) — a WPF app (.NET Framework 4.7.2, **x86 only**) that authenticates against the Ubisoft/TSO web portal, downloads/updates `client.swf`, unpacks a bundled Adobe AIR application, and starts it with a `tso://…` argument string.
2. **AIR app** (`client/files/content/`) — an unpacked Adobe AIR app (`index.html` + `client.exe`/`client64.exe` runtime) that loads the game's `client.swf` and injects JavaScript mods (`scripts/`, `userscripts/`) that reach into the Flash game via `swmmo.getDefinitionByName(...)`.

`client.swf` / `client_upstream.swf` / `client_testing.swf` at the repo root are **patched game binaries committed to git** — the launcher downloads them from GitHub raw, not from Ubisoft.

## Build

No test suite exists. Build = MSBuild on the solution:

```powershell
nuget restore client.sln
msbuild /m /p:Configuration=Release /property:Platform=x86 client.sln
# output: client\bin\Release\client.exe (single file; Costura.Fody embeds all DLLs)
```

`Any CPU` configurations map to x86 — always build x86. NuGet packages are committed under `packages/` (old-style `packages.config`).

**The pre-build event zips `client/files/content` into `client/files/content.zip`**, which is embedded as `Properties.Resources.content`; the post-build event deletes the zip. Consequence: **any change under `client/files/content/` (including `scripts/*.js` and `index.html`) requires a full rebuild** to reach the runtime. Editing the extracted copy in the runtime folder is pointless — see "Runtime folder" below.

CI (`.github/workflows/msbuild.yml`) additionally replaces the `#TESTTAG#` placeholder in `Main.xaml.cs` with the short commit SHA; local builds leave it as-is and the code strips it.

APK build (`android/` is an apktool-decoded wrapper around the same AIR content):

```sh
cp client.swf android/assets && cd android
java -jar -Duser.language=en -Dfile.encoding=UTF8 apktool.jar b .
jarsigner -verbose -keystore sign.keystore -storepass qwerty dist/client.apk abc.keystore
```

`library.swf` (ActionScript sources in `library/`) is built separately — see `library/README.md` (docker `jeko/airbuild:15.0`, `compc`). The built artifact is committed at `client/files/content/library.swf`.

## Fork-specific: update endpoints

This is a fork. `client/Main.xaml.cs` points self-update and SWF download at **`pkozak2/tso_client`**:

- `changelog.xml` (AutoUpdater.NET version feed)
- `upstream.json` (list of region codes that should use `client_upstream.swf`)
- `client*.swf` via `raw.githubusercontent.com` + the GitHub contents API for the SHA check

`Main.appversion` must match `<version>` in `changelog.xml`, or AutoUpdater will nag/never trigger. Releasing a new SWF means committing it to `master` of the fork the launcher points at.

The in-game script manager (`client/files/content/scripts/0-manager.js`) still fetches `userscripts/` and `info.json` from **`fedorovvl/tso_client`**. These two sources are independent; changing one does not change the other.

## Launcher architecture

- **`Main.xaml.cs`** — main window, settings persistence, SWF update check, and the handoff to AIR.
  - `checkVersion()` (background thread) extracts the embedded `content.zip` into `ClientDirectory`, then picks a SWF: region in `upstream.json` → `client_upstream.swf`; region `ts` → `client_testing.swf`; else `client.swf`. Freshness is a **git blob SHA1** (`"blob <len>\0" + bytes`) compared against the GitHub contents API `sha`. Whatever is downloaded is always saved locally as `client.swf`.
  - `makeTsoUrl()` is the single place where the `tso://` query string is assembled (lang, window, clientconfig, dropbox keys, gfxcache, debug). Command-line args override settings-file values.
  - `run_tso()` rewrites `META-INF/AIR/application.xml` `<id>` to `TSO-<5 random letters>` before each start so multiple AIR instances can coexist, then `Process.Start`s `client.exe`/`client64.exe` with the `tso://` string as its argument.
  - `clientSettings` (same file) is the settings model, serialized to JSON and stored in `settings.dat` via DPAPI (`ProtectedData`, `LocalMachine` scope, entropy `{2,1,8,4,2}`); the legacy `|`-separated format is auto-converted on read.
- **`login.xaml.cs`** — three auth paths, chosen in `MainAuth()`:
  - `FastAuth()` — replays the saved `dsoAuthUser`/`dsoAuthToken` from `settings.tsoArg` against the game backend; falls back to full auth on failure (or when `Main.forceFullAuth` is set, e.g. after a region change).
  - `CipAuth()` — the long Ubisoft OAuth chain (`/oauth/start` → `connect.ubisoft.com` → `api.partners.ubisoft.com` token/consent/callback), with TOTP 2FA via OtpSharp when `twoFactorAuthenticationTicket` comes back.
  - `CipMigratedAuth()` — for accounts migrated to CipSoft: direct POST to the portal's `/api/user/login`, then main page, then play page.
  - All paths end in `PrepareFlash()`, which **regex-scrapes the `/play` HTML** for the `lang…` query string and `loggedInUserName`. When login suddenly breaks, this regex and the auth endpoint URLs are the first suspects.
- **`Servers.cs`** — the one registry of region → domain/`uplay`/`main`/`play` paths, region → game language code, and all launcher UI translations. `getTrans()` falls back to `en-uk`, so **every key used anywhere must exist in the `en-uk` dictionary**. Adding a region needs an entry in `_servers`, `_langs`, and the `region_list` ComboBox in `Main.xaml` (its `Uid` is the index stored in `settings.region`).
- **`PostSubmitter.cs`** — hand-rolled HTTP client used for every request. `useBC = true` routes the request through a BouncyCastle TLS stack instead of `HttpWebRequest`, but **only when `Main.winver.Major < 10`** (TLS 1.2 on old Windows). It returns the sentinel string `" CAPCHA "` on certain failures — callers compare against that literal.
- `Crypt.cs` (3DES, used for the Dropbox-stored fast-login blob), `Unzip.cs`, `Arguments.cs` (`--name value` / `--name=value` parser), `WinVersion.cs`, `ExceptionDumper.cs` are support code.

Supported command-line flags are listed in `args_help` in `Main.xaml.cs`; keep that list in sync when adding one.

## Runtime folder

`ClientDirectory` = `%LOCALAPPDATA%\<settings.tsofolder>` (default `tso_portable`/`tso_portable2`), or next to the launcher if `tsoFolderNearLauncher`. On every start:

- `content.zip` is re-extracted over it,
- **`scripts/` is wiped first** — bundled scripts are always the ones compiled into the exe,
- `userscripts/` is *not* wiped — user-installed scripts and `settings.json` survive.

So: iterate on bundled scripts by editing `client/files/content/scripts/` and rebuilding. (`--debug` skips the extraction, which is how you test a hand-edited runtime folder.)

## AIR / in-game JS architecture

`client/files/content/index.html` is the AIR entry point. It loads `client.swf` with `air.Loader`, polls until the game GUI is up, then sets up the globals every mod script relies on:

- `swmmo` — the loaded game SWF; `swmmo.getDefinitionByName("…")` is how AS3 classes are reached
- `swmmo.application` — game interface; `globalFlash`, `loca` (localization), `assets`
- `game` — thin facade defined inline in `index.html` (`getBuildings`, `getSpecialists`, `getBuffs`, `chatMessage`, `getTracker`, …)
- `air` / `window.runtime` — AIR APIs (`AIRAliases.js`); `window.runtime.ClientBuff` comes from `library.swf`
- `rawArgs` — the parsed `tso://` parameters, reused by `runNewApplication()` when the game spawns an extra instance (adventure zones)

It then appends every file in `scripts/` as a `<script>` tag. **Filename numeric prefixes are the load order**: `0-common.js` (helpers + the `mainSettings` defaults and `settings.json` persistence), `0-manager.js` (userscript installer UI), `0-lang-*.js`, `0-keybinds.js`, feature scripts, and `99-menu.js` last (builds the in-game menu, `menu` global). `0-common.js#reloadScripts` injects `userscripts/` afterwards, honouring `enabledScripts`.

`mainSettings` in `0-common.js` is the schema for the in-game `settings.json`; new options belong there with a default.

## Userscripts

`userscripts/` at the repo root is the distributable catalog, described by `userscripts/info.json` (name/author/descriptions/url). These are **not** bundled into the exe — the in-game script manager downloads them from GitHub into `ClientDirectory/userscripts`. Only `99-example.js` ships in `content/userscripts` as a template; it documents the available globals and helpers (`addToolsMenuItem`, `createModalWindow`, `showAlert` — defined near the end of `0-common.js`). A script can self-describe instead of using `info.json` by assigning to `customScripts['user_myscript.js']` (see `userscripts/README.md`).

## Other

- `mapping/mapping.py` — UnityPy-based extractor that turns a Unity `.data` bundle into the mapping file the SWF consumes plus a JSON dump (`mapping/README.md`).
- `TSOlastBreath.user.js` — a browser userscript, unrelated to the launcher build.
