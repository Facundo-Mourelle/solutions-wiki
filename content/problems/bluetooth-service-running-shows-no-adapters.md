---
title: "Bluetooth Service Running Shows No Adapters"
date: 2026-09-08T22:33:10Z
draft: false
description: "bluetooth.service active but system shows No adapters / No default controller due to rfkill soft block"
categories: ["System Administration"]
tags: ["bluetooth", "bluez", "rfkill", "fedora", "niri"]
---

## Problem

On a Dell Latitude E7470 (Intel Wireless 8260, Fedora, niri + DankMaterialShell), the navigation/powermenu Bluetooth section showed "No adapters" despite `bluetooth.service` being `enabled` and `active (running)`.

`bluetoothctl list` returned `No default controller available`, `lsusb` showed no Bluetooth device, and `ls -l /sys/class/bluetooth/` was empty.

```
systemctl status bluetooth  # active (running), Bluetooth daemon 5.87
bluetoothctl list           # No default controller available
lsusb                       # only webcam + Broadcom smartcard reader, no 8087:0a2b
```

## Root Cause

The `dell-bluetooth` rfkill switch was soft-blocked, which hides the HCI adapter from BlueZ and powers down the USB interface.

```
rfkill list
0: dell-wifi: Wireless LAN     Soft blocked: no
1: dell-bluetooth: Bluetooth   Soft blocked: yes   # <- culprit
2: phy0: Wireless LAN          Soft blocked: no
```

With `Soft blocked: yes`, the kernel does not expose the Intel 8260 Bluetooth USB device (`8087:0a2b`) on the bus. `btusb`, `btintel` etc. were loaded but had no device to bind. `bluetoothd` starts successfully ("Bluetooth management interface 1.23 initialized") but has no controller to register, so BlueZ reports no adapters and DMS correctly shows "No adapters". The block is independent of the systemd service state.

`dmesg` was empty for bluetooth until unblocked, confirming the device was gated by rfkill rather than missing firmware/driver.

## Attempted Solutions

- **Checked service state** — `systemctl is-enabled bluetooth` / `is-active bluetooth` and `journalctl -u bluetooth` confirmed the daemon was running and initializing the management interface. Ruled out service/config failure; pointed to hardware/rfkill layer.
- **Inspected hardware enumeration** — `lsusb`, `lsusb -t`, `lspci -nnk`, `lsmod | grep btusb|bluetooth` showed `btusb`/`btintel` loaded but no `8087:0a2b` device. `lspci` showed Intel Wireless 8260 (`8086:24f3` with `iwlwifi`) — combo WiFi+BT card should provide BT over USB, so missing USB device indicated RF-kill gating, not absent hardware.
- **First unblock attempt needed verification** — initial `rfkill unblock bluetooth` via session bus succeeded (exit 0, `/dev/rfkill` writable for `wheel`), flipping `dell-bluetooth` to `Soft blocked: no` and immediately exposing `hci0`. A `sudo`-based attempt had failed with `a terminal is required to read the password`, but the non-sudo `rfkill` worked.

## Final Solution

Run the rfkill unblock and verify the adapter appears:

```bash
rfkill unblock bluetooth
# if the above still shows Soft blocked: yes, use:
sudo rfkill unblock bluetooth

rfkill list
# expect:
# 1: dell-bluetooth: Bluetooth  Soft blocked: no
# 3: hci0: Bluetooth            Soft blocked: no

bluetoothctl list
# Controller XX:XX:XX:XX:XX:XX <hostname> [default]

lsusb
# Bus 001 Device 005: ID 8087:0a2b Intel Corp. Bluetooth wireless interface

bluetoothctl show
# Powered: yes
```

After ~1 second, DankMaterialShell / niri powermenu refreshes and the Bluetooth section lists the adapter instead of "No adapters". No reboot or service restart required — `bluetoothd` picks up `hci0` hotplug automatically (visible in `journalctl -u bluetooth` as `Endpoint registered` lines and `Battery Provider Manager created`).

If the block returns after reboot, make it persistent:

```bash
sudo rfkill unblock bluetooth
systemctl status systemd-rfkill  # saves/restores rfkill state
# or add `rfkill unblock bluetooth` to a startup script / niri `spawn-at-startup`
```

Also check BIOS airplane-mode / hardware toggle on Dell laptops (Fn+wireless key) which can re-assert `Hard blocked: yes`.

## Why It Works

`rfkill` soft blocks are enforced in the kernel rfkill subsystem. For the `dell-bluetooth` / `dell_laptop` platform driver, a soft block powers down the Bluetooth USB function on the Intel 8260 combo card and suppresses the `hci0` device. BlueZ only exposes adapters that the kernel exports under `/sys/class/bluetooth/` via the management interface; with `hci0` suppressed, `bluetoothd` has nothing to advertise, hence "No adapters" regardless of service state.

`rfkill unblock bluetooth` clears the soft block for all `type: bluetooth` rfkill indices (`dell-bluetooth` and the subsequent `hci0`). The xHCI bus re-enumerates `8087:0a2b`, `btusb` binds to `1-8:1.0`/`1-8:1.1`, the kernel creates `hci0`, and `bluetoothd` registers the controller over mgmt — `bluetoothctl` and DMS then see a powered, discoverable controller.
