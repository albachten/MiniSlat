# MiniSlat Software Specification

2026-10-14 · Rudy Albachten

## Overview

MiniSlat firmware runs on an ATSAMD21G18A (48 MHz Cortex-M0+, 256 KB flash, 32 KB RAM). It receives fire commands over 915 MHz LoRa and fires up to two pyro cues from a 2S pack of two 18650 LiPo cells in series (6.0 to 8.4 V).

Firmware builds with the custom LineSlat board variant (`PlatformIO/Pyro/LineSlat/variants/lineslat/variant.cpp`). Its pin numbering matches the Adafruit Feather M0 / Arduino Zero. Every pin is referred to below by its Arduino number.

This spec covers what the firmware must do on this hardware: pin use, peripheral setup, I2C addresses, and the firing and safety rules. It is based on schematic rev v1.0 (12/3/2025).

The KiCad design files and this spec are in the [albachten/MiniSlat](https://github.com/albachten/MiniSlat) repo on GitHub.

## Functional requirements

### Power and arm states

SW2 is a center-off switch that sets the board's state. Firmware cannot read SW2 directly. It infers the armed state from A0, because the A0 divider sits on V_ARM.

| SW2 position | Logic (3V3) | V_ARM | ARM LED | A0 voltage |
| --- | --- | --- | --- | --- |
| Center (off) | Off, unless USB is connected | 0 V | Off | 0 V |
| Position 1 (safe) | On | 0 V | Off | 0 V |
| Position 3 (armed) | On | Battery, 6.0 to 8.4 V | On | Battery ÷ 3 |

- The battery voltage can only be measured while armed. In the safe state, A0 reads 0 V.
- 3V3 stays regulated across the whole battery range. Even a flat pack (6.0 V, minus about 0.4 V for D2 or D3) stays above U3's 3.8 V minimum input.
- When USB is connected, VBUS also powers the logic through D1. V_ARM stays at 0 V unless SW2 is armed.

### Battery and arm monitoring

- A0 (PA02) reads V_ARM through a 200K / 100K divider: V_ARM = 3 × V(A0).
- Use the default VDDANA reference (3.3 V full scale). 3V3 stays regulated, so the reference is accurate. A0 reaches 2.8 V at a full 8.4 V pack.
- AREF (PA03) has only a decoupling capacitor, so it cannot serve as an external reference.
- Treat the board as armed when V_ARM is above 5.0 V, and as safe below 1.0 V. Between those values, report the state as unknown and refuse to fire.
- Sample at least once per second while idle, and report the result with every status message.

### Firing

Each cue is fired by two switches in series. D9 drives the shared high-side switch Q2, which connects V_ARM to the common (red) clips. D6 or D7 drives the low-side switch for one cue's black clip. Both pins are active high. Neither switch can fire a cue alone.

1. Check that the board is armed (A0) and that the command is valid for this board and cue.
2. Set D9_FIRE high to turn on the high-side switch. No current flows yet.
3. Set the cue's select pin high (D6 for cue 1, D7 for cue 2) for the fire duration, so the low-side NMOS FET switches the fire current, not the PMOS. The default is 100 ms. E-matches light in 10–20 ms at 1 A, so 100 ms is enough, and it limits the energy into R17 if a cue is shorted.
4. OPTIONAL: While firing, read the current from the INA219. Current = (V_ARM − 0.9 V for D6) ÷ (2.1 Ω + match resistance). Into a 1 Ω match, expect about 2.4 A from a full pack and 1.6 A from a flat one.
5. Set the select pin low, then D9_FIRE low.
6. OPTIONAL: Report the peak current and whether the match went open circuit (current falling to near zero).

### Continuity test

With D9_FIRE low, setting one cue's select pin high routes a small test current through that cue's match: 3V3 → D4 → R16 (150 Ω) → shunt → match → low-side FET.

- The test current is about 20 mA through a good match and near 0 mA through an open one. The matches' no-fire current is at least 200 mA, so the test cannot fire them.
- The INA219 shunt current shows whether the cue is open or closed. Its bus voltage (IN− to GND) is the voltage across the match and FET, so match resistance ≈ bus voltage ÷ current. The bus voltage LSB is 4 mV, so this resistance estimate is coarse.
- Test one cue at a time. Keep the select pin on only as long as the measurement needs, for example 10 ms.
- The continuity test works in both the safe and armed states.

### Radio

- U2 is an RFM95W (SX1276) LoRa module at 915 MHz on the default SPI bus.
- Only DIO0 is wired (to D3), so the driver must use a single interrupt line. RadioHead RH_RF95 and arduino-LoRa both work this way.
- Pull the radio's RESET (D4) low for at least 1 ms at startup, then wait 5 ms before using SPI.
- The command protocol is defined in a separate spec.

### Indicators

- **PWR (green LED5)** and **ARM (red LED4):** driven by hardware only.
- **Red user LED (LED3) on D13:** active high.
- **TX and RX LEDs (yellow LED1 and LED2) on D26 and D25:** active low. The Arduino core blinks them on USB traffic.
- **Two WS2812B-2020 NeoPixels on D12:** chained, with one pixel per cue. Pixel 0 (D7) shows cue 1 and pixel 1 (D8) shows cue 2. Each pixel shows its cue's status, for example its continuity result or that it has fired.

### USB, serial and debug

- Native USB (PA24/PA25) gives CDC serial for debug messages and bootloader uploads.
- Install the bootloader through the Cortex SWD header J2. Double-tap reset (SW1) to enter it.
- Serial1 (SERCOM0) is on header J3 at D0 (RX) and D1 (TX). It is not currently used, and the LineSlat variant does not instantiate it.

## Pin assignments

The pin numbers come from the LineSlat variant, which matches the Feather M0 / Arduino Zero numbering. All signals are 3.3 V logic.

| Arduino pin | SAMD21 port | Net | Direction | Function and notes |
| --- | --- | --- | --- | --- |
| D0 | PA11 | D0_RX | In | Serial1 RX (SERCOM0), J3 pin 4. Not used |
| D1 | PA10 | D1_TX | Out | Serial1 TX (SERCOM0), J3 pin 5. Not used |
| D3 | PA09 | D3_RADIO_IRQ | In | RFM95 DIO0. Interrupt on rising edge |
| D4 | PA08 | D4_RADIO_RST | Out | RFM95 reset, active low |
| D5 | PA15 | D5_RADIO_CS | Out | RFM95 SPI chip select, active low |
| D6 | PA20 | D6_CUE1 | Out | Cue 1 low-side FET Q3 (CLIP2). Active high, 1M pull-down |
| D7 | PA21 | D7_CUE2 | Out | Cue 2 low-side FET Q4 (CLIP4). Active high, 1M pull-down |
| D9 | PA07 | D9_FIRE | Out | Shared high-side switch (Q1 → Q2). Active high |
| D12 | PA19 | D12_NEO | Out | WS2812B data in, 2 pixels |
| D13 | PA17 | D13_LED | Out | Red user LED, active high |
| A0 (D14) | PA02 | A0_VBAT | Analog in | V_ARM ÷ 3 |
| D20 / SDA | PA22 | SDA | I/O | I2C data (SERCOM3), 10K pull-up |
| D21 / SCL | PA23 | SCL | Out | I2C clock (SERCOM3), 10K pull-up |
| D22 / MISO | PA12 | MISO | In | SPI (SERCOM4) to RFM95 |
| D23 / MOSI | PB10 | MOSI | Out | SPI (SERCOM4) to RFM95 |
| D24 / SCK | PB11 | SCK | Out | SPI (SERCOM4) to RFM95 |
| D25 | PB03 | RXLED | Out | Yellow RX LED, active low |
| D26 | PA27 | TXLED | Out | Yellow TX LED, active low |
| D28 / D29 | PA24 / PA25 | USB_D− / USB_D+ | I/O | Native USB |
| — | PA00 / PA01 | XTAL1 / XTAL2 | — | 32.768 kHz crystal (XOSC32K) |
| — | PA03 | AREF | — | 1 µF to GND only |
| — | PA30 / PA31 | SWCLK / SWDIO | — | SWD on J2 |
| — | RESET | ~RESET | In | SW1 and J2, 10K pull-up |

These pins have net labels but connect to nothing: D2 (PA14), D8 (PA06), D10 (PA18), D11 (PA16), A1 (PB08), A2 (PB09), A3/D17 (PA04), A4/D18 (PA05), A5/D19 (PB02), D27 USBHOSTEN (PA28), D30/D31 (PB22/PB23) and D38 (PA13). Configure them as inputs with pull-downs to keep current draw low.

## I2C bus

One I2C bus (Wire, SERCOM3, SDA on D20 and SCL on D21) connects the on-board INA219 to two connectors. No OLED or other I2C accessory is used at this time. The bus has 10K pull-ups to 3V3. Run it at 100 kHz. 400 kHz is supported by the INA219B but is not necessary because sample intervals will dominate performance.

| Address (7-bit) | Device | Where | Notes |
| --- | --- | --- | --- |
| 0x40 | INA219B current monitor (U4) | On board | A0 and A1 both tied to GND |
| Any other | Future accessories | J4 (QWIIC) or J5 (header) | Off board, not used at this time. Must avoid 0x40 |

J4 (QWIIC) pinout: 1 GND, 2 3V3, 3 SDA, 4 SCL. J5 (header) pinout: 1 GND, 2 3V3, 3 SDA, 4 SCL.

### INA219 setup

- **Config register (0x00):** write 0x199F. This sets a 16 V bus range, PGA ÷8 (±320 mV, ±3.2 A on the 0.1 Ω shunt), 12-bit samples and continuous shunt and bus conversion.
- **Current:** read the shunt voltage register (0x01) directly. Its LSB is 10 µV, which is 100 µA on R18. That resolves both the ~20 mA continuity current and the 1.6–2.4 A fire current, so no calibration register or range switching is needed. A shorted cue draws up to about 3.6 A, which exceeds the ±3.2 A range. Treat a reading at full scale as a short.
- **Bus voltage (0x02):** shift right by 3, then multiply by 4 mV. This is the voltage at the cue's common (red) clip relative to GND.
- One conversion takes about 0.53 ms (12-bit). During a fire, sample about every 1 ms. A match lights in 10–20 ms, so this catches the moment it goes open.

## SPI radio interface

The RFM95W is the only SPI device. It sits on the default Arduino SPI bus (SERCOM4).

| Setting | Value |
| --- | --- |
| SPI pins | MISO D22 (PA12), MOSI D23 (PB10), SCK D24 (PB11) |
| Chip select | D5 (PA15), active low |
| Reset | D4 (PA08), active low |
| Interrupt | D3 (PA09), DIO0 only. DIO1–DIO5 are not connected |
| SPI mode and clock | Mode 0, MSB first, 8 MHz or less (the SX1276 allows up to 10 MHz) |
| Frequency | 915 MHz (US 902–928 MHz band) |
| TX output | PA_BOOST pin, up to +20 dBm. Lower it if range allows, to save battery |
| Suggested modem | LoRa, 125 kHz BW, SF7–SF9, CR 4/5, private sync word 0x12 |
| Antenna | On-board 915 MHz coil (ANT1) |

The RadioHead RH_RF95 driver works with this pinout: `RH_RF95 rf95(5, 3);`, after pulsing reset on D4. The over-the-air command protocol is outside this spec.

## Safety requirements

Firing needs two independent switches, a high-side switch and a cue's low-side switch, plus the physical arm switch. Firmware must never be the only thing preventing a fire.

1. **Safe outputs at boot.** The first lines of `setup()` set D9, D6 and D7 as outputs driven low, before any other setup. All three switches stay off until then: Q3 and Q4 have 1M gate pull-downs, and Q1 (a bipolar transistor) needs base current, which the MCU's reset-state input pins don't supply.
2. **Fire only when armed.** Refuse every fire command unless A0 shows V_ARM above 5.0 V at the moment of the command.
3. **Valid commands only.** Act only on a fire command addressed to this board and cue. Discard duplicates and stale commands, using a sequence number or a timestamp.
4. **Duration limit.** The fire pulse defaults to 100 ms and must be at least 100 ms, and the hard limit is 2 s. Into a shorted cue on a full pack, R17 dissipates about 26 W, five times its 5 W rating, or about 2.6 J over a 100 ms pulse. Stop firing as soon as the INA219 reads full scale.
5. **One cue at a time.** Both cues share R17. Firing both at once splits the fire current between them, which may not light either match.
6. **Continuity only with D9 low.** Never set D9 high during a continuity test.
7. **Watchdog.** Enable the WDT with a timeout of about 1 s. A reset leaves all pins high-impedance, which turns Q3 and Q4 off.
8. **Brown-out.** Enable BOD33 at about 2.7 V. 3V3 stays regulated on a 2S pack, so the BOD should only trip on a fault.
9. **Clean up after firing.** After every fire, or any fault such as a command timeout or radio error, drive D6 and D7 low first, then D9, so the low-side FET breaks the current.
