# The Self-Referential Pi

*A collaboration by Bill and Sui*

## The idea

Take a Raspberry Pi. Wire a bunch of sensors to it quasi-randomly — no documentation, no pinout notes, attention paid to power rails only (3.3V on GPIO inputs, nothing that lets the smoke out). Add a Pi camera **pointed at the Pi itself**, and maybe a monitor. Turn it on.

Then give an AI agent (Claude, or another instance) shell access and let it loose. It has to:

- Work out which pins are connected to what
- Determine which sensors actually work (some wiring may be a dead end — that's part of it)
- Build and run software to read and control everything

The camera closes the loop: the machine can visually inspect some of its own components to determine state. For example seeing if an LED has lit up to the correct colour or looking to see if there is a light shining on its light sensor.   
This way, by understanding itself, the agent/pi system will be able to independently prototype

## Goal

The goal is a repo that provides two things:
- A default configuration for a raspberry-pi that can be controlled by an agent
- Skills for a coding agent (such as Claude Code) to be able to access a raspberry-pi by SSH, learn the configuration of components, test them and write software to control them as intended. 

## Rules

- The agent may ask a human for help **only** when something is physically *wrong* — broken hardware, a sensor that never powers up. Everything else is its problem.
- No documentation is provided. No parts list, ideally.
- Every hypothesis and dead end gets logged. The transcript of self-discovery is a research artefact.

## How

- **I2C devices announce themselves.** `i2cdetect` returns addresses, and addresses map to chip families (0x68 → MPU6050, 0x76/0x77 → BMP280, …). Easy wins.
- **Analog sensors are a trap.** A bare Pi has no ADC, so an analog sensor wired straight to GPIO is useless — unless the agent notices an ADS1115 on the I2C bus and realises *that's* the path.
- **Digital one-wire and event sensors** require watching pins for state changes while things happen in the room. Detective work.
- **Display + camera** gives verifiable output: show a pattern, see the pattern, confirm the body works.

## Why bother

The usual setup is inverted: instead of a human wiring carefully and documenting pins for the AI, the AI must build an accurate model of attached peripherals by probing. It's a closed sensorimotor loop to investigate a question — *can we use agents to abstract all the understanding of the workings of hardware?* 

## Wiring landmines (for whoever builds it)

"Pay attention to power" needs to be sharper than just the 3.3V rail. Protocol by protocol:

- **I2C (SDA/SCL)** — fairly forgiving. Both lines are open-drain with pull-ups, so swapping SDA/SCL or landing on the wrong pin usually just fails to enumerate (nothing shows in `i2cdetect`) rather than damaging anything, *as long as everything involved is 3.3V logic*.
- **SPI (MOSI/MISO/SCLK/CS)** — push-pull, less forgiving than I2C. A random wiring that ends up with two active outputs on the same line (e.g. a sensor output landing on a pin already driven elsewhere) causes a current fight — not usually fatal at low duty, but a real risk over hours of operation.
- **UART (TX/RX)** — safest of the bunch. Swapped TX/RX just means silence, no contention.
- **Any protocol, digital sensors generally** — output-to-output shorts (two things actively driving the same pin) are the generic risk. Annoying, heat-generating, rarely instantly fatal on 3.3V CMOS.
- **The one true landmine: 5V-logic devices.** Most breakout boards from the Arduino world are 5V logic, not 3.3V-native — often true even when the datasheet buries it. A 5V signal on *any* Pi GPIO pin, on any line, in any protocol, is not something the agent can "discover gracefully" — it risks silently degrading or killing that GPIO, potentially taking a whole bank with it via the shared 3.3V rail. This is a build-time check, not a let-the-agent-find-out situation: **only 3.3V-native boards go on GPIO**, full stop, checked before power-on.

## Two documents

- **README.md** (this file) — for humans only. It tells the whole plot, including everything the agent must discover for itself (the camera, the monitor, the traps). **Never let this file reach the Pi.**
- **[PI.md](PI.md)** — the only document the machine gets: the task, the diary protocol, and the ask-a-human rule. Spoiler-free by design; it goes on the Pi as its `CLAUDE.md`.

## Build-day checklist

- [ ] Pi flashed, on network, SSH enabled
- [ ] A remote machine for the agent to access the pi
- [ ] Sensors wired quasi-randomly (3.3V discipline on GPIO inputs — the one mistake that ends the experiment rather than making it interesting)
- [ ] Pi camera mounted, pointed back at some interesting components
    - Maybe we draw on a desk with labels and arrows pointing at the components so that Claude's image model can easily understand what's going on?
- [ ] Agent dropped in with nothing but access and "figure out what you are"


## Going forward
- What if we restrict it to a smaller number of components? Could we plug the components into a pi 5 that runs an agent on boot and which has a local copy of all the documentation required.
- Instead of the pi's own camera pointing down onto the initial wiring, why not build an agentic nursery by giving the agent access to a camera pointed at a pen in which we have a freshly wired robot that the agent can control through SSH? The agent should be able to figure out how these bots can be moved around regardless of what form of locomotion they use. This is a bit like my [Wayward Robot Refuge](https://github.com/bill-mca/SIBAnetics/blob/docs/Group_Build_Design_Criteria.md#wayward-robot-refuge-) idea. 