Hardware notes: M5 Atom S3 Lite + FOSV Fuji-Atom-Interface
==========================================================

This fork adapts the firmware for the **M5 Atom S3 Lite (ESP32-S3)** paired with
the [FOSV Fuji-Atom-Interface](https://github.com/FOSV/Fuji-Atom-Interface) PCB
(MCP2025-330 LIN transceiver, powered from the air conditioner's 12 V).

The upstream project targets the original **Atom Lite (ESP32)**. The Atom S3 Lite
uses a different chip (ESP32-S3), so a prebuilt Atom-Lite `.bin` will **not** boot
on it — it must be built from source (see below).

Board / build
-------------
`platformio.ini`:
- `board = m5stack-atoms3` (was `m5stack-atom`)
- `build_flags = -DARDUINO_USB_MODE=1 -DARDUINO_USB_CDC_ON_BOOT=1`
  so `Serial` (the HomeSpan CLI / serial monitor) works over the S3's native USB.

Build & flash from the repo root:
```
pio run                 # compile
pio run -t upload       # flash over USB-C
pio device monitor      # serial console (115200)
```
The S3 enumerates as `/dev/cu.usbmodem*` (native USB). No NodeMCU PyFlasher needed.

Pin map
-------
Pins are hardcoded in `src/fuji-hk.cpp`. The FOSV PCB wires the transceiver to the
Atom's **G19 (RX) / G22 (TX)** header pins; on the Atom S3 Lite those physical
positions are different GPIOs. FOSV documents the translation
(`Atom G19 -> AtomS3 G6`, `Atom G22 -> AtomS3 G5`):

| Function      | Atom Lite | Atom S3 Lite | Note                                   |
|---------------|-----------|--------------|----------------------------------------|
| Status LED    | GPIO 27   | **GPIO 35**  | on-board WS2812C RGB LED               |
| Button        | GPIO 39   | **GPIO 41**  | front button                           |
| LIN RX        | GPIO 19   | **GPIO 6**   | FOSV: Atom G19 → AtomS3 G6             |
| LIN TX        | GPIO 22   | **GPIO 5**   | FOSV: Atom G22 → AtomS3 G5             |

Note: the old LIN RX (GPIO 19) is the ESP32-S3's native-USB D- pin, so it could
not be reused — hence the move to G5/G6.

Controller mode (secondary) and the wired wall controller
---------------------------------------------------------
The firmware binds as a **secondary** controller (`hp.connect(&Serial2, true, ...)`),
which requires an existing **wired wall controller acting as MASTER** on the 3-wire
LIN bus. On Fujitsu wired controllers the master/slave selection is a DIP switch
(e.g. UTY-RNNUM SW1-2: **Master = OFF, Slave = ON**) — the wall controller must be
Master; the adapter is the electronic slave. Consult the manual for your exact
controller model, as switch numbering/polarity varies.

Bring-up gotchas:
- If the wall controller shows **E:EE** (inter-controller comms error) after adding
  the adapter, it usually means a master/slave role conflict. Verify the wall
  controller is Master. Clear E:EE with the unit off by pressing Temp-Up + Temp-Down
  together for ~3 s.
- The master polls for a secondary only once at power-up, so power-cycle the AC
  after connecting the adapter so the unit re-runs controller discovery.

Fan speed mapping (Home Assistant)
----------------------------------
Home Assistant's HomeKit integration exposes any HeaterCooler fan as exactly
**three** speeds (Low / Medium / High) and writes RotationSpeed 33 / 66 / 100 %.
Fujitsu has five (Auto, Quiet, Low, Medium, High), so only three are selectable
from HA. This fork maps HA's three slots to **Quiet / Low / High**:

| HA setting | Fuji speed |
|------------|------------|
| Low        | Quiet (1)  |
| Medium     | Low (2)    |
| High       | High (4)   |

Auto and Medium remain settable from the physical wall remote. The mapping in
`src/fuji-hk.cpp` (`fanModeToRotation` / `rotationToFanMode`) is range-based, so it
tolerates whatever percentage HA sends. Adjust those two helpers to change which
three speeds are exposed. After reflashing, reload the HA HomeKit Controller
integration so the speed range refreshes.
