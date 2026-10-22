# win-surRa1n

This project ports the **Just Boot (`just_boot`)** feature of [surrealra1n](https://github.com/pwnerblu/surrealra1n) to Windows as a standalone executable. It is **intended only for A12 / A13 devices** (iPhone 11 / SE 2, etc.) by MYD.

The program performs two tasks:

1. **Automatically extracts, decrypts, and patches `iBSS.boot` from the IPSW firmware you provide.**
2. **Sends `iBSS.boot` to a pwned DFU device via the usbliter8 protocol**, allowing the device to complete a tethered boot.

---

## Supported Devices

| SoC | CPID   | Representative Models              |
| --- | ------ | ---------------------------------- |
| A12 | 0x8020 | iPhone XS / XR, iPad 8 (A12)       |
| A13 | 0x8030 | iPhone 11 / SE 2 (iPhone12,8)      |

---

## Prerequisites

1. **Firmware file (IPSW)**: Prepare the **Target IPSW** for the iOS version you want to boot (required) and, optionally, a **Base IPSW** (only required for a full custom IPSW; not used by `just_boot`, so it may be left blank).
   - iBSS is always extracted from the **Target IPSW** (consistent with surrealra1n).
   - The IV+Key used for decryption is provided by the program's bundled `keys/` directory (public firmware keys from Apple Wiki / theiphonewiki), covering many versions for iPhone12,8 and other devices. If your version is not included, the program will report a missing key.
2. **USB driver (required once before sending iBSS)**: DFU devices are bound to Apple's driver by default, and libusb **cannot** perform control transfers on them (all USB operations fail with `Input/Output Error` / `Errno 5`). You must use [Zadig](https://zadig.akeo.ie/) to replace the DFU device driver with **WinUSB** (or libusbK):
   - Put the iPhone into DFU mode and connect it.
   - Open Zadig → menu **Options → List All Devices**.
   - Select **Apple Mobile Device (DFU Mode)** (or `Apple Mobile Device USB Device`, i.e., VID 05AC / PID 1227).
   - Select **WinUSB** as the driver and click **Replace Driver** (or Install Driver).
   - This only needs to be done once. If the device re-enumerates later and loses PWND, reinstall the driver and re-pwn with the Pi Pico.
   - The program bundles `libusb-1.0.dll`; **no Python installation is required**.
3. **RP2350 / Pi Pico 2 development board**: Flash the usbliter8 firmware (see the UF2 in the usbliter8 repository Releases).
   The checkm8 pwn for A12/A13 must be performed by this board; this program **does not** perform the exploit on the PC side.
4. `keys/` is distributed with the program.

---

## Usage

Double-click `justboot.exe` (without any arguments). The program will display **native Windows file selection dialogs** and ask you to select the following in order:

1. **Target IPSW**
2. **Base IPSW**

After selection, the program will:

- Automatically identify the device model / version / build (e.g., `iPhone12,8 iOS 15.2 (build 19C56)`);
- Automatically **extract iBSS from the Target IPSW → decrypt offline using public keys → patch** (skip signature verification / inject boot-args / force LOCAL_BOOT);
- Generate `boot/<IDENTIFIER>/<VERSION>/iBSS.boot`;
- Ask whether to boot the device immediately (the device must be in DFU mode and pwned).

### Common Parameters

| Parameter                  | Description                                                                 |
| -------------------------- | --------------------------------------------------------------------------- |
| `--target-ipsw <path>`     | Target IPSW (the iOS version you want to boot; required for extraction)     |
| `--base-ipsw <path>`       | Base IPSW (optional; only for a full custom IPSW; not required by `just_boot`) |
| `--identifier iPhone12,8`  | Manually specify the device model (skip automatic detection)                |
| `--version 15.2`           | Manually specify the iOS version (skip automatic detection)                 |
| `--extract-only`           | Extract iBSS only; do not boot the device                                   |
| `--bootargs "-v"`          | boot-args to inject (default: `-v`)                                         |
| `--ecid <decimal ECID>`    | Manually specify ECID (skip automatic detection)                            |
| `--ibss D:\path\iBSS.boot` | Directly specify an existing iBSS.boot and skip extraction                  |
| `--bootdir D:\path\boot`   | Specify the boot directory location                                         |
| `--timeout 600`            | DFU / pwn wait timeout (seconds)                                            |
| `--probe`                  | Detect once, print device information (PID/serial/ECID/CPID/PWND), and exit; useful for confirming that the tool can see the device |

---

## Complete Device Boot Procedure

1. **Detect current state**: Identify whether the device is in Normal / Recovery / DFU / pwned state, and print PID, serial number, ECID, and CPID.
2. **Wait for DFU**: If the device is not in DFU mode, display "Please put the device into DFU mode (screen completely black)" and keep polling until DFU is detected (PID 0x1227). You must perform the steps to enter DFU mode yourself.
3. **Wait for pwn**: If the device is not yet pwned, prompt the user to run the usbliter8 exploit using the RP2350 / Pi Pico, then reconnect it to the computer; the program continuously checks for `PWND:[...]`.
4. **Send and boot**: Once a pwned DFU device is detected, send `iBSS.boot` (equivalent to `liter8ctl boot`), and the device resets and begins booting.

---

## Troubleshooting

- **Continually prompts "waiting to enter DFU"**: This tool identifies DFU by **USB PID 0x1227** (a real DFU serial number does not contain `MODE:DFU`; irecovery's MODE field is inferred from the device type). First run `justboot.exe --probe` to see whether `mode: DFU/PWND` can be detected:
  - No device detected → Check the USB cable/port and whether Zadig has installed WinUSB.
  - Device detected but `--probe` serial number is `(unreadable)` → The Apple driver has not been replaced with WinUSB; see above.
- **`Input/Output Error` / `Errno 5` when sending**: **The DFU device is not bound to WinUSB**, so libusb cannot perform control transfers. Use Zadig to install WinUSB/libusbK for `Apple Mobile Device (DFU Mode)` (VID 05AC / PID 1227), then rerun.
- **"could not confirm a Pwned DFU device"**: The device serial number does not contain `PWND:[...]`. First run the usbliter8 exploit with the RP2350/Pi Pico; if PWND disappears after reinstalling the driver with Zadig, re-pwn the device.
- **"No usable KBAG/key" / missing key**: The `keys/` directory does not contain the public key for your `<IDENTIFIER>` or version. Find the corresponding 96-bit hexadecimal (IV+Key) for `ibss-<version>` on Apple Wiki and append it to `keys/<IDENTIFIER>.txt`.
- **"decrypted data does not look like an iBoot image"**: The decrypted data is not a valid iBoot image. This is usually caused by an incorrect key or an unsupported compression format in the IPSW.
- **Device does not boot after sending**: Confirm that `iBSS.boot` was generated for the correct ECID / version of this device, that the device is indeed an A12/A13 device and has been pwned, and that the DFU device is bound to WinUSB. The "No such device" message at the end of sending is normal (the device resets and begins booting).

## Acknowledgements / Sources

- surrealra1n (just_boot flow): <https://github.com/pwnerblu/surrealra1n>
- usbliter8 (A12/A13 exploit and `usbliter8ctl` sending logic): <https://github.com/ahmadkamal09999-tech/usbliter8>
- `tools/usbliter8ctl.py` is a local archive of the upstream script for verifying the sending logic.
- iBSS decryption keys are from public firmware keys on Apple Wiki / theiphonewiki.
