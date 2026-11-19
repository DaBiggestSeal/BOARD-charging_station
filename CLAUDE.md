# Charging station — design notes

DIY desk charging station: ESP32-C3 controls two USB-C outputs (wireless phone pad + headphones). KiCad project.

Working style: guide me through problems, don't hand over complete solutions. Keep it direct, no over-engineering.

## Power budget
- Supply: Motorola wall brick, **5 V / 2 A (10 W), no PD**. Everything runs on a single 5 V rail.
- Wireless pad: E-Load item 75.019.66 (input 5 V/3 A or 9 V/2 A). On 5 V expected ~5–7.5 W, ~1.2–1.8 A — **to be measured** with USB meter.
- Headphones: SteelSeries Arctis Nova 5 Wireless, USB-C power-only, likely ignores Rp. Actual draw **to be measured**.
- Priority: phone pad. Headphones capped at ~400 mA by load switch (constant-current limit type, not hiccup/latch).

## Decided NOT to use
PD sink/source chips, boost converters, buck converter, MOSFET level shifters, Bluetooth.

## Input section (done)
- J1 USB-C 16-pin receptacle. CC1/CC2: 5.1 kΩ Rd to GND (not read by MCU).
- D+ (A6+B6) → IO19, D− (A7+B7) → IO18 (native USB for flashing).
- VBUS → F1 polyfuse (1812, 2 A hold — set Value field) → +5V.
- Shunt to GND on +5V: D1 SMF5.0A TVS (cathode on +5V), C7 10 µF, C8 100 nF. Keep ≤10 µF directly on VBUS (USB inrush).
- Shield and GND to ground. All grounds currently use `power:Earth` symbol (one net, consistent).

## 3.3 V
- U1 AP2112K-3.3 (600 mA, VIN 2.5–6 V), C2 1 µF in, C3 1 µF out.

## ESP32-C3-WROOM-02
- 3V3: C4 10 µF + C1 100 nF right at the pin.
- EN: R3 10 kΩ pull-up + C5 1 µF to GND (auto-start on power).
- IO8: R4 10 kΩ pull-up. IO2: R5 10 kΩ pull-up (strapping pins).
- IO9 (done): SW1 to GND + R6 10 kΩ pull-up to 3V3, C6 removed. SW1/R6 still need footprints (if 4-pin tactile, map symbol pins to opposite sides).
- Optional: reset button on EN (SW + 0.1 µF + 0 Ω, per Espressif ref fig. 9-1).

## Output ports (TODO) — same block ×2
- USB-C 6-pin power-only receptacle.
- TPS2553 load switch: IN 100 nF, OUT 10 µF, EN from GPIO with 100 kΩ pull to OFF state, ILIM resistor from datasheet equation, FAULT open-drain optional with 10 kΩ pull-up.
  - EN is **active high** on TPS2553 → 100 kΩ EN→GND. TODO: check IO5/IO6 reset-state pulls in ESP32-C3 datasheet (an internal pull-up would fight the 100 kΩ).
  - Datasheet SLVS841G: limit adjustable 75 mA–1.7 A typ, but **max continuous 1.5 A** (Tj ≤105 °C), 1.2 A (≤125 °C). R_ILIM 15–232 kΩ. Pad target ~1.7 A does NOT fit → measure pad, then cap at ≤1.5 A or pick another switch. Headphones ~0.4 A fine.
  - Symbol: project lib `charging_station:TPS2553DBV` (`charging_station.kicad_sym` + `sym-lib-table` in project folder), pinout 1 IN, 2 GND, 3 EN, 4 FAULT, 5 ILIM, 6 OUT. Footprint `Package_TO_SOT_SMD:SOT-23-6`.
  - USB-C: `Connector:USB_C_Receptacle_PowerOnly_6P`, footprint `Connector_USB:USB_C_Receptacle_GCT_USB4125-xx-x_6P_TopMnt_Horizontal` (match to bought part).
  - Power budget question open: pad + headphones + ESP vs. 2 A brick/fuse — firmware or ILIM must handle both ports on.
- Rp from CC1 and CC2 to **+3.3V** (no divider needed): pad 12 kΩ (advertises 1.5 A), headphones 36 kΩ (default).
- CC lines straight to ADC pins.

| Function | Pad | Headphones |
|---|---|---|
| CC1 / CC2 (ADC1) | IO0 / IO1 | IO3 / IO4 |
| EN | IO5 | IO6 |
| FAULT (optional) | IO7 | IO10 |

Don't put CC lines on IO2 (strapping). No spare ADC1 pins for input CC — not needed.

## Firmware idea
- Compute CC voltage thresholds: open (≈3.3 V) vs. Rd 5.1 kΩ attached against 12 k / 36 k Rp.
- Attach detected → enable load switch; CC rises back → disable.
- Optional later: pad current sense (INA219 or shunt + INA180) to know when phone is actually charging.

## Open items
- Set F1 value, draw output ports (next: decide pad current limit → R_ILIM).
- Measure pad and headset current draw.
