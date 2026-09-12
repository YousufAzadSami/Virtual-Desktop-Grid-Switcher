# Windows 11 24H2 Compatibility Notes

This document records the Windows build-specific COM work incorporated into this fork. It is deliberately narrower than `Documentation/Technical/TECHNICAL_DOCUMENTATION.md`, which is the canonical architecture and navigation guide.

## Current status

| Windows family | Build | Provider selected | Status in this repository |
|---|---:|---|---|
| Windows 10 | 19041–19045 | `Build10240` | The app target begins at build 19041; Windows 10 22H2 has an explicit IID set. Runtime verification is still required. |
| Windows Server 2022 | 20348 | `Build20348` | Provider and IID set present; not verified during consolidation. |
| Windows 11 21H2 | 22000 | `Build22000` | Provider and IID set present. |
| Windows 11 22H2/23H2 | 22621/22631 | `Build22621` | Multiple revision-specific IID sets present. |
| Windows 11 24H2 | 26100 | `Build26100` | Uses Slion's 26100.0 ABI and IID set. |
| Windows 11 25H2 | 26200 | `Build26100` | Read-only COM smoke test passed on 26200.8037. |

The provider and IID selectors use the newest known version that is not greater than the current OS version. A separate provider is not required for every cumulative update; one is needed only when Microsoft changes an undocumented interface GUID or ABI.

## Why 24H2 required a change

Virtual desktop creation, removal, switching, and window movement depend on undocumented Explorer COM interfaces. Windows 11 build 26100 changed the `IVirtualDesktopManagerInternal` interface identity and layout. Reusing the 22621/22631 definitions can lead to initialization errors or process-level access violations.

The earlier repository snapshot contained an experimental `Build26100_863` directory, but it was incomplete:

- Embedded interface files used the `Build22631` namespace.
- The 26100 IID set was absent from `Properties/Settings.settings` and its generated designer.
- The only attempted IID entry was in `app.config`, while `IID.GetIIDs()` enumerates properties exposed by the generated `Settings` class.
- The attempted manager IID did not match the later Slion 26100 implementation.

## Integrated solution

The current implementation follows Slion's April 2025 build-26100 support:

- Source: <https://github.com/Slion/VirtualDesktop/commit/cab02a9c02586656a359da5a304d2f735023c45a>
- Build directory: `VirtualDesktop-master/src/VirtualDesktop/Interop/Build26100_0000/`
- Provider threshold: `10.0.26100.0`
- `IVirtualDesktopManagerInternal` IID: `{4970BA3D-FD4E-4647-BEA3-D89076EF4B9C}`

The following representations are synchronized:

1. `VirtualDesktop.system.cs` provider selection.
2. `Interop/Build26100_0000/` wrappers and exact interface layouts.
3. Embedded-resource entries in `VirtualDesktop.csproj`.
4. `app.config` entry `v_26100_0000`.
5. `Properties/Settings.settings` entry `v_26100_0000`.
6. Generated `Properties/Settings.Designer.cs` property `v_26100_0000`.

`HString.cs` also uses the C#/WinRT `WinRT.HString` marshalling type required by the interface definitions.

## Verification performed

The solution builds successfully on .NET 8. A separate STA diagnostic program was used instead of launching the grid-switcher application because normal application startup changes the machine's desktop count.

On Windows 11 25H2 build `26200.8037`, the diagnostic:

- Disabled generated-assembly persistence and used a new empty cache location.
- Initialized the VirtualDesktop library.
- Confirmed selection of `WindowsDesktop.Interop.Build26100.VirtualDesktopProvider26100`.
- Queried all existing desktops.
- Resolved the current desktop among them.
- Did not create, remove, switch, rename, or move anything.

This verifies provider selection, IID loading, runtime Roslyn compilation, and read-only COM calls on that build. It does not verify every mutating operation or native notifications. Native VirtualDesktop notification registration remains disabled in this application; desktop changes are detected by polling.

## Testing before release

Perform the following manual checks on each Windows build advertised as supported:

1. Start with disposable or backed-up virtual-desktop state.
2. Confirm the selected provider and full `CurrentBuild.UBR` in diagnostics.
3. Read the current desktop and enumerate all desktops.
4. Create one temporary desktop and remove it with an explicit fallback.
5. Switch in both directions.
6. Move a normal Win32 window between desktops.
7. Exercise pinned-window and always-on-top behavior.
8. Switch externally through Task View and confirm polling detects it.
9. Restart Explorer and confirm the application either recovers or exits clearly.
10. Verify settings and generated COM assembly behavior after an application upgrade.

## Supporting a future Windows build

Do not add a build number merely because a cumulative update was released. First reproduce an ABI or IID failure on the exact build.

When a change is required:

1. Record Windows edition, display version, `CurrentBuild`, and `UBR`.
2. Compare maintained implementations such as Slion/VirtualDesktop and MScholtes/VirtualDesktop.
3. Verify GUIDs from reliable sources or local registration/debug symbols.
4. Preserve exact COM method order, parameter types, and marshalling.
5. Add a versioned build directory only when interface source actually differs.
6. Update provider selection, embedded resources, `app.config`, `Settings.settings`, and `Settings.Designer.cs` together.
7. Clear or bypass cached generated assemblies during testing.
8. Run read-only initialization first, then explicit mutating tests on disposable desktop state.
9. Document the exact build and operations tested; avoid claiming support for “all Windows versions.”

Because these APIs are undocumented, graceful failure and diagnostic logging remain necessary even for tested builds.
