# The Self-Referential Pi

*A collaboration by Bill and Sui*

## The idea

Take a Raspberry Pi. Wire a bunch of sensors to it quasi-randomly — no documentation, no pinout notes, attention paid to power rails only (3.3V on GPIO inputs, nothing that lets the smoke out). Add a Pi camera **pointed at the Pi itself**, and maybe a monitor. Turn it on.

Then give an AI agent (Claude, or another instance) shell access and let it loose. It has to:

- Work out which pins are connected to what
- Determine which sensors actually work (some wiring may be a dead end — that's part of it)
- Build and run software to read and control everything
- Drive screen output, if a monitor is attached

The camera closes the loop: the machine can visually inspect its own wiring to disambiguate what bus scans can't tell it, and — if a monitor is present — display test patterns and watch them through its own eye. A mirror test for machines.

## Ground rules

- The agent may ask a human for help **only** when something is physically *wrong* — broken hardware, a sensor that never powers up. Everything else is its problem.
- No documentation is provided. No parts list, ideally. The less it knows going in, the better the experiment.
- Every hypothesis and dead end gets logged. The transcript of self-discovery is a research artefact in its own right.

## Why it's tractable (and where it gets hard)

- **I2C devices announce themselves.** `i2cdetect` returns addresses, and addresses map to chip families (0x68 → MPU6050, 0x76/0x77 → BMP280, …). Easy wins.
- **Analog sensors are a trap.** A bare Pi has no ADC, so an analog sensor wired straight to GPIO is useless — unless the agent notices an ADS1115 on the I2C bus and realises *that's* the path.
- **Digital one-wire and event sensors** require watching pins for state changes while things happen in the room. Detective work.
- **Display + camera** gives verifiable output: show a pattern, see the pattern, confirm the body works.

## Why bother

The usual setup is inverted: instead of a human wiring carefully and documenting pins for the AI, the AI must build an accurate model of its own body by probing. It's a closed sensorimotor loop and a small, sharp question — *can an agent discover its own nervous system?* — with an answer you can watch unfold.

## Build-day checklist

- [ ] Pi flashed, on network, SSH enabled
- [ ] Sensors wired quasi-randomly (3.3V discipline on GPIO inputs — the one mistake that ends the experiment rather than making it interesting)
- [ ] Pi camera mounted, pointed at the Pi and its wiring
- [ ] Optional: monitor connected, in camera view
- [ ] Ideally: wiring done by someone who won't be in the chat, so no hints can leak
- [ ] Agent dropped in with nothing but access and "figure out what you are"
