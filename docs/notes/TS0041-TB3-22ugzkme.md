# TS0041 TB3 (_TZ3000_22ugzkme) — Development Notes

## Hardware

| Property | Value |
|----------|-------|
| Chip | TLSR8258F1KAT32 (1MB flash, QFN32) |
| Module | ZT3L |
| Button pin | C2 (pin 7) |
| LED pin | D7 (pin 4), inverted (transistor-driven) |
| Battery ADC | C4 (pin 2) |

## Unique Hardware Design

**This is the ONLY known TS0041 variant with active-HIGH, VCC-switched button input.**

- C2 has a **series capacitor** between the pin and the physical switch
- The switch connects to **VCC** (2.7V), not GND
- Stock firmware uses `bt1_lv:1` (active HIGH)
- Press: VCC couples through capacitor → C2 rising edge
- Release: capacitor discharges → C2 falling edge

## Known Chip Bug

**TLSR8258 PC2 internal pull-down diode** (fixed in lots with Date code ≥ 1844).
The diode pulls C2 to ~0V permanently, defeating any internal pull-up.
This is actually the *correct direction* for this board's design — the diode
keeps C2 at 0V idle, and the VCC-switched button pulls it HIGH on press.

The `PULL_WAKEUP_SRC_PC2=PM_PIN_PULLDOWN_100K` in `app_cfg.h` reinforces
the diode's pull at boot via `gpio_analog_resistance_init()`.
`PC2_INPUT_ENABLE=1` ensures input is enabled despite the precompiled
`libdrivers_8258.a` possibly mishandling Port C analog registers.

## Config

```
22ugzkme;TS0041-TB3;SC2d;ID7i;BTC4;M;
```

- `SC2d` — C2 pull-down 100K, `pressed_when_high=1` (active HIGH)
- `ID7i` — D7 inverted LED
- `BTC4` — battery ADC on C4
- `M` — momentary switch mode

## Stock Firmware Reference

- Golden reference dump saved at `bin/stock_full_dump_TZ3000_22ugzkme_1MB.bin`
- Flashed via CH340G for circuit analysis: confirmed pull-down + active HIGH behavior

## Extraction Script

`extract_pinout_from_telink_fw.py` was fixed to:
1. Skip false-positive `{` in the binary firmware header (0x7B byte in `0xE87B` word)
2. Handle inner `{` within binary-wrapped config blob
3. Add `i` suffix for inverted LEDs when `ledX_lv:0`

## Flashing Hardware

**CH340G** USB-UART adapter with TX→SWS via 470Ω resistor.
FT232RL does NOT work for read-back (RX LED loads SWS line — documented limitation).

## Future Enhancements

### LED Press Feedback
The indicator LED (`ID7i`) currently only shows network status.
For tactile feedback, the switch cluster could pulse the LED on
press/hold/release events. Not required for operation — cosmetic UX improvement.

Implementation hint: `switch_cluster.c` callbacks could toggle
`switch_clusters[index].indicator_led` via `led_on()`/`led_off()`.
Would need debounce-aware timing and power-conscious duty cycling since
this is a battery-powered device.
