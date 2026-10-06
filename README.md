# DeskPet-Doom — BLE controller + ESP32 Doom setup guide

This repo hosts the **phone controller** for Doom running on the ESP32 Cheap Yellow
Display (CYD). The page deploys as `index.html` from this repo via GitHub Pages:

- Controller: `https://ignatiusvanniekerk.github.io/doom-remote/`
- Firmware: [github.com/HenrysCat/cyd-doom](https://github.com/HenrysCat/cyd-doom) (a GBADoom/doomhack fork, itself based on PrBoom)

> The controller needs **no changes to the game**. The ESP32 side only adds a
> Bluetooth peripheral that feeds ordinary key events into Doom's existing input
> queue, right next to the touchscreen and serial input.

---

## 1. What you need

| Item | Detail |
|---|---|
| Board | ESP32-2432S028R "Cheap Yellow Display" (CYD) |
| Chip | ESP32-D0WD-V3, 240 MHz, **no PSRAM**, 4 MB flash |
| Toolchain | [PlatformIO Core](https://platformio.org/) |
| Platform | `espressif32@6.9.0`, framework `arduino`, board `esp32dev` |
| Firmware | Clone of `HenrysCat/cyd-doom` |
| Controller | This repo, served over HTTPS (Web Bluetooth requires it) |

Please note the no-PSRAM constraint: everything runs out of ~320 KB of DRAM, and
after BLE + the Doom zone allocator there is about **14 KB of heap** left. The
whole setup below is shaped by that budget.

---

## 2. Build the firmware

```
# in the cyd-doom repo root
pio run
pio run -t upload                   # flashes the app (default port)
pio run -t upload --upload-port COM3  # if your board is on COM3
python tools/flashwad.py             # flashes the WAD into its own partition
```

`platformio.ini` essentials (already set up in the working fork):

```
[env:cyd]
platform = espressif32@6.9.0
board = esp32dev
framework = arduino
board_build.partitions = partitions.csv
board_build.f_cpu = 240000000L
board_build.f_flash = 80000000L
board_build.flash_mode = dio
monitor_speed = 115200
build_flags = -O2 -I include
lib_deps = h2zero/NimBLE-Arduino@^2.3.2
```

Partition layout (`partitions.csv`) used to fit everything in 4 MB:

```
nvs,      data, nvs,     0x9000,   0x5000,
phy_init, data, phy,     0xe000,   0x1000,
factory,  app,  factory, 0x10000,  0x100000,   ; app + all the game assets
sram,     data, 0x40,    0x110000, 0x8000,     ; high-score save
wad,      data, 0x41,    0x118000, 0x2e8000,   ; main WAD (3,042,648 bytes)
```

---

## 3. BLE controller protocol

Service `d00d0001-0000-4000-8000-000000000000`
Characteristic `d00d0002-0000-4000-8000-000000000000` (write, 1 byte mask):

| Bit | Hex | Action |
|---|---|---|
| 0 | 0x01 | ▲ forward |
| 1 | 0x02 | ▼ back |
| 2 | 0x04 | ◀ turn left |
| 3 | 0x08 | ▶ turn right |
| 4 | 0x10 | FIRE |
| 5 | 0x20 | USE |
| 6 | 0x40 | MAP (automap) |
| 7 | 0x80 | MUTE (momentary — firmware toggles on the **rising edge**) |

Device advertises as **`DeskPet-Doom`**.

iPhone/iPad: pair `DeskPet-Doom` once in Settings → Bluetooth first (iOS only
lets the browser use already-paired devices), then Connect from the page.

---

## 4. What had to change in the firmware

All of it lives in `src/port/`; `src/doom/*` (the actual game) is untouched.

### 4.1 BLE peripheral — `src/port/ble.cpp` (new), `ble.h`
- NimBLE peripheral advertising `DeskPet-Doom`.
- **Name discovery gotcha:** NimBLE-Arduino 2.x only puts the name in the
  advertisement if you call `advertising->setName(name)`. The name goes in the
  **scan response** (it can't fit beside the 128-bit UUID in the 31-byte ADV).
  Without `setName()`, the phone can't find/filter the device.
- One write characteristic holds the 1-byte mask.

### 4.2 Feeding the game — `src/port/input.cpp`
`blePoll()` diffs the mask each Doom tic and calls `postKey(ev_keydown|ev_keyup,
KEYD_*)` — the same event pipeline the touchscreen/serial already use. Triggered
from `InputPoll()`, running in the Doom task.

### 4.3 Init order — `src/port/main.cpp`
`BleInit()` is **not** in Arduino `setup()`. It runs inside the doom task after
`I_PreInitGraphics()`:

```
Z_Init             // zone allocator grabs the big chunk first  (~110 KB)
I_PreInitGraphics
BleInit            // BLE stack (~53 KB static + ~12 KB runtime)
D_DoomMain
```

### 4.4 RAM budget (measured, serial boot log)

```
free=172,752 largest=110,580   <- after NimBLE's static BSS
Z_Init: Heapsize is 110,580
after zone:    free=62,056  largest=40,948
after graphics:free=26,052  largest=11,252
runtime heap:  ~14 KB        (game + BLE)
```

That works because:
1. **Zone first.** `Z_Init` must grab its chunk before anything else fragments the
   heap. A graphics-first reorder shrinks the zone to ~73 KB and Doom dies at the
   difficulty screen with
   `I_Error: Z_Malloc: failed on allocation of 17128 bytes`.
2. **Framebuffer before display init.** The 38,400-byte framebuffer is `malloc`'d
   in `I_PreInitGraphics` *before* `DisplayInit()` claims its DMA line chunks.
3. **Smaller DMA chunks.** `ROWS_PER_CHUNK 8 -> 4` cuts each chunk 10,240 → 5,120
   bytes so the framebuffer + chunks fit the 40,948 largest block.
4. **NimBLE trims** in `platformio.ini` (all guarded by `#ifndef` in NimBLE):

   ```
   CONFIG_BT_NIMBLE_MAX_CONNECTIONS=1
   CONFIG_BT_NIMBLE_MAX_BONDS=1
   CONFIG_BT_NIMBLE_MAX_CCCDS=2
   CONFIG_BT_NIMBLE_GATT_MAX_PROCS=2
   CONFIG_BT_NIMBLE_WHITELIST_SIZE=4
   CONFIG_BT_NIMBLE_TRANSPORT_ACL_FROM_LL_COUNT=4
   CONFIG_BT_NIMBLE_TRANSPORT_EVT_COUNT=10
   CONFIG_BT_NIMBLE_TRANSPORT_EVT_DISCARD_COUNT=4
   CONFIG_BT_NIMBLE_MSYS1_BLOCK_COUNT=6
   CONFIG_BT_NIMBLE_ATT_PREFERRED_MTU=23
   CONFIG_BT_NIMBLE_HOST_TASK_STACK_SIZE=3072
   ```

5. **One trap:** `TRANSPORT_EVT_COUNT`, `TRANSPORT_EVT_DISCARD_COUNT` and
   `GATT_MAX_PROCS` are *unconditional* `#define`s in
   `.pio/libdeps/cyd/NimBLE-Arduino/src/nimconfig.h` (no `#ifndef`), so `-D`
   cannot override them. Patch the header directly:
   `EVT_COUNT 30→10`, `EVT_DISCARD_COUNT 8→4`, `GATT_MAX_PROCS 4→2`.
   (Lib re-installs wipe it, but there's enough headroom that a vanilla build
   also fits at 53 KB BSS — it's just tighter.)

### 4.5 Audio / MUTE — `src/port/audio.cpp` (new), `audio.h`
- SFX synthesized through LEDC (ch0, GPIO26), byte-fed from the game mixer.
- An inaudible **20 kHz idle carrier at low duty** keeps the CYD speaker's amp
  watchdog engaged so alert tones still sound.
- `AudioSetMute(bool)`: while muted, SFX are dropped but the idle carrier keeps
  running, so unmute is instant. The MUTE button is bit 7, edge-detected in
  `ble.cpp` (momentary pulse from the phone, ~120 ms).

---

## 5. Inspecting it yourself

Reset the board and capture the boot log:

```python
import serial, time
s = serial.Serial('COM3', 115200, timeout=0.5)
s.setDTR(False); s.setRTS(True); time.sleep(0.15); s.setRTS(False); s.close()
```

Then read COM3 at 115200. Key lines to look for: `[ble] advertising as
DeskPet-Doom`, `Z_Init: Heapsize is 110580`, `[stat] fps=... heap_free=...`,
`[ble] mask=0x20`, `[audio] mute ON/OFF`.

---

## 6. Fast path: hand all of this to an LLM

Want to reproduce this without reading the details above? Paste this into a
coding agent (Claude Code, opencode, etc.) — it encodes every pitfall found
while getting this working:

> You are building a wireless controller for Doom on an ESP32 "Cheap Yellow
> Display" (ESP32-2432S028R, ESP32-D0WD, **no PSRAM**, 4 MB flash) using
> PlatformIO (`espressif32@6.9.0`, arduino, `h2zero/NimBLE-Arduino`). The
> game already boots and plays via touchscreen; add a Bluetooth peripheral
> (service `d00d0001-0000-4000-8000-000000000000`, write characteristic
> `d00d0002-...`) whose 1-byte value is a button mask: bit0 ▲, bit1 ▼, bit2 ◀,
> bit3 ▶, bit4 FIRE, bit5 USE, bit6 MAP, bit7 MUTE (toggle on rising edge only).
> Advertise the name `DeskPet-Doom` and NOTE: NimBLE-Arduino 2.x will not put
> the name in the scan response unless you call `advertising->setName(name)`,
> so call it or phones can't find the device.
>
> The chip has no PSRAM and BLE costs ~53 KB static RAM + ~12 KB heap, so the
> whole design must fit a ~110 KB zone plus ~14 KB runtime heap. Constraint
> list, learned the hard way:
> - Initialize audio (LEDC on GPIO26) and audio mute gating; keep an inaudible
>   20 kHz idle carrier at low duty so the speaker amp watchdog stays open, and
>   do not stop it on mute.
> - In the Doom task, keep this exact order: `Z_Init` FIRST (it must claim its
>   ~110 KB zone before anything fragments the heap — a graphics-first order
>   shrinks the zone to ~73 KB and the game dies at difficulty select with
>   `Z_Malloc: failed on allocation of 17128 bytes`), then graphics init with
>   the 38,400-byte framebuffer `malloc`'d *before* `DisplayInit()` claims its
>   DMA chunks, then `BleInit()`. Never call BleInit from Arduino `setup()`.
> - Keep DMA display chunks ≤5,120 bytes (`ROWS_PER_CHUNK=4`) so framebuffer +
>   chunks fit the largest free block (~41 KB).
> - Trim NimBLE: set MAX_CONNECTIONS=1, MAX_BONDS=1, MAX_CCCDS=2,
>   GATT_MAX_PROCS=2, WHITELIST_SIZE=4, TRANSPORT_ACL_FROM_LL_COUNT=4,
>   TRANSPORT_EVT_COUNT=10, TRANSPORT_EVT_DISCARD_COUNT=4, MSYS1_BLOCK_COUNT=6,
>   ATT_PREFERRED_MTU=23, HOST_TASK_STACK_SIZE=3072. `EVT_COUNT`,
>   `EVT_DISCARD_COUNT` and `GATT_MAX_PROCS` are unguarded `#define`s in
>   NimBLE's `nimconfig.h`, so `-D` won't override them — patch that header.
> - The game engine (`src/doom/*`) must NOT be modified. Map the button mask to
>   key-down/key-up events and post them into the port's existing input queue
>   (beside touch/serial) every game tick.
> - Result to verify in the boot log: `Z_Init: Heapsize is 110580`,
>   `[ble] advertising as DeskPet-Doom`, `[stat] fps≈28 heap_free≈14000`, and
>   button presses log `[ble] mask=0x..`.

---

## 7. Layout of this repo

```
index.html            <- the controller (diamond D-pad left, FIRE/USE/MAP/MUTE right)
.github/workflows/
  static.yml          <- deploys the repo root to GitHub Pages on push to master
README.md             <- this file
```

The workflow generates nothing: `index.html` sits at the repo root and Pages
serves it directly, so the final URL is `https://<user>.github.io/doom-remote/`.