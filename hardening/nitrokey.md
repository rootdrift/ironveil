# Nitrokey 3A NFC — FIDO2 Hardware Key Integration

Living build — `MANUAL INPUT REQUIRED` markers flag values still to be captured from the running machine; `FUTURE WORK` markers flag planned enhancements.

## Hardware

| Attribute | Value |
|-----------|-------|
| Device | Nitrokey 3A NFC |
| Firmware | 1.8.3 |
| Role in build | FIDO2 keyslot for LUKS2 full-disk encryption unlock |

## Key roles

Two physical keys are enrolled against the LUKS2 volume (see
[luks2-setup.md](luks2-setup.md) for keyslot detail):

| Key | LUKS2 slot | Role |
|-----|-----------|------|
| NK#1 | Slot 1 | **Primary** — daily driver; touch required for every unlock |
| NK#2 | Slot 2 | **Backup** — kept offline; activated only if NK#1 is lost |

A separate offline emergency passphrase (slot 0) guarantees recovery even if both keys are
unavailable.

## FIDO2 LUKS enrollment

Enrollment uses `systemd-cryptenroll --fido2-device=auto`. The Nitrokey produces a FIDO2
credential combined with a stored key file to derive the slot key; the LUKS2 passphrase is never
sent to the key.

```
# Confirm the device is detected
nitropy fido2 list                  # or: fido2-token -L

# Enrol NK#1 (inserted), then repeat with NK#2 inserted
sudo systemd-cryptenroll /dev/sda3 --fido2-device=auto
```

## Touch-only constraint

Firmware 1.8.3 on the Nitrokey 3A NFC does **not** expose `clientPin`. Unlock therefore depends
on **physical touch confirmation** rather than a PIN. This is a deliberate security property: a
key that could be activated remotely over USB (via clientPin) would defeat the physical-presence
guarantee that underpins the whole disk-encryption model.

## Physical recovery and custody

| Item | Detail |
|------|--------|
| NK#1 | Carried / on the workstation for daily unlock |
| NK#2 | Stored offline as backup |
| Recovery case | **Pelican 1200** protective case for physical storage of the backup key and recovery material |

The Pelican 1200 provides a crush- and water-resistant enclosure for offline custody of NK#2
and the emergency passphrase material, keeping the recovery path physically protected and
separated from the running machine.

## Current state

| Item | Status |
|------|--------|
| NK#1 FIDO2 enrolment (firmware 1.8.3, touch-only) | Operational |
| NK#2 FIDO2 backup enrolment | Enrolled; stored offline |
| Pelican 1200 recovery custody | In place |

<!-- MANUAL INPUT REQUIRED: if non-root access to the Nitrokey is needed, record the udev rule in use — typically shipped by the nitrokey-udev-rules package under /usr/lib/udev/rules.d/, or a custom /etc/udev/rules.d/ entry; get device attributes with `udevadm info -a -n /dev/hidrawX` -->
<!-- FUTURE WORK: document the NFC enrolment/unlock path if NFC unlock is added (current build is USB touch-only) -->
