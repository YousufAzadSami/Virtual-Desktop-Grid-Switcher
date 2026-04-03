# Error Handling & Logging Implementation Plan

**Date:** April 3, 2026  
**Status:** Planning Phase  
**Priority:** P0 (High Impact, Low Effort)

---

## Current State Analysis

### Existing Error Handling Patterns

#### ✅ What's Working
1. **Desktop initialization error** - Shows MessageBox with exception details (line 127-136 in VirtualDesktopGridManager.cs)
2. **Hotkey registration failures** - Shows user-friendly warnings about conflicts (line 695-700)
3. **Settings validation** - Warns about too many desktops (>20)

#### ❌ What's Missing

1. **Silent failures** - Multiple empty catch blocks swallow errors:
   - Line 144: `catch { return false; }` - Hotkey registration
   - Line 428: `catch { }` - Pin/unpin window operations
   - Line 563: `catch { return false; }` - Desktop switching
   - Line 349: `catch { return false; }` - Settings save

2. **No logging infrastructure**
   - Only `Debug.WriteLine()` calls (11 instances in VirtualDesktop library)
   - Debug output only visible in debugger, not in production
   - No persistent logs for troubleshooting

3. **No startup error handling**
   - `Program.Main()` has zero error handling
   - Application crashes silently if initialization fails
   - No graceful degradation

4. **Poor error context**
   - Generic error messages without actionable information
   - No error codes or categories
   - No troubleshooting guidance

---

## Problem Scenarios (From Documentation)

### Critical Issues We Need to Handle

1. **Windows 11 24H2 Compatibility Crash**
   - **Current:** Silent crash with AccessViolationException
   - **Impact:** Application won't start, no error message
   - **Need:** Detect incompatible Windows build, show helpful error

2. **Hotkey Conflicts**
   - **Current:** Warning shown, but hotkey just doesn't work
   - **Impact:** User doesn't know which app is conflicting
   - **Need:** Better diagnostics, suggest alternatives

3. **COM Interface Failures**
   - **Current:** Generic "error initializing desktops"
   - **Impact:** User doesn't know if it's Windows version, permissions, or corruption
   - **Need:** Specific error messages with troubleshooting steps

4. **Settings File Corruption**
   - **Current:** Silent failure, uses defaults
   - **Impact:** User loses settings without knowing why
   - **Need:** Detect corruption, offer backup/reset

---

## Proposed Solution

### Architecture

```
┌─────────────────────────────────────────────────────────┐
│                   Application Layer                      │
│  (Program.cs, VirtualDesktopGridManager.cs, etc.)       │
└────────────────┬────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────┐
│              Error Handling Layer                        │
│  ┌──────────────────┐  ┌──────────────────┐            │
│  │  ErrorHandler    │  │  UserErrorDialog │            │
│  │  - Categorize    │  │  - User-friendly │            │
│  │  - Log           │  │  - Actionable    │            │
│  │  - Report        │  │  - Copy details  │            │
│  └──────────────────┘  └──────────────────┘            │
└────────────────┬────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────┐
│                  Logging Layer                           │
│  ┌──────────────────┐  ┌──────────────────┐            │
│  │  File Logger     │  │  Event Viewer    │            │
│  │  - Rolling files │  │  - Critical only │            │
│  │  - Structured    │  │  - System events │            │
│  └──────────────────┘  └──────────────────┘            │
└─────────────────────────────────────────────────────────┘
```

### Components to Implement

#### 1. Logging Framework Selection

**Option A: Serilog** ⭐ RECOMMENDED
- Lightweight, structured logging
- File sinks with rolling
- Easy configuration
- Popular, well-maintained

**Option B: NLog**
- More features, more complex
- Excellent performance
- Good for enterprise apps

**Option C: Microsoft.Extensions.Logging**
- Built into .NET
- Requires more setup for file logging
- Good for ASP.NET, overkill for WinForms

**Decision: Serilog**
- Best balance of simplicity and features
- Perfect for desktop apps
- Minimal dependencies

#### 2. Error Categories

```csharp
public enum ErrorCategory
{
    Startup,              // Application initialization
    WindowsCompatibility, // OS version issues
    VirtualDesktop,       // Desktop creation/switching
    Hotkey,              // Hotkey registration
    Settings,            // Configuration load/save
    WindowManagement,    // Window operations
    COMInterop,          // COM interface failures
    Unknown              // Uncategorized
}
```

#### 3. Error Severity Levels

```csharp
public enum ErrorSeverity
{
    Debug,      // Development info
    Info,       // Normal operations
    Warning,    // Recoverable issues
    Error,      // Failures with fallback
    Critical    // Application cannot continue
}
```

---

## Implementation Plan

### Phase 1: Infrastructure Setup

**Files to Create:**
1. `Logging/Logger.cs` - Serilog wrapper
2. `Logging/LoggerConfig.cs` - Configuration
3. `ErrorHandling/ErrorHandler.cs` - Centralized error handling
4. `ErrorHandling/ErrorDialog.cs` - User-friendly error UI
5. `ErrorHandling/ErrorCategory.cs` - Error categorization

**Files to Modify:**
1. `Program.cs` - Add global error handler
2. `VirtualDesktopGridManager.cs` - Replace empty catches
3. `SettingValues.cs` - Add logging to save/load

### Phase 2: Logging Integration

**Where to Add Logging:**

| Location | Event | Level | Example |
|----------|-------|-------|---------|
| Program.Main() | Application start | Info | "Application starting v4.0.2.4" |
| Program.Main() | Unhandled exception | Critical | "Unhandled exception: {ex}" |
| VirtualDesktopGridManager.Start() | Desktop creation | Info | "Creating {count} desktops" |
| VirtualDesktopGridManager.Start() | COM failure | Error | "Failed to initialize COM: {ex}" |
| RegisterHotkey() | Success | Debug | "Registered hotkey: {key}" |
| RegisterHotkey() | Conflict | Warning | "Hotkey conflict: {key}" |
| SettingValues.Load() | File not found | Info | "Settings file not found, using defaults" |
| SettingValues.Save() | Save failure | Error | "Failed to save settings: {ex}" |
| VirtualDesktop_CurrentChanged() | Desktop switch | Debug | "Switched to desktop {index}" |

### Phase 3: Error Handling Improvements

**Replace Silent Catches:**

```csharp
// BEFORE (Line 144)
try {
    RegisterHotKeys();
} catch {
    return false;
}

// AFTER
try {
    RegisterHotKeys();
} catch (Exception ex) {
    Logger.Error(ErrorCategory.Hotkey, "Failed to register hotkeys", ex);
    ErrorHandler.ShowError(
        "Hotkey Registration Failed",
        "Could not register keyboard shortcuts. They may be in use by another application.",
        ex,
        showTroubleshooting: true
    );
    return false;
}
```

**Add Startup Error Handling:**

```csharp
// Program.Main()
[STAThread]
static void Main() {
    try {
        Logger.Initialize();
        Logger.Info("Application starting");
        
        Application.SetUnhandledExceptionMode(UnhandledExceptionMode.CatchException);
        Application.ThreadException += OnThreadException;
        AppDomain.CurrentDomain.UnhandledException += OnUnhandledException;
        
        // ... existing code ...
        
    } catch (Exception ex) {
        Logger.Critical(ErrorCategory.Startup, "Fatal startup error", ex);
        ErrorHandler.ShowFatalError(ex);
    }
}
```

### Phase 4: User-Friendly Error Dialogs

**Features:**
- Clear, non-technical primary message
- Technical details in expandable section
- "Copy to Clipboard" button
- Troubleshooting suggestions based on error category
- Link to documentation/GitHub issues

**Example Dialog:**

```
┌─────────────────────────────────────────────────────┐
│  ⚠️  Virtual Desktop Initialization Failed          │
├─────────────────────────────────────────────────────┤
│                                                      │
│  Could not initialize virtual desktops.             │
│                                                      │
│  This may be caused by:                             │
│  • Incompatible Windows version                     │
│  • Missing Windows updates                          │
│  • Corrupted Windows virtual desktop settings       │
│                                                      │
│  ▼ Show Technical Details                           │
│                                                      │
│  [ View Troubleshooting Guide ]  [ Copy Details ]   │
│                                                      │
│                                    [ Close ]         │
└─────────────────────────────────────────────────────┘
```

---

## Log File Strategy

### Location
```
%LOCALAPPDATA%\VirtualDesktopGridSwitcher\Logs\
  ├── vdgs-20260403.log      (today)
  ├── vdgs-20260402.log      (yesterday)
  └── vdgs-20260401.log      (2 days ago)
```

### Retention Policy
- Keep last 7 days
- Max 10 MB per file
- Rolling daily

### Log Format
```
2026-04-03 15:30:45.123 [INF] Application starting v4.0.2.4
2026-04-03 15:30:45.456 [DBG] Windows version: 10.0.26100.7705
2026-04-03 15:30:45.789 [INF] Creating 9 virtual desktops (3x3 grid)
2026-04-03 15:30:46.012 [WRN] Hotkey Ctrl+Alt+Left already registered
2026-04-03 15:30:46.234 [ERR] Failed to pin window 0x12345: Access denied
```

---

## Configuration

### appsettings.json (New File)
```json
{
  "Logging": {
    "MinimumLevel": "Information",
    "FilePath": "%LOCALAPPDATA%\\VirtualDesktopGridSwitcher\\Logs\\vdgs-.log",
    "RetainedFileCountLimit": 7,
    "FileSizeLimitBytes": 10485760,
    "WriteToEventLog": true,
    "EventLogSource": "VirtualDesktopGridSwitcher"
  },
  "ErrorHandling": {
    "ShowTechnicalDetails": false,
    "EnableCrashReporting": false,
    "AutoOpenLogOnError": false
  }
}
```

---

## NuGet Packages to Add

```xml
<PackageReference Include="Serilog" Version="3.1.1" />
<PackageReference Include="Serilog.Sinks.File" Version="5.0.0" />
<PackageReference Include="Serilog.Sinks.EventLog" Version="3.1.0" />
<PackageReference Include="Serilog.Formatting.Compact" Version="2.0.0" />
```

---

## Testing Strategy

### Error Scenarios to Test

1. **Startup Errors**
   - [ ] Incompatible Windows version
   - [ ] Missing permissions
   - [ ] Corrupted settings file
   - [ ] COM interface unavailable

2. **Runtime Errors**
   - [ ] Hotkey conflicts
   - [ ] Desktop creation failure
   - [ ] Window move failure
   - [ ] Settings save failure

3. **Recovery Scenarios**
   - [ ] Settings reset to defaults
   - [ ] Graceful degradation (some features disabled)
   - [ ] Automatic retry logic

---

## Success Criteria

### Must Have
- ✅ No silent failures (all exceptions logged)
- ✅ User-friendly error messages
- ✅ Persistent log files
- ✅ Global exception handler
- ✅ Startup error handling

### Nice to Have
- ⭐ Automatic crash reporting
- ⭐ Performance metrics logging
- ⭐ Log viewer UI
- ⭐ Error analytics

---

## Estimated Effort

| Phase | Effort | Priority |
|-------|--------|----------|
| Infrastructure Setup | 2-3 hours | P0 |
| Logging Integration | 1-2 hours | P0 |
| Error Handling Improvements | 2-3 hours | P0 |
| User-Friendly Dialogs | 1-2 hours | P1 |
| Testing | 1-2 hours | P0 |
| **Total** | **7-12 hours** | |

---

## Next Steps

1. **Review this plan** - Get approval on approach
2. **Add Serilog packages** - Update .csproj files
3. **Create logging infrastructure** - Logger.cs, LoggerConfig.cs
4. **Add global error handler** - Update Program.cs
5. **Replace empty catches** - Add proper error handling
6. **Create error dialog** - User-friendly UI
7. **Test error scenarios** - Verify all paths work

---

**Document Version:** 1.0  
**Last Updated:** April 3, 2026  
**Status:** Ready for Review
