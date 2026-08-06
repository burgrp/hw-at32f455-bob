# AT32F455 breakout-board pin assignment

This assignment keeps every movable onboard peripheral off GPIO port A. It assumes the AT32F455CxT7 LQFP-48 package; the AF/mux values remain the same in larger packages, but package pin numbers differ.

Sources:

- AT32F455/456/457 datasheet, version 2.01, especially Table 10 (pin definitions)
- AT32F455/456/457 reference manual, version 2.01, especially Tables 6-1 and 6-2 (GPIO mux values) and Table 22-1 (OTGFS pins)
- `rs485-st3485eb.pdf`
- `can-xl1040.pdf`

## Recommended assignment

| Peripheral | Signal | GPIO | LQFP-48 pin | GPIO mux / AF | Notes |
|---|---|---:|---:|---:|---|
| HEXT | Crystal input | PH0 | 5 | Dedicated | HEXT_IN |
| HEXT | Crystal output | PH1 | 6 | Dedicated | HEXT_OUT |
| OLED | SDA | PH3 | 23 | MUX4 / AF4 | I2C2_SDA, open-drain |
| OLED | SCL | PH2 | 35 | MUX4 / AF4 | I2C2_SCL, open-drain |
| Encoder | A | PB6 | 42 | MUX2 / AF2 | TMR4_CH1 |
| Encoder | B | PB7 | 43 | MUX2 / AF2 | TMR4_CH2 |
| Encoder | Push switch | PB4 | 40 | GPIO | 10 kΩ/100 nF debounce network |
| Debug UART | RX | PB8 | 45 | MUX8 / AF8 | USART5_RX |
| Debug UART | TX | PB9 | 46 | MUX8 / AF8 | USART5_TX |
| CAN | RXD | PB12 | 25 | MUX9 / AF9 | CAN2_RX |
| CAN | TXD | PB13 | 26 | MUX9 / AF9 | CAN2_TX |
| CAN | STB | PB15 | 28 | GPIO | Low = high-speed mode |
| RS-485 | DI | PB10 | 21 | MUX7 / AF7 | USART3_TX |
| RS-485 | RO | PB11 | 22 | MUX7 / AF7 | USART3_RX |
| RS-485 | DE + /RE | PB14 | 27 | MUX7 / AF7 | USART3_RTS_DE; tie ST3485EB DE and /RE together |
| BleRiot | CSN | PB0 | 18 | GPIO | Active low; add about 10 kΩ pull-up to 3.3 V |
| BleRiot | SCK | PB3 | 39 | MUX5 / AF5 | SPI1_SCK, or ordinary GPIO for bit-banging |
| BleRiot | DATA | PB5 | 41 | MUX5 / AF5 | SPI1_MOSI used bidirectionally, or ordinary GPIO for bit-banging |
| WS2812B | DIN | PB1 | 19 | MUX2 / AF2 | TMR3_CH4; direct 3.3 V logic through 60 Ω |
| Status LED | Red | PC13 | 2 | GPIO | Active low; LED and 1 kΩ resistor to 3.3 V |
| Status LED | Green | PC14 | 3 | GPIO | Active low; LED and 1 kΩ resistor to 3.3 V |
| USB | D− | PA11 | 32 | Dedicated USB pin | OTGFS1_D−; no GPIO mux value is programmed |
| USB | D+ | PA12 | 33 | Dedicated USB pin | OTGFS1_D+; no GPIO mux value is programmed |
| SWD | SWDIO | PA13 | 34 | Dedicated / MUX0 | Debug data |
| SWD | SWCLK | PA14 | 37 | Dedicated / MUX0 | Debug clock |

This uses all bonded Port B GPIOs except PB2, which is not available in the LQFP-48 package. It keeps PA0–PA10 and PA15 free of onboard peripheral connections. PA11/PA12 are fixed USB pins, while PA13/PA14 retain their reset-default SWD debug roles.

## Port A and USB limitation

Native USB is the unavoidable exception to the “no onboard peripheral on port A” goal. On AT32F455, OTGFS1_D− and OTGFS1_D+ are fixed on PA11 and PA12. There is no non-port-A remap and no AF number for these two data pins: enable the OTGFS clock and set `PWRDOWN=1` in the USB controller as described by reference-manual Table 22-1.

Options are therefore:

1. Keep native USB on PA11/PA12 and omit those two pins from an electrically exclusive port-A header.
2. Put PA11/PA12 on the header as shared signals, clearly label them, and keep the header routing extremely short to avoid USB stubs.
3. If every PA pin must be header-only, omit native USB and use an external USB-to-UART bridge connected to a non-A USART instead.

## Peripheral notes

### Encoder

The hardware encoder interface requires timer inputs C1IN and C2IN, so use TMR4_CH1 and TMR4_CH2 rather than channels 3/4. Configure PB6 and PB7 as AF2. External pull-ups and small RC filtering footprints are useful if the encoder is connected by a cable; avoid excessive capacitance that distorts fast edges.

### OLED I2C

Configure PH2/PH3 as AF4, open-drain. Fit one set of pull-ups on the board, typically 4.7 kΩ to 3.3 V for a short local bus. Verify that an OLED module does not pull SCL/SDA up to 5 V.

### HEXT crystal

Reserve PH0 for HEXT_IN and PH1 for HEXT_OUT. These are dedicated oscillator pins and do not require GPIO AF configuration. Place the crystal and its load capacitors immediately beside the MCU, keep both traces short and approximately symmetric, and avoid routing fast signals beneath them.

Choose the load capacitors from the selected crystal's specified load capacitance. For equal capacitors, a useful estimate is $C_L \approx C/2 + C_{stray}$. For example, an 8 pF crystal often starts with two 12 pF capacitors when stray capacitance is approximately 2 pF; verify against the crystal data and during bring-up.

### SWD and debug UART

PA13 is SWDIO and PA14 is SWCLK in their reset-default configuration. Expose both along with NRST, GND, and a 3.3 V target-voltage reference. The voltage-reference pin is not intended to power the board unless that is explicitly designed into the debugger connection.

Use USART5 on PB8/PB9 for the debug UART: PB8 is MCU RX and PB9 is MCU TX, both AF8. Label connector TX/RX from the MCU perspective and also provide GND. This UART remapping frees the original PB8/PB9 CAN1 pair for debugging while keeping the interface off Port A.

### CAN (XL1040)

The XL1040 requires 5 V ±10% on VCC. Its TXD and STB high-level threshold is 2 V, so AT32F455 3.3 V outputs are valid. Use CAN2 on PB12/PB13: PB12 is CAN2_RX and PB13 is CAN2_TX, both AF9. PB12 is marked FT in the AT32F455 datasheet and can accept the transceiver's 5 V RXD output.

PB15 can control STB; drive it low for normal high-speed CAN. If standby/wake is not required, STB may instead be tied low. The XL1040 has an internal pull-up on STB, so ensure the firmware drives PB15 low early if it is fitted.

### RS-485 (ST3485EB)

The ST3485EB runs directly from 3.0–3.6 V. Tie DE and /RE together and drive them from USART3_RTS_DE on PB14:

- low: driver disabled, receiver enabled
- high: driver enabled, receiver disabled

Use USART hardware driver-enable timing if supported by the firmware; otherwise PB14 can be controlled as an ordinary GPIO. Add termination only at bus ends, and provide bias/TVS footprints according to the intended cabling and installation environment.

### BleRiot PAN211x

PAN211x uses three-wire, half-duplex SPI: CSN, SCK, and one bidirectional DATA wire. It has no separate MISO, reset, busy, or mandatory IRQ pin.

The current BleRiot approach should use PB0/PB3/PB5 as GPIOs and bit-bang the transfer. SPI1 AF5 is recorded for PB3/PB5 so hardware half-duplex SPI can be evaluated later; PB0 remains ordinary GPIO for CSN. Do not assume hardware SPI releases PB5 soon enough during read turnaround without oscilloscope verification. Use SPI mode 0 and stay at or below 8 MHz pending validation against the exact PAN211x silicon documentation.

### WS2812B

Use PB1 as TMR3_CH4 with AF2. A timer channel with DMA can generate the approximately 800 kbit/s waveform without CPU-sensitive bit-banging.

Power the WS2812B from 3.3 V and drive DIN directly from PB1 through a 60 Ω series resistor. This supply voltage is outside the original WS2812B specification but is an intentional design choice. Place a 100 nF ceramic capacitor directly across the LED supply pins and budget sufficient current on the 3.3 V regulator.

### Status LEDs

Use PC13 for the red LED and PC14 for the green LED. Wire both for active-low operation: connect each LED anode to 3.3 V through a 1 kΩ resistor and its cathode to the GPIO. Driving the GPIO low turns the LED on.

PC13–PC15 are supplied through a limited-current power switch, and the datasheet specifically prohibits using them as current sources for LEDs. The active-low connection makes the GPIO sink current instead. A 1 kΩ resistor keeps each LED near 1 mA with typical modern indicators; do not reuse the 330 Ω status-LED resistors on these pins. PC14 cannot be used for an LEXT crystal while assigned to the green LED; this does not conflict with the HEXT crystal on PH0/PH1.

## AF configuration summary

- HEXT: `PH0/PH1`, dedicated oscillator pins
- OLED: `PH2/PH3 -> AF4` (`I2C2`), open-drain
- Encoder: `PB6/PB7 -> AF2`
- Debug UART: `PB8/PB9 -> AF8` (`USART5_RX/TX`)
- CAN: `PB12/PB13 -> AF9` (`CAN2_RX/TX`)
- USART3: `PB10/PB11/PB14 -> AF7`
- Optional hardware SPI1: `PB3/PB5 -> AF5`
- WS2812B: `PB1 -> AF2` (`TMR3_CH4`)
- Status LEDs: `PC13/PC14`, active-low GPIO outputs
- SWD: `PA13/PA14`, dedicated reset-default debug pins
- GPIO-only signals: PB0, PB4, PB15
- USB D−/D+: PA11/PA12 dedicated PHY pins, not AF-configured
