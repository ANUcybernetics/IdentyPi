# Wiring plan (humans only — never let this file near the Pi)

This is the concrete pin-by-pin build plan. It contains exact GPIO assignments, which is exactly the information [PI.md](PI.md) exists to keep from the agent. Keep this file, README.md, and any wiring photos/labels off the Pi filesystem entirely.

Bus assignments (SDA/SCL, SPI0) are fixed by hardware — there's no meaningful "random" choice there, they're real buses. Everything else is scattered across free GPIO deliberately, not grouped by function, so there's no pattern for the agent to lean on.

**As-built, 2026-08-17.** Reconciled against photos of the actual assembly (IMG_0445 "not included", IMG_0447 "the setup"). Install got hard partway through, and scope got trimmed as a result — that's part of the point, not a failure (dead ends are iteration). Confirmed dropped entirely: SEN0171 PIR, the relay (SRD-03VDC-SL-C), the Keyestudio joystick, TEMT6000, the Keyestudio IR module (IR_01), the DS18B20 probe, and the TCA9548A mux with both OLEDs. Still unconfirmed and left in the table below pending a look: which of AHT20/SGP40 actually made it in (possibly both, possibly one), whether the cap touch module is wired, and whether the ESP32+MPU-6050 subsystem is actually connected.

## What's actually wired

Ultrasonic (HC-SR04) was dropped earlier — 5V echo pin straight into a GPIO input, not worth the risk.

### I2C bus — SDA = GPIO2 (pin 3), SCL = GPIO3 (pin 5)

No mux in the build — everything left on this bus has a unique address anyway.

| Device | Address | Notes |
|---|---|---|
| SEN0528 (AHT20 temp/humidity) | 0x38 | **unconfirmed which of AHT20/SGP40 is actually in — check before relying on this row** |
| SEN0394 (SGP40 VOC) | 0x59 | **unconfirmed which of AHT20/SGP40 is actually in — check before relying on this row** |
| APDS-9960 (Adafruit breakout — gesture/proximity/light) | 0x39 | direct on bus |
| HuskyLens | 0x32 | direct on bus — **must set its onboard protocol switch to I2C mode first**, it doesn't default there |
| PiicoDev RFID Module | 0x2C (default, switchable 0x2C–0x2F) | direct on bus |

### SPI0 — epaper screen (WeAct 4.2")

The board's own silkscreen labels these pins SCL/SDA/CS/D-C/RES/BUSY — that's the driver chip's alternate I2C-mode pin names, not actual I2C. It's SPI: SCL on the silkscreen is the SPI clock (SCLK), SDA is MOSI. Confirmed by full pin list (BUSY, RES, D/C, CS, SCL, SDA, GND, VCC).

| Signal | Silkscreen label | GPIO (BCM) | Physical pin |
|---|---|---|---|
| MOSI | SDA | GPIO10 | 19 |
| SCLK | SCL | GPIO11 | 23 |
| CE0 (chip select) | CS | GPIO8 | 24 |
| DC (data/command) | D/C | GPIO12 | 32 |
| RST | RES | GPIO16 | 36 |
| BUSY | BUSY | GPIO20 | 38 |

No MISO — the panel is write-only from the host (BUSY reports status instead of a data-back line), so this board doesn't break one out at all. GPIO9 (pin 21) is free.

### Bit-banged 3-wire — 8x8 LED matrix (MAX7219-style breakout)

Not real SPI (no MISO, it's a shift-register protocol) — deliberately kept off the real SPI0 bus and given its own GPIO so it doesn't get mistaken for genuine SPI.

| Signal | GPIO (BCM) | Physical pin |
|---|---|---|
| DIN | GPIO19 | 35 |
| CLK | GPIO26 | 37 |
| CS | GPIO21 | 40 |

Note: MAX7219 nominally wants closer to 5V for full brightness. At 3.3V it should still register/respond, just possibly dim — not a safety issue, just a "why won't this fully light up" puzzle rather than a landmine.

### Scattered digital/PWM GPIO (the "random" bucket)

| Device | Signal | GPIO (BCM) | Physical pin |
|---|---|---|---|
| TowerPro SG92R servo | PWM signal | GPIO18 | 12 |
| Keyestudio 3W LED module | PWM/digital signal | GPIO24 | 18 |
| Keyestudio cap touch module (**unconfirmed if actually wired**) | digital out | GPIO5 | 29 |
| Piezo buzzer | digital out | GPIO6 | 31 |

### Nested subsystem — ESP32 SuperMini + 2x MPU-6050 (via USB, from Groundskeeper) — unconfirmed

Status in the actual build unclear — a small board with a visible antenna trace appears in the "setup" photo, tentatively this, but not confirmed connected or powered. If it is in: connects over USB, not Pi GPIO at all, enumerates as a serial device. The two MPU-6050s would live on the ESP32's own I2C bus, not the Pi's (one at default address 0x68, the other with AD0 tied high → 0x69).

**Open decision, if this subsystem is actually included:** whether to fully wipe the ESP32's current firmware before handing it over. A full wipe turns this into a genuinely nested bring-up challenge — the Pi's agent has to notice a blank USB device, realize nothing runs on it yet, and write + flash firmware onto a second, unknown microcontroller before it can learn anything from it at all. A thin stub firmware (streams raw I2C scan output on boot) keeps a real puzzle without requiring blind toolchain bring-up.

### Camera, monitor, and physical layout — built

Confirmed built as a cardboard rig: two cameras (Pi Camera v2.1 on CSI ribbon, HuskyLens with its own onboard camera + AI + screen) mounted on angled cardboard walls above the assembly, looking down at the board and each other.

Confirmed physical orientation: **the Pi's HDMI ports face toward the Pi Camera, and beyond it toward the HuskyLens** — so "front" of the Pi, camera, and HuskyLens all roughly line up along one axis.

Practical note: both cameras are **fixed-focus, not macro** — the Pi Camera v2.1's lens is tuned roughly 1m-to-infinity, HuskyLens similar. Up close on a cluttered desk of wiring, images will likely be soft.

## Dropped entirely (confirmed 2026-08-17, not in the build)

| Device | Was assigned | Now |
|---|---|---|
| SEN0171 PIR motion | GPIO17 | free |
| Keyestudio joystick (button + X + Y) | GPIO27, GPIO22, GPIO23 | free |
| Duinotech IR proximity | GPIO25 | free (dropped earlier — turned out not to be on hand at all) |
| Keyestudio TEMT6000 | GPIO4 | free |
| Keyestudio IR module (IR_01) | GPIO13 | free |
| DS18B20 1-Wire probe | GPIO7 | free |
| Relay (SRD-03VDC-SL-C) | GPIO14 | free — **the mains-relay safety concerns in README.md no longer apply to this build unless it goes back in** |
| TCA9548A mux + OLED 1.3" + OLED 1.5" | GPIO2/3 (shared I2C bus) | free — no I2C address collisions to manage anymore |

## General wiring rules

- **Common ground, always.** Anything powered from an external supply (servo, LED driver board, any 5V-rail sensor) must share ground with the Pi, or its digital signal is meaningless even if the voltage levels are otherwise fine.
- **Servo and 3W LED module get their own external power**, never the Pi's 5V or 3.3V rail — both can pull more current than the Pi's regulator is meant to supply, especially the servo under load.

## Status

- [x] Ultrasonic dropped
- [x] As-built reconciliation against photos — most items confirmed (2026-08-17)
- [ ] Confirm which of AHT20/SGP40 is actually wired (or both)
- [ ] Confirm whether cap touch module is wired
- [ ] Confirm whether ESP32+MPU-6050 subsystem is actually connected — if yes, decide firmware wipe vs. thin stub
- [ ] Camera + monitor layout dry-run (check focus/framing before fixing in place)
- [ ] PI.md copied onto the Pi as CLAUDE.md, README/WIRING kept off it
- [ ] Bootstrap via gen bridge (see README.md "Bootstrapping" section)
