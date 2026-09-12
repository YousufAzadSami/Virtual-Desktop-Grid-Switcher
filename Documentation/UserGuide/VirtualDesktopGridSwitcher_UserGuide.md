# Virtual Desktop Grid Switcher User Guide

Virtual Desktop Grid Switcher is a Windows notification-area application that arranges the system's virtual desktops as a row-major grid. It provides global hotkeys for switching desktops, moving the foreground window, pinning a window across desktops, and toggling always-on-top.

> **Desktop-count warning:** the current application enforces exactly `Rows × Columns` real virtual desktops whenever it starts or applies settings. It creates missing desktops and removes surplus desktops. The default 3 × 3 grid therefore enforces nine desktops.

See the [installation guide](../Installation/VirtualDesktopGridSwitcher_Installation.md) before the first launch.

## Notification-area menu

The application has no main window. While it is running, a numbered icon identifies the current desktop in the Windows notification area.

![Numbered Virtual Desktop Grid Switcher icon](attachment/image1.png)

Right-click the icon to open its menu:

![Virtual Desktop Grid Switcher notification-area menu](attachment/image2.png)

- **Settings** opens grid and hotkey configuration.
- **About** displays application information.
- **Exit** stops the application and unregisters its hotkeys.

If the icon is hidden, use Windows taskbar settings to move it out of the notification-area overflow menu.

## Grid layout

Desktops are numbered from left to right and then top to bottom. The default 3 × 3 grid is:

```text
1  2  3
4  5  6
7  8  9
```

Moving right from desktop 3 remains on desktop 3 unless **Wrap Around** is enabled. With wrapping enabled, it moves to desktop 1. Vertical movement follows the same rule within each column.

### Changing rows or columns

Open **Settings**, change **Rows** or **Columns**, and select **Apply**. Applying settings immediately restarts desktop management and changes the real desktop count:

- If `Rows × Columns` is larger than the current count, desktops are created.
- If it is smaller, surplus desktops are removed.

Use positive values only. The current validation is limited, and very large grids can make Windows slow or unstable.

The application ships numbered icons only for desktops 1 through 12. Do not configure more than 12 desktops in the current version; switching to a desktop without a corresponding icon is not handled safely.

## Default hotkeys

| Action | Default shortcut |
|---|---|
| Switch left/right/up/down | **Ctrl+Alt+Arrow key** |
| Move the foreground window and switch left/right/up/down | **Ctrl+Alt+Shift+Arrow key** |
| Switch directly to desktop 1–9 | **Ctrl+Alt+1–9** |
| Move the foreground window and switch to desktop 1–9 | **Ctrl+Alt+Shift+1–9** |
| Toggle always-on-top for the foreground window | **Ctrl+Alt+Space** |
| Toggle sticky/pinned state for the foreground window | **Ctrl+Alt+Shift+Space** |

“Move” always acts on the current foreground window and then switches to the destination desktop.

A sticky window is pinned through the Windows virtual-desktop API so that it appears on every desktop. Always-on-top is independent: it controls whether the window remains above ordinary windows.

Some packaged applications, special system windows, or applications with unusual window ownership may not support moving or pinning correctly.

## Configuring hotkeys

The Settings dialog separates shortcuts into these groups:

- **By Direction**: switch or move using arrows or configured direction keys.
- **By Position**: switch or move directly using numbers, F1–F12, or custom keys.
- **Sticky Window** and **Always On Top**: configure each toggle independently.

Each action group has its own enable checkbox and Ctrl, Win, Alt, and Shift modifiers.

Additional behavior:

- **Arrow / Direction Keys** enables the standard arrow keys.
- **Numbers 1–9** enables direct shortcuts for existing desktops up to desktop 9.
- **F1–F12** enables direct shortcuts for existing desktops up to desktop 12.
- The twelve desktop key fields allow custom action keys.
- Select a key field and press a key to assign it. Press **Delete** while a value is selected to clear it.

Only assign custom keys for desktop positions that exist in the configured grid. The current application does not guard custom shortcuts that target a nonexistent desktop.

Select **Apply** to register the new hotkeys, apply the grid, and save settings. Select **Cancel** to close the dialog without applying the displayed changes.

### Hotkey conflicts

Windows permits only one application to own a particular global hotkey. If registration fails:

1. Check whether the same shortcut has multiple assignments in this application.
2. Change the shortcut in Settings.
3. Check graphics, keyboard, window-management, launcher, and automation utilities for conflicting shortcuts.

A failed shortcut is unavailable, but other successfully registered shortcuts may continue to work.

## Wrap-around behavior

When **Wrap Around** is disabled, moving beyond a grid edge keeps you on the current desktop. When enabled:

- Left/right wraps within the current row.
- Up/down wraps within the current column.

The same destination calculation is used for switching and moving a window.

## Detecting switches made outside the application

The application polls Windows to detect desktop changes made through Task View or other shortcuts. Polling is enabled by default with a 500 ms interval.

Leave polling enabled in the current version. The shutdown/restart path is not yet safe when polling is disabled. The interval must be a positive number of milliseconds; excessively small values add unnecessary work.

Avoid adding or removing desktops through another tool while Virtual Desktop Grid Switcher is running. Its desktop lookup is built when management starts and is not designed to track external changes to the desktop collection.

## Window activation and document-opening workarounds

The application remembers recently active windows and attempts to restore focus when switching from an empty desktop.

It also contains a workaround for Word, Excel, and Adobe Acrobat Reader windows that may initially open on the desktop containing another document window. By default, these executable names are monitored:

- `WINWORD.EXE`
- `EXCEL.EXE`
- `AcroRd32.exe`

The detection window defaults to 5000 ms. These advanced values are not exposed in the Settings dialog; they can be changed in `VirtualDesktopGridSwitcher.Settings` using `MoveOnNewWindowExeNames` and `MoveOnNewWindowDetectTimeoutMs`.

Legacy default-browser activation logic also exists but is disabled by default because modern browsers generally handle cross-desktop links correctly.

## Settings file

Settings are serialized as XML beside the executable:

```text
VirtualDesktopGridSwitcher.Settings
```

Exit the application and back up the file before editing it manually. Invalid XML or unsupported values can prevent startup or cause runtime errors. Starting without a settings file restores defaults, including the 3 × 3 desktop grid, so deleting the file is not a harmless reset.

The most important advanced settings are:

| Setting | Purpose | Default |
|---|---|---:|
| `TimerEnabled` | Detect desktop switches made outside this application | `true` |
| `IntervalMs` | Polling interval in milliseconds | `500` |
| `MoveOnNewWindowDetectTimeoutMs` | Detection window for document-opening workaround | `5000` |
| `MoveOnNewWindowExeNames` | Executables monitored by that workaround | Word, Excel, Acrobat Reader |
| `ActivateWebBrowserOnSwitch` | Legacy browser-activation workaround | `false` |

## Multiple monitors

Windows virtual desktops are system-wide in this application. The grid cannot switch an independent desktop on only one monitor.

## Troubleshooting and support

### Desktop initialization error

Do not repeatedly relaunch the application. Record the Windows edition, display version, build, and UBR shown by `winver`, then check the [Windows compatibility notes](../Technical/WINDOWS_COMPATIBILITY.md).

### Wrong number of desktops

The application intentionally synchronizes Windows to the configured `Rows × Columns` count. Exit the application before manually reorganizing desktops if you do not want it to enforce that count again.

### Window does not move or become sticky

Make sure the intended window is in the foreground. Some system, packaged, elevated, or specially owned windows cannot be manipulated through the same APIs as ordinary desktop windows.

### Reporting a problem

Use the repository's [GitHub issue tracker](https://github.com/YousufAzadSami/Virtual-Desktop-Grid-Switcher/issues). Include:

- Application version
- Windows edition, display version, build, and UBR
- Grid dimensions and relevant hotkey settings
- Exact steps and error text
- Whether the operation was switching, moving, pinning, or changing the desktop collection

Do not include confidential window titles, settings, or logs without reviewing them first.
