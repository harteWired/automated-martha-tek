# DIY Martha Tent Controller — Full Build Guide

> **⚠️ Work in progress — not yet built or tested**
>
> This guide was written collaboratively with an AI assistant and has not been physically built or validated. The design is reasonable but unverified. Review every circuit carefully before building — especially anything near mains voltage. Wait for the human author to report back if you want a tested reference before committing.

A highly-featured ESP32-S3-based controller for a Martha tent fruiting chamber. Replaces ~$233 worth of off-the-shelf CO2 and humidity controllers with a single board that gives you more accurate sensors, full data logging, and a web dashboard.

**What you replace:** dccrens' CO2 controller ($161) + Inkbird IHC200 humidity controller ($42) + Inkbird IBS-TH2 Plus monitor ($30).

**What you get instead:** real-time CO2, per-shelf humidity and temperature gradients, substrate temperature per shelf, spectral light monitoring, continuous water level, VPD calculation, web dashboard, and optional automatic reservoir top-off.

**Hardware cost: ~$290–310** (base build) — roughly the same price as the off-the-shelf controllers it replaces, with dramatically more capability.

> This guide covers hardware assembly. Firmware is in [`firmware/`](../firmware/) — see [`firmware/README.md`](../firmware/README.md) for build, flash, and configuration instructions.
>
> Reference build credit: u/mettalmag (r/MushroomGrowers, Jan 2025) — adapted from a 200sqm commercial greenhouse to a home Martha tent.
> Original post: https://www.reddit.com/r/MushroomGrowers/comments/1rao1ms/

**→ [Shopping List](https://harteWired.github.io/mushroom-growing-automation/controller-build/hardware/diy-controller-shopping-list.html)** — interactive checklist with persistent notes, syncs to your GitHub account.

## Contents

- [Safety Warning — Mains Voltage](#safety-warning--mains-voltage)
- [How It Works](#how-it-works)
- [Why ESP32-S3](#why-esp32-s3)
- [Parts List](#parts-list)
- [Tools Required](#tools-required)
- [Enclosure Layout](#enclosure-layout)
- [Wiring Guide](#wiring-guide)
- [GPIO Full IO Reference](#gpio-full-io-reference)
- [Initial Power-On Test](#initial-power-on-test)
- [Sensor Placement in the Tent](#sensor-placement-in-the-tent)
- [Security Note](#security-note)
- [Alternatives Considered](#alternatives-considered)
- [Optional Add-on: Reservoir Top-Off Pump](#optional-add-on-reservoir-top-off-pump)
- [Firmware](#firmware)

---

## ⚠️ Safety Warning — Mains Voltage

This build switches 120V AC (or 240V depending on region) mains loads. **Mains voltage can kill.** Before you start:

- All mains wiring must be inside the sealed half of the enclosure, physically separated from the low-voltage electronics
- Use 14 AWG minimum wire for mains runs; 18 AWG minimum for load leads under 10A
- Use insulated ferrules or properly rated terminals on all mains connections — no bare wire ends in screw terminals
- Always power down and unplug before touching any wiring
- If you are not comfortable with mains wiring, have a qualified electrician do the line-side portion
- Ground the enclosure

---

## How It Works

The ESP32-S3 reads all sensors and drives an 8-channel relay module that switches the tent's loads. The firmware runs two control loops:

- **Humidity loop:** SHT45 sensors measure RH → fogger + tub fan relay fires when RH drops below 85%
- **CO2/FAE loop:** SCD30 measures CO2 → exhaust + intake fan relays fire when CO2 exceeds 950 ppm

Everything else (UVC, lights) runs on timers. All data streams to a web dashboard at `http://martha.local` in real time. Physical failsafe switches on the panel face let you control any load group manually, independent of the ESP32-S3.

```
                    ┌───────────────────────────────────────┐
    SENSORS         │         ESP32-S3 DevKitC-1             │    RELAY OUTPUTS
                    │                                         │
  SCD30 (CO2) ─────┤ I2C (GPIO 21/9)        GPIO 38 ├──── Ch1: Fogger
  SHT45 ×3 ────────┤ via TCA9548A mux        GPIO 39 ├──── Ch2: Tub fan
  AS7341 (light) ──┤                         GPIO 18 ├──── Ch3: Exhaust fan
                   │                         GPIO 19 ├──── Ch4: Intake fan
  DS18B20 ×5 ──────┤ 1-Wire (GPIO 4)         GPIO 40 ├──── Ch5: UVC lights
                   │                         GPIO 41 ├──── Ch6: Grow lights
  Water level ─────┤ ADC (GPIO 7)            GPIO 42 ├──── Ch7: Pump (optional)
                   │                         GPIO 47 ├──── Ch8: Spare
                    └───────────────────────────────────────┘
                               │
                      Failsafe panel switches
                      (bypass relay board in MANUAL mode)
```

![Controller Architecture](diy-controller-wiring.svg)

---

## Why ESP32-S3

This guide is written for the **ESP32-S3 DevKitC-1**. Earlier drafts used the classic ESP32 DevKit V1; the S3 is the recommended platform for new builds because it eliminates several hardware problems that required workarounds on the V1:

| Problem (V1) | How S3 resolves it |
|---|---|
| GPIO 16/17 are UART2 — TX briefly pulses LOW during boot and flashing, firing connected relays | GPIO 38/39 (used here for Ch1/Ch2) have safe boot states and no peripheral conflicts |
| CH340 USB bridge oxidizes in 80–95% RH enclosure — USB port degrades over months | S3 has native USB-C on the chip itself; no external UART bridge or micro-USB port |
| GPIO 25/26 are DAC outputs — non-deterministic until firmware initialises them | Not used for DAC here; clean GPIO with defined boot state |
| GPIO 34 is input-only — no ESD tolerance; easy to destroy with wiring faults | GPIO 7 used instead; bidirectional with better fault tolerance (protection circuit still recommended) |

The S3 costs the same as the V1 (~$10–15). Classic ESP32 DevKit V1 can still run this firmware with the `esp32dev` PlatformIO environment — the GPIO mapping differs and the pull-ups on relay IN pins matter more.

---

## Parts List

### Core Electronics

| Component | Part | Source | Qty | Est. Cost |
|-----------|------|--------|-----|-----------|
| Microcontroller | ESP32-S3 DevKitC-1 (38-pin) | [Amazon](https://www.amazon.com/s?k=ESP32-S3+DevKitC-1) / [Adafruit](https://www.adafruit.com/product/5312) | 1 | ~$10–15 |
| Relay module | 8-channel **3.3V-compatible** opto-isolated relay | [Amazon](https://www.amazon.com/s?k=8+channel+relay+module+3.3V+optocoupler) | 1 | ~$10–12 |
| CO2 sensor | Sensirion SCD30 | [Adafruit #4867](https://www.adafruit.com/product/4867) | 1 | $58.95 |
| Temp/RH sensor | Sensirion SHT45 breakout (PTFE filter) | [Adafruit #6174](https://www.adafruit.com/product/6174) | 3 | $40.50 |
| I2C multiplexer | TCA9548A 8-channel | [Adafruit #2717](https://www.adafruit.com/product/2717) | 1 | $6.95 |
| Water level sensor | Submersible pressure sensor + converter | [DFRobot KIT0139](https://www.dfrobot.com/product-1863.html) | 1 | ~$44 |
| Substrate temp | DS18B20 waterproof probe (1m cable) | [Adafruit #381](https://www.adafruit.com/product/381) | 5 | ~$50 |
| Light sensor | AS7341 11-channel spectral | [Adafruit #4698](https://www.adafruit.com/product/4698) | 1 | ~$12 |

> **Relay module note:** The ESP32-S3 outputs 3.3V logic. Standard PC817-based relay boards are designed for 5V — at 3.3V they pull only ~2.1 mA through the optocoupler LED, which is below the PC817's reliable operating range. Source a module explicitly rated for 3.3V logic (sometimes labelled "3.3V compatible" or "3.3V trigger"). Bench-test every relay click before connecting any mains loads (see Initial Power-On Test).

### Power

| Component | Spec | Source | Qty | Est. Cost |
|-----------|------|--------|-----|-----------|
| 5V PSU | **Meanwell or Mornsun**, 5V 3A, DIN rail | [DigiKey](https://www.digikey.com/en/products/filter/power-supplies-board-mount/740?s=N4IgTCBcDaIDIBUDKBaAjABjAZgSygEYQBdAXyA) | 1 | ~$20–30 |
| 12V PSU | 12V 1A, DIN rail | [Amazon](https://www.amazon.com/s?k=12v+1a+power+supply+DIN+rail) | 1 | ~$10 |

> **PSU quality matters:** Four relays switching simultaneously draws several hundred mA in transient spikes. Generic "5V 2A" supplies from Amazon have poor transient response and can dip enough to reset the ESP32-S3 or cause relay chatter. Meanwell HDR-15-5 or Mornsun equivalents are specified here because their transient response is measured and published. Budget accordingly.

### Enclosure

| Component | Spec | Source | Qty | Est. Cost |
|-----------|------|--------|-----|-----------|
| IP65 enclosure | Min 250×200×100mm | [Amazon](https://www.amazon.com/s?k=IP65+waterproof+enclosure+250x200mm) | 1 | ~$20–30 |
| DIN rail (35mm) | Cut to enclosure width | Hardware store | 1 | ~$5 |
| Cable glands (PG9/PG11/PG16) | Sensor cables, power in, load out | [Amazon](https://www.amazon.com/s?k=PG11+cable+gland+nylon) | assorted | ~$8 |

### Safety & Protection (Required)

| Component | Spec | Purpose | Qty | Est. Cost |
|-----------|------|---------|-----|-----------|
| **GFCI outlet or DIN-rail RCD** | **30 mA trip** | Electrocution protection — first device on mains feed | **1** | **~$15–40** |
| Blade fuse block | DIN rail, 4–8 position | Per-load overcurrent protection (2–3A per channel) | 1 | ~$12–15 |
| 1kΩ resistor | ¼W | Series protection, GPIO 7 ADC input | 1 | <$1 |
| 3.3V Zener or SMBJ3V3A TVS diode | 3.3V, 500mW+ | Overvoltage clamp on ADC input | 1 | <$1 |
| 100kΩ resistor | ¼W | GPIO 7 pull-down (prevents float noise) | 1 | <$1 |

> **GFCI/RCD:** This build operates at 80–95% RH with mains-connected loads including a fogger that contacts standing water. A fuse trips at thousands of milliamps — well above the lethal threshold. A GFCI/RCD trips at 5–30 mA in under 40 ms. **In a wet environment, a GFCI/RCD is not optional.** Mount it as the first device the mains feed reaches inside the enclosure, before everything else.

### Panel & Wiring

| Component | Spec | Qty | Est. Cost |
|-----------|------|-----|-----------|
| Panel switches | DPDT toggle (master) + SPST toggle ×4 | 5 | ~$12 |
| 10kΩ resistor | Relay IN pin pull-ups, one per channel | 8 | <$1 |
| 2.2kΩ resistor | 1-Wire pull-up | 1 | <$1 |
| 100nF ceramic capacitor | 1-Wire bus decoupling | 1 | <$1 |
| Wire | 22 AWG (low voltage); 14 AWG (mains) | — | ~$10 |
| Wago 221 connectors | Push-in mains terminals | 1 pack | ~$10 |
| Ferrule crimp kit | Clean screw terminal ends (22 AWG + 14 AWG) | 1 | ~$10 |
| Terminal block strip | DIN rail, 10-position | 2 | ~$8 |
| Fuse holder + 5A fuse | Panel mount, mains feed input | 1 | ~$5 |

### Misc

| Component | Notes | Est. Cost |
|-----------|-------|-----------|
| M3 standoffs (10mm) | Mount ESP32-S3 + relay board | ~$3 |
| Zip ties / velcro | Cable management | ~$3 |
| Heat shrink tubing | Insulate splices and solder joints | ~$3 |
| Label maker tape | Label every wire at both ends | ~$2 |

**Base build total: ~$320–360** (includes GFCI/RCD and per-load fuse block — both required)

---

### Optional Add-on: Reservoir Top-Off Pump

> Relay Ch 7 and the 12V rail are pre-wired for this in the base build. Add the pump at any time — no enclosure rewiring needed.

| Component | Part | Source | Est. Cost |
|-----------|------|--------|-----------|
| 12V DC submersible pump | 3–5 L/min, food-safe | [Amazon](https://www.amazon.com/s?k=12v+DC+submersible+water+pump+aquarium) | ~$10 |
| Food-safe silicone tubing | 3/8" or 1/2" ID | Hardware/aquarium store | ~$5 |
| Check valve | Prevents backflow into pump | [Amazon](https://www.amazon.com/s?k=aquarium+check+valve+3%2F8+inch) | ~$5 |
| **Add-on total** | | | **~$20** |

When added, the firmware monitors the water level sensor and triggers Ch 7 to top off the reservoir automatically.

---

## Tools Required

- Soldering iron + solder
- Wire strippers (22 AWG and 14 AWG)
- Ferrule crimping tool
- Multimeter (essential — test every connection before powering on)
- Drill + step bit (for enclosure cutouts)
- Screwdrivers (flathead + Phillips)
- Label maker or permanent marker

---

## Enclosure Layout

Use an IP65 box of at least 250×200×100mm. Divide it into two logical zones:

```
┌─────────────────────────────────────────────────────────────┐
│  MAINS ZONE (left 1/3)         │  LOW VOLTAGE ZONE (right)  │
│                                 │                             │
│  ┌──────────────┐               │  ┌─────────────────────┐   │
│  │  5V PSU      │               │  │  ESP32-S3 DevKitC-1 │   │
│  │  12V PSU     │               │  └─────────────────────┘   │
│  └──────────────┘               │                             │
│                                 │  ┌─────────────────────┐   │
│  ┌──────────────┐               │  │  8-ch Relay Module  │   │
│  │  GFCI / RCD  │               │  └─────────────────────┘   │
│  │  (30 mA)     │               │                             │
│  └──────────────┘               │  ┌─────────────────────┐   │
│                                 │  │  Sensor breakouts   │   │
│  ┌──────────────┐               │  │  (TCA9548A, SCD30,  │   │
│  │  Mains fuse  │               │  │   AS7341)           │   │
│  │  Blade fuse  │               │  └─────────────────────┘   │
│  │  block (per- │               │                             │
│  │  load fuses) │               │                             │
│  │  Mains term- │               │                             │
│  │  inal blocks │               │                             │
│  └──────────────┘               │                             │
│   ← Loads exit via              │                             │
│     cable glands                │                             │
│     on bottom edge              │                             │
│─────────────────────────────────│                             │
│  HIGH VOLTAGE — DO NOT TOUCH    │  Relay board bridges both  │
│  UNLESS FULLY UNPLUGGED         │  zones (coil LV, contact HV)│
└─────────────────────────────────────────────────────────────┘
                    │ Panel face (lid):
                    │  [MASTER AUTO/MAN] [HUMIDITY] [FAE] [UVC] [LIGHTS]
```

> **GFCI/RCD position:** Mount the GFCI outlet or DIN-rail RCD as the first device the mains feed reaches inside the enclosure — before the input fuse, before the PSUs, before anything else. Every mains-connected load must be downstream of it.

**Cable entry points:**
- Bottom or side of enclosure via cable glands
- Mains cable: PG16 gland
- Load cables (fans, fogger, lights): PG11 glands, one per load
- Sensor cables: PG9 glands (low voltage)
- Reservoir water level sensor cable: PG9 or PG11

Mount the relay board so its **load terminals (COM/NO/NC)** face the mains zone and its **signal header (IN1–IN8, VCC, GND)** faces the low-voltage zone. This is the natural bridge between the two halves.

---

## Wiring Guide

### Step 1 — Power Supply

Wire both PSUs from a single mains input cable (via the fuse block):

```
Mains In (fused 5A)
    ├── L (Hot) ──── 5V PSU L-in ──── 12V PSU L-in
    ├── N (Neutral) ─ 5V PSU N-in ──── 12V PSU N-in
    └── PE (Earth) ── 5V PSU PE ─────── 12V PSU PE ─── Enclosure chassis

5V PSU out:
    + ──── ESP32-S3 5V pin and Relay VCC/JD-VCC
    − ──── Common GND rail

12V PSU out:
    + ──── Water level sensor supply (+ terminal)
    − ──── Common GND rail (shared with 5V −)
```

> **Important:** Share a common negative/GND between both PSUs. This ensures the water level sensor's 4-20mA return current has a proper path and the ESP32-S3 ADC reads correctly.

---

### Step 2 — Relay Module

#### Opto-isolation setup (important — don't skip)

Most 8-channel relay boards have a 3-pin header: **VCC**, **JD-VCC**, and **GND**, with a jumper between VCC and JD-VCC. **Remove this jumper.**

```
Relay board:
  VCC     ──── 3.3V from ESP32-S3 (3V3 pin)   ← optocoupler logic side
  JD-VCC  ──── 5V from PSU                      ← relay coil side
  GND     ──── Common GND
```

This creates true electrical isolation between the ESP32-S3 signal ground and the relay coil circuit, protecting the ESP32-S3 if a relay coil spikes.

> **Confirm 3.3V compatibility:** Even with the VCC/JD-VCC jumper removed, the optocoupler LED still runs from the 3.3V signal. Verify your relay module is explicitly rated for 3.3V logic before ordering. Bench-test every relay click before connecting any loads (see Initial Power-On Test).

#### Pull-up resistors on relay IN pins (belt-and-suspenders safety)

Add a 10 kΩ pull-up resistor from each relay IN pin to the relay VCC rail (3.3V). This holds every relay firmly OFF before the ESP32-S3 initialises its GPIO pins — providing a hardware backstop independent of firmware:

```
Relay VCC (3.3V) ─── 10kΩ ─── Relay IN1   (repeat for IN2 through IN8)
```

Eight resistors total. They can be soldered to the relay board's IN pin headers or placed on a small piece of stripboard.

#### Signal connections (ESP32-S3 → Relay)

These run through the **failsafe DPDT switch** in normal operation (see Step 4 before soldering these).

| Relay Channel | Load | ESP32-S3 GPIO | Notes |
|---------------|------|----------------|-------|
| IN1 | Fogger | GPIO 38 | Safe boot state; no peripheral conflicts |
| IN2 | Tub fan | GPIO 39 | Safe boot state; no peripheral conflicts |
| IN3 | Exhaust fan | GPIO 18 | Safe default HIGH |
| IN4 | Intake fan | GPIO 19 | Safe default HIGH; shares USB D− pin (inactive when UART programming is used) |
| IN5 | UVC lights | GPIO 40 | Safe default HIGH; firmware enforces 10s boot guard |
| IN6 | Grow lights | GPIO 41 | Safe default HIGH |
| IN7 | Top-off pump (optional) | GPIO 42 | Safe default HIGH |
| IN8 | Spare | GPIO 47 | Safe default HIGH |

> **Active LOW:** Relay modules trigger when the IN pin is pulled LOW. Pull-up resistors hold all IN pins HIGH (relays OFF) until the ESP32-S3 actively drives them. The firmware adds a 5-second relay-arm delay in `setup()` as an additional safety margin — no relay can fire in the first 5 seconds after boot regardless of GPIO state.

> **UVC note:** The firmware enforces a 10-second boot guard on the UVC channel specifically (5s general + 5s additional). A boot glitch that fires UV lights is a human safety hazard. Never wire UVC to a channel without this protection in firmware.

---

### Step 3 — Sensor Wiring

#### I2C bus

All I2C sensors share two wires: SDA (GPIO 21) and SCL (GPIO 9), plus 3.3V and GND. Run a 4-wire bus from the ESP32-S3 to each breakout board.

**Power all I2C breakouts from the ESP32-S3's 3V3 pin, not from the 5V PSU rail.** All sensors in this build are 3.3V native. Adafruit breakouts accept 5V on VIN via an on-board regulator, but some of their I2C pull-up resistors reference VIN rather than the regulated 3.3V rail — if that is the case, SDA/SCL lines can float to 5V, which will damage the ESP32-S3's GPIO pins and the AS7341.

> Before first power-on, check the schematic of each Adafruit breakout you are using and confirm that the on-board I2C pull-up resistors reference the 3.3V regulated rail, not VIN.
> If you use a **Waveshare AS7341** board (alternative to the Adafruit version), add an I2C level shifter — the Waveshare board does not include on-board level shifting.

```
ESP32-S3 GPIO 21 (SDA) ───┬── TCA9548A SDA
ESP32-S3 GPIO  9 (SCL) ───┼── TCA9548A SCL
                           ├── SCD30 SDA/SCL (direct on bus — I2C addr 0x61)
                           └── AS7341 SDA/SCL (direct on bus — I2C addr 0x39)
ESP32-S3 3V3 ─────────────┬── TCA9548A VIN  (3.3V — do not connect to 5V rail)
                           ├── SCD30 VIN
                           └── AS7341 VIN (3.3V only — not 5V tolerant)
ESP32-S3 GND ──────────────── all sensor GND
```

> **SCD30 SEL pin:** Tie LOW (connect to GND) to select I2C mode. On the Adafruit breakout this is handled by a default solder jumper. On a bare SCD30 module, wire the SEL pin explicitly.

**SHT45 sensors via TCA9548A multiplexer:**

All three SHT45 sensors share I2C address 0x44. The TCA9548A puts each on its own bus channel:

```
TCA9548A channel 0 ── SHT45 #1 (top shelf)    — SDA/SCL/3V3/GND
TCA9548A channel 1 ── SHT45 #2 (middle shelf) — SDA/SCL/3V3/GND
TCA9548A channel 2 ── SHT45 #3 (bottom shelf) — SDA/SCL/3V3/GND
```

TCA9548A I2C address: **0x70** (default, A0/A1/A2 all LOW).

**SCD30 placement reminder:** Mount inside the tent at mid-height on the back wall, with a small plastic baffle to deflect direct fog. Feed the I2C + power wires through a PG9 cable gland.

---

#### 1-Wire bus (DS18B20 substrate temperature probes)

```
                        100nF ceramic
                 ┌──────┤├──────┐
ESP32-S3 GPIO 4 ─┴─── 2.2kΩ ──── 3.3V    ← pull-up resistor
     │                                      (100nF suppresses cable ringing)
     └── DS18B20 data pin (yellow wire)
             ├── Probe 1 (shelf 1)
             ├── Probe 2 (shelf 2)
             ├── Probe 3 (shelf 3)
             ├── Probe 4 (shelf 4)
             └── Probe 5 (shelf 5)

DS18B20 power (red): 3.3V
DS18B20 ground (black): GND
```

All 5 probes connect to the **same three wires** — data, power, and GND — daisy-chained along the tent. Each probe has a unique factory address so the firmware reads them individually. Label each probe with its shelf number at install time.

**2.2kΩ pull-up:** Place between GPIO 4 and 3.3V. Only one is needed for the whole chain. The DS18B20 datasheet recommends 2.2 kΩ for multi-sensor chains — a 4.7 kΩ pull-up gives marginal RC rise time with 5 probes on 5–8 m of cable and can cause intermittent dropout.

**100nF decoupling capacitor:** Place a 100nF ceramic capacitor between the pull-up junction (where the 2.2kΩ connects to the data line) and GND. This suppresses cable ringing that can corrupt 1-Wire transactions.

---

#### Water level sensor (4-20mA analog)

The DFRobot KIT0139 includes the sensor and a Gravity 4-20mA converter board. Wire the converter board output to the ESP32-S3 ADC. The KIT0139 Gravity board includes overvoltage protection internally — use it.

```
12V PSU + ──────── KIT0139 sensor supply
12V PSU − / GND ── KIT0139 return / GND
                        │
               Gravity converter board (internal protection)
                        │
                0–3.3V output ─── 1kΩ ─── GPIO 7 (ADC1_CH6)
                                            │
                                       3.3V Zener            ← belt-and-suspenders clamp
                                     (anode GND,               even with KIT0139
                                      cathode GPIO 7)          internal protection
                                            │
                                       100kΩ to GND           ← prevents ADC float
                                                                  when sensor unplugged
                GND ────────────── ESP32-S3 GND
                3.3V VCC ────────── ESP32-S3 3V3
```

> **Why the external 1kΩ + Zener even with KIT0139?** The KIT0139 protects against sensor faults. The Zener on the GPIO side protects against wiring errors — e.g. accidentally connecting the 12V supply line to the signal output wire during installation. The ESP32-S3 GPIO 7 absolute maximum is 3.6V. A 3.3V Zener costs under $0.50 and is good insurance.

> **ADC non-linearity:** The ESP32-S3 ADC has meaningful non-linearity without calibration. Use `esp_adc_cal_characterize()` from ESP-IDF, 32+ sample rolling average, and 10% hysteresis on pump thresholds. The firmware implements all three. For sub-mm accuracy, an ADS1115 16-bit I2C ADC (~$3) on the same I2C bus is a clean upgrade.

Drop the sensor probe to the bottom of the 19-gallon reservoir. Route the cable through a PG9 cable gland in the enclosure.

---

### Step 4 — Failsafe Panel Switches

The failsafe allows manual load control when the ESP32-S3 is dead, rebooting, or in a bad state. Wire it using a DPDT master toggle plus 4 SPST group switches.

#### How it works — wiring topology

A single DPDT switch cannot individually control all 8 relay IN signal lines. What the DPDT actually switches is the **common reference wire** that the SPST group switches use to pull relay IN pins LOW.

- **AUTO position:** The manual SPST switches' common wire is connected to VCC (3.3V). Closing a manual switch has no effect — it can only pull an IN pin toward 3.3V, which is already its pulled-up state.
- **MANUAL position:** The DPDT connects the manual SPST switches' common wire to GND. The ESP32-S3 should have its relay GPIOs released to high-Z (the firmware does this when it detects MANUAL mode). Closing a manual switch pulls the corresponding relay IN pins LOW, firing those relays.

```
                            ┌── Pole A: manual switch common wire
DPDT switch ────────────────┤   AUTO  throw → VCC (3.3V) — manual switches inactive
                            │   MANUAL throw → GND       — manual switches active
                            │
                            └── Pole B: optional indicator LED

ESP32-S3 GPIOs are always wired to relay IN pins.
In MANUAL, firmware releases GPIOs to INPUT (high-Z); pull-ups hold relays OFF
until a manual switch is closed.
```

```
SPST group switches (each connected between manual common wire and relay IN pins):

  HUMIDITY switch ── common wire ──► Relay IN1 (fogger) + IN2 (tub fan)
  FAE switch      ── common wire ──► Relay IN3 (exhaust) + IN4 (intake)
  UVC switch      ── common wire ──► Relay IN5 (UVC)
  LIGHTS switch   ── common wire ──► Relay IN6 (grow lights)

In MANUAL mode (DPDT common = GND):
  Closing HUMIDITY switch pulls IN1 + IN2 LOW → fogger + fan fire
  Opening it releases → pull-up resistors hold IN1 + IN2 HIGH → relays off
```

**Why this is safe:** The 10 kΩ pull-up resistors on each IN pin (see Step 2) ensure that in the absence of an active LOW signal from either the ESP32-S3 or a closed manual switch, all relays default to OFF. When the ESP32-S3 is dead or rebooting, the pull-ups hold everything off until you manually override.

Mount all 5 switches on the enclosure lid face, clearly labeled.

---

### Step 5 — Mains Load Wiring

> ⚠️ Work in the mains zone only. Double-check the enclosure is unplugged.

#### 5a — GFCI/RCD (mandatory first device)

The GFCI outlet or DIN-rail RCD must be **the first device on the mains feed** when it enters the enclosure. Wire it before the fuse holder and before the PSUs:

```
Mains in (from wall) ──── GFCI/RCD (30 mA) ──── Fuse holder (5A) ──── PSUs + relay loads
```

Test the GFCI test button after installation to confirm it trips. Do not proceed to load wiring until this is confirmed functional.

#### 5b — Per-load fusing

A single 5A fuse on the mains feed protects the input cable but will not blow for a 3A fault in one load. Add a blade fuse block with one 2–3A fuse per load:

```
After relay COM terminals:
  Relay COM (Ch1 Fogger)  ──── 2A fuse ──── Fogger Live
  Relay COM (Ch2 Tub fan) ──── 2A fuse ──── Fan Live
  Relay COM (Ch3 Exhaust) ──── 2A fuse ──── Exhaust fan Live
  Relay COM (Ch4 Intake)  ──── 2A fuse ──── Intake fan Live
  Relay COM (Ch5 UVC)     ──── 2A fuse ──── UVC Live
  Relay COM (Ch6 Lights)  ──── 3A fuse ──── Lights Live
```

Size fuses at ~125% of normal load current per channel, rounded up to the next standard fuse value.

#### 5c — Load wiring

Each load uses the relay's **Normally Open (NO)** and **Common (COM)** contacts. The relay switches the **Hot (Live)** wire only. Neutral runs straight through to the load.

```
For each load:

  Mains L (Hot) ──── Relay COM ──── per-load fuse ──── Load Live wire
  Mains N (Neutral) ─ straight through ──── Load Neutral wire
  Mains PE (Earth) ── Load ground (if load is earthed)
```

**Use Wago 221 lever connectors** for all mains connections. Do not use wire nuts in an enclosure.

**Wire gauge:**
- Mains in to GFCI, fuse block, and PSUs: 14 AWG
- Load leads (fans, fogger): 18 AWG minimum; 16 AWG preferred
- Terminate all wire ends with crimped ferrules before inserting into any terminal

---

### Step 6 — Final Assembly Checklist

Before first power-on:

**Low voltage side:**
- [ ] VCC/JD-VCC jumper removed from relay board
- [ ] Relay module confirmed 3.3V compatible — bench-test required before mains loads
- [ ] 10 kΩ pull-up resistors fitted on all 8 relay IN pins (to relay VCC rail)
- [ ] All I2C sensors powered from ESP32-S3 3V3 pin (not 5V rail)
- [ ] I2C breakout pull-up resistors confirmed to reference 3.3V regulated rail (not VIN)
- [ ] 2.2kΩ pull-up on 1-Wire line (GPIO 4)
- [ ] 100nF ceramic cap at 1-Wire pull-up junction to GND
- [ ] Water level sensor output: 1kΩ series + Zener/TVS clamp fitted before GPIO 7
- [ ] 100kΩ pull-down on GPIO 7 to GND
- [ ] All sensor GNDs tied to common GND rail
- [ ] 5V PSU − and 12V PSU − both tied to common GND

**Mains side:**
- [ ] GFCI/RCD fitted and tested (Test button trips; Reset restores) — **first device on mains feed**
- [ ] All mains connections terminated with ferrules
- [ ] Wago connectors fully seated (lever closed)
- [ ] Main fuse fitted (5A) downstream of GFCI/RCD
- [ ] Per-load blade fuse block installed with correct fuse ratings per channel
- [ ] No bare wire visible at any terminal
- [ ] Relay board's COM/NO contacts on mains side, IN header on low-voltage side
- [ ] Earth/PE continuity from mains to enclosure chassis

**Enclosure:**
- [ ] All cable glands tightened with cable seated (IP65 seal requires clamping the cable)
- [ ] Lid gasket seated properly
- [ ] All switch labels clearly marked

---

## GPIO Full IO Reference

Complete pin-by-pin reference for the ESP32-S3 DevKitC-1 (38-pin variant, J1 + J3 headers).

> **Why some GPIO numbers changed from earlier drafts:** GPIO 22, 23, 24, and 25 **do not exist** in the ESP32-S3 silicon — the chip skips those numbers entirely compared to the classic ESP32. GPIO 26–32 are internally wired to the QSPI flash inside the WROOM-1 module and cannot be used for user I/O. Original assignments that fell on these numbers have been corrected throughout this guide.
>
> **Firmware update required:** `firmware/include/config.h` must be updated to the GPIO numbers below before building. See the corrected/original comparison table.

---

### Corrected vs. Original Assignments

| Function | Original GPIO | Why invalid on ESP32-S3 | Corrected GPIO |
|----------|--------------|------------------------|----------------|
| I2C SCL | 22 | GPIO 22 does not exist (S3 skips 22–25) | 9 |
| Relay Ch5 — UVC lights | 23 | GPIO 23 does not exist | 40 |
| Relay Ch6 — Grow lights | 25 | GPIO 25 does not exist | 41 |
| Relay Ch7 — Pump | 26 | Internal QSPI flash line (SPICS1) | 42 |
| Relay Ch8 — Spare | 27 | Internal QSPI flash line (SPIHD) | 47 |
| Relay Ch1 — Fogger | 32 | Internal QSPI flash line (SPID/MOSI) | 38 |
| Relay Ch2 — Tub fan | 33 | PSRAM data line in R8 variants | 39 |

---

### Pins Used by This Build

| GPIO | Direction | Function | External circuit | Notes |
|------|-----------|----------|-----------------|-------|
| 4 | Open-drain | 1-Wire data — DS18B20 ×5 | 2.2kΩ pull-up to 3.3V; 100nF ceramic at junction to GND | One pull-up + cap serves all 5 probes on the chain; 2.2kΩ sized for 5–8m cable runs |
| 7 | Input — ADC1_CH6 | Water level sensor (4–20mA converter) | 1kΩ series; 3.3V Zener/TVS to GND; 100kΩ pull-down to GND | Absolute max input 3.6V; pull-down prevents ADC float when sensor is disconnected |
| 9 | Bidirectional — I2C SCL | I2C clock bus (all sensors) | Pull-ups on sensor breakout boards | Default SCL pin in ESP32-S3 Arduino framework; replaces non-existent GPIO 22 |
| 18 | Output | Relay Ch3 — Exhaust fan | 10kΩ pull-up to relay VCC on IN3 | Active LOW; relay holds OFF when GPIO floats or is HIGH |
| 19 | Output | Relay Ch4 — Intake fan | 10kΩ pull-up to relay VCC on IN4 | Active LOW; also USB D−; only usable as GPIO when native USB CDC is inactive (normal with UART programming) |
| 21 | Bidirectional — I2C SDA | I2C data bus (all sensors) | Pull-ups on sensor breakout boards | Pull-ups must reference 3.3V regulated rail, not 5V VIN |
| 38 | Output | Relay Ch1 — Fogger | 10kΩ pull-up to relay VCC on IN1 | Active LOW; clean general-purpose GPIO; safe boot state |
| 39 | Output | Relay Ch2 — Tub fan | 10kΩ pull-up to relay VCC on IN2 | Active LOW; clean general-purpose GPIO; safe boot state |
| 40 | Output | Relay Ch5 — UVC lights | 10kΩ pull-up to relay VCC on IN5 | Active LOW; firmware enforces 10s boot guard (5s general + 5s UVC-specific) |
| 41 | Output | Relay Ch6 — Grow lights | 10kΩ pull-up to relay VCC on IN6 | Active LOW; safe boot state |
| 42 | Output | Relay Ch7 — Top-off pump (optional) | 10kΩ pull-up to relay VCC on IN7 | Active LOW; pre-wired in base build; activate with pump add-on |
| 47 | Output | Relay Ch8 — Spare | 10kΩ pull-up to relay VCC on IN8 | Active LOW; available for future expansion |

---

### Strapping Pins — Boot State Requirements

Sampled at reset. The DevKitC-1 holds them at safe defaults if left unconnected. Do not drive them with external circuits that could force the wrong state during power-on.

| GPIO | Required boot state | DevKitC-1 mechanism | Usable after boot | Notes |
|------|--------------------|--------------------|------------------|-------|
| 0 | HIGH = boot from flash; LOW = download mode | 10kΩ pull-up to 3.3V; BOOT button shorts to GND | Yes, with caution | Avoid driving with a totem-pole output; do not connect relay or other active load |
| 3 | HIGH = JTAG enabled; LOW = JTAG disabled | Internal weak pull-up | Yes | Leave NC unless hardware JTAG debugging is needed; default HIGH causes no problems |
| 45 | LOW = 3.3V VDD_SPI (normal); HIGH = 1.8V | NC — internal weak pull-down | Yes, with caution | Do NOT add an external pull-up — wrong VDD_SPI level can destroy the onboard flash |
| 46 | LOW = normal | NC — internal weak pull-down | Yes | Treat as input-only during the boot sequence; do not connect to a totem-pole output |

---

### Special-Function Pins

| GPIO | Peripheral | Status in this build | Notes |
|------|-----------|---------------------|-------|
| 19 | USB D− (native USB CDC) | In use as Relay Ch4 — Intake fan | Shared function; native USB CDC must remain inactive (use UART programming via GPIO 43/44) |
| 20 | USB D+ (native USB CDC) | NC | Leave unconnected; do not attach any load if native USB CDC might be needed |
| 43 | UART0 TX — default serial | Leave as UART | Used for firmware download and Serial debug output via the USB bridge; do not reassign |
| 44 | UART0 RX — default serial | Leave as UART | Pairs with GPIO 43 |
| 48 | On-board WS2812 RGB LED | NC — not used in firmware | Official DevKitC-1 v1.0 connects GPIO 48 to the RGB LED; safe to use as general output or to drive the LED |

---

### Available Expansion Pins

Unused GPIOs exposed on the DevKitC-1 38-pin headers. No external termination is required for pins left fully disconnected — the ESP32-S3 GPIO matrix supports configurable internal pull-up/pull-down on every pin, which can be set in firmware. In a high-humidity environment, configure unused input pins with internal pull-down to prevent noise pickup.

| GPIO | ADC / special capability | Notes |
|------|-------------------------|-------|
| 0 | Strapping / BOOT button | Spare input — caution: strapping pin; avoid driving HIGH/LOW at reset |
| 1 | ADC1_CH0, touch sensor | General purpose; good for additional analog or digital sensor |
| 2 | ADC1_CH1, touch sensor | General purpose |
| 3 | Strapping, JTAG | Leave NC unless JTAG debugging needed |
| 5 | ADC1_CH4, touch sensor | General purpose |
| 6 | ADC1_CH5, touch sensor | General purpose |
| 8 | ADC1_CH7, touch sensor | Default SDA in ESP32-S3 Arduino; general purpose if I2C stays on GPIO 21 |
| 10 | ADC1_CH9, touch sensor | General purpose |
| 11 | ADC2_CH0, touch sensor | General purpose |
| 15 | ADC2_CH4 | General purpose |
| 16 | ADC2_CH5 | General purpose |
| 17 | ADC2_CH6 | General purpose |
| 20 | USB D+ | Leave NC; do not load if native USB unused |
| 35 | PSRAM data (R8 variant only) | **NC if using an R8 module** (8MB PSRAM); usable as general GPIO on N8 (no PSRAM) |
| 36 | PSRAM data (R8 variant only) | Same as GPIO 35 |
| 37 | PSRAM data (R8 variant only) | Same as GPIO 35 |
| 45 | Strapping (VDD_SPI) | Leave NC; do not add pull-up |
| 46 | Strapping | Leave NC |
| 48 | On-board RGB LED | Drive the DevKitC-1 onboard LED, or leave NC |

---

### Do Not Use — Reserved GPIO

These GPIO numbers are wired internally inside the ESP32-S3-WROOM-1 module and cannot be used for user I/O. Attempting to write to them will corrupt flash or trigger undefined behavior.

| GPIO | Reserved for | Consequence if used |
|------|-------------|---------------------|
| 22, 23, 24, 25 | Non-existent — not implemented in ESP32-S3 silicon | Writes are silently ignored or undefined; reads return garbage |
| 26 | QSPI flash SPICS1 (chip-select) | Flash corruption; immediate crash |
| 27 | QSPI flash SPIHD (hold line) | Flash corruption |
| 28 | QSPI flash SPIWP (write-protect line) | Flash corruption |
| 29 | QSPI flash SPICS0 | Flash corruption |
| 30 | QSPI flash SPICLK (clock) | Flash corruption |
| 31 | QSPI flash SPIQ (MISO) | Flash corruption |
| 32 | QSPI flash SPID (MOSI) | Flash corruption |
| 33–37 | Octal PSRAM in R8 variants (e.g. WROOM-1 N16R8) | PSRAM corruption; may be usable on N8 no-PSRAM modules, but not recommended |

---

### Power and Control Pins

| Pin | Function | Notes |
|-----|----------|-------|
| 3V3 | 3.3V regulated output (500mA max from on-board LDO) | Powers all sensors and relay optocoupler side (VCC on relay board) |
| 5V | 5V input | Relay coil side (JD-VCC on relay board); also powers the board via VIN or USB |
| GND | Common ground | Tie both PSU negatives here; ADC accuracy depends on a solid common ground |
| EN / RST | Active-low chip reset | Pulled HIGH by on-board resistor; short to GND momentarily to reset |

---

### I2C Device Addresses

| Device | Address | Connected via |
|--------|---------|---------------|
| TCA9548A (I2C mux) | 0x70 | Direct on bus (GPIO 21/9) |
| SHT45 ×3 (shelf sensors) | 0x44 each | TCA9548A channels 0 / 1 / 2 |
| SCD30 (CO2 / temp / RH) | 0x61 | Direct on bus |
| AS7341 (spectral light) | 0x39 | Direct on bus |

---

## Initial Power-On Test

Do this before connecting any mains loads:

1. **Unplug all mains load cables** from the relay output terminals
2. Power on
3. Open a serial monitor at 115200 baud
4. **Relay boot-state check:** Immediately after power-on, before firmware runs, verify that **no relay clicks**. Any relay that fires during boot indicates a pull-up or GPIO problem — do not proceed until resolved
5. **Relay compatibility test:** Wait 10 seconds for the firmware's boot guard to expire, then manually trigger each relay from the serial console or web UI. Confirm every relay **audibly clicks** and its indicator LED illuminates. If any relay fails to click (coil hums or no response), you have a 3.3V logic incompatibility with your specific relay module — source a different module before proceeding
6. Verify each I2C sensor is detected (SCD30 at 0x61, TCA9548A at 0x70, 3× SHT45 via mux, AS7341 at 0x39)
7. Verify 5 DS18B20 addresses appear on the 1-Wire bus
8. Verify water level ADC reads a plausible value; measure voltage at GPIO 7 — should be ≤3.3V under all sensor conditions
9. Flip the MASTER switch to MANUAL, test each group switch — confirm the correct relay fires; confirm other relays stay off
10. **GFCI test:** Press the Test button — power should cut. Press Reset to restore. Do not skip this step.

Only after all sensors report, all relays confirm, and GFCI is tested: reconnect loads and test under power.

---

## Sensor Placement in the Tent

| Sensor | Position | Notes |
|--------|----------|-------|
| SCD30 (CO2) | Mid-height, back wall | Small baffle to deflect direct fog; feeds I2C cable through gland |
| SHT45 #1 | Top shelf height | Clip to tent frame |
| SHT45 #2 | Middle shelf height | Clip to tent frame |
| SHT45 #3 | Bottom shelf height | Near drip tray, away from fogger pipe |
| DS18B20 ×5 | One per shelf | Rest probe against or slightly into a fruiting block |
| AS7341 | Top of tent, center | Faces down toward blocks; measures light reaching the canopy |
| Water level | Bottom of reservoir tub | Drop sensor to floor of tub; cable exits through tub lid gland |

---

## Security Note

The web dashboard must **not** be exposed outside your local network. A relay controller accessible from the internet is a fire and safety risk.
- Use WPA2 on your local WiFi; do not set up port forwarding to this device
- For OTA firmware updates, use the built-in ElegantOTA interface at `http://martha.local/update` — this runs only on your local network

---

## Alternatives Considered

See [diy-controller-hardware-reference.md](diy-controller-hardware-reference.md) for full comparison tables and rejection reasoning for every sensor category.

**Short version:**
- CO2: **SCD30** over SCD41 (photoacoustic is sensitive to fan vibration) and MH-Z19x (no reference channel, drifts)
- Temp/RH: **SHT45 + PTFE** over SHT3x series (SHT45 has on-chip heater that recovers from high-RH creep drift; SHT3x doesn't)
- Water level: **Submersible pressure sensor** over ultrasonic JSN-SR04T (fogger mist destroys ultrasonic readings at the water surface) and XKC-Y25 capacitive (binary only — can't give continuous level)
- MCU: **ESP32-S3** over classic ESP32 DevKit V1 (native USB, no UART boot glitch on relay GPIO, better ADC support)

---

## Optional Add-on: Reservoir Top-Off Pump

The base build pre-wires relay Ch 7 and the 12V rail for this. To add the pump:

1. Mount the 12V submersible pump in a secondary reservoir (or dedicate one corner of the main tub as a clean water supply)
2. Run silicone tubing from the pump outlet to the main reservoir, terminated with a check valve
3. Connect pump leads: 12V+ → relay Ch 7 NO contact; 12V− → GND
4. Firmware monitors reservoir level continuously and runs short top-off cycles to keep it in range — no manual filling needed

---

## Firmware

Firmware is in [`firmware/`](../firmware/). See [`firmware/README.md`](../firmware/README.md) for:
- Build and flash instructions (PlatformIO)
- OTA update procedure
- Config reference (RH setpoint, CO2 thresholds, light schedules)
- REST API reference
- Hardware bring-up checklist (`firmware/test/embedded/README.md`)
