# Glorious Forge Series One Pro Wireless Technical Specification

This document details the hardware architecture, MCU pinout, sensor SPI configuration, flash memory map, and resident USB HID bootloader of the **Glorious Forge Series One Pro Wireless** gaming mouse (`FCC ID: 2AXP2-SP22`, PCB silkscreen `CM2852-BAMF 20220516`), intended for hardware modding and ZMK/Zephyr firmware development.

## 1. Key Integrated Circuits & Hardware Overview

| **Component** | **Model / Marking** | **Purpose** | **Package / Notes** |
| --- | --- | --- | --- |
| **Microcontroller (`U1`)** | **Nordic `nRF52840-QIAA-D0`** (`N52840 QIAAD0 2144MX`) | Main 64 MHz ARM Cortex-M4F MCU, USB 2.0 FS controller, and 2.4 GHz radio transceiver | `aQFN-73` (`7x7 mm`), `1 MB` Flash, `256 KB` SRAM |
| **Optical Sensor (`U4`)** | **Glorious BAMF** (marking `BAMF 2152`; PixArt `PAW3370` derivative, inferred from the package and the stock driver's register sequence) | High-speed optical tracking sensor (`50` – `19,000` DPI, `400` IPS, `50G`) | `SPI2` @ `4 Mbps`, Mode 3. **No PAW3370 driver exists in Zephyr or ZMK yet.** |
| **Main PCB** | **Compx `CM2852-BAMF 20220516`** | Main switch, sensor, power, and radio board | Manufactured by Compx Technology |
| **Battery** | `3.7 V` Li-Po (`300 mAh`, `1.11 Wh`) | Rechargeable internal battery | `3-pin` JST connector (`+`, `NTC`, `-`) |

### USB VID:PID & Interface Layout

- **Wired USB Application**: `258a:2018` (`USB bcdDevice 1.01`, product string `Series One PRO Wireless`, full-speed USB, 3 HID interfaces with 91 / 163 / 20-byte report descriptors; Interface 2 exposes a 64-byte vendor Feature report with Report ID `0x00` on Usage Page `0xFFFF`, Usage `0x0000`).
- **Wireless 2.4 GHz Receiver Application**: `258a:2031` (`USB bcdDevice 1.00`, same product string and identical descriptors).
- **Wired Mouse Resident USB HID Bootloader**: `258a:2103` (product string `Glorious bootloader`; seen on a retail unit).
- **Wireless Receiver Resident USB HID Bootloader**: `258a:2113` (PID constant in the official updater; not seen on USB yet).

## 2. nRF52840 GPIO Pinout Mapping (`CM2852-BAMF`, from Firmware Disassembly)

The pin assignments below are inferred from disassembly of the official `CM2856_Mouse_USB_v1.00.00_20220810` firmware (`0x0000C5B8`–`0x0000C648` GPIO init, `0x00016B30`–`0x00016CE0` and `0x00009614`–`0x00009690` button and wheel scanner, `0x000184C8`–`0x0001855C` SPI2 sensor setup). None of them has been probed on hardware, and no ZMK image has been flashed yet.

| **Subsystem** | **Function** | **nRF52840 Pin** | **Direction / Pull** | **Active State & Notes** |
| --- | --- | --- | --- | --- |
| **Switches** | **Left Button (LMB)** | P1.15 | Input, Pull-Up | Active Low (`0` = pressed) |
| | **Right Button (RMB)** | P1.13 | Input, Pull-Up | Active Low (`0` = pressed) |
| | **Middle Button (MMB)** | P1.10 | Input, Pull-Up | Active Low (`0` = pressed) |
| | **Back Button (Side Btn 4)** | P0.09 (`NFC1`) | Input, Pull-Up | Active Low (`0` = pressed); requires `nfct-pins-as-gpios;` on `&uicr` (Zephyr 4.x / ZMK main; older trees: `CONFIG_NFCT_PINS_AS_GPIOS=y`) |
| | **Forward Button (Side Btn 5)** | P0.10 (`NFC2`) | Input, Pull-Up | Active Low (`0` = pressed); requires `nfct-pins-as-gpios;` on `&uicr` (Zephyr 4.x / ZMK main; older trees: `CONFIG_NFCT_PINS_AS_GPIOS=y`) |
| | **DPI Cycle Button** | P1.00 | Input, Pull-Up | Active Low (`0` = pressed) |
| **Scroll Encoder** | **Quadrature Phase A** | P0.29 | Input, Pull-Up | Quadrature input |
| | **Quadrature Phase B** | P0.02 | Input, Pull-Up | Quadrature input |
| **Sensor (`SPI2`)** | **SPI Chip Select (`NCS`)** | P0.04 | Output | Active Low (GPIO toggled in software, idle High) |
| | **SPI Clock (`SCK`)** | P0.12 | Output | `NRF_SPI2->PSEL.SCK = 12` (`4 Mbps`, Mode 3: `CPOL=1, CPHA=1`) |
| | **SPI MOSI** | P0.08 | Output | `NRF_SPI2->PSEL.MOSI = 8` |
| | **SPI MISO** | P0.11 | Input, Pull-Up | `NRF_SPI2->PSEL.MISO = 11` |
| | **Motion Interrupt (`MOTION`)** | P0.06 | Input, Pull-Up | Active Low (`0` = motion data ready in burst register `0x16`) |
| **Power & Status** | **Sensor / Power Rail Enable 1** | P0.26 | Output | Sensor power enable; polarity unconfirmed |
| | **LED / Auxiliary Rail Enable 2** | P0.14 | Output | LED power enable; polarity unconfirmed |
| | **Charger Status Input** | P1.06 | Input, Pull-Up | Battery charger status indicator |
| | **Power / Mode Switch Input** | P1.04 | Input, Pull-Up | Power switch state sense |
| | **Battery Voltage Sense (`ADC`)** | Not mapped | — | No SAADC input identified yet (`P0.04`/`AIN2` is the sensor CSN) |
| **LEDs / RGB** | **LED / RGB Control Lines** | P0.13, P0.15, P0.17, P0.20, P0.22, P0.24 | Output | PWM / GPIO indicator and RGB LED outputs; polarity unconfirmed |

## 3. Optical Sensor (`PAW3370` / Glorious `BAMF`) SPI Details

- **SPI Peripheral**: `NRF_SPI2` (`0x40023000`)
- **Frequency**: `4 Mbps` (`FREQUENCY = 0x40000000`)
- **SPI Mode**: Mode 3 (`CONFIG = 0x06`: `CPOL = ActiveLow`, `CPHA = Trailing`, `MSB first`)
- **Key Registers Used by Stock Firmware**:
  - DPI (`0x00017860`–`0x00017960`): `0x4D` = `0x50` (50-DPI steps, 50–10,000) or `0xD0` (100-DPI steps, 10,100–19,000); `0x4E`/`0x4F` = step (`DPI/50 - 1` or `DPI/100 - 1`)
  - `0x5A` = `0x90` at `DPI >= 5,000`, else `0x10`; `0x56` = `0x84`/`0x04` (angle snapping on/off); `0x42` = `0x0C`/`0x0D` and `0x2B` = `0x0A`/`0x18` (LOD 1 mm / 2 mm)
  - Timing: read waits `20 us` after the address byte and `20 us` after CSN rises; write waits `180 us` after CSN rises; burst waits `5 us` after `0x16`
  - `0x16` (`Motion_Burst`): 12-byte burst read (`0x0001708C`) returning `Motion` (`byte[0]`, bit 7 = motion), `Observation` (`byte[1]`), `Delta_X_L/H` (`int16_t` LE at `bytes[2..3]`), `Delta_Y_L/H` (`int16_t` LE at `bytes[4..5]`), `SQUAL` (`byte[6]`), `Raw_Data_Sum` (`byte[7]`), `Maximum_Raw_Data` (`byte[8]`), `Minimum_Raw_Data` (`byte[9]`), `Shutter_Upper/Lower` (`bytes[10..11]`).

## 4. Flash Memory Map & Resident Compx USB HID Bootloader (`258a:2103`)

Unlike mice that require an external SWD debugger to install custom firmware, the Glorious Series One Pro includes a **resident 24 KB USB HID bootloader** at `0x00000000`–`0x00005FFF` (inferred: the official image starts at `0x00006000`) that performs **no cryptographic signature check** (a modified application image was flashed over USB and booted on a retail unit) and, as far as is known, never erases itself during USB updates:

| **Flash Address Range** | **Size** | **Region / Contents** |
| --- | --- | --- |
| `0x00000000` – `0x00005FFF` | `24 KB` | **Resident Compx USB HID Bootloader (`258a:2103`)** — Do not overwrite |
| `0x00006000` – `0x0007DFFF` | `480 KB` | **Application Firmware Partition** (Vector table at `0x00006000`; stock FW ends at `0x0001CC8F`). Over USB the bootloader only reads back `0x00006000`–`0x0004D7FF`. |
| `0x0007E000` – `0x0007FFFF` | `8 KB` | Two pages not used by the stock firmware; the draft ZMK board keeps its `storage` partition here (`Erase` does not clear them: data saved there survived two full USB flashes) |
| `0x00088000` – `0x00088FFF` | `4 KB` | **Factory 2.4 GHz Radio Pairing & Calibration Page** — Preserve to keep stock dongle pairing |
| `0x0008B000` – `0x000DFFFF` | `340 KB` | **Stock Onboard Profile & Macro Storage** |

Regions from `0x00088000` up are inferred from the stock firmware and cannot be read over USB.

### Bootloader Re-Entry Mechanism (`0x2003FFFC = 0xA55AA55A`)

1. **SRAM Magic Word**:
   - The stock application writes `0xA55AA55A` to the top word of SRAM (`0x2003FFFC`) and resets (`0x0001484C`, disassembly); the bootloader then enumerates as `258a:2103` (`Glorious bootloader`). That the bootloader checks this word at reset is inferred; doing it from custom firmware has not been tried yet.
   - Any custom ZMK/Zephyr firmware flashed via USB should:
     1. Write the magic word with interrupts locked and reset immediately, so nothing touches `0x2003FFFC` in between. (No RAM reservation is needed.)
     2. Check a hardware boot-chord (e.g., `LMB P1.15 + RMB P1.13 + DPI P1.00` held low at `PRE_KERNEL_1`) and write `0xA55AA55A` to `0x2003FFFC` before resetting. The chord only runs when the MCU comes out of reset: plugging in USB does not reset a running, battery-powered mouse, so disconnect the battery (or let the mouse power down fully) first.
2. **USB HID Bootloader Commands (`258a:2103`, 64-byte Feature Report `0x00` on Usage Page `0xFFFF`)**. Byte strings are the 64-byte payload `body[0..63]` (report ID not included); the 65-byte `HidD_SetFeature`/`HIDIOCSFEATURE` buffer has the report ID at `buf[0]`, so `buf[i] = body[i-1]`.
   - **Enter Bootloader from Stock App (`258a:2018`)**: Send `00 00 00 01 00 00 B0 ...` (`body[2]` = device ID, `0x00` for the wired mouse; `body[3]=0x01`, `body[6]=0xB0`; every other byte `0x00`). Read from `C52840Dongle::SetBLDeviceMode` in the official updater (`buf[3]`, `buf[4]`, `buf[7]`) and tested on a retail unit.
   - **`GetBLFW` (`0x80`)**: `00 00 00 06 B0 80 ...` -> returns `0xA1` and 4-byte bootloader version (`b0 01 00 00`) at `body[8..11]`.
   - **`Erase` (`0x01`)**: `00 00 00 08 B0 01 ...` -> on a retail unit it cleared `0x0001E000`–`0x0004D7FF` (the highest address USB can read back) but **kept `0x00006000`–`0x0001DFFF`**, including the old vector table. It does not reach `0x7E000` (data saved there survived two full flashes); whether it reaches the pairing page is unknown. Before programming, write `0xFF` over the vector table at `0x00006000`, and write the vector table last. The `0xFF` write protects against a half-written vector table only once it has landed: an interruption during or right after `Erase` leaves the old vector table with `0x00006000`–`0x0001DFFF` intact and the range `Erase` clears (from `0x0001E000`) erased, or only partly erased if `Erase` itself was cut short. That is still a complete image for the stock firmware (it ends at `0x0001CC8F`), but a broken one for an old image that reached `0x0001E000`, as a ZMK build may. Whether the bootloader boots it is unknown.
   - **`Program` (`0x02`)**: `00 00 00 (len+5) B0 02 len [addr_be32] [data ^ 0x55 ...]` -> writes up to 32 bytes (XORed with `0x55` on the wire) to `addr_be32` (keep `addr >= 0x00006000`; whether the bootloader refuses lower addresses is unknown). It is read-modify-write: the bytes land exactly as sent and the rest of the 4 KB page is kept, so no prior erase is needed.
   - **`VerifyByAddress` (`0x83`)**: `00 00 00 20 B0 83 len [addr_be32] ...` -> reads back `len` (`<= 32`) bytes from `addr_be32` (XORed with `0x55` on the wire) at `body[11..11+len-1]`. Works for `0x00006000`–`0x0004D7FF`; from `0x0004D800` up it answers `0xA2`, so the pairing page cannot be backed up over USB (only over SWD).
   - **`Exit` (`0x04`)**: `00 00 00 01 B0 04 01 ...` -> boots the application at `0x00006000` (any validation it performs is unknown).

Tools that implement this sequence ([`compx_usb_flasher.py`](https://github.com/0zLloen/mouse-firmware-re/tree/main/platforms/compx-nrf52/tools): a read-only `dump` backup, and a `flash` command with read-back verification that blanks the vector table first and writes it last, used on a retail unit to flash and boot a patched image and to restore the stock one), the full reverse-engineering notes, and a draft HWMv2 ZMK board (builds on ZMK main with the sensor node disabled; never flashed) are in [mouse-firmware-re](https://github.com/0zLloen/mouse-firmware-re/tree/main/devices/glorious-series-one-pro).
