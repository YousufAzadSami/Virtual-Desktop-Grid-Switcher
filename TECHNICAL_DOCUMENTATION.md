# Virtual Desktop Grid Switcher — Technical Documentation

This is the canonical architecture and code-navigation guide for humans and coding agents. It describes the current source tree; when historical prose disagrees with executable code, treat the code and project files as authoritative.

## What this repository is

A Windows-only, self-contained .NET 8 WinForms tray application that presents Windows virtual desktops as a row-major grid. Global hotkeys switch desktops, move the foreground window, pin a window across desktops, or toggle always-on-top.

The app has no main window. It runs a WinForms message loop, displays a numbered notification-area icon, and exposes **Settings**, **About**, and **Exit** from the tray menu.

The repository contains two logical products:

1. **The grid-switcher app** in `VirtualDesktopGridSwitcher/`.
2. **A vendored VirtualDesktop library** in `VirtualDesktop-master/`, based on Slion/Grabacr07 code and modified locally for Windows build-specific, undocumented COM APIs. This is source in the repository, not a Git submodule or a NuGet dependency.

Project lineage:

- Original application: <https://sourceforge.net/projects/virtual-desktop-grid-switcher/>
- GitHub parent repository: <https://github.com/kloned/Virtual-Desktop-Grid-Switcher>
- Vendored library lineage: <https://github.com/Grabacr07/VirtualDesktop> and <https://github.com/Slion/VirtualDesktop>

## Read these first

| Order | File | Why |
|---|---|---|
| 1 | `VirtualDesktopGridSwitcher/Program.cs` | Entire startup/lifetime flow. |
| 2 | `VirtualDesktopGridSwitcher/VirtualDesktopGridManager.cs` | Most application behavior: grid, switching, moving, hooks, browser/document workarounds, hotkeys. |
| 3 | `VirtualDesktopGridSwitcher/Settings/SettingValues.cs` | Defaults, persisted XML schema, migrations, and the Apply event. |
| 4 | `VirtualDesktop-master/src/VirtualDesktop/VirtualDesktop.system.cs` | OS provider selection and COM initialization. |
| 5 | `VirtualDesktop-master/src/VirtualDesktop/Interop/ComInterfaceAssemblyBuilder.cs` | Runtime generation of build-specific COM interface assemblies. |
| 6 | `VirtualDesktop-master/src/VirtualDesktop/VirtualDesktop.cs` | Public virtual-desktop API used by the app. |

## Repository layout

```text
.
├── VirtualDesktopGridSwitcher.sln              # Main solution; use this for the app
├── VirtualDesktopGridSwitcher/                  # WinForms tray application
│   ├── Program.cs                               # [STAThread] entry point
│   ├── VirtualDesktopGridManager.cs             # Main controller and behavior
│   ├── VirtualDesktopTimer.cs                   # Polling-based desktop-change detector
│   ├── SysTrayProcess.cs                        # NotifyIcon and numbered icon loading
│   ├── ContextMenus.cs                          # Settings/About/Exit menu
│   ├── WinAPI.cs                                # App-specific user32/kernel32/psapi P/Invoke
│   ├── Settings/
│   │   ├── SettingValues.cs                     # Settings model, XML load/save/migrations
│   │   ├── SettingsDialog.cs                    # Settings UI behavior
│   │   └── SettingsDialog.Designer.cs           # Generated WinForms layout; avoid hand edits
│   ├── AboutBox/                                # About dialog and generated layout
│   ├── Icons/                                   # Runtime icons 1.ico through 12.ico
│   ├── App.config                               # Legacy application config
│   ├── app.manifest                             # asInvoker + Windows 10/11 compatibility
│   └── VirtualDesktopGridSwitcher.csproj        # net8.0 Windows, self-contained WinExe
├── GlobalHotkeysWithDotNET/
│   └── Hotkey.cs                                # Linked into the app project; WM_HOTKEY filter
├── VirtualDesktop-master/                       # Vendored VirtualDesktop library repository
│   ├── src/VirtualDesktop/                      # Core library included by main solution
│   │   ├── VirtualDesktop*.cs                   # Public API, initialization, events
│   │   ├── Interop/                             # COM generation/proxies/build adapters
│   │   ├── Properties/                          # Library metadata and GUID settings
│   │   └── app.config                           # Build-to-IID mappings
│   ├── src/VirtualDesktop.WPF/                  # Not in the main solution
│   ├── src/VirtualDesktop.WinForms/             # Not in the main solution
│   ├── samples/                                 # Showcase; not in the main solution
│   └── .github/workflows/                       # Vendored library CI, not root-repo CI
├── Documentation/                               # Installation and user guides, plus PDFs/images
├── Package/                                     # Historical distribution payload/icons/docs
├── TECHNICAL_DOCUMENTATION.md                   # This canonical architecture/navigation guide
├── WINDOWS_11_24H2_FIX_GUIDE.md                 # Windows compatibility implementation/test notes
├── MIGRATION_TO_NET8.md                         # Fork's .NET 8 migration record
├── ERROR_HANDLING_LOGGING_PLAN.md               # Detailed proposal; not implemented
├── ROADMAP.md                                   # Prioritized work and release prerequisites
└── Windows10SDKVS13_*.props, .hg*               # Historical migration artifacts
```

The main solution contains only:

- `VirtualDesktopGridSwitcher/VirtualDesktopGridSwitcher.csproj`
- `VirtualDesktop-master/src/VirtualDesktop/VirtualDesktop.csproj`

## Runtime architecture

```text
Program.Main
  ├─ SettingValues.Load()
  ├─ SysTrayProcess
  │    ├─ NotifyIcon
  │    ├─ ContextMenus
  │    └─ Icons/<desktop-number>.ico
  └─ VirtualDesktopGridManager
       ├─ WindowsDesktop.VirtualDesktop public API
       │    └─ build-specific COM provider/runtime-generated interfaces
       ├─ GlobalHotkeyWithDotNET.Hotkey
       │    └─ RegisterHotKey + WinForms IMessageFilter
       ├─ VirtualDesktopTimer
       │    └─ WinForms Timer polling VirtualDesktop.Current
       └─ WinAPI.SetWinEventHook
            └─ foreground-window tracking and workarounds
```

### Startup and shutdown

1. `Program.Main()` enables WinForms styles/DPI behavior and loads settings.
2. `SysTrayProcess` creates the visible tray icon/menu and loads every `Icons/*.ico`, sorting filenames numerically.
3. `VirtualDesktopGridManager` installs an `EVENT_SYSTEM_FOREGROUND` hook and calls `Start()`.
4. `Start()` changes the actual Windows desktop count to exactly `Rows * Columns`: surplus desktops are removed from the end and missing desktops are created.
5. It snapshots desktop GUID/index lookups and per-desktop window-tracking arrays.
6. If enabled, `VirtualDesktopTimer` starts polling; global hotkeys are registered.
7. `Application.Run()` supplies the UI/message loop required by the tray icon, timer, hook callbacks, and `WM_HOTKEY` filter.
8. Disposal stops polling/hotkeys and removes the foreground-event hook.

**Safety:** merely launching the application can create or remove real Windows virtual desktops. Do not use application startup as a smoke test on a workstation whose desktop state must be preserved.

## Main application behavior

### Grid model

`VirtualDesktopGridManager` treats desktop positions as zero-based row-major indexes:

```text
0 1 2
3 4 5     (default 3 rows × 3 columns)
6 7 8
```

- `ColumnOf(index)` and `RowOf(index)` normalize candidate positions.
- `Left`, `Right`, `Up`, and `Down` return the adjacent index.
- At an edge they return `Current`, unless `WrapAround` is enabled.
- `Switch(index)` records the current foreground HWND and changes desktop.
- `Move(index)` moves the foreground HWND and then switches to the target.

### State worth knowing

| Field | Meaning |
|---|---|
| `desktopIdLookup[index]` | Grid index to Windows desktop GUID. |
| `desktopNumberLookup[id]` | Windows desktop GUID to grid index. |
| `activeWindows[index]` | Last foreground window observed on each desktop. |
| `lastActiveBrowserWindows[index]` | Last browser window observed on each desktop. |
| `_current` / `Current` | App's current grid index; the setter performs the COM switch. |
| `movingWindow` | Suppresses normal activation bookkeeping during a move. |
| `activatingBrowserWindow` | Suppresses bookkeeping while applying the browser workaround. |
| `callbackMutex` | Shared lock for timer and foreground-hook callbacks. |

### Desktop-change detection

Native VirtualDesktop notifications are intentionally not active:

- The app's `VirtualDesktop.CurrentChanged` subscription is commented out in `VirtualDesktopGridManager.Start()`.
- The library's notification registration is commented out in `VirtualDesktop.InitializeCore()`.
- External desktop changes are therefore detected by `VirtualDesktopTimer` polling `VirtualDesktop.Current` at `SettingValues.IntervalMs` (default 500 ms).
- App-initiated changes invoke `VirtualDesktop_CurrentChanged(...)` manually from the `Current` setter.

`VirtualDesktopGridManager` also contains older private `StartWinFormsTimer`, `DoOnTimerTick`, and `StopWinFormsTimer` methods. They are currently unused; the separate `VirtualDesktopTimer` class is the active implementation.

### Foreground-window hook and workarounds

`ForegroundWindowChanged(...)` is the center of window bookkeeping:

- Resolves the HWND's desktop through `VirtualDesktop.FromHwnd`.
- Updates last-active window/browser arrays.
- Supports the optional default-browser activation workaround.
- Detects Word, Excel, and Acrobat-style “new document opened on another desktop” sequences and tries to move the new HWND back to the initiating desktop.

Executable/class defaults and timeout settings live in `SettingValues.SetListDefaults()`.

### Hotkeys

`VirtualDesktopGridManager.RegisterHotKeys()` expands settings into concrete actions:

- Directional switch/move: arrow keys and/or custom direction keys.
- Positional switch/move: digits 1–9, F1–F12, and up to 12 custom keys.
- Sticky window: `VirtualDesktop.PinWindow` / `UnpinWindow`.
- Always-on-top: `GetWindowLongPtr` + `SetWindowPos`.

`GlobalHotkeysWithDotNET/Hotkey.cs` calls `RegisterHotKey` and installs each instance as an `Application` message filter. `VirtualDesktopGridManager.RegisterHotkey(...)` detects duplicate configured combinations and reports OS-level registration conflicts with message boxes.

## Settings lifecycle

- **Storage:** `VirtualDesktopGridSwitcher.Settings` beside the executable (`AppDomain.CurrentDomain.BaseDirectory`).
- **Format:** public fields serialized with `XmlSerializer`.
- **Defaults:** field initializers plus `SetListDefaults()`.
- **Migration:** `LoadOldSettings()` supports old element names; `ApplyVersionUpdates()` migrates through `SettingsVersion == 4`.
- **Apply:** `SettingsDialog.SaveValues()` mutates the in-memory object, calls `settings.ApplySettings()`, then saves. The manager subscribes `Restart` to `SettingValues.Apply`, so Apply tears down and recreates desktops/timer/hotkeys before persistence.
- **UI:** behavior is in `SettingsDialog.cs`; layout/control declarations are in generated `SettingsDialog.Designer.cs` and `.resx`.

Settings are portable-install style, not stored under AppData. Running from a non-writable installation directory can make `Save()` fail.

## VirtualDesktop COM layer

Windows exposes only limited public virtual-desktop functionality. Creation, removal, switching, pinning, and cross-process moves here use undocumented Explorer COM interfaces whose GUIDs and vtable layouts vary by Windows build.

### Call path

```text
VirtualDesktopGridManager
  → WindowsDesktop.VirtualDesktop                public facade
  → VirtualDesktopProvider                       provider contract
  → BuildNNNNN wrapper                           stable proxy API → build ABI adapter
  → ComWrapperBase.InvokeMethod                  reflected call on generated COM type
  → ImmersiveShell/VirtualDesktop COM service
```

### Initialization path

1. `Utils/OS.cs` combines `Environment.OSVersion.Version` with registry `UBR` to obtain `10.0.<build>.<revision>`.
2. `VirtualDesktop.system.cs::CreateProvider()` selects the newest provider threshold not greater than the OS: 10240, 20348, 22000, 22621.2215, or 26100.0.
3. `ComInterfaceAssemblyBuilder` finds proxy interfaces marked `[ComInterface]` and all embedded `.interfaces/*.cs` variants.
4. For each interface, it selects the newest embedded source whose build is not greater than the current OS.
5. `IID.GetIIDs()` selects the newest configured IID set not greater than the current OS and falls back to `HKCR\Interface` for missing names.
6. Placeholder zero GUIDs in the selected C# sources are replaced, and Roslyn compiles a build-specific assembly.
7. By default it is cached under `%LOCALAPPDATA%\slions.net\VirtualDesktop\assemblies` and shadow-copied when reused in Release builds.
8. A provider creates wrapper objects for the public `Interop/Proxy` contracts; `ComWrapperBase` invokes methods by matching wrapper method names to generated COM methods.

### Build-specific files

Each `Interop/Build<build>_<revision>/` may contain:

- `.interfaces/*.cs`: exact COM ABI/vtable definitions embedded as source resources.
- `.Provider.cs`: provider construction and shared wrappers.
- `VirtualDesktop.cs`: build-specific desktop wrapper.
- `VirtualDesktopManagerInternal.cs`: maps stable proxy calls to that build's ABI.
- `VirtualDesktopNotificationService.cs`: build-specific notification bridge.

Folder suffixes are selection versions, while namespaces omit the revision (for example, `Build26100_0000` uses `WindowsDesktop.Interop.Build26100`). A wrong namespace or method order can select/invoke the wrong ABI and cause process-level access violations, not just managed exceptions.

### Adding or repairing Windows build support

Check all of these together:

1. Provider threshold/imports in `VirtualDesktop.system.cs`.
2. New or reused build folder under `Interop/`.
3. Exact method order/signatures in `.interfaces/`.
4. Wrapper behavior in the non-hidden `.cs` files.
5. Embedded-resource entries in `VirtualDesktop.csproj`.
6. IID mappings in `app.config`.
7. Settings declarations in `Properties/Settings.settings` and generated `Properties/Settings.Designer.cs`.
8. Cached generated assemblies under LocalAppData; remove them when testing ABI changes.
9. Runtime tests on the exact Windows build/revision.

The IID data has multiple representations. `IID.GetIIDs()` enumerates properties exposed by the generated `WindowsDesktop.Properties.Settings` type, so do not update only `app.config`; keep `Settings.settings` synchronized and regenerate/check `Settings.Designer.cs`. The current 26100 implementation uses `v_26100_0000` consistently in all three files.

## Feature-to-file index

| Task | Start here | Also inspect |
|---|---|---|
| Change grid navigation/wrapping | `VirtualDesktopGridManager.cs` (`Left/Right/Up/Down`, `ColumnOf`, `RowOf`) | `SettingValues.cs`, settings dialog |
| Change switch/move behavior | `VirtualDesktopGridManager.cs` (`Current`, `Switch`, `Move`, `MoveWindow`) | `WinAPI.cs`, `VirtualDesktop.cs` |
| Add/change a hotkey option | `SettingValues.cs` | `SettingsDialog.cs` + Designer, `RegisterHotKeys()`, `Hotkey.cs` |
| Change tray menu/icon | `SysTrayProcess.cs`, `ContextMenus.cs` | `Icons/`, project Content entries |
| Change desktop polling | `VirtualDesktopTimer.cs` | `VirtualDesktopGridManager.Start/Stop`, timer settings UI |
| Change foreground tracking | `ForegroundWindowChanged()` | `WinAPI.cs`, browser/document helper methods |
| Change settings schema/default | `SettingValues.cs` | migration version, dialog load/save, user guide |
| Fix a P/Invoke | `VirtualDesktopGridSwitcher/WinAPI.cs` | Call sites in `VirtualDesktopGridManager.cs` |
| Fix virtual-desktop public API | `VirtualDesktop-master/src/VirtualDesktop/VirtualDesktop.cs` | `Interop/Proxy`, every affected build wrapper |
| Support a Windows build | `VirtualDesktop.system.cs`, `Interop/Build*/` | builder, IID/settings files, cache |
| Change app version | `VirtualDesktopGridSwitcher/Properties/AssemblyInfo.cs` | package/docs |
| Change library package version | `VirtualDesktop-master/src/Directory.Build.props` | nested publish workflow/package metadata |
| Work on error handling/logging | `ERROR_HANDLING_LOGGING_PLAN.md` | startup, empty catches, settings load/save |

## Build, test, and run

Requirements: Windows, .NET 8 SDK, and a compatible Windows SDK/Visual Studio 2022 installation.

```powershell
dotnet restore VirtualDesktopGridSwitcher.sln
dotnet build VirtualDesktopGridSwitcher.sln --configuration Release
```

Observed app output directory:

```text
bin/Release/net8.0-windows10.0.19041.0/win-x64/
```

The project is a self-contained `WinExe`; the project reference builds the vendored `VirtualDesktop` library. `Hotkey.cs` is linked as compile source rather than built as a separate project.

There are **no test projects**. Do not mistake a successful build for complete COM/runtime compatibility. A full local Release build during consolidation completed with **0 errors and 314 warnings**. Most concern Windows platform annotations and nullable analysis; the build also warns that the legacy explicit `System.Data.Linq` reference cannot be resolved.

A separate read-only COM smoke test passed on Windows 11 25H2 build 26200.8037. It selected `VirtualDesktopProvider26100`, compiled interfaces without using an existing cache, enumerated desktops, and resolved the current desktop. It did not create, remove, switch, or move anything; see `WINDOWS_11_24H2_FIX_GUIDE.md` for scope.

There is no active root-level GitHub Actions workflow. Workflows under `VirtualDesktop-master/.github/` belong to the vendored library snapshot, target its nested solution/branch conventions, and still mention .NET 7.

## Known sharp edges

These are navigation warnings, not an exhaustive bug list:

- `Start()` removes surplus real desktops, despite older user documentation saying reduction leaves them for manual deletion.
- Only 12 numbered tray icons ship, but grid settings can exceed 12 desktops; the UI merely warns above 20. `ShowIconForDesktop()` indexes the icon array directly.
- If timer polling is disabled, `virtualDesktopTimer` is never assigned, but `Stop()` calls `virtualDesktopTimer.Stop()` unconditionally.
- `Start()` catches desktop-initialization failures and then continues into icon/hotkey setup, so partially initialized state is possible. `Program.Main()` has no global exception handling.
- `Current` currently invokes a desktop switch inside its null check and then invokes `Switch()` again afterward. Review this carefully before changing switch timing.
- Native notification plumbing exists but is disabled; enabling it means coordinating both library registration and app subscription and reconsidering manual/polling callbacks.
- Several empty or lossy `catch` blocks suppress pin/move/save/initialization details. Persistent logging is only planned, not implemented.
- `GetWindowExeName()` opens process handles with broad access and has no corresponding `CloseHandle` declaration/call.
- `WinAPI.cs` contains an unused overload of `SetWindowPos` that throws `NotImplementedException`.
- Settings validation does not enforce positive grid dimensions or a tray-icon-sized maximum, and timer parsing shares the rows/columns error catch.
- Virtual desktops are system-wide rather than independently selectable per monitor.
- Some packaged/UWP application windows may not expose handles or move like conventional desktop windows.
- The COM ABI is undocumented and version-sensitive. A compile-successful interface change can still crash Explorer-facing calls at runtime.

## Documentation trust and legacy files

- This file is the single canonical architecture and code-navigation guide.
- `Documentation/UserGuide/...md` and `Documentation/Installation/...md` explain user-facing behavior, but contain historical statements; compare them with current code.
- `WINDOWS_11_24H2_FIX_GUIDE.md` records the exact compatibility implementation and test scope; it is not proof for every future Windows revision.
- `MIGRATION_TO_NET8.md` records the current framework migration; project files are the final authority.
- `ROADMAP.md` and `ERROR_HANDLING_LOGGING_PLAN.md` describe proposed work, not implemented features.
- `.hgignore`, `.hgtags`, root `Windows10SDKVS13_*.props`, and much of `Package/` are historical/migration artifacts.
- `VirtualDesktop-master/README.md` and its nested workflows reflect the vendored library snapshot and are stale regarding this application fork's .NET 8-only target.

## Glossary

| Term | Meaning |
|---|---|
| COM | Component Object Model, the binary interface used by Explorer's internal desktop services. |
| CLSID | Identifier for a COM class/service. |
| IID | Identifier for a particular COM interface layout. |
| HWND | Native Windows handle identifying a window. |
| HSTRING | Windows Runtime string representation used by several desktop methods. |
| P/Invoke | Managed declaration used to call a native Windows function. |
| UBR | Update Build Revision, the fourth component of the full Windows build. |
| vtable | Ordered COM method table; incorrect ordering can invoke the wrong native function. |

## Editing conventions for future agents

- Keep changes in the smallest responsible layer: app policy in `VirtualDesktopGridSwitcher`, OS ABI handling in `VirtualDesktop-master`.
- Do not edit `*.Designer.cs`, `*.designer.cs`, `*.resx`, or `Properties/Settings.Designer.cs` casually; use the WinForms/settings designers or regenerate and inspect their output.
- Treat `.interfaces/*.cs` as embedded runtime compiler input, even though they look like ordinary source files.
- When changing settings, update defaults, load/save UI, migration logic, and documentation together.
- When changing desktop-switch callbacks, account for three paths: manual callback, timer polling, and currently disabled native notification.
- Build artifacts (`bin/`, `obj/`) are ignored. The tracked `Package/` directory is also ignored for new files because it predates the current `.gitignore`.
- Never launch the app as a routine automated check; use build/static tests unless desktop mutation is explicitly intended.
