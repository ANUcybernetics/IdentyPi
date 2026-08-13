# Wiring plan (humans only — never let this file near the Pi)

This is the concrete pin-by-pin build plan. It contains exact GPIO assignments, which is exactly the information [PI.md](PI.md) exists to keep from the agent. Keep this file, README.md, and any wiring photos/labels off the Pi filesystem entirely.

Bus assignments (SDA/SCL, SPI0) are fixed by hardware — there's no meaningful "random" choice there, they're real buses. Everything else is scattered across free GPIO deliberately, not grouped by function, so there's no pattern for the agent to lean on.

Split into what's actually on the desk now (**Phase 1**) and what's ordered/pending (**Phase 2**) — Phase 2 pins are pre-reserved so adding them later is a plug-in, not a redesign.

## Phase 1 — wiring now (original list, minus ultrasonic)

Ultrasonic (HC-SR04) dropped — 5V echo pin straight into a GPIO input, not worth the risk.

### I2C bus — SDA = GPIO2 (pin 3), SCL = GPIO3 (pin 5)

No duplicate addresses in Phase 1, so everything sits directly on the bus — no mux needed yet.

| Device | Address | Notes |
|---|---|---|
| SEN0528 (AHT20 temp/humidity) | 0x38 | direct on bus |
| SEN0394 (SGP40 VOC) | 0x59 | direct on bus |
| APDS-9960 (gesture/proximity/light) | 0x39 | direct on bus — confirm breakout is 3.3V-native before powering |
| HuskyLens | 0x32 | direct on bus — **must set its onboard protocol switch to I2C mode first**, it doesn't default there |

### Scattered digital/PWM/analog GPIO (the "random" bucket)

| Device | Signal | GPIO (BCM) | Physical pin |
|---|---|---|---|
| SEN0171 (PIR motion) | digital out | GPIO17 | 11 |
| Keyestudio joystick | button (digital in) | GPIO27 | 13 |
| Keyestudio joystick | X axis (analog — **dead end, no ADC**) | GPIO22 | 15 |
| Keyestudio joystick | Y axis (analog — **dead end, no ADC**) | GPIO23 | 16 |
| TowerPro SG92R servo | PWM signal | GPIO18 | 12 |
| Keyestudio 3W LED module | PWM/digital signal | GPIO24 | 18 |
| Duinotech IR proximity | digital out | GPIO25 | 22 |
| Keyestudio TEMT6000 | analog out (**dead end, no ADC**) | GPIO4 | 7 |
| Keyestudio cap touch module | digital out | GPIO5 | 29 |

Analog dead ends are wired anyway rather than left off the desk — the agent finding a live GPIO pin that reads pure noise, and working out *why*, is part of the exercise.

### Cameras, monitor, and layout

Both the Pi Camera v2.1 (CSI) and HuskyLens (its own onboard camera + AI + screen) go up at once, arranged so they can see: each other, a monitor, and the rest of the parts on the table. Two independent vision systems perceiving the same shared scene, one of which has its own display — deliberately not explained to the agent anywhere, including here-derived docs.

Practical note: both cameras are **fixed-focus, not macro** — the Pi Camera v2.1's lens is tuned roughly 1m-to-infinity, HuskyLens similar. Up close on a cluttered desk of wiring, images will likely be soft. Do a live dry-run of both feeds before gluing anything down; the "sees the monitor and the other camera" distance is probably naturally about right, close-up wiring inspection may need to be pulled back further than instinct suggests.

## Phase 2 — reserved, wire in once acquired (mux, 2x OLED, epaper, buzzer)

Not a killer to add later — pins below are reserved now so nothing in Phase 1 needs to move.

### I2C, once the mux and OLEDs arrive

| Device | Address | Notes |
|---|---|---|
| TCA9548A multiplexer | 0x70 | direct on bus, joins the Phase 1 I2C devices above |
| OLED #1 | 0x3C (typical SSD1306) | direct on bus, verify once part confirmed |
| OLED #2 | unknown yet | **route through mux (channel 6)** by default in case it shares 0x3C with OLED #1 |
| "bunch of temp sensors" (identical, duplicate address) | same address as each other | one per mux channel (0, 1, 2, …) |

### SPI0, once the epaper screen arrives (the project's one SPI device)

| Signal | GPIO (BCM) | Physical pin |
|---|---|---|
| MOSI | GPIO10 | 19 |
| MISO | GPIO9 | 21 |
| SCLK | GPIO11 | 23 |
| CE0 (chip select) | GPIO8 | 24 |
| DC (data/command) | GPIO12 | 32 |
| RST | GPIO16 | 36 |
| BUSY | GPIO20 | 38 |

DC/RST/BUSY aren't part of the SPI standard — they're just GPIO the epaper module needs, drawn from the same reserved pool.

### Buzzer, once acquired

| Device | Signal | GPIO (BCM) | Physical pin |
|---|---|---|
| Buzzer | digital out | GPIO6 | 31 |

## General wiring rules

- **Common ground, always.** Anything powered from an external supply (servo, LED driver board, any 5V-rail sensor) must share ground with the Pi, or its digital signal is meaningless even if the voltage levels are otherwise fine.
- **Servo and 3W LED module get their own external power**, never the Pi's 5V or 3.3V rail — both can pull more current than the Pi's regulator is meant to supply, especially the servo under load.
- Confirm Duinotech IR proximity sensor's output voltage before connecting — check whether it can run at 3.3V, or add a level shifter if not (see landmines section in README.md).

## Status

- [x] Ultrasonic dropped
- [x] Phase 1 pin plan drafted (this file)
- [x] Phase 2 pins pre-reserved for mux/OLEDs/epaper/buzzer
- [ ] Confirm Duinotech IR sensor logic voltage
- [ ] Physical wiring — Phase 1
- [ ] Camera + monitor layout dry-run (check focus/framing before fixing in place)
- [ ] PI.md copied onto the Pi as CLAUDE.md, README/WIRING kept off it
- [ ] Phase 2 parts acquired and wired in when ready
