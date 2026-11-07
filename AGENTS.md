# AGENTS.md

Guidance for AI agents working in this repository.

## What this repo is

- AFK Realm builds and manages a Conquest of Azeroth (AzerothCore + Playerbots) game server from a GUI.
- `app/` — the released product since 1.1.6: a .NET 8 local web server (`app/Web`), UI in `app/wwwroot` (HTML/JS), shown in an app window or browser. Cross-platform: win-x64, linux-x64, linux-arm64.
- `src/` — classic WinForms exe (C# 5 / .NET Framework 4.x). CI no longer builds or releases it; the released `AFK-Realm.exe` is the web build, which 1.1.5 and older classic installs update themselves to.
- `engine/engine.ps1` — does all actual server work (install, update, rebuild, setup, backup/restore, modules). Runs standalone; the apps call it and parse its output.

## Builds and verification

```sh
dotnet publish app/AfkRealm.csproj -c Release -r win-x64 --self-contained true -p:PublishSingleFile=true -p:IncludeNativeLibrariesForSelfExtract=true -p:EnableCompressionInSingleFile=true
```

- This is the real verification step — there are no tests or linter. Use `linux-x64` / `linux-arm64` for Linux; needs the .NET 8 SDK per `global.json`. CI runs it for all three RIDs on every PR.
- Beyond publishing, CI only syntax-parses `engine/engine.ps1` (on the Windows runner); it never runs the app.
- Classic build: `build.bat` (Windows csc.exe) or `build.sh` (Mono `mcs`, needs `mono-devel`). CI never builds it, so C# 6+ syntax in `src/` passes CI but breaks the classic build.
- No NuGet packages anywhere; `app/Shims/` reimplements classic-only APIs (JavaScriptSerializer, ProtectedData) so shared code compiles on .NET 8.

## Shared-source rules (easy to break)

- All of `src/` is compiled as C# 5 by the classic build, even though the app build uses LangVersion 12: no string interpolation, `?.`, expression-bodied members, `nameof`, `out var`, tuples, etc. C# 12 is fine in `app/**` but not in `src/**`; write classic property bodies and format with `L.F`.
- The web build compiles only the explicit `<Compile Include="../src/...">` list in `app/AfkRealm.csproj` (default compile covers only `app/**`). Add every new shared `src/*.cs` file there or the web build won't compile it; the other `src/*.cs` files are WinForms-only. `EnableDefaultContentItems=false` affects content, not compile.
- Platform differences inside shared files go behind `#if AFK_WEB` / `#if !AFK_WEB` (defined by the csproj). Web-only code lives in `app/Web/`.

## Web app

- `app/Web/Api.*.cs` are partial `Api` classes; routes are registered in `Api.Map` (`app/Web/Api.cs`). Browser code is plain JS in `app/wwwroot/js` with no build step.
- `app/wwwroot/**` is embedded as `web/*` in the exe. To iterate on the page without rebuilding, serve it from disk: `AFK-Realm --web-root app/wwwroot` or `AFK_WEB_ROOT`.
- Ports: 47810 by default, 47800 with `--listen` (override with `--port`). `--listen` requires `--password` or `AFK_REALM_PASSWORD`. `--no-window` runs the web server only; `--server <folder>` opens a known server.
- Headless path: `AFK-Realm cli install|status|start|stop|restart|update|repair|backup|account|command` (see `app/Web/Cli.cs`).

## Embedded assets and engine updates

- Embedded files use fixed LogicalNames (`CoAInstaller.*`, `web/*`); renaming or moving a file breaks the lookup. Adding a non-`wwwroot` asset means editing `app/AfkRealm.csproj`, `build.bat` and `build.sh` — separate lists; `build.sh` already embeds fewer (e.g. no dxvk zip). `wwwroot/**` is globbed.
- Keep `engine/engine.ps1` UTF-8 with BOM: Windows PowerShell 5.1 reads BOM-less scripts as ANSI.
- The app re-extracts the engine to `<server>/Builder/AFK-Realm-Engine.ps1` every time it runs a job, overwriting edits. To test engine changes without rebuilding, run `pwsh engine/engine.ps1 ...` directly.

## Running and debugging

- Windows: without administrator rights the app relaunches itself elevated (installing Visual Studio, firewall rules and the database need them); Linux never elevates.
- One instance at a time: a second start shows or reports the first (via `web-instance.json`) and exits. Stop the running instance before testing a new build.
- Settings and diagnostics: `%APPDATA%\AFK-Realm` on Windows, `~/.config/AFK-Realm` on Linux. A failed start writes `startup-error.txt`; every start logs `startup.log`. On Linux the app fetches PowerShell 7 into `~/.local/share/AFK-Realm` when it needs the engine and `pwsh` is absent.

## engine.ps1 contract

- Must parse under Windows PowerShell 5.1 and pwsh 7 (`#requires -Version 5.1`). CI parses it with pwsh 7 only, so check 5.1 compatibility yourself (no `??`, ternary `? :`, `-Parallel`).
- Modes: Install, Update, Rebuild, Setup, Backup, Restore, RestoreParts, Modules. Run standalone, e.g. `pwsh engine/engine.ps1 -InstallRoot <dir> -NonInteractive -Mode Update`.
- In `-NonInteractive` mode the DB password comes from `$env:AC_DB_PASSWORD`, and progress is machine-readable lines the app depends on: `##AC|PHASE|…`, `##AC|CHANGE|…`, `##AC|SOURCES|…`, `##AC|NOTE|…`, `##AC|DONE|…`, `##AC|FAIL|…`. Keep this format stable.

## Versions and releases

- Version lives in two places — `Product.Version` in `src/Core.cs` and `<Version>` in `app/AfkRealm.csproj` — keep them identical.
- Release flow: bump both, add a top `## X.Y.Z` entry to `CHANGELOG.md`, add a top user-facing section to `RELEASE_NOTES.md` (CI uses the whole file as the GitHub release body), commit as `Release X.Y.Z: <summary>`, push tag `vX.Y.Z`; CI builds and publishes. A `-` in the tag marks a prerelease (for the same version, a release beats a `-preview`).
- Keep release asset names: the updater downloads `AFK-Realm.exe`, `AFK-Realm-linux-x64` or `AFK-Realm-linux-arm64`; `AFK-Realm-web.exe` (same file) updates 1.1.0–1.1.5 web builds. Tag builds also attach `assets/coa64-native/CoA64.zip` (a missing file fails the job).
- CI fails if the win-x64 `AFK-Realm.exe` reaches 50 MiB (52,428,800 bytes): classic exes refuse larger self-updates. Watch size when adding embedded assets.

## UI text and translations

- The English text is the key in `lang/<code>.json` on three surfaces: C# `L.T("…")` / `L.F("… {0} …", value)`; web JS `T('…')`, `F('… {0} …', …)` and `C('context', '…')` from `app/wwwroot/js/core.js`; engine `Tr '…'` (keys are the English text, or `context|text` for `C`). Unwrapped strings are not extracted and stay English.
- `python3 tools/lang.py keys` regenerates `lang/template.json`; `check` reports missing/unused texts and mismatched `{0}` places; `missing de`, `merge de <file>`, `prune`, `todo` for the rest. Module folders: same commands with `--module <folder>` (module `keys` needs `mcs`/`mono` when it has `conf/*.conf.dist`).
- Language files are auto-discovered from embedded `lang/*.json`; add a new language's code to `LANGS` in `tools/lang.py` so `check`/`prune` cover it. CI does not run any of this — run `check` after UI text changes.

## Docs

- `docs/MANUAL.md` is the user guide keyed to the UI; feature changes visible to users belong in `README.md` (feature list), `docs/MANUAL.md`, `CHANGELOG.md` together.
- `docs/RACE-IDS.md` documents CoA Custom source-row race IDs; the runtime-authoritative list is in the app (Custom races → Show race IDs).
