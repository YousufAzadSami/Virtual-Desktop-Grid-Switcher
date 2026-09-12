# .NET 8 Migration Record

**Completed:** April 3, 2026

**Status:** Complete

This document records the migration of the application and its vendored VirtualDesktop library from .NET 5/6/7 targets to .NET 8. It is a migration record, not a guarantee of compatibility with every Windows build; the undocumented COM layer must be validated separately.

## Project changes

### Application

`VirtualDesktopGridSwitcher/VirtualDesktopGridSwitcher.csproj` now targets:

```xml
<TargetFramework>net8.0-windows10.0.19041.0</TargetFramework>
```

Updated packages:

- `Microsoft.Windows.Compatibility`: 6.0.0 → 8.0.0
- `Microsoft.DotNet.UpgradeAssistant.Extensions.Default.Analyzers`: 0.3.330701 → 0.4.421302

The application remains a self-contained, 64-bit Windows executable:

```xml
<RuntimeIdentifier>win-x64</RuntimeIdentifier>
<SelfContained>true</SelfContained>
<PublishSingleFile>false</PublishSingleFile>
<PublishTrimmed>false</PublishTrimmed>
```

Single-file publishing and trimming remain disabled because the VirtualDesktop library discovers embedded interface definitions and compiles COM interface assemblies at runtime.

### Vendored VirtualDesktop library

`VirtualDesktop-master/src/VirtualDesktop/VirtualDesktop.csproj` now targets only:

```xml
<TargetFrameworks>net8.0-windows10.0.19041.0</TargetFrameworks>
```

Updated packages and build settings:

- `Microsoft.CodeAnalysis.CSharp`: 4.0.1 → 4.11.0
- `Microsoft.Windows.CsWinRT`: unified on 2.1.1
- C#/WinRT projection generation explicitly disabled
- .NET 5, 6, and 7 conditional build groups removed

The vendored project is built as part of the application solution; this migration does not claim continued multi-targeted library-package compatibility.

## Compatibility boundaries

- The application declares Windows `10.0.19041.0` (Windows 10 version 2004) as its minimum supported platform.
- It compiles against Windows SDK contracts for the same `10.0.19041.0` platform version.
- End users do not need to install the .NET runtime because the application is self-contained.
- Developers need a .NET 8 SDK and a compatible Windows SDK/Visual Studio 2022 installation.
- Windows virtual-desktop compatibility is controlled independently by build-specific COM providers and IID settings. Successful compilation does not prove runtime support on a particular Windows update.

## Build verification

```powershell
dotnet restore VirtualDesktopGridSwitcher.sln
dotnet build VirtualDesktopGridSwitcher.sln --configuration Release
```

Expected output location:

```text
bin/Release/net8.0-windows10.0.19041.0/win-x64/
```

The repository currently has no automated tests. A manual runtime check should cover:

- Tray icon creation
- Settings load/save
- Hotkey registration and conflict reporting
- Desktop switching and window movement
- Sticky-window and always-on-top commands
- External desktop-change polling
- The exact Windows builds listed as supported

**Caution:** normal application startup changes the machine's real virtual-desktop count to match the configured grid. Do not use it as an unattended smoke test.

## Rollback

Use Git to revert the migration commit rather than manually changing only the target-framework elements. The package versions and C#/WinRT build settings are part of the same migration and must remain synchronized.

## Maintenance

.NET 8 is an LTS release supported through November 2026. Plan the next LTS migration before end of support and validate the runtime COM compilation path again when changing framework, Roslyn, or C#/WinRT versions.
