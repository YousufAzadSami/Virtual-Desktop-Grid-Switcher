# Virtual Desktop Grid Switcher Installation

Virtual Desktop Grid Switcher is currently a portable application; it does not have an installer. Keep the complete application folder in a stable, user-writable location.

> **Important:** starting the application immediately changes the machine's real virtual-desktop count to exactly the configured grid size. A fresh installation defaults to a 3 × 3 grid, so its first launch creates desktops until there are nine and removes any desktops beyond nine. Do not launch the current version unless that behavior is acceptable.

## Requirements

- 64-bit Windows 10 build 19041 or later, or Windows 11
- A Windows build covered by the current [compatibility notes](../Technical/WINDOWS_COMPATIBILITY.md)
- The .NET runtime is not required when using a self-contained release/build

Support for the undocumented virtual-desktop COM API depends on the exact Windows build. A listed provider does not necessarily mean every operation has been tested on that release.

## Install from a release archive

If a suitable package is available on the repository's [Releases page](https://github.com/YousufAzadSami/Virtual-Desktop-Grid-Switcher/releases):

1. Download the ZIP archive from a trusted release.
2. Before extracting, right-click the ZIP, select **Properties**, and use **Unblock** if Windows displays that option.

   ![Unblock option in ZIP properties](media/Unblock.png)

3. Extract the complete archive to a stable, user-writable directory, such as:

   ```text
   C:\Users\<your-name>\Programs\VirtualDesktopGridSwitcher
   ```

4. Keep `VirtualDesktopGridSwitcher.exe`, its DLLs, configuration files, and the `Icons` directory together.
5. Read the startup warning above, then run `VirtualDesktopGridSwitcher.exe` only when you are ready for the configured desktop count to be applied.

If no prebuilt archive is available, build the application from source.

## Build and install from source

Developers need the .NET 8 SDK and a compatible Windows SDK or Visual Studio 2022 installation.

From the repository root:

```powershell
dotnet restore VirtualDesktopGridSwitcher.sln
dotnet build VirtualDesktopGridSwitcher.sln --configuration Release
```

The application output is produced under:

```text
bin\Release\net8.0-windows10.0.19041.0\win-x64\
```

Copy the complete output directory to the chosen installation location. Do not copy only the executable.

## First run and notification-area icon

The application has no main window. When running, it displays a numbered icon in the Windows notification area. Right-click the icon for **Settings**, **About**, and **Exit**.

If the icon is hidden, use Windows taskbar settings to show Virtual Desktop Grid Switcher directly rather than in the notification-area overflow menu.

## Settings and upgrades

Settings are stored beside the executable in:

```text
VirtualDesktopGridSwitcher.Settings
```

The file is created after settings are successfully applied or migrated. Because it is stored in the application directory, install the application somewhere your account can write to.

Before upgrading:

1. Exit the application from its notification-area menu.
2. Back up `VirtualDesktopGridSwitcher.Settings` and any customized `Icons` directory.
3. Replace the application files as a complete set.
4. Restore only deliberate local customizations.

## Start automatically at login

Automatic startup is not built into the application. Enable it only after manually verifying the application and accepting its desktop-count behavior.

1. Press **Win+R**.
2. Enter `shell:startup` and press **Enter**.

   ![Opening the Startup folder from the Run dialog](media/image2.png)

3. Create a shortcut in that folder pointing to `VirtualDesktopGridSwitcher.exe`.

Remove the shortcut to disable automatic startup.

## Troubleshooting

### A hotkey cannot be registered

Another application probably owns the same global key combination. Change the shortcut in **Settings**, or disable the conflicting shortcut in graphics, keyboard, window-management, or automation software.

### Settings cannot be saved

Confirm that your account can write to the installation directory and that the settings file is not read-only.

### Windows blocks the executable

Use only artifacts obtained from a source you trust. If Windows preserved download-zone information, unblock the ZIP before extracting it rather than unblocking individual files afterward.

### Desktop initialization fails

Record the complete Windows build and UBR (`winver`), then consult the [Windows compatibility notes](../Technical/WINDOWS_COMPATIBILITY.md). Do not repeatedly relaunch the application if initialization is failing.

## Uninstall

1. Exit the application.
2. Remove any shortcut from `shell:startup`.
3. Delete the application directory.

The application is portable and does not currently register an installed product with Windows.
