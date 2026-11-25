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
- J1 USB-C 16-pin receptacle (HRO TYPE-C-31-M-12). CC1/CC2: R1/R2 5.1 kΩ Rd to GND (not read by MCU).
- D+ (A6+B6) → IO19, D− (A7+B7) → IO18 (native USB for flashing).
- VBUS → F1 polyfuse 1812 → +5V. **Value field still says "1812" — set to hold current (2 A hold).**
- On +5V: D1 SMF5V0A TVS, C3 10 µF, C4 100 nF. Keep ≤10 µF directly on VBUS (USB inrush).
- Shield and GND to ground. All grounds use `power:Earth` symbol (one net, consistent).
- All passives 0805 SMD.

## 3.3 V
- U1 AP2112K-3.3 (600 mA, VIN 2.5–6 V), EN tied to +5V. C1 1 µF in (+5V), C2 1 µF out (+3.3V).

## ESP32-C3-WROOM-02
- 3V3: C5 10 µF + C6 100 nF right at the pin.
- EN: R10 10 kΩ pull-up + C7 1 µF to GND (auto-start on power).
- IO8: R9 10 kΩ pull-up. IO2: R8 10 kΩ pull-up (strapping pins).
- IO9: SW1 (PTS645 SMD tactile) to GND + R6 10 kΩ pull-up — hold for download mode.
- Unused: IO7, IO10, IO20/RXD, IO21/TXD (keep UART free for debug).
- Optional: reset button on EN (SW + 0.1 µF + 0 Ω, per Espressif ref fig. 9-1).

## Output ports (schematic done) — hierarchical sheet `output.kicad_sch` used ×2 (`output-wireless`, `output-headphones`)
- Board-mounted USB-C **plugs** (intentional — devices plug straight onto the board): P1 headphones (JingTuoJin vertical plug), P2 pad (Molex 105444). Plug has one CC (A5); B5 = VCONN, left open.
- TPS2553 load switch (U4 pad, U3 headphones): IN 0.1 µF (C10/C8), OUT 120 µF electrolytic (C11/C9, CP_Elec_6.3x7.7), EN pull-down 20 kΩ (R14/R12), FAULT pull-up 100 kΩ to 3V3 (R13/R11).
  - EN is **active high**. TODO: check IO5/IO1 reset-state pulls in ESP32-C3 datasheet (internal pull-up would fight the 20 kΩ).
  - Datasheet SLVS841G: limit 75 mA–1.7 A typ, **max continuous 1.5 A** (Tj ≤105 °C). R_ILIM 15–232 kΩ.
  - Symbol: project lib `charging_station:TPS2553DBV` (`charging_station.kicad_sym` + `sym-lib-table`), pinout 1 IN, 2 GND, 3 EN, 4 FAULT, 5 ILIM, 6 OUT. Footprint SOT-23-6.
- R_ILIM: pad R3 20 kΩ → 1.20/1.28/1.38 A (min/nom/max), headphones R5 75 kΩ → 0.31/0.35/0.40 A. If pad needs more after measuring: R3 → 16 kΩ (~1.5 A min), and firmware delays headphones while pad draws a lot.
- Rp from CC (A5) to **+3.3V**: pad R4 12 kΩ (advertises 1.5 A — slightly above the 1.28 A limit; only matters if pad reads Rp, 36 kΩ would match), headphones R7 36 kΩ (default).
- Worst case total ≈ 1.38 + 0.40 + ESP 0.35 ≈ 2.1 A peak (typ ~1.7–1.8 A) vs. 2 A brick — accepted.
- CC line straight to ADC pin.

| Function | Pad | Headphones |
|---|---|---|
| CC (ADC1) | IO4 | IO0 |
| EN | IO5 | IO1 |
| FAULT | IO6 | IO3 |

Don't put CC lines on IO2 (strapping).

## Firmware idea
- Compute CC voltage thresholds: open (≈3.3 V) vs. Rd 5.1 kΩ attached against 12 k / 36 k Rp.
- Attach detected → enable load switch; CC rises back → disable.
- Optional later: pad current sense (INA219 or shunt + INA180) to know when phone is actually charging.

## Open items
- Set F1 Value (2 A hold).
- PCB: run Update PCB from Schematic (F8, replace footprints) — C1/C2 are still THT disc footprints on the board.
- Check IO5/IO1 reset-state pulls.
- Measure pad and headset current draw → confirm R3.
- Then PCB layout.
