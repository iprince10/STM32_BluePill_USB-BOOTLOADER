# Blue Pill New-Project Cheat Sheet (detailed)

**STM32F103 · CubeMX + VS Code + CMake · HID bootloader**

---

## Contents

1. Prerequisites
2. Reading the placeholders
3. Hardware reminders
4. New project: CubeMX
5. Open in VS Code
6. Edits by board
7. `.vscode/tasks.json`
8. Build and check
9. Programming routes
10. Troubleshooting
11. Gotchas
12. Quick reference card
13. Glossary

---

## 1. Prerequisites

Set up once, assumed present from here on:

- VS Code with the **STM32CubeIDE** extension, and **STM32CubeMX**.
- **STM32CubeProgrammer**, for writing the bootloader or rescuing a board.
- `D:\USBboot\hid_flash\hid-flash.exe` and `hid_generic_pc13.bin` (low/medium density).
- The **HID bootloader is installed on the 128 KB board**.
- Use the ST-Link with serial `56FF6C064…` in CubeProgrammer. The `000000000001` one works only in VS Code.

---

## 2. Reading the placeholders

Anything in angle brackets is a **blank to fill in with your own value**, not text to type:

| Placeholder | Meaning | Example |
| --- | --- | --- |
| `<project>` | Your project / binary name | `BluePill_USB_Blink` |
| `<path>` | A real folder path | `C:\Users\Prince Jha\STM32_BluePill` |

PowerShell treats `<` and `>` as special characters, so pasting a placeholder verbatim gives:

```
Resolve-Path : Illegal characters in path.
```

That is not a broken command — it is an unfilled blank. Two fixes: replace the placeholder, or use the auto-detecting versions in section 8, which need no editing at all.

---

## 3. Hardware reminders

### ST-Link wiring

| ST-Link | Board |
| --- | --- |
| SWCLK | PA14 |
| SWDIO | PA13 |
| GND | GND |
| 3V3 | 3V3 |
| RST | **R** (recommended) |

- **Onboard LED:** PC13, **active-low** (low = on).
- **Pins to avoid:** PA13/PA14 (SWD), PB2 (BOOT1), PA11/PA12 (USB, on the bootloader board), PA9/PA10 (UART rescue). Check any new peripheral against this list before assigning it.
- **Jumpers:** the one **next to the reset button** enters the USB bootloader. The other stays at **0** always. They are read only at reset, so press **reset** after moving one.

| Action | Jumper (next to reset) | Other jumper | Connected |
| --- | --- | --- | --- |
| Run app | 0 | 0 | USB (power) |
| Flash over USB | 1 | 0 | USB |
| Debug with ST-Link | 0 | 0 | ST-Link |
| Rescue via CP2102 | 0 | 1 | CP2102 |

---

## 4. New project: CubeMX

1. STM32Cube panel → **Launch STM32CubeMX** → **Access to MCU Selector**.
2. MCU: **STM32F103CBTx** for the 128 KB board, **STM32F103C8Tx** for the 32 KB clone.
3. **RCC → HSE:** Crystal/Ceramic Resonator.
4. **SYS → Debug:** Serial Wire.
5. **Clock Configuration:** type **72** into HCLK and press Enter.
6. Set your pins (PC13 as GPIO_Output with label `LED`). **Don't enable the USB peripheral.**
7. **Project Manager:** name and location, toolchain **CMake**, then **Generate Code**.

---

## 5. Open in VS Code

**File → Open Folder** → accept *Configure as STM32Cube project* (or **STM32Cube panel → Setup STM32Cube project(s)**) → device → **Configure** → preset **Debug**.

---

## 6. Edits by board

### 32 KB clone (plain ST-Link)

In the `_FLASH.ld` file, set the real sizes (`FLASH` 32K, `RAM` 6K). Otherwise the stack points outside real RAM and you get a HardFault right at start.

### 128 KB board with the USB bootloader

```c
/* _FLASH.ld */
MEMORY
{
  RAM    (xrw) : ORIGIN = 0x20000000, LENGTH = 20K
  FLASH  (rx)  : ORIGIN = 0x08000800, LENGTH = 126K
}
```

```c
/* main.c */
/* USER CODE BEGIN 1 */
SCB->VTOR = FLASH_BASE | 0x800U;
/* USER CODE END 1 */
...
/* USER CODE BEGIN 2 */
__enable_irq();
/* USER CODE END 2 */
```

```cmake
# end of the top-level CMakeLists.txt
add_custom_command(TARGET ${CMAKE_PROJECT_NAME} POST_BUILD
    COMMAND ${CMAKE_OBJCOPY} -O binary $<TARGET_FILE:${CMAKE_PROJECT_NAME}> ${CMAKE_PROJECT_NAME}.bin
    COMMENT "Creating .bin")
```

---

## 7. `.vscode/tasks.json`

Create the folder `.vscode` in the project root and add `tasks.json` with the contents below. The file name comes from `${workspaceFolderBasename}`, so it fills itself in and nothing needs typing.

```json
{
  "version": "2.0.0",
  "tasks": [
    {
      "label": "Build",
      "type": "shell",
      "command": "cmake",
      "args": [
        "--build",
        "${workspaceFolder}/build/Debug"
      ],
      "group": {
        "kind": "build",
        "isDefault": true
      },
      "problemMatcher": [
        "$gcc"
      ]
    },
    {
      "label": "Flash over USB",
      "type": "shell",
      "command": "D:\\USBboot\\hid_flash\\hid-flash.exe",
      "args": [
        "${workspaceFolder}\\build\\Debug\\${workspaceFolderBasename}.bin",
        "COM99"
      ],
      "dependsOn": "Build",
      "problemMatcher": []
    },
    {
      "label": "Check vector table",
      "type": "shell",
      "command": "powershell",
      "args": [
        "-NoProfile",
        "-Command",
        "$f = (Get-ChildItem '${workspaceFolder}/build/Debug/*.bin' | Select-Object -First 1).FullName; $b = [System.IO.File]::ReadAllBytes($f); '{0:X8}  {1:X8}' -f [BitConverter]::ToUInt32($b,0), [BitConverter]::ToUInt32($b,4)"
      ],
      "dependsOn": "Build",
      "problemMatcher": []
    }
  ]
}
```

Notes:

- **Build** is the default build task, so `Ctrl+Shift+B` runs it.
- **Flash over USB** builds first, then invokes `hid-flash.exe` with the dummy `COM99` this version requires. Put the board in bootloader mode (jumper next to reset = 1, press reset) before running it.
- **Check vector table** is the sanity check from section 8, wired in as a task so it needs no editing.
- If your binary or build folder is named differently, adjust the paths in the `args` arrays only.

---

## 8. Build and check (bootloader board)

Build, then in the terminal:

```powershell
$b = [System.IO.File]::ReadAllBytes((Resolve-Path build\Debug\<project>.bin).Path)
"{0:X8}  {1:X8}" -f [BitConverter]::ToUInt32($b,0), [BitConverter]::ToUInt32($b,4)
```

With the placeholder filled in, that becomes:

```powershell
$b = [System.IO.File]::ReadAllBytes((Resolve-Path build\Debug\BluePill_USB_Blink.bin).Path)
```

### Version that needs no editing

This one picks up whichever `.bin` is in `build\Debug`, so it works in any project:

```powershell
$f = (Get-ChildItem build\Debug\*.bin | Select-Object -First 1).FullName
$b = [System.IO.File]::ReadAllBytes($f)
"{0:X8}  {1:X8}" -f [BitConverter]::ToUInt32($b,0), [BitConverter]::ToUInt32($b,4)
```

Run it from the project folder. Expect the first value to be `20005000` and the second something like `0800xxxx`, at least `08000800`, ending in an odd digit.

### Reading the output

The check reads the first two 32-bit words of the image, which are the first two entries of the ARM vector table:

| Word | Offset | Meaning | Expected |
| --- | --- | --- | --- |
| 1 | 0x00 | Initial main stack pointer | `20005000` |
| 2 | 0x04 | Reset handler address | `0800xxxx`, ≥ `08000800`, odd |

---

## 9. Programming routes

### ST-Link (debug)

Jumpers 0/0, wire the ST-Link, press `F5`. **Never add** `--erase-all` **on the bootloader board.**

### USB (no ST-Link)

1. Move the jumper next to reset to **1**, then press **reset**. The LED blinks fast.
2. Run the **Flash over USB** task, or run the command manually:

```powershell
& "D:\USBboot\hid_flash\hid-flash.exe" "C:\Users\Prince Jha\STM32_BluePill\BluePill_USB_Blink\build\Debug\BluePill_USB_Blink.bin" COM99
```

The `COM99` is a dummy that this version needs. The VS Code task uses `${workspaceFolderBasename}` for the file name, so it fills itself in and you don't type anything.

3. Move the jumper back to **0** and press **reset**.

### CP2102 rescue

CP2102 TXD to PA10, RXD to PA9, GND, 5V. BOOT0 = 1, press reset. In CubeProgrammer switch to **UART** (Even parity, 115200). Set BOOT0 back to 0 afterward.

### Reinstall the bootloader

With the `56FF…` ST-Link, run **Full chip erase**, then write `hid_generic_pc13.bin` at `0x08000000` with **Verify programming**.

---

## 10. Troubleshooting

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| `Illegal characters in path` | Unfilled `<placeholder>` | Replace it, or use the auto-detecting command |
| `Cannot find path … because it does not exist` | Wrong file name or build not run | Check `build\Debug\*.bin` exists; build first |
| HardFault immediately at start | `.ld` sizes don't match real silicon | Correct FLASH/RAM sizes for the board |
| Nothing runs after USB flash | VTOR not relocated, or jumper left at 1 | Set `SCB->VTOR`, jumpers back to 0/0, reset |
| Vector table check shows `08000000` | Image linked for the wrong flash origin | `FLASH ORIGIN = 0x08000800` for bootloader board |
| Second word ends in an even digit | Not a valid Thumb address / bad image | Rebuild; something corrupted the entry point |
| Upload appears to "malfunction" in Windows | Expected after HID flash | Harmless — confirm via the LED pattern |
| Board unresponsive after a bad flash | App overwrote the bootloader | Reinstall bootloader, or use CP2102 rescue |

---

## 11. Gotchas

- After CubeMX regenerates code, re-check the `.ld` file. Keep your own code inside the `USER CODE` markers.
- "Malfunctioned" in Windows after an upload is harmless. The LED tells you the state: fast blink is the bootloader, your own pattern is the app.
- Check any new peripheral pins against the avoid list.
- Copy a working project folder as a template and just rename it.

---

## 12. Quick reference card

```text
Open project       File > Open Folder, preset Debug
Build              Ctrl+Shift+B
Flash over USB     1) jumper next-to-reset = 1, press reset  2) run task  3) jumper = 0, reset
Sanity check       expect 20005000 + 0800xxxx (odd last digit)
Rescue via UART    BOOT0 = 1, UART Even/115200 in CubeProgrammer, then BOOT0 = 0
Never              --erase-all on the bootloader board
```

---

## 13. Glossary

- **HID bootloader** — A small program that lives at the bottom of flash and talks the USB HID protocol, so a PC can push a new application image over plain USB without an ST-Link.
- **Flash offset 0x08000800** — The bootloader reserves the first 2 KB, so the application must be linked to start 2 KB higher.
- **Vector table** — The array at the start of the image: entry 0 is the initial stack pointer, entry 1 is the reset handler. Its two first words are what the sanity check reads.
- **VTOR** — The CPU register telling the core where the vector table lives. When the application is not at `0x08000000`, the startup code must point VTOR at the application's own table.
- **`.bin` vs `.elf`** — `.elf` carries debug and symbol info for the debugger; `.bin` is the raw image the bootloader writes.
- **Linker script (`_FLASH.ld`)** — Tells the linker which memory regions exist and where code and data go; if it disagrees with real hardware, the stack ends up outside RAM.
- **Thumb bit** — The lowest bit of an address on Cortex-M is always 1 for code; a reset handler address that ends in an even digit is a red flag.
- **POST_BUILD** — A custom build step that produces something extra (`objcopy` to `.bin`) after the linked `.elf` is ready.
