# Project Roadmap

This roadmap records proposed work; checked items are implemented in the current tree. Priorities account for the fact that application startup can change real virtual desktops and that the controlling Windows COM interfaces are undocumented.

## Completed foundation

- [x] Migrate the application and vendored library to .NET 8.
- [x] Synchronize the official build-26100 provider, interfaces, and IID settings.
- [x] Verify read-only COM initialization on Windows 11 25H2 build 26200.8037.
- [x] Create one source-based technical navigation guide.

## 1. Safety and reliability

Complete this before automatic startup or broad installer distribution.

- [ ] Stop removing surplus desktops automatically by default.
- [ ] Require explicit confirmation for any operation that removes desktops.
- [ ] Make initialization transactional so failures do not leave partially initialized state.
- [ ] Add a single-instance guard.
- [ ] Fix the null timer shutdown path when polling is disabled.
- [ ] Remove the duplicate COM switch in the `Current` setter.
- [ ] Close native process handles and reduce requested process access rights.
- [ ] Validate positive grid dimensions and desktop/icon limits.
- [ ] Add production logging and global managed exception handling; see `ERROR_HANDLING_LOGGING_PLAN.md`.
- [ ] Report unsupported or untested Windows builds with actionable diagnostics.

## 2. Windows compatibility

- [ ] Define a public support matrix by Windows edition, display version, build, architecture, and test level.
- [ ] Certify Windows 10 22H2/ESU and supported LTSC editions on real or virtual machines before advertising support.
- [ ] Complete mutating-operation tests on Windows 11 24H2 and 25H2.
- [ ] Add a non-destructive diagnostics command that reports OS build/UBR, provider, IID set, and generated assembly.
- [ ] Monitor Microsoft Release Health and maintained VirtualDesktop implementations for ABI changes.
- [ ] Warn on builds newer than the newest tested build without blindly blocking compatible updates.
- [ ] Document exact test evidence in `WINDOWS_11_24H2_FIX_GUIDE.md` or a future compatibility matrix.

A new provider should be added only when an interface GUID or ABI actually changes, not for every monthly cumulative update.

## 3. Tests and continuous integration

- [ ] Extract grid navigation into a Windows-independent component and unit-test edges and wrap-around behavior.
- [ ] Unit-test settings defaults, migration, validation, and malformed XML recovery.
- [ ] Unit-test hotkey expansion and duplicate detection behind an abstraction.
- [ ] Wrap the VirtualDesktop API so application behavior can be tested with a fake provider.
- [ ] Add non-destructive COM smoke tests that are opt-in and clearly separated from unit tests.
- [ ] Add a root GitHub Actions workflow for restore, Release build, tests, and artifact creation.
- [ ] Treat new compiler/analyzer warnings as failures after reducing the existing warning baseline.
- [ ] Add Markdown/link checks and `git diff --check` to CI.

## 4. Installer and distribution

Recommended first implementation: an Inno Setup per-user installer using the existing self-contained `win-x64` publish output.

Before installing under a protected directory:

- [ ] Move settings to `%LOCALAPPDATA%\VirtualDesktopGridSwitcher\`.
- [ ] Migrate an existing portable settings file without losing user configuration.
- [ ] Keep logs and generated COM assemblies under application-specific LocalAppData paths.

Installer work:

- [ ] Add repeatable `dotnet publish` and installer scripts.
- [ ] Add Start Menu and optional desktop shortcuts.
- [ ] Preserve settings on upgrade and offer explicit removal on uninstall.
- [ ] Remove startup registration during uninstall.
- [ ] Produce checksums and sign the executable and installer.
- [ ] Publish release artifacts through GitHub Releases and optionally WinGet.
- [ ] Keep trimming disabled; test single-file publishing separately before enabling it.
- [ ] Consider `win-arm64` only after native interop testing on Windows on ARM.

## 5. Start at login

Implement only after startup is non-destructive and single-instance protection exists.

- [ ] Add a default-off **Start with Windows** setting.
- [ ] Register per user through `HKCU\Software\Microsoft\Windows\CurrentVersion\Run`.
- [ ] Quote the executable path and use a recognizable value name.
- [ ] Add a `--startup` argument for delayed, low-noise initialization and diagnostics.
- [ ] Make the installer option and in-application checkbox use the same registration mechanism.
- [ ] Reflect the actual registry state in the UI rather than trusting a duplicated XML value.
- [ ] Recover cleanly when an installation is moved or upgraded.

## 6. User experience and maintainability

- [ ] Generate numbered tray icons dynamically or enforce the number of icons that ship.
- [ ] Show configured hotkey conflicts before applying settings.
- [ ] Add settings export/import/reset and automatic corrupt-file backup.
- [ ] Display app version, Windows build/UBR, selected provider, and log location in About/Diagnostics.
- [ ] Improve high-DPI, keyboard navigation, screen-reader labels, and dark-mode behavior.
- [ ] Replace dead timer code and stale P/Invoke overloads.
- [ ] Separate UI, grid policy, hotkeys, and native integrations behind small interfaces.
- [ ] Track the vendored VirtualDesktop source explicitly and document local deviations.
- [ ] Add an opt-in update check only after signed, reproducible releases exist.

## Release gates

Before the first installer-backed release:

1. No automatic desktop deletion under default settings.
2. Single-instance behavior verified.
3. Settings and logs work from a non-writable installation directory.
4. Release build and automated tests pass.
5. Windows support matrix records the exact tested builds.
6. Installer install, upgrade, startup, repair, and uninstall paths are tested.
7. Executables and installer are signed, or unsigned status is disclosed clearly.
