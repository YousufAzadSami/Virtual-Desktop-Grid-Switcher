# Virtual Desktop Grid Switcher - Comprehensive Technical Documentation

**Version:** 4.0.2.4 (kloned fork)  
**Original Project:** https://sourceforge.net/projects/virtual-desktop-grid-switcher/  
**Forked Repository:** https://github.com/kloned/Virtual-Desktop-Grid-Switcher  
**Last Updated:** March 2026

---

## Table of Contents

1. [Project Overview](#project-overview)
2. [Architecture & Design](#architecture--design)
3. [Core Components](#core-components)
4. [Virtual Desktop Library Integration](#virtual-desktop-library-integration)
5. [Windows API & COM Interop](#windows-api--com-interop)
6. [The Windows 11 24H2 Problem](#the-windows-11-24h2-problem)
7. [Build-Specific Implementations](#build-specific-implementations)
8. [Settings & Configuration](#settings--configuration)
9. [Known Issues & Limitations](#known-issues--limitations)
10. [Development Guide](#development-guide)

---

## Project Overview

### What is Virtual Desktop Grid Switcher?

Virtual Desktop Grid Switcher (VDGS) is a Windows 10/11 application that enhances the native virtual desktop functionality by organizing desktops in a **grid layout** (e.g., 3x3) and enabling keyboard-based navigation using arrow keys with configurable modifier keys.

### Key Features

- **Grid-based desktop layout** (configurable rows × columns)
- **Keyboard shortcuts** for switching desktops (directional and positional)
- **Move active windows** between desktops via hotkeys
- **Always on Top** toggle for windows
- **Sticky Windows** (visible on all desktops)
- **Browser activation fixes** for Chrome/Firefox
- **Document opening fixes** for Word/Excel/Acrobat Reader
- **System tray integration** with desktop indicator icons
- **Wrap-around navigation** (optional)

### Technology Stack

- **Language:** C# (.NET 7.0)
- **Target Framework:** `net7.0-windows10.0.19041.0`
- **UI Framework:** Windows Forms
- **Deployment:** Self-contained executable
- **Dependencies:**
  - Custom VirtualDesktop library (modified from Grabacr07/VirtualDesktop)
  - GlobalHotkeysWithDotNET (modified)
  - Windows COM Interop

---

## Architecture & Design

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Program.cs (Entry Point)                  │
│                  - Initializes Application                   │
│                  - Loads Settings                            │
└────────────────────────┬────────────────────────────────────┘
                         │
         ┌───────────────┴───────────────┐
         │                               │
┌────────▼────────┐            ┌────────▼──────────┐
│  SysTrayProcess │            │ VirtualDesktop    │
│  - System Tray  │            │ GridManager       │
│  - Context Menu │            │ - Core Logic      │
│  - Icon Display │            │ - Hotkey Handling │
└─────────────────┘            │ - Desktop Control │
                               └────────┬──────────┘
                                        │
                    ┌───────────────────┼───────────────────┐
                    │                   │                   │
         ┌──────────▼──────────┐ ┌─────▼─────┐  ┌─────────▼────────┐
         │ VirtualDesktop Lib  │ │  WinAPI   │  │ VirtualDesktop   │
         │ (WindowsDesktop)    │ │  Wrapper  │  │     Timer        │
         │ - COM Interop       │ │ - P/Invoke│  │ - Polling Fallback│
         │ - Build Detection   │ │ - Window  │  └──────────────────┘
         │ - Interface Wrapper │ │   Mgmt    │
         └─────────────────────┘ └───────────┘
                    │
         ┌──────────▼──────────────────────────┐
         │  Windows Internal COM Interfaces    │
         │  - IVirtualDesktopManagerInternal   │
         │  - IVirtualDesktop                  │
         │  - IVirtualDesktopPinnedApps        │
         └─────────────────────────────────────┘
```

### Application Flow

1. **Startup:**
   - `Program.Main()` initializes Windows Forms
   - Loads `SettingValues` from XML file
   - Creates `SysTrayProcess` (system tray icon)
   - Creates `VirtualDesktopGridManager` (main controller)
   - Enters Windows Forms message loop

2. **Initialization:**
   - VirtualDesktop library detects Windows build
   - Loads appropriate COM interface GUIDs from `app.config`
   - Creates/removes desktops to match grid configuration
   - Registers global hotkeys
   - Starts desktop change detection timer (if enabled)
   - Hooks foreground window change events

3. **Runtime:**
   - Listens for hotkey presses
   - Monitors foreground window changes
   - Tracks active windows per desktop
   - Handles desktop switching
   - Manages window movement between desktops

---

## Core Components

### 1. Program.cs

**Location:** `VirtualDesktopGridSwitcher/Program.cs`

**Purpose:** Application entry point

**Key Responsibilities:**
- Initialize Windows Forms application
- Load settings from `VirtualDesktopGridSwitcher.Settings` file
- Create and manage `SysTrayProcess` and `VirtualDesktopGridManager`
- Run application message loop

```csharp
static void Main() {
    Application.EnableVisualStyles();
    Application.SetHighDpiMode(HighDpiMode.SystemAware);
    
    var settings = SettingValues.Load();
    using (SysTrayProcess sysTrayProcess = new SysTrayProcess(settings)) {
        using (VirtualDesktopGridManager gridManager = 
                   new VirtualDesktopGridManager(sysTrayProcess, settings)) {
            Application.Run();
        }
    }
}
```

---

### 2. VirtualDesktopGridManager.cs

**Location:** `VirtualDesktopGridSwitcher/VirtualDesktopGridManager.cs`

**Purpose:** Core business logic for grid-based desktop management

**Key Responsibilities:**

#### Desktop Management
- Creates/removes desktops to match configured grid size (`Rows × Columns`)
- Maintains lookup tables:
  - `desktopIdLookup`: Array mapping grid index → Desktop GUID
  - `desktopNumberLookup`: Dictionary mapping Desktop GUID → grid index
- Tracks current desktop position in grid

#### Navigation Logic
```csharp
public int Left {
    get {
        if (ColumnOf(Current - 1) < ColumnOf(Current)) {
            return Current - 1;
        } else {
            return settings.WrapAround ? Current + settings.Columns - 1 : Current;
        }
    }
}
```

Similar logic for `Right`, `Up`, `Down` with wrap-around support.

#### Hotkey Registration
- Registers global hotkeys using `GlobalHotkeyWithDotNET` library
- Supports multiple hotkey schemes:
  - **Direction keys:** Arrow keys or custom keys with modifiers
  - **Position keys:** Number keys (1-9), F-keys (F1-F12), or custom keys
  - **Special functions:** Always on Top, Sticky Window

#### Window Tracking
- `activeWindows[]`: Tracks last active window per desktop
- `lastActiveBrowserWindows[]`: Tracks browser windows per desktop
- Uses Windows event hooks to monitor foreground window changes

#### Browser Activation Fix
When switching desktops, activates the last-used browser window on the target desktop to ensure links open in the correct window.

#### Document Opening Fix
Detects when Word/Excel/Acrobat opens a document on the wrong desktop and moves it back within a configurable timeout (`MoveOnNewWindowDetectTimeoutMs`).

---

### 3. SysTrayProcess.cs

**Location:** `VirtualDesktopGridSwitcher/SysTrayProcess.cs`

**Purpose:** System tray icon management

**Key Responsibilities:**
- Creates `NotifyIcon` in system tray
- Loads numbered icons (1.ico - 12.ico) from `Icons/` folder
- Updates icon to show current desktop number
- Provides context menu (Settings, About, Exit)

```csharp
public void ShowIconForDesktop(int desktopIndex) {
    notifyIcon.Icon = desktopIcons[desktopIndex];
}
```

---

### 4. SettingValues.cs

**Location:** `VirtualDesktopGridSwitcher/Settings/SettingValues.cs`

**Purpose:** Configuration management

**Key Settings:**

| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| `Columns` | int | 3 | Grid columns |
| `Rows` | int | 3 | Grid rows |
| `WrapAround` | bool | false | Enable wrap-around navigation |
| `SwitchDirModifiers` | Modifiers | Ctrl+Alt | Modifiers for directional switching |
| `MoveDirModifiers` | Modifiers | Ctrl+Alt+Shift | Modifiers for moving windows |
| `ArrowKeysEnabled` | bool | true | Enable arrow keys |
| `NumbersEnabled` | bool | true | Enable number keys (1-9) |
| `FKeysEnabled` | bool | false | Enable F-keys (F1-F12) |
| `AlwaysOnTopHotkey` | Hotkey | Ctrl+Alt+Space | Toggle always on top |
| `StickyWindowHotKey` | Hotkey | Ctrl+Alt+Shift+Space | Toggle sticky window |
| `ActivateWebBrowserOnSwitch` | bool | false | Browser activation feature |
| `MoveOnNewWindowDetectTimeoutMs` | int | 5000 | Document move detection timeout |
| `TimerEnabled` | bool | true | Enable desktop change polling |
| `IntervalMs` | int | 500 | Polling interval |

**Persistence:**
- Saved to `VirtualDesktopGridSwitcher.Settings` (XML format)
- Auto-migrates from older versions

---

### 5. VirtualDesktopTimer.cs

**Location:** `VirtualDesktopGridSwitcher/VirtualDesktopTimer.cs`

**Purpose:** Fallback desktop change detection

**Why It Exists:**
The VirtualDesktop library's event-based notification (`VirtualDesktop.CurrentChanged`) is unreliable on some Windows builds. The timer provides a polling-based fallback.

**How It Works:**
```csharp
private void DoOnTimerTick(object sender, EventArgs e) {
    if (VirtualDesktop.Current != VirtualDesktop.FromId(desktopIdLookup[getCurrentDesktop()])) {
        onDesktopChanged(null, 
            new VirtualDesktopChangedEventArgs(
                VirtualDesktop.FromId(desktopIdLookup[getCurrentDesktop()]),
                VirtualDesktop.Current));
    }
}
```

Polls every 500ms (configurable) to detect external desktop switches (e.g., via Windows Task View).

---

### 6. WinAPI.cs

**Location:** `VirtualDesktopGridSwitcher/WinAPI.cs`

**Purpose:** Windows API P/Invoke wrappers

**Key Functions:**

| Function | Purpose |
|----------|---------|
| `GetForegroundWindow()` | Get currently active window |
| `SetForegroundWindow()` | Activate a window |
| `FindWindowEx()` | Find child windows by class name |
| `GetWindowPlacement()` | Get window state (minimized, etc.) |
| `SetWindowPos()` | Set window Z-order (for Always on Top) |
| `SetWinEventHook()` | Hook window events (foreground changes) |
| `GetWindowThreadProcessId()` | Get process ID from window handle |
| `GetProcessImageFileName()` | Get executable path from process |

---

## Virtual Desktop Library Integration

### Library Origin

The project uses a **modified version** of the VirtualDesktop library originally created by **Manato KAMEYA** (Grabacr07), further maintained by **Slion**.

**Original:** https://github.com/Grabacr07/VirtualDesktop  
**Fork Used:** https://github.com/Slion/VirtualDesktop

### Library Architecture

The VirtualDesktop library is a C# wrapper around **undocumented Windows COM interfaces** that control virtual desktops.

#### Key Classes

**1. VirtualDesktop.cs**

Main API surface:

```csharp
public partial class VirtualDesktop {
    public Guid Id { get; }
    public string Name { get; set; }
    public string WallpaperPath { get; set; }
    
    public void Switch();
    public void Remove();
    public VirtualDesktop? GetLeft();
    public VirtualDesktop? GetRight();
    
    public static VirtualDesktop Current { get; }
    public static VirtualDesktop[] GetDesktops();
    public static VirtualDesktop Create();
    public static VirtualDesktop? FromId(Guid desktopId);
    public static VirtualDesktop? FromHwnd(IntPtr hWnd);
    
    public static void MoveToDesktop(IntPtr hWnd, VirtualDesktop desktop);
    public static bool PinWindow(IntPtr hWnd);
    public static bool UnpinWindow(IntPtr hWnd);
}
```

**2. VirtualDesktopProvider**

Abstract base class for build-specific implementations:

```csharp
internal abstract class VirtualDesktopProvider {
    public abstract IApplicationViewCollection ApplicationViewCollection { get; }
    public abstract IVirtualDesktopManager VirtualDesktopManager { get; }
    public abstract IVirtualDesktopManagerInternal VirtualDesktopManagerInternal { get; }
    public abstract IVirtualDesktopPinnedApps VirtualDesktopPinnedApps { get; }
    public abstract IVirtualDesktopNotificationService VirtualDesktopNotificationService { get; }
}
```

Each Windows build has its own provider implementation (e.g., `Build26100.Provider`, `Build22621.Provider`).

**3. ComInterfaceAssemblyBuilder**

Dynamically generates COM interface assemblies at runtime:

1. Detects current Windows build version
2. Loads appropriate interface GUIDs from `app.config`
3. Selects matching interface definitions from embedded resources
4. Compiles them into a temporary assembly using Roslyn
5. Caches the assembly in `%LocalAppData%\VirtualDesktop\`

---

## Windows API & COM Interop

### The Undocumented API Problem

Windows does **not** provide a public API for programmatically managing virtual desktops. The VirtualDesktop library works by:

1. **Reverse-engineering** internal COM interfaces
2. **Discovering GUIDs** via registry inspection
3. **Invoking methods** via COM interop

### Key COM Interfaces

#### IVirtualDesktopManagerInternal

The main interface for desktop management:

```csharp
public interface IVirtualDesktopManagerInternal {
    IEnumerable<IVirtualDesktop> GetDesktops();
    IVirtualDesktop GetCurrentDesktop();
    IVirtualDesktop GetAdjacentDesktop(IVirtualDesktop pDesktopReference, AdjacentDesktop uDirection);
    IVirtualDesktop FindDesktop(Guid desktopId);
    IVirtualDesktop CreateDesktop();
    void MoveDesktop(IVirtualDesktop pMove, int nIndex);
    void RemoveDesktop(IVirtualDesktop pRemove, IVirtualDesktop pFallbackDesktop);
    void SwitchDesktop(IVirtualDesktop desktop);
    void MoveViewToDesktop(IntPtr hWnd, IVirtualDesktop desktop);
    void SetDesktopName(IVirtualDesktop desktop, string name);
    void SetDesktopWallpaper(IVirtualDesktop desktop, string path);
}
```

#### IVirtualDesktop

Represents a single virtual desktop (COM object).

#### IVirtualDesktopPinnedApps

Manages pinned windows/applications:

```csharp
public interface IVirtualDesktopPinnedApps {
    bool IsViewPinned(IntPtr hWnd);
    void PinView(IntPtr hWnd);
    void UnpinView(IntPtr hWnd);
    bool IsAppIdPinned(string appUserModelId);
    void PinAppID(string appUserModelId);
    void UnpinAppID(string appUserModelId);
}
```

### GUID Discovery Process

**From app.config:**

```xml
<setting name="v_26100_863" serializeAs="Xml">
  <value>
    <ArrayOfString>
      <string>IVirtualDesktop,{3F07F4BE-B107-441A-AF0F-39D82529072C}</string>
      <string>IVirtualDesktopManagerInternal,{A3175F2D-239C-4BD2-8AA0-EEBA8B0B138E}</string>
      ...
    </ArrayOfString>
  </value>
</setting>
```

**Fallback via Registry:**

If GUIDs aren't in config, the library searches `HKEY_CLASSES_ROOT\Interface` for matching interface names.

---

## The Windows 11 24H2 Problem

### The Issue

**Build:** 26100.x (Windows 11 24H2)  
**Error:** `System.AccessViolationException: Attempted to read or write protected memory.`  
**Location:** `IVirtualDesktopManagerInternal.CreateDesktop()`

### Root Cause

Microsoft **completely rewrote** the virtual desktop implementation in Windows 11 24H2:

1. **Changed COM interface GUIDs**
2. **Modified method signatures**
3. **Altered memory layouts**
4. **Changed vtable offsets**

The existing `Build26100_863` implementation targets **build 26100.863**, but later patches (e.g., 26100.7627+) have further changes.

### Why It Crashes Silently

1. Application starts
2. Attempts to call `VirtualDesktop.Create()` during initialization
3. COM method invocation points to **wrong memory address**
4. **Access violation** → immediate crash
5. No error dialog because crash happens before UI initialization

### Historical Pattern

This happens **every major Windows update**:

| Windows Version | Build | Issue | Fix |
|----------------|-------|-------|-----|
| Windows 10 1607 | 14393 | New GUIDs | Community updated |
| Windows 10 1703 | 15063 | Method signature changes | Community updated |
| Windows 10 1809 | 17763 | Interface restructure | Community updated |
| Windows 10 21H2 | 19044 | GUID changes | Community updated |
| Windows 11 21H2 | 22000 | Major rewrite | Community updated |
| Windows 11 22H2 | 22621 | Multiple sub-builds | Multiple updates |
| **Windows 11 24H2** | **26100** | **Major rewrite** | **Ongoing** |

### The Fix Process

To fix for a new build:

1. **Obtain the new build** (Insider Preview or release)
2. **Use debugging tools** (WinDbg, IDA Pro, Ghidra)
3. **Inspect system DLLs** (`twinui.dll`, `twinui.pcshell.dll`, `shell32.dll`)
4. **Find COM class registrations** in registry
5. **Reverse-engineer vtables** to find method offsets
6. **Extract new GUIDs** for each interface
7. **Test method signatures** (parameters, calling conventions)
8. **Create new Build folder** (e.g., `Build26100_1234/`)
9. **Add GUIDs to app.config**
10. **Compile and test**

### Current Status (March 2026)

- **kloned's v4.0.2.4** targets build 26100.863
- **Later builds** (26100.7627+) are **broken**
- Community is working on reverse-engineering latest changes
- No official Microsoft support (undocumented API)

---

## Build-Specific Implementations

### Directory Structure

```
VirtualDesktop-master/src/VirtualDesktop/Interop/
├── Build10240_0000/          # Windows 10 RTM
├── Build20348_0000/          # Windows Server 2022
├── Build22000_0000/          # Windows 11 21H2
├── Build22621_2215/          # Windows 11 22H2 (early)
├── Build26100_863/           # Windows 11 24H2 (early)
└── Proxy/                    # Interface definitions
```

### Build Detection

**File:** `VirtualDesktop-master/src/VirtualDesktop/Interop/IID.cs`

```csharp
public static Dictionary<string, Guid> GetIIDs(string[] interfaceNames) {
    var orderedProps = Settings.Default.Properties
        .OfType<SettingsProperty>()
        .Select(prop => {
            if (Version.TryParse(OS.VersionPrefix + 
                _osBuildRegex.Match(prop.Name).Groups["build"].ToString().Replace('_','.'), 
                out var build)) {
                return new OsBuildSettings(build, prop);
            }
            return null;
        })
        .OrderByDescending(s => s.osBuild);
    
    // Find first prop with build version <= current OS version
    var selectedSettings = orderedProps.FirstOrDefault(p => p.osBuild <= OS.Build);
    
    // Load GUIDs from selected settings
    foreach (var str in (StringCollection)Settings.Default[selectedSettings.prop.Name]) {
        var pair = str.Split(',');
        result.Add(pair[0], Guid.Parse(pair[1]));
    }
}
```

### Example: Build26100_863

**File:** `Build26100_863/VirtualDesktopManagerInternal.cs`

```csharp
internal class VirtualDesktopManagerInternal : 
    ComWrapperBase<IVirtualDesktopManagerInternal>, 
    IVirtualDesktopManagerInternal {
    
    public IVirtualDesktop CreateDesktop()
        => this.InvokeMethodAndWrap();
    
    public void SwitchDesktop(IVirtualDesktop desktop)
        => this.InvokeMethod(Args(((VirtualDesktop)desktop).ComObject));
    
    public void MoveViewToDesktop(IntPtr hWnd, IVirtualDesktop desktop)
        => this.InvokeMethod(Args(
            this._factory.ApplicationViewFromHwnd(hWnd).ComObject, 
            ((VirtualDesktop)desktop).ComObject));
}
```

The `InvokeMethod` calls use reflection to invoke COM methods by name, with the correct parameter marshalling for this build.

---

## Settings & Configuration

### Settings File Format

**Location:** `VirtualDesktopGridSwitcher.Settings` (same directory as .exe)

**Format:** XML (serialized via `XmlSerializer`)

**Example:**

```xml
<?xml version="1.0" encoding="utf-8"?>
<SettingValues>
  <Columns>3</Columns>
  <Rows>3</Rows>
  <WrapAround>false</WrapAround>
  <SwitchDirModifiers>
    <Ctrl>true</Ctrl>
    <Win>false</Win>
    <Alt>true</Alt>
    <Shift>false</Shift>
  </SwitchDirModifiers>
  <SwitchDirEnabled>true</SwitchDirEnabled>
  <ArrowKeysEnabled>true</ArrowKeysEnabled>
  <NumbersEnabled>true</NumbersEnabled>
  <FKeysEnabled>false</FKeysEnabled>
  <AlwaysOnTopHotkey>
    <Key>Space</Key>
    <Modifiers>
      <Ctrl>true</Ctrl>
      <Alt>true</Alt>
      <Shift>false</Shift>
    </Modifiers>
  </AlwaysOnTopHotkey>
  <MoveOnNewWindowExeNames>
    <ExeName>WINWORD.EXE</ExeName>
    <ExeName>EXCEL.EXE</ExeName>
    <ExeName>AcroRd32.exe</ExeName>
  </MoveOnNewWindowExeNames>
  <SettingsVersion>4</SettingsVersion>
</SettingValues>
```

### Settings Dialog

**File:** `VirtualDesktopGridSwitcher/Settings/SettingsDialog.cs`

Provides GUI for configuring:
- Grid dimensions
- Hotkey assignments
- Feature toggles
- Advanced settings

---

## Known Issues & Limitations

### 1. Windows 11 24H2 Compatibility

**Status:** BROKEN on builds > 26100.863  
**Workaround:** None currently  
**Fix:** Requires reverse-engineering new COM interfaces

### 2. Hotkey Conflicts

**Issue:** Graphics drivers (Intel, NVIDIA, AMD) often use Ctrl+Alt+Arrow keys  
**Solution:** Disable in graphics control panel or change VDGS hotkeys

### 3. Maximum Desktop Limit

**Issue:** Windows becomes unstable with >20 desktops  
**Recommendation:** Keep grid ≤ 4×4 (16 desktops)

### 4. Event Notification Reliability

**Issue:** `VirtualDesktop.CurrentChanged` event doesn't always fire  
**Workaround:** `VirtualDesktopTimer` polls for changes

### 5. UWP App Compatibility

**Issue:** Some UWP apps don't report window handles correctly  
**Impact:** May not move between desktops properly

### 6. Multi-Monitor Behavior

**Issue:** Virtual desktops are system-wide, not per-monitor  
**Limitation:** Cannot have different desktops on different monitors

### 7. Others

- Out of support .NET libraries used (7, 5, 4)
- No proper setup files 

---

## Development Guide

### Building the Project

**Prerequisites:**
- Visual Studio 2022
- .NET 7.0 SDK
- Windows 10 SDK (10.0.19041.0)

**Steps:**

```bash
git clone https://github.com/kloned/Virtual-Desktop-Grid-Switcher.git
cd Virtual-Desktop-Grid-Switcher
# Open VirtualDesktopGridSwitcher.sln in Visual Studio
# Build → Build Solution (F6)
```

**Output:** `bin\Release\VirtualDesktopGridSwitcher.exe` (self-contained)

### Project Structure

```
Virtual-Desktop-Grid-Switcher/
├── VirtualDesktopGridSwitcher/          # Main application
│   ├── Program.cs                       # Entry point
│   ├── VirtualDesktopGridManager.cs     # Core logic
│   ├── SysTrayProcess.cs                # System tray
│   ├── WinAPI.cs                        # P/Invoke wrappers
│   ├── VirtualDesktopTimer.cs           # Polling fallback
│   ├── ContextMenus.cs                  # Context menu
│   ├── Settings/                        # Configuration
│   │   ├── SettingValues.cs
│   │   ├── SettingsDialog.cs
│   │   └── SettingsDialog.Designer.cs
│   ├── AboutBox/                        # About dialog
│   └── Icons/                           # Desktop icons (1-12.ico)
├── VirtualDesktop-master/               # Virtual desktop library
│   └── src/VirtualDesktop/
│       ├── VirtualDesktop.cs            # Main API
│       ├── Interop/                     # COM interop
│       │   ├── Build10240_0000/
│       │   ├── Build22621_2215/
│       │   ├── Build26100_863/
│       │   ├── Proxy/                   # Interface definitions
│       │   ├── ComInterfaceAssemblyBuilder.cs
│       │   ├── VirtualDesktopProvider.cs
│       │   └── IID.cs                   # GUID management
│       └── app.config                   # Build-specific GUIDs
├── GlobalHotkeysWithDotNET/             # Hotkey library
│   └── Hotkey.cs
└── Documentation/                       # User guides
```

### Adding Support for New Windows Build

**Example: Adding Build 26100.9999**

1. **Create new build folder:**
   ```
   VirtualDesktop-master/src/VirtualDesktop/Interop/Build26100_9999/
   ```

2. **Copy files from previous build:**
   ```
   Build26100_9999/
   ├── .Provider.cs
   ├── VirtualDesktop.cs
   ├── VirtualDesktopManagerInternal.cs
   ├── VirtualDesktopNotificationService.cs
   └── .interfaces/
       ├── IVirtualDesktop.cs
       ├── IVirtualDesktopManagerInternal.cs
       ├── IVirtualDesktopNotification.cs
       └── IVirtualDesktopNotificationService.cs
   ```

3. **Update GUIDs in app.config:**
   ```xml
   <setting name="v_26100_9999" serializeAs="Xml">
     <value>
       <ArrayOfString>
         <string>IVirtualDesktop,{NEW-GUID-HERE}</string>
         <string>IVirtualDesktopManagerInternal,{NEW-GUID-HERE}</string>
         ...
       </ArrayOfString>
     </value>
   </setting>
   ```

4. **Update .csproj to embed new interfaces:**
   ```xml
   <EmbeddedResource Include="Interop\Build26100_9999\.interfaces\*.cs" />
   ```

5. **Test on target build**

### Debugging Tips

**Enable Debug Output:**

The VirtualDesktop library uses `Debug.WriteLine()` extensively. View output in Visual Studio's Output window (Debug → Windows → Output).

**Check Event Viewer:**

Crashes appear in:
```
Event Viewer → Windows Logs → Application
Source: .NET Runtime
```

**Test COM Interface:**

```csharp
try {
    var desktop = VirtualDesktop.Create();
    Debug.WriteLine($"Created desktop: {desktop.Id}");
    desktop.Remove();
} catch (Exception ex) {
    Debug.WriteLine($"Failed: {ex}");
}
```

### Contributing

**Original Project:** https://sourceforge.net/p/virtual-desktop-grid-switcher/  
**Fork:** https://github.com/kloned/Virtual-Desktop-Grid-Switcher

For Windows 11 24H2 fixes, coordinate with the community on GitHub Issues.

---

## Appendix: Key Files Reference

| File | Purpose | Lines |
|------|---------|-------|
| `Program.cs` | Entry point | 33 |
| `VirtualDesktopGridManager.cs` | Core logic | 746 |
| `SysTrayProcess.cs` | System tray | 51 |
| `SettingValues.cs` | Configuration | 381 |
| `WinAPI.cs` | Windows API wrappers | 304 |
| `VirtualDesktopTimer.cs` | Polling fallback | 79 |
| `VirtualDesktop.cs` (lib) | Main API | 357 |
| `ComInterfaceAssemblyBuilder.cs` | Dynamic compilation | 217 |
| `IID.cs` | GUID management | 102 |
| `app.config` | Build-specific GUIDs | 156 |

---

## Glossary

- **COM:** Component Object Model - Microsoft's binary interface standard
- **GUID:** Globally Unique Identifier - 128-bit identifier for COM interfaces
- **IID:** Interface Identifier - GUID for a COM interface
- **P/Invoke:** Platform Invocation Services - .NET mechanism for calling native code
- **vtable:** Virtual method table - COM interface method dispatch table
- **HWND:** Window Handle - unique identifier for a window
- **CLSID:** Class Identifier - GUID for a COM class

---

**End of Documentation**

This documentation should be updated as the project evolves and new Windows builds are supported.
