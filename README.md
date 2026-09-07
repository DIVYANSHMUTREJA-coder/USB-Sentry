# USB-Sentry
An air-gapped hardware USB security proxy built on an embedded Linux SBC to neutralize BadUSB keystroke injection attacks and automate read-only malware scanning on untrusted mass storage devices.
# USB-Sentry

An air-gapped hardware security gateway built on an embedded RISC-V Single Board Computer (Milk-V Duo). USB-Sentry acts as an active, isolated proxy between untrusted USB peripherals and host workstations to neutralize BadUSB keystroke injection attacks and inspect external flash storage before host enumeration.

---

## Architecture Overview

Standard computing endpoints implicitly trust peripherals enumerated across physical USB ports. USB-Sentry eliminates this implicit trust model by breaking the direct physical connection between the accessory and the target machine:

```
[ Untrusted Peripheral ] (Keyboard / Flash Drive)
           │
           ▼ (Downstream Host Port)
[ USB-Sentry Gateway ] (Milk-V Duo / RISC-V Linux)
   │
   ├── HID Path: Inline evdev packet timing & shortcut suppression
   │     ├── Drops keystroke intervals below human thresholds (<30ms)
   │     └── Intercepts automated payload hotkeys (e.g., GUI+R)
   │
   └── Mass Storage Path: Read-only sandbox & file header inspection
         ├── Auto-mounts partition strictly with ro, noexec, nosuid
         ├── Detects disguised binaries (PE header / "MZ" magic-byte checks)
         └── Matches SHA-256 hashes & flags weaponized extensions (.scr, .bat, .lnk)
           │
           ▼ (Upstream USB-C Port / Gadget Mode)
[ Target Host Workstation ]

```

---

## Features

* **Inline HID Injection Defense:** Analyzes incoming keystroke timing deltas via Linux `evdev`. Enforces a hardware lockout if sustained typing rates exceed human limits (>180 WPM) or if unauthorized elevated shortcuts are dispatched.
* **Hardened Storage Sandbox:** Auto-intercepts mass storage devices using strict Linux mount policies (`ro, noexec, nosuid, nodev`), preventing malicious auto-run scripts or firmware exploits from executing.
* **Low-Footprint Heuristic Scanner:** Engineered specifically for resource-constrained embedded systems (under 30MB RAM footprint). Inspects raw binary headers (magic bytes) to catch executables masquerading as harmless formats (`.pdf`, `.jpg`) alongside known-bad hash lookups.
* **Physical Hardware Interlock:** Includes hardware interrupt controls via GPIO push-buttons, requiring physical human authorization before releasing flagged hotkey sequences.
* **Real-Time Telemetry:** Integrates an I2C SSD1306 OLED display for real-time reporting of typing speeds (WPM), packet drop status, and file scan verdicts.

---

## Hardware Requirements

* **SBC:** Milk-V Duo (or Milk-V Duo S / 256M)
* **USB Interface:** USB Type-C Data Cable (Host connection via USB Gadget Mode)
* **Peripheral Input:** Micro-USB / Type-C OTG Host Adapter Cable
* **Display:** 0.96-inch I2C OLED Display (SSD1306)
* **Physical Input:** 1x Tactile Push Button + Jumper Wires
* **Storage:** 16GB / 32GB Class 10 MicroSD Card

---

## Repository Structure

```text
├── drivers/
│   ├── setup_gadget.sh        # Configures USB composite device (HID & Storage)
│   └── udev_rules/
│       └── 99-usb-sentry.rules # Dynamic peripheral detection & mounting rules
├── src/
│   ├── hid_firewall.py        # Keystroke velocity & combination filter
│   ├── storage_scanner.py     # Magic-byte inspector, extension & hash scanner
│   ├── display_driver.py      # SSD1306 OLED telemetry & UI renderer
│   └── main.py                # Core orchestration service
├── tests/
│   ├── simulate_badusb.py     # Inhuman-rate keystroke injection test script
│   └── create_test_img.sh     # Generates mock FAT32 drive images with test payloads
├── README.md
└── LICENSE

```

---

## Quickstart (Simulation Mode)

You can run and evaluate the core defense logic inside an Ubuntu/Debian terminal without physical hardware attached.

### 1. Prerequisites

```bash
sudo apt update
sudo apt install -y python3 python3-pip dosfstools
pip3 install evdev

```

### 2. Test Storage Sanitizer

Generate a simulated 32MB FAT32 flash drive containing a disguised executable:

```bash
# Create disk image
fallocate -l 32M untrusted_test.img
mkfs.vfat untrusted_test.img

# Mount temporarily to plant test files
sudo mkdir -p /mnt/test_setup
sudo mount untrusted_test.img /mnt/test_setup
echo "Legitimate report text" | sudo tee /mnt/test_setup/annual_report.txt
# Fake an executable masquerading as a PDF (contains DOS header 'MZ')
printf "MZfakeexecutablepayload" | sudo tee /mnt/test_setup/invoice.pdf
sudo umount /mnt/test_setup

# Run the quarantine scanner
sudo python3 src/storage_scanner.py --image untrusted_test.img

```

### 3. Test Keystroke Velocity Defense

Run the HID proxy in one terminal:

```bash
python3 src/hid_firewall.py

```

Simulate automated keystroke injection bursts in a second terminal:

```bash
python3 tests/simulate_badusb.py

```

---

## Threat Model & Mitigations

| Attack Vector | Mechanism | USB-Sentry Mitigation |
| --- | --- | --- |
| **BadUSB / Rubber Ducky** | High-speed automated typing to drop payloads via terminal | Drops packets arriving with $<30\text{ ms}$ intervals; locks down after 3 consecutive violations. |
| **Autorun / Script Exploits** | Flash drive executes scripts immediately upon OS mounting | Linux sandbox mounts partitions strictly with `noexec, nosuid, ro`. |
| **Extension Spoofing** | Executable renamed to `document.pdf` to trick user into launching | Binary magic-byte inspection flags DOS/PE executable headers (`MZ`) on non-binary formats. |
| **Elevation Sequences** | Quick launch of run dialogs (`GUI+R` / `Super+Enter`) | Intercepts modifier key combinations; requires physical button press to clear. |

---

## License

This project is licensed under the MIT License - see the LICENSE file for details.
