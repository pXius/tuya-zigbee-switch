# TS0041 TB3 — _TZ3000_22ugzkme 1-Button Zigbee Remote

> **Config:** `22ugzkme;TS0041-TB3;SC2d;ID7i;BTC4;M;`
> **Z2M model:** `TS0041` (external converter required)
> **Status:** Working — 3 devices confirmed

![Product](https://ae01.alicdn.com/kf/Sc2b2a91ab4824cb9975ec12a8c6986f89.jpg)
*[AliExpress listing](https://www.aliexpress.com/item/1005005401760152.html)*

---

## Hardware

| Property | Value |
|----------|-------|
| **Chip** | TLSR8258F1KAT32 (1MB flash, QFN32, BLE 5.4 + Zigbee 3.0) |
| **Module** | ZT3L (Tuya low-power Zigbee module) |
| **Battery** | CR2430 (3V coin cell) |
| **Button** | 1x momentary, C2 pin |
| **LED** | 1x status, D7 pin (inverted via transistor) |
| **Battery ADC** | C4 pin |

### Pinout (ZT3L module)

| Module Pin | Signal | IC Pin | Direction | Active |
|-----------|--------|---------|-----------|--------|
| 7 (C2) | Button | C2 | Input | HIGH (VCC-switched) |
| 4 (D7) | LED | D7 | Output | LOW (inverted) |
| 2 (C4) | Battery ADC | C4 | Analog | — |
| 7 → 1 | RST | — | Input | LOW pulse |

### Unique Hardware Design

**This is the ONLY known TS0041/TLSR8258 variant with active-HIGH, VCC-switched button input.**

- C2 has a **series capacitor** between the MCU pin and the physical switch
- The switch connects to **VCC** (2.7V), NOT ground
- **Stock firmware** uses `bt1_lv:1` (active HIGH) — the MCU keeps C2 at 0V idle (diode + pull-down), and pressing the button couples 2.7V through the capacitor creating a rising edge
- **PC2 silicon bug** (TLSR8258 lots before date code 1844): an internal pull-down diode permanently sinks C2 to near-ground, making `SC2u` (pull-up) impossible. The `SC2d` (pull-down) config works WITH the diode, not against it
- Stock firmware config extracted: `bt1_pin:10, led1_pin:16, led1_lv:0, bt1_lv:1, module:ZT3L`

---

## Custom Firmware

### Config String

```
22ugzkme;TS0041-TB3;SC2d;ID7i;BTC4;M;
```

| Segment | Meaning |
|---------|---------|
| `22ugzkme` | Zigbee manufacturer name |
| `TS0041-TB3` | Custom model identifier |
| `SC2d` | Switch on **C2**, 100K **pull-down**, active HIGH (`pressed_when_high=1`) |
| `ID7i` | Indicator LED on **D7**, inverted (active LOW) |
| `BTC4` | Battery voltage ADC on **C4** |
| `M` | Momentary switch mode |

### Required SDK Changes

**`src/telink/configs/app_cfg.h`:**
```c
#define PULL_WAKEUP_SRC_PC2  PM_PIN_PULLDOWN_100K
#define PC2_INPUT_ENABLE     1
```

These are REQUIRED because the precompiled `libdrivers_8258.a` functions (`gpio_set_input_en`, `gpio_setup_up_down_resistor`) do NOT correctly handle Port C analog registers on the TLSR8258. Without these defines:

- `PC2_INPUT_ENABLE=1` ensures the global `gpio_init()` enables C2 input at boot (bypassing `gpio_set_input_en` in the library)
- `PULL_WAKEUP_SRC_PC2=PM_PIN_PULLDOWN_100K` ensures C2 has 100K pull-down configured at boot (reinforcing the internal diode)

No other firmware changes are needed. Standard edge-interrupt button detection works correctly once the pull-down and input are configured.

---

## Flashing

### Hardware Required

- **CH340G** USB-UART adapter (FTDI adapters do NOT work — FT232RL RX LED loads SWS line)
- **470Ω resistor** (220Ω–1KΩ acceptable)
- CR2430 battery (installed)
- Jumper wires

### Wiring

```
CH340G                  ZT3L Module
-------                 -----------
TXD ----[470Ω]-----+--- SWS (pin 17)
RXD ---------------+
GND -------------------- GND (pin 9)
RTS -------------------- RST (pin 1)
                       [Battery installed]
```

**Important:** The resistor is REQUIRED. Direct TXD→SWS splice creates a loopback that prevents the MCU's SWS responses from being received.

### Flash Command

From within the project directory:
```bash
.venv/bin/python /home/pxius/dev/TlsrComSwireWriter/TLSR825xComFlasher.py \
  -p /dev/ttyUSB0 -b 460800 -t 3000 \
  wf 0 bin/end_device/REMOTE_TUYA_BUTTON_TS0041_3_END_DEVICE/tlc_switch-1.1.2-b2596413.bin
```

Takes ~110 seconds for 150KB. Full 1MB emergency restore takes ~35 minutes.

### Recovery

Stock firmware dump (1MB golden reference) available from the project repository.

### IEEE Address Duplicate Fix

If multiple devices have the same IEEE address (from cloning flash dumps), erase the MAC address sector on the affected device:

```bash
.venv/bin/python /home/pxius/dev/TlsrComSwireWriter/TLSR825xComFlasher.py \
  -p /dev/ttyUSB0 -b 460800 -t 3000 es 0xFF000 0x1000
```

On next boot, the firmware generates a new random IEEE (keeping Telink OUI `38:C1:A4` in bytes 5-7).

---

## Z2M Setup

### External Converter

Since Z2M 2.0, the `external_converters:` config in `configuration.yaml` is no longer used. Instead:

1. Create `external_converters/` directory **next to** your Z2M `configuration.yaml`
2. Place `switch_custom_ts0041_tb3.js` inside it
3. Restart Z2M

The converter is available from:
```
https://raw.githubusercontent.com/pXius/tuya-zigbee-switch/device/ts0041-tb3-22ugzkme-v2/z2m_external/switch_custom_ts0041_tb3.js
```

Z2M will display the device as `Tuya-custom Custom switch (TS0041)` with model `TS0041-TB3`.

### Z2M Device Features

After pairing, the device exposes:
- **Battery percentage** — diagnostic
- **Switch press action** — `released`, `press`, `long_press`
- **Switch mode** — `momentary` (default, configurable to `toggle`, `momentary_nc`)
- **Switch action mode** — how switch controls bindings (`toggle_simple`, `on_off`, etc.)
- **Switch binded mode** — when to fire bindings (`short_press`, `long_press`, `press_start`)
- **Long press duration** — 0-5000ms (default: 800ms)
- **Level move rate** — for dimming bound lights (1-255 steps/ms)
- **Multi press reset count** — factory reset via rapid presses (default: 10)
- **Device config** — read/write the config string
- **OTA updates** — supported via Zigbee OTA cluster
- **Link quality** — LQI diagnostic

### Button Behavior

| Action | Z2M Event |
|--------|-----------|
| Short press (<800ms) | `press` → `released` |
| Long press (≥800ms) | `press` → `long_press` → `released` |
| Rapid 10 presses | Factory reset |

---

## Extraction Script

The `extract_pinout_from_telink_fw.py` helper script was fixed for this device:

1. **Skips false-positive `{`** in binary firmware header (0x7B byte within 0xE87B word)
2. **Handles inner `{`** within binary-wrapped config blobs
3. **Adds `i` suffix** for inverted LEDs when `ledX_lv:0`

Run against the stock dump:
```bash
.venv/bin/python helper_scripts/extract_pinout_from_telink_fw.py \
  bin/stock_full_dump_TZ3000_22ugzkme_1MB.bin
```

Output:
```
{'bt1_pin': '10', 'led1_pin': '16', ..., 'led1_lv': '0', 'bt1_lv': '1', ...}
SC2d;ID7i;
```

---

## Known Issues

### PC2 Internal Diode
Some TLSR8258 lots have a silicon bug where PC2 has an internal pull-down diode. This makes pull-up (`SC2u`) impossible. The `SC2d` config works with the diode. Fixed in chips with date code ≥ 1844.

### Series Capacitor
The external series capacitor between C2 and the switch requires the MCU to provide a reference voltage. With `SC2d`, C2 is pulled to 0V idle and rises on button press — no output drive needed. The capacitor serves as a debounce filter and DC blocker.

### IEEE Address Clones
Writing a full 1MB flash dump from one device to another clones the MAC/IEEE address stored at `0xFF000`. Erase that sector on the clone device to regenerate a unique address.

### NVM Compatibility
The `NV_MODULE_APP` is stored at `0x8C000` (module ID 6). Precompiled library write functions may fail silently for this module. The Zigbee stack handles empty NVM gracefully by re-initializing. No special handling needed in firmware.

---

## Tested Devices

| # | IEEE | Network | Status |
|---|------|---------|--------|
| A | `0xa4c1381a229280aa` | 0x0EDE | Working |
| B | `0x4D9EE9AFEE38C1A4` | 0xAEEE | Working |
| C | `0x8BA464A59938C1A4` | — | Working |
