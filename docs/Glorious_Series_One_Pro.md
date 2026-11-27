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
- **Wired Mouse Resident USB HID Bootloader**: `258a:2103` (`USB bcdDevice 2.00`, product string `Glorious bootloader`, single interface 0 with the same 20-byte vendor Feature report descriptor; verified on a retail unit).
- **Wireless Receiver Resident USB HID Bootloader**: `258a:2113` (wireless `Config.ini` `BLPID = 2113`, same `C52840Dongle` updater class; not seen on USB yet).

## 2. nRF52840 GPIO Pinout Mapping (`CM2852-BAMF`, from Firmware Disassembly)

The pin assignments below are inferred from disassembly of the official `CM2856_Mouse_USB_v1.00.00_20220810` firmware (`0x0000C5B8`–`0x0000C648` GPIO init, `0x00016B30`–`0x00016CE0` and `0x00009614`–`0x00009690` button and wheel scanner, `0x000184C8`–`0x0001855C` SPI2 sensor setup, `0x0000E088` battery SAADC, `0x0000D91C` PWM LEDs). None has been continuity-probed on the PCB; the three boot-chord pins (`P1.15`, `P1.13`, `P1.00`) and the bootloader's own chord pins (`P0.09`, `P0.10`, `P1.00`) are verified active-low on a retail unit, and button indices 2–4 (Middle/Back/Forward) and LED RGB channel order are verified on hardware. No ZMK image has been flashed yet.

| **Subsystem** | **Function** | **nRF52840 Pin** | **Direction / Pull** | **Active State & Notes** |
| --- | --- | --- | --- | --- |
| **Switches** | **Left Button (LMB)** | P1.15 | Input, Pull-Up | Active Low (`0` = pressed; verified low in boot chord) |
| | **Right Button (RMB)** | P1.13 | Input, Pull-Up | Active Low (`0` = pressed; verified low in boot chord) |
| | **Middle Button (MMB)** | P1.10 | Input, Pull-Up | Active Low (`0` = pressed) |
| | **Back Button (Side Btn 4)** | P0.09 (`NFC1`) | Input, Pull-Up | Active Low (`0` = pressed; verified low in bootloader chord); requires `nfct-pins-as-gpios;` on `&uicr` (Zephyr 4.x / ZMK main; older trees: `CONFIG_NFCT_PINS_AS_GPIOS=y`) |
| | **Forward Button (Side Btn 5)** | P0.10 (`NFC2`) | Input, Pull-Up | Active Low (`0` = pressed; verified low in bootloader chord); requires `nfct-pins-as-gpios;` on `&uicr` (Zephyr 4.x / ZMK main; older trees: `CONFIG_NFCT_PINS_AS_GPIOS=y`) |
| | **DPI Cycle Button** | P1.00 | Input, Pull-Up | Active Low (`0` = pressed; verified low in both chords) |
| **Scroll Encoder** | **Quadrature Phase A** | P0.29 | Input, Pull-Up | Quadrature input |
| | **Quadrature Phase B** | P0.02 | Input, Pull-Up | Quadrature input |
| **Sensor (`SPI2`)** | **SPI Chip Select (`NCS`)** | P0.04 | Output | Active Low (GPIO toggled in software, idle High) |
| | **SPI Clock (`SCK`)** | P0.12 | Output | `NRF_SPI2->PSEL.SCK = 12` (`4 Mbps`, Mode 3: `CPOL=1, CPHA=1`) |
| | **SPI MOSI** | P0.08 | Output | `NRF_SPI2->PSEL.MOSI = 8` |
| | **SPI MISO** | P0.11 | Input, Pull-Up | `NRF_SPI2->PSEL.MISO = 11` |
| | **Motion Interrupt (`MOTION`)** | P0.06 | Input, Pull-Up | Active Low (`0` = motion data ready in burst register `0x16`) |
| **Power & Status** | **Unknown Output 1** | P0.26 | Output | Driven high once at init (`0xC61A`–`0xC632`), never changed again; role unconfirmed |
| | **Unknown Output 2** | P0.14 | Output | Driven low once at init (`0xC636`–`0xC64A`), untouched by LED or sleep code; role unconfirmed |
| | **Charger Status Input** | P1.06 | Input, Pull-Up | Only charging input (`Source\battery.c`, `0xE1C4`); **Low = charging**, debounced ~2.2 s |
| | **Unknown Input** | P1.04 | Input, Pull-Up | Configured at init (`0xC60A`), never read in stock firmware; role unconfirmed |
| | **Battery Voltage Sense (`ADC`)** | P0.31 (`AIN7`) | Analog Input | `SAADC` ch 0 single-ended (`0xE09C`), 10-bit, gain `1/4`, ref `VDD/4` (full scale = `VDD`), 200 ms period, 16-sample average; divider ratio and `VDD` unmeasured |
| **LEDs / RGB** | **LED 0 (Status, `PWM0` ch 0/1/2)** | P0.13, P0.15, P0.17 | PWM Output | **R / G / B**, active-high, ~490 Hz (`PRESCALER=7`, `COUNTERTOP=255`) |
| | **LED 1 (DPI LED next to sensor, `PWM1` ch 0/1/2)** | P0.20, P0.22, P0.24 | PWM Output | **R / G / B** (colours & location verified on hardware), active-high, ~490 Hz |

## 3. Optical Sensor (`PAW3370` / Glorious `BAMF`) SPI Details

- **SPI Peripheral**: `NRF_SPI2` (`0x40023000`)
- **Frequency**: `4 Mbps` (`FREQUENCY = 0x40000000`)
- **SPI Mode**: Mode 3 (`CONFIG = 0x06`: `CPOL = ActiveLow`, `CPHA = Trailing`, `MSB first`)
- **Key Registers Used by Stock Firmware**:
  - DPI (`0x00017860`–`0x00017960`): `0x4D` = `0x50` (50-DPI steps, 50–10,000) or `0xD0` (100-DPI steps, 10,100–19,000); writes `uxtb(DPI/step - 1)` to **`0x4E` only** (never writes `0x4F` outside the one-time power-up table)
  - `0x5A` = `0x90` at `DPI >= 5,000`, else `0x10`; `0x56` = `0x84`/`0x04` (angle snapping on/off); `0x42` = `0x0C`/`0x0D` and `0x2B` = `0x0A`/`0x18` (LOD 1 mm / 2 mm)
  - Timing: read waits `20 us` after the address byte and `20 us` after CSN rises; write waits `180 us` after CSN rises; burst waits `5 us` after `0x16`
  - `0x16` (`Motion_Burst`): 12-byte burst read (`0x0001708C`) returning `Motion` (`byte[0]`, bit 7 = motion, bit 3 = lifted — verified on hardware), `Observation` (`byte[1]`), `Delta_X_L/H` (`int16_t` LE at `bytes[2..3]`), `Delta_Y_L/H` (`int16_t` LE at `bytes[4..5]`), `SQUAL` (`byte[6]`), `Raw_Data_Sum` (`byte[7]`), `Maximum_Raw_Data` (`byte[8]`), `Minimum_Raw_Data` (`byte[9]`), `Shutter_Upper/Lower` (`bytes[10..11]`).

## 4. Flash Memory Map & Resident Compx USB HID Bootloader (`258a:2103`)

Unlike mice that require an external SWD debugger to install custom firmware, the Glorious Series One Pro includes a **resident 24 KB USB HID bootloader** at `0x00000000`–`0x00005FFF` (read from a retail unit and disassembled on 2026-11-19: code up to `0x4FFF`, flag page at `0x5000`) that performs **no cryptographic signature check** (a modified application image was flashed over USB and booted on a retail unit) and never writes below `0x4D800` except when copying a staged image to `0x6000` at `Exit`.

> **Important (`APPROTECT` and `0x36800` image-size limit):**
> - `UICR.APPROTECT` reads **`0xFFFFFF00` (enabled)** on a retail unit, so SWD cannot read flash and unlocking it (`ERASEALL`) permanently destroys the 24 KB bootloader. Use the USB HID bootloader instead.
> - **Custom images MUST end at or below `0x00036800` (`194 KiB` max).** `Program` and `VerifyByAddress` add `0x47800` to every address, staging the image in a **second copy at `0x4D800`–`0x94FFF`** before `Exit` copies it to `0x6000`. That second copy overlaps the settings pages: image bytes from `0x36800` up overwrite `0x7E000` (ZMK/custom `storage`), from `0x43800` up overwrite `0x8B000` (stock profiles), and from `0x44800` up overwrite `0x8C000` (macro directory).

| **Flash Address Range** | **Size** | **Region / Contents** |
| --- | --- | --- |
| `0x00000000` – `0x00005FFF` | `24 KB` | **Resident Compx USB HID Bootloader (`258a:2103`)** — code `0x0000`–`0x4FFF`, flag page `0x5000`; never written by USB updates |
| `0x00006000` – `0x000367FF` | `194 KB` | **Safe Custom Application Partition** (vector table at `0x00006000`; stock FW occupies `0x6000`–`0x1CC8F`). Ending at or below `0x36800` keeps the second copy (`+0x47800`) below `0x7E000`. |
| `0x0004D800` – `0x00094FFF` | `286 KB` | **Bootloader Second Copy** — `Program` / `VerifyByAddress` act at `addr + 0x47800` (`0x6000`–`0x4D7FF` maps here); `Exit` copies the touched span to `0x6000`. Overlaps the settings pages below. |
| `0x0007E000` – `0x0007FFFF` | `8 KB` | Unused by stock firmware; custom firmware / draft ZMK board keeps its `storage` partition here (preserved as long as flashed images end ≤ `0x36800`) |
| `0x00088000` – `0x00088FFF` | `4 KB` | **Blank on retail unit** (earlier notes called this a pairing page; the radio link actually uses the chip's `FICR DEVICEID[0]`, not flash) |
| `0x0008B000` – `0x0008CFFF` | `8 KB` | **Stock Onboard Profile Config (`0x8B000`, all 3 profiles) & Macro Directory (`0x8C000`)** |
| `0x00090000` – `0x000DFFFF` | `320 KB` | **Stock Macro Data Pages** (blank on retail unit) |

### Bootloader Re-Entry & Recovery

1. **Built-in Bootloader Recovery Chord (Verified on hardware)**:
   - The resident bootloader itself samples **`P0.09` (Back) + `P0.10` (Forward) + `P1.00` (DPI)** for ~640 ms at power-up (`0x24E8`). With the power switch off and USB unplugged, hold **Back + Forward + DPI** while plugging in USB to enter `258a:2103`. This works regardless of what application is flashed (plugging USB in does not reset an already-running mouse; unplug with the switch off first).
2. **SRAM Magic Word (`0x2003FFFC = 0xA55AA55A`)**:
   - After a **soft reset** (`POWER.RESETREAS` soft-reset bit set), the bootloader checks `*(uint32_t *)0x2003FFFC == 0xA55AA55A`, clears it, and stays in `258a:2103`.
   - Custom ZMK/Zephyr firmware should still expose a runtime combo (e.g. `&compx_bl`) and/or `PRE_KERNEL_1` boot chord that locks interrupts, writes `0xA55AA55A` to `0x2003FFFC`, and calls `NVIC_SystemReset()` (no RAM reservation needed).
3. **USB HID Bootloader Commands (`258a:2103`, 64-byte Feature Report `0x00` on Usage Page `0xFFFF`)**. Byte strings are the 64-byte payload `body[0..63]` (report ID not included); the 65-byte `HidD_SetFeature`/`HIDIOCSFEATURE` buffer has the report ID at `buf[0]`, so `buf[i] = body[i-1]`.
   - **Enter Bootloader from Stock App (`258a:2018`)**: Send `00 00 00 01 00 00 B0 ...` (`body[2]=0x00`, `body[3]=0x01`, `body[6]=0xB0`; rest `0x00`). Verified on a retail unit.
   - **`GetBLFW` (`0x80`)**: `00 00 00 06 B0 80 ...` -> returns `0xA1`, app PID `20 18` at `body[6..7]`, and 4-byte bootloader version (`b0 01 00 00`) at `body[8..11]`.
   - **`Erase` (`0x01`)**: `00 00 00 08 B0 01 ...` -> **changes no flash** on this bootloader; it only drops the RAM page buffer.
   - **`Program` (`0x02`)**: `00 00 00 (len+5) B0 02 len [addr_be32] [data ^ 0x55 ...]` -> accepts `0x6000 <= addr < 0x4D800` (else `0xA2`), XORs payload with `0x55`, and writes through a 4 KB read-modify-write page buffer to **`addr + 0x47800`**. The running application at `0x6000` is untouched until `Exit`. Keep images `<= 0x36800` to preserve `0x7E000`+.
   - **`VerifyByAddress` (`0x83`)**: `00 00 00 20 B0 83 len [addr_be32] ...` -> flushes the pending page buffer and reads `len` (`<= 32`) bytes from **`addr + 0x47800`** (XOR `0x55` on the wire) at `body[11..11+len-1]`. Refuses (`0xA2`) below `0x6000` and from `0x4D800` up.
   - **`Exit` (`0x04`)**: `00 00 00 01 B0 04 01 ...` -> flushes the page buffer, marks flag page `0x5000` copy-pending, **copies the touched span from `0x4D800` → `0x6000`**, marks `0x5000` valid, and resets. Checks nothing about the image. Because of the two-copy design, an interrupted `Program` leaves the old application intact, and an interrupted `Exit` copy is resumed automatically on the next power-up.

### 2.4 GHz Stock Receiver Note (`258a:2031`)
The stock 2.4 GHz link is not Nordic ESB (it uses raw `NRF_RADIO` `MODE = 4` `Ble_2Mbit`, 8-bit preamble, 8-bit length, whitening seed `0x40`, address `FICR DEVICEID[0]` + prefix `0xB1`, fixed 4×4 hop table). Because pairing is tied to `FICR DEVICEID[0]` rather than flash, open firmware implementing the raw radio framing can talk to the unchanged stock receiver without re-pairing; see [`radio-protocol.md`](https://github.com/0zLloen/mouse-firmware-re/blob/main/platforms/compx-nrf52/radio-protocol.md).

Tools that implement the USB bootloader protocol ([`compx_usb_flasher.py`](https://github.com/0zLloen/mouse-firmware-re/tree/main/platforms/compx-nrf52/tools), which enforces the `0x36800` image cap), the full reverse-engineering notes, and a draft HWMv2 ZMK board (builds on ZMK main with the sensor node disabled; never flashed) are in [mouse-firmware-re](https://github.com/0zLloen/mouse-firmware-re/tree/main/devices/glorious-series-one-pro).
