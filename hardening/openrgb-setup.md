# OpenRGB Peripheral Configuration

Living build — `MANUAL INPUT REQUIRED` markers flag values still to be captured from the running machine; `FUTURE WORK` markers flag planned enhancements.

## Rationale

RGB peripheral control is handled entirely by **OpenRGB**, an open-source, vendor-agnostic
controller. No vendor daemons (Razer Synapse, Corsair iCUE) are installed. This is a deliberate
operational-security decision, not a cosmetic one:

- Vendor RGB suites of this class commonly require account registration, run persistent background
  services, and have historically shipped telemetry that cannot be fully disabled without
  network-level blocking.
- They typically run with elevated privileges and auto-update from vendor infrastructure —
  expanding the trusted-software footprint on a workstation that is otherwise minimised.

OpenRGB replaces all of that with a single local tool that talks to the devices directly over USB
HID, with no cloud dependency and no account.

## Controlled hardware

| Device | Type | Connection |
|--------|------|------------|
| Razer Huntsman V2 | Keyboard | USB HID |
| Razer Basilisk | Mouse | USB HID |
| 6 × Corsair fans via Corsair Commander Pro | Chassis fans (RGB) | Commander Pro Channel 1 |

**Total addressable LEDs: 96** across the six Corsair fans on Commander Pro Channel 1. The
Commander Pro itself is detected as a USB HID device; OpenRGB drives all six fan channels through
it. No Corsair software is installed.

## Channel layout

| Controller | Channel | Devices | LED count |
|------------|---------|---------|-----------|
| Corsair Commander Pro | Channel 1 | 6 × RGB fans (daisy-chained) | 96 LEDs total |
| Razer (direct) | — | Huntsman V2 keyboard | per-key (vendor daemon not required) |
| Razer (direct) | — | Basilisk mouse | onboard zones |

## Configuration approach

- Devices are configured through OpenRGB's profile system; settings are applied locally and
  persisted to an OpenRGB profile rather than to vendor cloud storage.
- The Commander Pro is addressed as a USB HID device — all six fans on Channel 1 are driven as a
  single addressable strip of 96 LEDs.
- Razer devices are driven directly by OpenRGB's Razer plugin; Synapse is never installed.

<!-- MANUAL INPUT REQUIRED: record the OpenRGB version used — run `openrgb --version` -->
<!-- MANUAL INPUT REQUIRED: record the saved OpenRGB profile name and colour scheme from the OpenRGB GUI -->

## Non-root access (udev)

OpenRGB requires udev rules to access HID devices without running as root. Running RGB control as
an unprivileged user is preferable to granting a cosmetic tool root on a hardened workstation.

```
# OpenRGB ships a udev rules file; install it and reload so a non-root user
# can access the controllers over HID.
sudo cp 60-openrgb.rules /usr/lib/udev/rules.d/
sudo udevadm control --reload-rules
sudo udevadm trigger
```

<!-- MANUAL INPUT REQUIRED: confirm the installed udev rules path (OpenRGB ships 60-openrgb.rules) and any device-specific entries required for the Razer/Corsair controllers -->

## Verification

```
openrgb --list-devices          # confirm keyboard, mouse, and Commander Pro detected
openrgb --list-devices | grep -i commander   # confirm Commander Pro + fan channel
```

Confirm no vendor services are present:

```
systemctl list-units | grep -iE 'razer|corsair|icue|synapse'   # expected: no results
```

## Current state

| Item | Status |
|------|--------|
| OpenRGB controlling Razer Huntsman V2 | Operational |
| OpenRGB controlling Razer Basilisk | Operational |
| 6 × Corsair fans via Commander Pro Channel 1 (96 LEDs) | Operational |
| Vendor daemons (Synapse, iCUE) | Absent — none installed |
| Non-root HID access via udev | Operational |
