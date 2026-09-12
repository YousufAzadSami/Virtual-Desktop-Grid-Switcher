# Error Handling and Logging Plan

**Status:** Proposed

This plan is not implementation documentation. Its goal is to replace silent failures with useful diagnostics while avoiding attempts to continue after unsafe COM initialization failures.

## Current risks

- `Program.Main()` has no top-level managed exception handling.
- Several `catch` blocks discard exceptions or return only `false`.
- Startup can continue after desktop initialization fails, leaving partial state.
- Settings load/save failures provide little context and can silently revert behavior.
- Debug output is unavailable to normal users.
- The COM layer can fail because an undocumented IID or vtable changed.
- An `AccessViolationException` may indicate corrupted native state and is not safely recoverable.

Existing hotkey-conflict and some settings-validation message boxes are useful, but they are not backed by persistent diagnostics.

## Principles

1. Prevent known-invalid COM calls through build/provider validation.
2. Fail closed when initialization is incomplete; do not continue into desktop mutation or hotkey registration.
3. Catch exceptions at the narrowest layer that can add context or recover.
4. Preserve the original exception, operation, Windows build, and selected provider.
5. Show concise user messages and make technical details easy to copy.
6. Never claim recovery from native memory corruption.
7. Avoid logging window titles, document names, command lines, or other potentially sensitive data by default.

## Logging design

Use structured logging behind an application-owned interface so the sink can change later. Serilog with a rolling file sink is a reasonable initial implementation, but package versions should be selected when implementation begins rather than pinned in this document.

Default location:

```text
%LOCALAPPDATA%\VirtualDesktopGridSwitcher\Logs\vdgs-YYYYMMDD.log
```

Recommended policy:

- Information level by default; Debug can be enabled for troubleshooting.
- Daily rolling files with a size limit.
- Keep seven recent files.
- Flush logs during normal shutdown and fatal-error handling.
- If file logging cannot initialize, continue with a minimal fallback such as `Trace` and a user-visible warning.
- Do not require administrator access or Event Log source registration.

Every startup record should include:

- Application and vendored-library versions
- OS display version, build, and UBR
- Process architecture
- Selected VirtualDesktop provider and IID settings version
- Generated COM assembly source/cache decision
- Settings path and schema version, without settings values that may be sensitive

## Error categories

| Category | Examples | Expected response |
|---|---|---|
| Startup | Settings, tray, hook, or manager initialization | Abort cleanly if core services are unavailable. |
| Windows compatibility | Unknown build, missing IID, provider failure | Show build/provider diagnostics and avoid desktop mutation. |
| COM/native interop | Desktop query/switch/move/pin failure | Log HRESULT and operation; stop if state may be unsafe. |
| Hotkeys | Duplicate configuration or OS registration conflict | Continue with affected hotkey disabled and identify it. |
| Settings | Missing, malformed, migration, or write failure | Back up corrupt data, use explicit defaults, and tell the user. |
| Window management | Foreground activation, process query, or move failure | Log at warning/error and continue when safe. |
| Shutdown | Timer, hook, hotkey, or logger disposal failure | Attempt remaining cleanup and record every failure. |

## Implementation sequence

### Phase 1: logging infrastructure

- Add `IAppLogger` or use `Microsoft.Extensions.Logging.ILogger` at application boundaries.
- Configure the rolling LocalAppData sink before loading settings.
- Add a diagnostics context containing versions, OS build/UBR, and process architecture.
- Ensure startup logging cannot itself terminate the application without a fallback message.

### Phase 2: startup boundary

In `Program.Main()`:

- Configure WinForms exception behavior before creating application services.
- Handle `Application.ThreadException` for UI-thread managed exceptions.
- Record `AppDomain.CurrentDomain.UnhandledException` and `TaskScheduler.UnobservedTaskException` for diagnostics.
- Wrap initial settings/tray/manager construction in a top-level managed exception boundary.
- Dispose successfully initialized resources in reverse order.
- Exit with a clear fatal dialog when the grid manager cannot initialize safely.

These handlers improve diagnostics; they do not make an invalid COM ABI recoverable.

### Phase 3: transactional manager startup

Refactor `VirtualDesktopGridManager.Start()` into explicit stages:

1. Detect and validate OS/provider support.
2. Query current desktops without mutation.
3. Validate the requested grid and calculate the proposed changes.
4. Obtain explicit authorization for desktop removal.
5. Apply desktop changes.
6. Build lookups and window-tracking state.
7. Start polling and hooks.
8. Register hotkeys.

On failure, undo only operations known to be reversible, dispose initialized services, and return a structured failure. Do not continue with icon or hotkey setup after core initialization fails.

### Phase 4: replace silent catches

For each existing catch block:

- Catch the narrowest expected exception where possible.
- Add operation-specific context, including desktop index/GUID or HWND when appropriate.
- Decide explicitly between retry, feature disablement, startup abort, and safe continuation.
- Preserve HRESULTs for COM exceptions.
- Avoid repetitive dialogs for timer or hook callbacks; rate-limit recurring log messages.

Priority locations:

- Settings load, migration, and save
- Desktop initialization and switching
- Pin/unpin and window movement
- Hotkey registration and disposal
- Foreground-hook callbacks
- Browser/document workarounds
- Timer startup and shutdown

### Phase 5: user-facing error dialog

Provide a reusable dialog with:

- Short plain-language summary
- Suggested next action
- Windows build and application version
- Expandable technical details
- **Copy details** and **Open log folder** actions
- Link to the relevant troubleshooting/support page

Do not expose stack traces in the primary message, but retain them in logs and copied details.

### Phase 6: settings recovery

- Store settings under application-specific LocalAppData.
- Write through a temporary file and replace atomically.
- Preserve a last-known-good backup before migration or replacement.
- On malformed XML, offer backup/reset rather than silently discarding it.
- Distinguish “first run, no file” from “existing file could not be read.”

## Validation

Automated tests should cover:

- Logger initialization and fallback behavior
- Log retention/configuration
- Missing and malformed settings
- Failed settings migration/write
- Duplicate and OS-rejected hotkeys
- Manager stage failures using a fake VirtualDesktop service
- Repeated timer/hook errors without dialog or log flooding
- Fatal-dialog detail generation without leaking configured sensitive values

Manual tests should cover:

- Non-writable installation directory
- Explorer restart
- Unsupported/newer Windows build
- COM initialization failure
- Installer upgrade and uninstall
- Start-at-login failure and duplicate process launch

Tests that create, remove, switch, or move desktops must be opt-in and run only on a disposable test machine.

## Completion criteria

- No exception is silently discarded without a documented reason.
- Fatal initialization failures stop before desktop mutation and hotkey registration.
- Users can copy useful diagnostic details without a debugger.
- Logs identify the Windows build, provider, IID set, and failing operation.
- Settings corruption and write failures are visible and recoverable.
- Recurring callbacks cannot flood logs or dialogs.
- Native access violations are prevented through compatibility checks rather than treated as recoverable exceptions.
