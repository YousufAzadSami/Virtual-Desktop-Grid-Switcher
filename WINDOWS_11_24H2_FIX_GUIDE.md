# Windows 11 24H2 Compatibility Fix Guide

**Date:** April 1-2, 2026  
**Windows Build:** 26100.7705 (Windows 11 24H2)  
**Status:** ✅ Fixed and Working

---

## Table of Contents

1. [Problem Overview](#problem-overview)
2. [Root Cause Analysis](#root-cause-analysis)
3. [Solution Summary](#solution-summary)
4. [Detailed Fix Steps](#detailed-fix-steps)
5. [Files Modified](#files-modified)
6. [Testing & Verification](#testing--verification)
7. [Future Windows Updates](#future-windows-updates)
8. [Technical Deep Dive](#technical-deep-dive)
9. [Community Resources](#community-resources)

---

## Problem Overview

### Symptoms
- Application crashes immediately on startup with `AccessViolationException`
- Error: "Attempted to read or write protected memory"
- No system tray icon appears
- Event Viewer shows crash in `IVirtualDesktopManagerInternal.CreateDesktop()`

### Affected Builds
- Windows 11 Build 26100.863 and higher (24H2 release)
- Specifically tested on Build 26100.7705

### Error Stack Trace
```
System.AccessViolationException: Attempted to read or write protected memory.
Stack:
   at WindowsDesktop.Interop.Build22621.IVirtualDesktopManagerInternal.CreateDesktop()
   at WindowsDesktop.Interop.Build26100.VirtualDesktopManagerInternal.CreateDesktop()
   at WindowsDesktop.VirtualDesktop.Create()
   at VirtualDesktopGridSwitcher.VirtualDesktopGridManager.Start()
```

---

## Root Cause Analysis

### Why It Broke

Microsoft completely rewrote the virtual desktop implementation in Windows 11 24H2, causing:

1. **Changed COM Interface GUIDs**
   - Each COM interface has a unique GUID identifier
   - Windows 11 24H2 uses different GUIDs than previous versions
   - Old GUIDs point to invalid memory locations → crash

2. **Changed Method Signatures**
   - COM interface vtable (virtual method table) layout changed
   - Method order and parameters may differ
   - Calling methods with wrong offsets → memory corruption

3. **Undocumented API**
   - Virtual Desktop API is not officially documented by Microsoft
   - No public documentation of GUID changes
   - Requires reverse-engineering or community research

### Key Technical Issue

The codebase had **three critical bugs**:

1. **Missing GUIDs in `app.config`** - No configuration for build 26100
2. **Missing NuGet package** - C#/WinRT package not referenced
3. **Wrong namespaces** - Build26100 interface files had `Build22631` namespace

---

## Solution Summary

### What We Fixed

| Issue | Fix | File(s) |
|-------|-----|---------|
| Missing Build 26100 GUIDs | Added `v_26100_0863` configuration section | `app.config` |
| Missing C#/WinRT package | Added conditional package references | `VirtualDesktop.csproj` |
| Wrong namespace in interfaces | Changed `Build22631` → `Build26100` | 4 interface files |
| Cached build artifacts | Clean rebuild | N/A |

### Build 26100 GUIDs (Windows 11 24H2)

```xml
IApplicationView:                   {372E1D3B-38D3-42E4-A15B-8AB2B178F513}
IApplicationViewCollection:         {1841C6D7-4F9D-42C0-AF41-8747538F10E5}
IObjectArray:                       {92CA9DCD-5622-4BBA-A805-5E9F541BD8C9}
IServiceProvider:                   {6D5140C1-7436-11CE-8034-00AA006009FA}
IVirtualDesktop:                    {3F07F4BE-B107-441A-AF0F-39D82529072C}
IVirtualDesktopManager:             {A5CD92FF-29BE-454C-8D04-D82879FB3F1B}
IVirtualDesktopManagerInternal:     {53F5CA0B-158F-4124-900C-057158060B27}  ⚠️ KEY CHANGE
IVirtualDesktopNotification:        {B9E5E94D-233E-49AB-AF5C-2B4541C3AADE}
IVirtualDesktopNotificationService: {0cd45e71-d927-4f15-8b0a-8fef525337bf}
IVirtualDesktopPinnedApps:          {4CE81583-1E4C-4632-A621-07A53543148F}
```

**Critical:** `IVirtualDesktopManagerInternal` GUID changed from `{A3175F2D-239C-4BD2-8AA0-EEBA8B0B138E}` (Build 22621) to `{53F5CA0B-158F-4124-900C-057158060B27}` (Build 26100).

---

## Detailed Fix Steps

### Step 1: Extract GUIDs from Community Sources

**Source:** MScholtes/VirtualDesktop repository  
**URL:** https://github.com/MScholtes/VirtualDesktop/blob/master/VirtualDesktop11-24H2.cs

**Method:**
1. Search for community repositories supporting Windows 11 24H2
2. Locate the interface definitions with `[Guid("...")]` attributes
3. Extract all 10 required interface GUIDs
4. Verify against multiple sources if possible

**Note:** Windows does not expose these GUIDs publicly - they are undocumented internal COM interfaces. The Registry does not contain this information, which is why community reverse-engineering is necessary.

### Step 2: Add GUIDs to app.config

**File:** `VirtualDesktop-master\src\VirtualDesktop\app.config`

**Location:** After the last `<setting>` block, before `</WindowsDesktop.Properties.Settings>`

**Code to Add:**
```xml
   <setting name="v_26100_0863" serializeAs="Xml">
    <value>
     <ArrayOfString xmlns:xsd="http://www.w3.org/2001/XMLSchema" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
      <string>IApplicationView,{372E1D3B-38D3-42E4-A15B-8AB2B178F513}</string>
      <string>IApplicationViewCollection,{1841C6D7-4F9D-42C0-AF41-8747538F10E5}</string>
      <string>IObjectArray,{92CA9DCD-5622-4BBA-A805-5E9F541BD8C9}</string>
      <string>IServiceProvider,{6D5140C1-7436-11CE-8034-00AA006009FA}</string>
      <string>IVirtualDesktop,{3F07F4BE-B107-441A-AF0F-39D82529072C}</string>
      <string>IVirtualDesktopManager,{A5CD92FF-29BE-454C-8D04-D82879FB3F1B}</string>
      <string>IVirtualDesktopManagerInternal,{53F5CA0B-158F-4124-900C-057158060B27}</string>
      <string>IVirtualDesktopNotification,{B9E5E94D-233E-49AB-AF5C-2B4541C3AADE}</string>
      <string>IVirtualDesktopNotificationService,{0cd45e71-d927-4f15-8b0a-8fef525337bf}</string>
      <string>IVirtualDesktopPinnedApps,{4CE81583-1E4C-4632-A621-07A53543148F}</string>
     </ArrayOfString>
    </value>
   </setting>
```

**Naming Convention:** `v_MAJOR_MINOR` where build is `MAJOR.MINOR` (e.g., 26100.863 → `v_26100_0863`)

### Step 3: Fix Missing C#/WinRT Package

**File:** `VirtualDesktop-master\src\VirtualDesktop\VirtualDesktop.csproj`

**Problem:** Build error:
```
error CS0246: The type or namespace name 'WinRT' could not be found
```

**Solution:** Add conditional package references for different .NET versions.

**Find this section:**
```xml
  <ItemGroup>
    <PackageReference Include="Microsoft.CodeAnalysis.CSharp" Version="4.0.1" />
  </ItemGroup>
```

**Replace with:**
```xml
  <ItemGroup>
    <PackageReference Include="Microsoft.CodeAnalysis.CSharp" Version="4.0.1" />
  </ItemGroup>

  <ItemGroup Condition="'$(TargetFramework)' == 'net5.0-windows10.0.19041.0'">
    <PackageReference Include="Microsoft.Windows.CsWinRT" Version="1.6.5" />
  </ItemGroup>

  <ItemGroup Condition="'$(TargetFramework)' == 'net6.0-windows10.0.19041.0' Or '$(TargetFramework)' == 'net7.0-windows10.0.19041.0'">
    <PackageReference Include="Microsoft.Windows.CsWinRT" Version="2.0.1" />
  </ItemGroup>
```

**Why Conditional?**
- C#/WinRT 2.0+ does not support .NET 5.0
- .NET 5.0 requires C#/WinRT 1.6.5
- .NET 6.0 and 7.0 use C#/WinRT 2.0.1

### Step 4: Fix Namespace Bug in Interface Files

**Problem:** All Build26100 interface files had wrong namespace `Build22631` instead of `Build26100`.

**Files to Fix:**
1. `Interop\Build26100_863\.interfaces\IVirtualDesktop.cs`
2. `Interop\Build26100_863\.interfaces\IVirtualDesktopManagerInternal.cs`
3. `Interop\Build26100_863\.interfaces\IVirtualDesktopNotification.cs`
4. `Interop\Build26100_863\.interfaces\IVirtualDesktopNotificationService.cs`

**Change in Each File:**

**Before:**
```csharp
namespace WindowsDesktop.Interop.Build22631
{
    [ComImport]
    [Guid("00000000-0000-0000-0000-000000000000")]
    public interface IVirtualDesktop
    {
        // ...
    }
}
```

**After:**
```csharp
namespace WindowsDesktop.Interop.Build26100
{
    [ComImport]
    [Guid("00000000-0000-0000-0000-000000000000")]
    public interface IVirtualDesktop
    {
        // ...
    }
}
```

**Why This Matters:**
- The provider selection logic correctly chose `VirtualDesktopProvider26100`
- But the interface files declared themselves as `Build22631`
- This caused the runtime to use Build22621 GUIDs instead of Build26100 GUIDs
- Result: Wrong GUID → Invalid memory access → Crash

### Step 5: Clean Rebuild

**Critical:** Old build artifacts cache the wrong namespace.

**PowerShell Commands:**
```powershell
Remove-Item -Path "d:\personal\Virtual-Desktop-Grid-Switcher\bin" -Recurse -Force -ErrorAction SilentlyContinue
Remove-Item -Path "d:\personal\Virtual-Desktop-Grid-Switcher\VirtualDesktop-master\src\VirtualDesktop\bin" -Recurse -Force -ErrorAction SilentlyContinue
Remove-Item -Path "d:\personal\Virtual-Desktop-Grid-Switcher\VirtualDesktop-master\src\VirtualDesktop\obj" -Recurse -Force -ErrorAction SilentlyContinue
Remove-Item -Path "d:\personal\Virtual-Desktop-Grid-Switcher\VirtualDesktopGridSwitcher\bin" -Recurse -Force -ErrorAction SilentlyContinue
Remove-Item -Path "d:\personal\Virtual-Desktop-Grid-Switcher\VirtualDesktopGridSwitcher\obj" -Recurse -Force -ErrorAction SilentlyContinue
```

**Visual Studio:**
1. Build → Clean Solution
2. Build → Rebuild Solution (Ctrl+Shift+B)

### Step 6: Test the Application

**Run:**
```
D:\personal\Virtual-Desktop-Grid-Switcher\bin\Release\net7.0-windows10.0.19041.0\win-x64\VirtualDesktopGridSwitcher.exe
```

**Verify:**
- ✅ System tray icon appears
- ✅ No crash on startup
- ✅ Ctrl+Alt+Arrow Keys switches desktops
- ✅ Ctrl+Alt+Shift+Arrow Keys moves windows
- ✅ Ctrl+Alt+Number jumps to desktop

---

## Files Modified

### 1. app.config
**Path:** `VirtualDesktop-master\src\VirtualDesktop\app.config`  
**Change:** Added `v_26100_0863` configuration section with Windows 11 24H2 GUIDs  
**Lines:** 154-169 (new section)

### 2. VirtualDesktop.csproj
**Path:** `VirtualDesktop-master\src\VirtualDesktop\VirtualDesktop.csproj`  
**Change:** Added conditional C#/WinRT package references  
**Lines:** 48-54 (new ItemGroups)

### 3. IVirtualDesktop.cs
**Path:** `Interop\Build26100_863\.interfaces\IVirtualDesktop.cs`  
**Change:** Namespace `Build22631` → `Build26100`  
**Line:** 5

### 4. IVirtualDesktopManagerInternal.cs
**Path:** `Interop\Build26100_863\.interfaces\IVirtualDesktopManagerInternal.cs`  
**Change:** Namespace `Build22631` → `Build26100`  
**Line:** 5

### 5. IVirtualDesktopNotification.cs
**Path:** `Interop\Build26100_863\.interfaces\IVirtualDesktopNotification.cs`  
**Change:** Namespace `Build22631` → `Build26100`  
**Line:** 5

### 6. IVirtualDesktopNotificationService.cs
**Path:** `Interop\Build26100_863\.interfaces\IVirtualDesktopNotificationService.cs`  
**Change:** Namespace `Build22631` → `Build26100`  
**Line:** 4

---

## Testing & Verification

### Build Verification

**Expected Output:**
```
========== Rebuild All: 2 succeeded, 0 failed, 0 skipped ==========
```

**If Build Fails:**
- Check that all namespace changes are applied
- Verify C#/WinRT package references are correct
- Clean solution and rebuild

### Runtime Verification

**Check Event Viewer:**
```powershell
Get-EventLog -LogName Application -Source ".NET Runtime" -Newest 5 | 
  Where-Object {$_.TimeGenerated -gt (Get-Date).AddMinutes(-5)} | 
  Format-List TimeGenerated, EntryType, Message
```

**Success:** No new error entries after running the application.

**Failure:** `AccessViolationException` entries indicate wrong GUIDs or namespace issues.

### Functional Testing

| Feature | Test | Expected Result |
|---------|------|-----------------|
| Startup | Launch application | System tray icon appears |
| Desktop Switch | Ctrl+Alt+Arrow Keys | Switches between desktops smoothly |
| Window Move | Ctrl+Alt+Shift+Arrow Keys | Moves active window to adjacent desktop |
| Direct Jump | Ctrl+Alt+1-9 | Jumps to specific desktop number |
| Grid Layout | Navigate in 2D grid | Desktops arranged in configured grid (default 3x3) |
| Desktop Creation | Auto-create on first run | Creates configured number of desktops |

---

## Future Windows Updates

### When Windows 11 Gets Updated Again

Microsoft will likely change COM interface GUIDs again in future major updates (e.g., 25H2, 26H1).

### Quick Fix Process

1. **Check Community Sources**
   - MScholtes: https://github.com/MScholtes/VirtualDesktop
   - Ciantic: https://github.com/Ciantic/VirtualDesktopAccessor
   - Slion: https://github.com/Slion/VirtualDesktop

2. **Extract New GUIDs**
   - Look for files like `VirtualDesktop11-25H2.cs` or similar
   - Find interface definitions with `[Guid("...")]` attributes
   - Note the new build number (e.g., 27000.x)

3. **Update app.config**
   - Add new `v_XXXXX_XXXX` section
   - Use the naming convention: build `27000.123` → `v_27000_0123`
   - Copy GUID format from existing sections

4. **Verify Provider Exists**
   - Check if `VirtualDesktop.system.cs` has version check for new build
   - Check if `Interop\BuildXXXXX_XXX\` folder exists
   - If not, may need to create new provider implementation

5. **Rebuild and Test**
   - Clean solution
   - Rebuild
   - Test on new Windows build

### Example: Adding Support for Build 27000.500

```xml
<setting name="v_27000_0500" serializeAs="Xml">
 <value>
  <ArrayOfString xmlns:xsd="http://www.w3.org/2001/XMLSchema" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
   <string>IApplicationView,{NEW-GUID-HERE}</string>
   <string>IApplicationViewCollection,{NEW-GUID-HERE}</string>
   <!-- ... etc ... -->
  </ArrayOfString>
 </value>
</setting>
```

---

## Technical Deep Dive

### How Build Detection Works

**File:** `VirtualDesktop.system.cs`

```csharp
private static VirtualDesktopProvider CreateProvider()
{
    Version v = OS.Build;  // Gets current Windows build (e.g., 10.0.26100.7705)

    if (v >= new Version(10, 0, 26100, 863))  // Check highest version first
    {
        return new VirtualDesktopProvider26100();
    }

    if (v >= new Version(10, 0, 22621, 2215))
    {
        return new VirtualDesktopProvider22621();
    }
    
    // ... more version checks ...
}
```

**Key Points:**
- Checks versions in **descending order** (highest first)
- Uses `>=` comparison, so build 26100.7705 matches 26100.863
- Returns first matching provider
- Each provider uses different COM interface implementations

### How GUID Loading Works

**File:** `Interop\IID.cs`

```csharp
public static Dictionary<string, Guid> GetIIDs()
{
    // 1. Load all v_XXXXX_XXXX settings from app.config
    var orderedProps = Settings.Default.Properties.OfType<SettingsProperty>()
        .Select(prop => {
            if (Version.TryParse(OS.VersionPrefix + _osBuildRegex.Match(prop.Name).Groups["build"].ToString().Replace('_','.'), out var build))
            {
                return new OsBuildSettings(build, prop);
            }
            return null;
        })
        .Where(s => s != null)
        .OrderByDescending(s => s.osBuild)  // Sort by build version descending
        .ToArray();

    // 2. Find first setting with build <= current OS build
    var selectedSettings = orderedProps.FirstOrDefault(p => p.osBuild <= OS.Build);
    
    // 3. Parse GUIDs from selected setting
    // 4. Fallback to registry search if GUIDs missing
}
```

**Process:**
1. Parse all `v_XXXXX_XXXX` entries from `app.config`
2. Sort by build version (highest first)
3. Select first entry where `build <= current_build`
4. For build 26100.7705, selects `v_26100_0863` (26100.863 <= 26100.7705)
5. Extract GUIDs from that section
6. Replace placeholder GUIDs in interface definitions

### Why Namespace Matters

**The Problem:**

```csharp
// File: Build26100_863\.interfaces\IVirtualDesktopManagerInternal.cs
namespace WindowsDesktop.Interop.Build22631  // WRONG!
{
    public interface IVirtualDesktopManagerInternal { ... }
}

// File: Build26100\VirtualDesktopManagerInternal.cs
namespace WindowsDesktop.Interop.Build26100
{
    // Tries to use: WindowsDesktop.Interop.Build26100.IVirtualDesktopManagerInternal
    // But interface file declares: WindowsDesktop.Interop.Build22631.IVirtualDesktopManagerInternal
    // Result: Falls back to actual Build22621 implementation!
}
```

**The Fix:**

```csharp
// File: Build26100_863\.interfaces\IVirtualDesktopManagerInternal.cs
namespace WindowsDesktop.Interop.Build26100  // CORRECT!
{
    public interface IVirtualDesktopManagerInternal { ... }
}

// Now the namespace matches and correct implementation is used
```

### COM Interface Compilation

The application **dynamically compiles** COM interfaces at runtime:

1. **Reads embedded interface files** (`.interfaces\*.cs`)
2. **Replaces placeholder GUIDs** with actual GUIDs from `app.config`
3. **Compiles into in-memory assembly** using Roslyn
4. **Caches compiled assembly** (optional, for performance)
5. **Uses compiled interfaces** for COM interop

This is why namespace bugs are critical - they affect the compiled assembly.

---

## Community Resources

### Primary Sources for GUIDs

1. **MScholtes/VirtualDesktop** (Most reliable)
   - URL: https://github.com/MScholtes/VirtualDesktop
   - Files: `VirtualDesktop11-24H2.cs`, `VirtualDesktop11-23H2.cs`, etc.
   - Actively maintained for each Windows release

2. **Scott Hanselman's MaximizeToVirtualDesktop** ✅
   - URL: https://github.com/shanselman/MaximizeToVirtualDesktop
   - Multi-build detection pattern implementation
   - Good reference for handling multiple Windows versions
   - Uses similar COM adapter approach

3. **Ciantic/VirtualDesktopAccessor**
   - URL: https://github.com/Ciantic/VirtualDesktopAccessor
   - C++ implementation with GUID definitions
   - Good for cross-referencing

4. **Slion/VirtualDesktop**
   - URL: https://github.com/Slion/VirtualDesktop
   - Fork of original library
   - Our codebase is based on this

### Reverse Engineering Tools (Advanced)

If community sources are unavailable:

1. **IDA Pro / Ghidra** - Disassemble `twinui.pcshell.dll`
2. **WinDbg** - Debug Windows shell processes
3. **Process Monitor** - Track COM registration
4. **Registry Editor** - Search `HKEY_CLASSES_ROOT\Interface` (limited success)

### Discussion Forums

- GitHub Issues on above repositories
- Reddit: r/Windows11, r/PowerShell
- Stack Overflow: Tag `windows-10-virtual-desktop`

---

## Troubleshooting

### Build Errors

**Error:** `The type or namespace name 'WinRT' could not be found`
- **Fix:** Add C#/WinRT package references (Step 3)

**Error:** `Support for .NET 5 ended with C#/WinRT 2.0`
- **Fix:** Use conditional package references with version 1.6.5 for .NET 5.0

**Error:** Namespace conflicts
- **Fix:** Ensure all Build26100 interface files use `Build26100` namespace

### Runtime Errors

**Error:** `AccessViolationException` on startup
- **Cause:** Wrong GUIDs or wrong namespace
- **Fix:** Verify GUIDs in `app.config`, check namespace in interface files, clean rebuild

**Error:** Application starts but hotkeys don't work
- **Cause:** Different issue (not GUID-related)
- **Fix:** Check hotkey configuration, check for conflicts with other software

**Error:** `ConfigurationException` about missing settings
- **Cause:** `app.config` not copied to output directory
- **Fix:** Rebuild solution, verify `app.config` in `bin\Release\` folder

### Debugging Tips

**Check which provider is selected:**
```csharp
// Add breakpoint in VirtualDesktop.system.cs, line 57
if (v >= new Version(10, 0, 26100, 863))
{
    return new VirtualDesktopProvider26100();  // <- Breakpoint here
}
```

**Check which GUIDs are loaded:**
```csharp
// Add breakpoint in IID.cs after GUID loading
var iids = GetIIDs();  // <- Inspect this dictionary
```

**Check COM interface compilation:**
- Look for generated DLL in `%LOCALAPPDATA%\VirtualDesktop\`
- Check if namespace in compiled assembly is correct

---

## Conclusion

This fix required three critical changes:

1. **Data:** Adding correct GUIDs for Windows 11 24H2
2. **Dependencies:** Adding C#/WinRT package with correct versions
3. **Code:** Fixing namespace bug in interface files

The multi-build detection system was already in place - it just needed the correct configuration data and bug fixes.

**Key Lesson:** When Microsoft updates Windows, the GUIDs change but the architecture remains the same. Future fixes should only require updating `app.config` with new GUIDs.

---

**Document Version:** 1.0  
**Last Updated:** April 2, 2026  
**Tested On:** Windows 11 Build 26100.7705 (24H2)  
**Status:** ✅ Working
