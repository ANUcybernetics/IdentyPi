# Wiring plan (humans only — never let this file near the Pi)

This is the concrete pin-by-pin build plan. It contains exact GPIO assignments, which is exactly the information [PI.md](PI.md) exists to keep from the agent. Keep this file, README.md, and any wiring photos/labels off the Pi filesystem entirely.

Bus assignments (SDA/SCL, SPI0) are fixed by hardware — there's no meaningful "random" choice there, they're real buses. Everything else is scattered across free GPIO deliberately, not grouped by function, so there's no pattern for the agent to lean on.

Split into what's actually on hand (**Phase 1**) and what's still pending (**Phase 2**) — Phase 2 pins are pre-reserved so adding them later is a plug-in, not a redesign.

## Phase 1 — on hand, wire now

Ultrasonic (HC-SR04) dropped — 5V echo pin straight into a GPIO input, not worth the risk.

### I2C bus — SDA = GPIO2 (pin 3), SCL = GPIO3 (pin 5)

No duplicate addresses in Phase 1, so everything sits directly on the bus — no mux needed yet.

| Device | Address | Notes |
|---|---|---|
| SEN0528 (AHT20 temp/humidity) | 0x38 | direct on bus |
| SEN0394 (SGP40 VOC) | 0x59 | direct on bus |
| APDS-9960 (Adafruit breakout — gesture/proximity/light) | 0x39 | direct on bus |
| HuskyLens | 0x32 | direct on bus — **must set its onboard protocol switch to I2C mode first**, it doesn't default there |
| PiicoDev RFID Module | 0x2C (default, switchable 0x2C–0x2F) | direct on bus |
| TCA9548A multiplexer | 0x70 | direct on bus |
| OLED 1.3" | address unconfirmed | **route through mux (channel 0)** until confirmed via `i2cdetect` — 1.3" and 1.5" panels may or may not share a default address |
| OLED 1.5" | address unconfirmed | **route through mux (channel 1)** until confirmed via `i2cdetect` |

### SPI0 — epaper screen (WeAct 4.2")

| Signal | GPIO (BCM) | Physical pin |
|---|---|---|
| MOSI | GPIO10 | 19 |
| MISO | GPIO9 | 21 |
| SCLK | GPIO11 | 23 |
| CE0 (chip select) | GPIO8 | 24 |
| DC (data/command) | GPIO12 | 32 |
| RST | GPIO16 | 36 |
| BUSY | GPIO20 | 38 |

### Bit-banged 3-wire — 8x8 LED matrix (MAX7219-style breakout)

Not real SPI (no MISO, it's a shift-register protocol) — deliberately kept off the real SPI0 bus and given its own GPIO so it doesn't get mistaken for genuine SPI.

| Signal | GPIO (BCM) | Physical pin |
|---|---|---|
| DIN | GPIO19 | 35 |
| CLK | GPIO26 | 37 |
| CS | GPIO21 | 40 |

Note: MAX7219 nominally wants closer to 5V for full brightness. At 3.3V it should still register/respond, just possibly dim — not a safety issue, just a "why won't this fully light up" puzzle rather than a landmine.

### 1-Wire — DS18B20 temperature sensor

Genuinely different protocol from everything else on this board — single data line, not a bus shared with I2C, and it won't enumerate at all without the `w1-gpio` kernel overlay loaded, which defaults to expecting the data line on GPIO4. Since GPIO4 is already taken (TEMT6000), the overlay needs its `gpiopin=` parameter set explicitly rather than relying on the default.

| Signal | GPIO (BCM) | Physical pin | Notes |
|---|---|---|---|
| DS18B20 data | GPIO7 | 26 | needs `dtoverlay=w1-gpio,gpiopin=7` in config.txt; needs a 4.7kΩ pull-up between data and 3.3V unless the specific breakout already has one onboard |

This was the last fully free general-purpose GPIO in the plan — everything else in the random bucket below is now spoken for.

### Scattered digital/PWM/analog GPIO (the "random" bucket)

| Device | Signal | GPIO (BCM) | Physical pin |
|---|---|---|---|
| SEN0171 (PIR motion) | digital out | GPIO17 | 11 |
| Keyestudio joystick | button (digital in) | GPIO27 | 13 |
| Keyestudio joystick | X axis (analog — **dead end, no ADC**) | GPIO22 | 15 |
| Keyestudio joystick | Y axis (analog — **dead end, no ADC**) | GPIO23 | 16 |
| TowerPro SG92R servo | PWM signal | GPIO18 | 12 |
| Keyestudio 3W LED module | PWM/digital signal | GPIO24 | 18 |
| Keyestudio TEMT6000 | analog out (**dead end, no ADC**) | GPIO4 | 7 |
| Keyestudio cap touch module | digital out | GPIO5 | 29 |
| Piezo buzzer | digital out | GPIO6 | 31 |
| Keyestudio IR module (IR_01, 3-pin: VCC/GND/OUT) | digital out | GPIO13 | 33 |

Analog dead ends are wired anyway rather than left off the desk — the agent finding a live GPIO pin that reads pure noise, and working out *why*, is part of the exercise.

GPIO25 (pin 22) is now free — the Duinotech IR proximity sensor it was reserved for turned out not to be on hand, dropped 2026-07-28.

### Relay module (SRD-03VDC-SL-C, single channel) — read the safety note before wiring

**This is a mains-rated relay (10A @ 125VAC / 240VAC on the switched contacts) — categorically different risk from everything else in this build.** Every other device here is 3.3–5V logic; the worst case elsewhere is a damaged GPIO pin. If the relay's switched terminals are ever connected to a real mains-powered load, an agent triggering it unpredictably becomes a genuine electrical safety issue, not a component-damage one.

**Rule for this project: the switched side stays disconnected, or wired only to something trivially safe (small 12V fan, LED, nothing) for the entire agent-exploration phase.** The relay's audible click and onboard indicator LED are enough to prove the agent successfully controlled an actuator — mains switching for a real purpose is a separate, deliberately human-supervised job, not something to leave for blind discovery.

Practically: the coil is 3V, low current (~80mA), so the module's VCC can run off the Pi's 3.3V rail. The IN/trigger pin is logic-level — **many boards like this are active-LOW** (pulling the pin low energizes the relay), worth confirming once wired since a floating/unconfigured GPIO at boot could otherwise cause a spurious click before any code runs.

Every general-purpose GPIO in this plan is already spoken for (GPIO7 was the last free one, used by the DS18B20 above). To fit the relay trigger in, free up GPIO14 by disabling the Pi's serial console — `raspi-config` → Interface Options → Serial Port → login shell **off**, hardware **on**. This turns GPIO14/15 back into ordinary GPIO without needing to touch anything already assigned.

| Signal | GPIO (BCM) | Physical pin | Notes |
|---|---|---|---|
| Relay IN (trigger) | GPIO14 | 8 | requires disabling serial console first; GPIO15 (pin 10) is then a spare |

### Nested subsystem — ESP32 SuperMini + 2x MPU-6050 (via USB, from Groundskeeper)

Not wired to Pi GPIO at all — connects over USB, enumerates as a serial device. The two MPU-6050s live on the ESP32's own I2C bus, not the Pi's (one at default address 0x68, the other with AD0 tied high → 0x69 — no mux needed even for two identical sensors, since the address pin resolves the collision at the hardware level).

**Open decision, not yet made:** whether to fully wipe the ESP32's current firmware before handing it over. A full wipe turns this into a genuinely nested bring-up challenge — the Pi's agent has to notice a blank USB device, realize nothing runs on it yet, and write + flash firmware onto a second, unknown microcontroller before it can learn anything from it at all. That's a much bigger task than reading a sensor and could dominate the whole session. A thin stub firmware (just streams raw I2C scan output on boot) keeps a real puzzle without requiring blind toolchain bring-up. Decide before this goes in — see [project_self_referential_pi memory] for the fuller tradeoff writeup.

### Camera, monitor, and physical layout

Both the Pi Camera v2.1 (CSI) and HuskyLens (its own onboard camera + AI + screen) go up at once, arranged so they can see: each other, a monitor, and the rest of the parts on the table.

Confirmed physical orientation: **the Pi's HDMI ports face toward the Pi Camera, and beyond it toward the HuskyLens** — so "front" of the Pi, camera, and HuskyLens all roughly line up along one axis. Use that as the layout's spine: Pi at the back, HDMI/camera side facing out toward the table, Pi Camera and HuskyLens further out along that same line, monitor beyond them facing back toward the Pi.

Practical note: both cameras are **fixed-focus, not macro** — the Pi Camera v2.1's lens is tuned roughly 1m-to-infinity, HuskyLens similar. Up close on a cluttered desk of wiring, images will likely be soft. Do a live dry-run of both feeds before gluing anything down.

## Phase 2 — not yet acquired

| Device | Notes |
|---|---|
| "bunch of temp sensors" (identical, duplicate address) | one per mux channel (2, 3, 4, …) once acquired — mux itself is already in Phase 1 |

## General wiring rules

- **Common ground, always.** Anything powered from an external supply (servo, LED driver board, any 5V-rail sensor) must share ground with the Pi, or its digital signal is meaningless even if the voltage levels are otherwise fine.
- **Servo and 3W LED module get their own external power**, never the Pi's 5V or 3.3V rail — both can pull more current than the Pi's regulator is meant to supply, especially the servo under load.
- **Relay module: switched side stays unloaded or on a trivially safe load for the whole agent-exploration phase — no mains, ever, during that phase.** See relay section above.

## Status

- [x] Ultrasonic dropped
- [x] Full Phase 1 parts identified from photo, pin plan drafted (this file)
- [x] Duinotech IR proximity sensor dropped — not actually on hand (GPIO25/pin 22 now free)
- [x] DS18B20 1-Wire thermometer added to Phase 1 (GPIO7, last free general-purpose pin)
- [x] Mux + 2x OLED (1.3", 1.5") + relay found, moved into Phase 1
- [ ] Confirm actual I2C addresses of both OLEDs via `i2cdetect` (currently both routed through mux as a safe default)
- [ ] Confirm relay active-high vs active-low before wiring
- [ ] Disable serial console (`raspi-config`) to free GPIO14 for the relay trigger
- [ ] Decide ESP32 firmware wipe vs. thin stub (see subsystem section above)
- [ ] Physical wiring — Phase 1
- [ ] Camera + monitor layout dry-run (check focus/framing before fixing in place)
- [ ] PI.md copied onto the Pi as CLAUDE.md, README/WIRING kept off it
- [ ] Phase 2 (duplicate temp sensors) wired in when ready
