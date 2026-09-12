# Virtual Desktop Grid Switcher

A Windows 10/11 notification-area application that arranges virtual desktops as a configurable grid. Global hotkeys can switch desktops, move the foreground window, pin a window across desktops, and toggle always-on-top.

> **Startup safety:** the current application changes the machine's real desktop count to exactly `Rows × Columns`, creating missing desktops and removing surplus desktops. Review this behavior and your settings before launching it.

## Documentation

- [Technical documentation and codebase navigation](Documentation/Technical/TECHNICAL_DOCUMENTATION.md)
- [Installation guide](Documentation/Installation/VirtualDesktopGridSwitcher_Installation.md)
- [User guide](Documentation/UserGuide/VirtualDesktopGridSwitcher_UserGuide.md)
- [Windows 11 24H2/25H2 compatibility notes](Documentation/Technical/WINDOWS_11_24H2_FIX_GUIDE.md)
- [.NET 8 migration](Documentation/Technical/MIGRATION_TO_NET8.md)
- [Project roadmap](ROADMAP.md)
- [Error-handling and logging plan](Documentation/Technical/ERROR_HANDLING_LOGGING_PLAN.md)

## Build

The application targets .NET 8 for 64-bit Windows and is self-contained.

```powershell
dotnet restore VirtualDesktopGridSwitcher.sln
dotnet build VirtualDesktopGridSwitcher.sln --configuration Release
```

See [`Documentation/Technical/TECHNICAL_DOCUMENTATION.md`](Documentation/Technical/TECHNICAL_DOCUMENTATION.md) before changing or testing the undocumented, Windows-build-specific COM layer.

## Troubleshooting

If Windows reports downloaded release files as blocked, open the ZIP file's **Properties**, select **Unblock**, and then extract it again.

If a configured hotkey cannot be registered, another application may already own that combination. Change the combination in Settings or disable the conflicting shortcut in graphics/keyboard utility software.
