# Plan: multi-display full screen on macOS

Status: plan only. Nothing here is implemented, built or tested. This file is a
working note and should be removed before any PR to UTM.

## Goal

Let a VM with several guest displays fill several host monitors in full screen,
and come back that way on the next start, for both QEMU and Apple
Virtualization VMs.

## Where things stand today

- Full screen is the standard macOS full screen on one window on one monitor
  (`Platform/macOS/Display/VMDisplayWindowController.swift:443`).
- **QEMU:** each guest display already gets its own window
  (`VMDisplayQemuDisplayController.swift:261`, `registerSecondaryWindow` in
  `VMDisplayWindowController.swift:384`). Each window saves and restores its own
  size and full-screen state, keyed by display id
  (`VMDisplayQemuMetalWindowController.swift:518-540`). Nothing remembers which
  monitor a window was on.
- **Apple:** only the first display is ever shown
  (`VMDisplayAppleDisplayWindowController.swift:118`, `//FIXME: add multiple
  displays`). The configuration already passes every display to the framework
  (`Configuration/UTMAppleConfiguration.swift:309-322`), but the settings UI
  only lets you add one (`VMAppleSettingsAddDeviceMenuView.swift:23-25`).

## Priorities

| Priority | Work | Backend | Size | PR |
|---|---|---|---|---|
| P0 | One window per guest display | Apple | Medium, plus a feasibility risk | 1 |
| P1 | Remember each window's monitor and restore full screen there | QEMU and Apple | Small | 2 |
| P2 | "Enter Full Screen on All Displays" menu item | Both | Small | 3, optional |

P1 does not depend on P0 and carries no feasibility risk, so it can ship first
if P0 stalls.

## P0: Apple multi-window

### Step 0: feasibility spike (on a Mac, before any design)

Two things are unverified:

- **Can a second display be shown at all?** `VZVirtualMachineView` takes a whole
  VM (`appleView.virtualMachine = appleVM.apple`,
  `VMDisplayAppleDisplayWindowController.swift:83`). It is not known whether
  there is a public way to point a view at a specific display. If there is not,
  the options are a private API or dropping P0. The code already uses one
  private key (`_supportsReconfiguration`, line 37), but that is the
  maintainers' call.
- **How many displays does the framework accept?** Check whether configuration
  validation accepts more than one entry in
  `VZMacGraphicsDeviceConfiguration.displays` (macOS guests) and
  `VZVirtioGraphicsDeviceConfiguration.scanouts` (Linux guests).

### Changes, if the spike passes

1. **Settings.** Allow adding more than one display, up to the limit the spike
   finds (`VMAppleSettingsAddDeviceMenuView.swift:23-25`).
2. **Window controller.** Give `VMDisplayAppleDisplayWindowController` a display
   index and a secondary initialiser, as the Apple terminal controller has
   (`secondaryForIndex:`). Replace the `displays.first` uses with that index
   (lines 34, 41, 63 and 117).
3. **Opening windows.** `VMDisplayAppleWindowController.swift:76-90` already
   opens secondary serial windows at start. Do the same for displays 2 and up.
4. **Per-display state.** Save and restore hard-code `resolutionSettings[0]`
   (lines 195 and 201). Key them by display index, as QEMU does.
5. **Window menu and titles.** Replace the single "Display" item
   (`VMDisplayAppleWindowController.swift:467`) with "Display N" items, and
   title secondary windows "(Display N)". Matching strings exist on the QEMU
   side.
6. **Input and capture.** `appleVM.inputDelegate` and the capture button assume
   a single display window. Decide which window owns them; the primary is the
   likely choice.
7. **Screenshots and close behaviour.** Keep the primary window as the
   screenshot source. Closing a secondary window must not stop the VM
   (`isSecondary` already handles this in the base class).

### Tests

- A configuration test that several displays produce several VZ displays or
  scanouts, alongside the existing Apple tests in `Tests/macOS`.
- Manual: macOS guest and Linux guest, a two-monitor Mac, and an older macOS
  host (dynamic resolution needs macOS 14).

## P1: remember the monitor (both backends)

1. Add an optional `screenUUID: String?` to `UTMRegistryEntry.Resolution`
   (`Services/UTMRegistryEntry.swift:545`), decoded with `decodeIfPresent` so
   existing registries still load. Use the display's persistent UUID
   (`CGDisplayCreateUUIDFromDisplayID`), which survives reboots and re-plugging.
2. **QEMU:** save it in `saveDynamicResolution` and on screen change; in
   `restoreDynamicResolution`, move the window to the saved screen before
   `toggleFullScreen` (`VMDisplayQemuMetalWindowController.swift:520-540`). The
   existing screen-change observer at line 118 listens to every window and
   should be filtered to this one.
3. **Apple:** the same in `VMDisplayAppleDisplayWindowController.swift:188-208`.
4. If the saved monitor is missing, fall back to today's behaviour.
5. Add a Codable round-trip test that includes a legacy entry without the new
   key.

Estimated at 100-150 lines for QEMU plus 20-30 for Apple.

### Risks to check on a real Mac

- **Move, then full screen.** `VMDisplayWindow` overrides `setFrame` and
  `toggleFullScreen` for the camera-housing handling. The move must finish
  before the transition starts and may need deferring a runloop turn.
- **Simultaneous transitions.** Several windows entering full screen at once on
  start may need sequencing.
- **"Displays have separate Spaces" off.** macOS then blanks the other
  monitors, so multi-monitor full screen cannot work. Document rather than work
  around.
- **Fixed-resolution displays.** Saving is skipped unless dynamic resolution is
  active. Keep that, so the feature applies only to guests with the display
  tools installed.

## P2: menu item

A View menu command that assigns guest display N to monitor N and full-screens
all of them. Build only once P0 and P1 are proven.

## Constraints

- Building and testing need macOS, Xcode and the sysroot artifacts (see
  `Documentation/MacDevelopment.md`).
- `CONTRIBUTING.md` requires one feature per PR, with test evidence, device and
  OS details, and screenshots. Hence three PRs.
- Run `/utm-review` then `/utm-submit` before each PR.

## Next action

Run the Step 0 spike on a Mac.
