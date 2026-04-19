# Automated Martha Tek

Fully automated Martha tent fruiting chamber — CO2-controlled FAE, humidity-controlled fogger, optional DIY ESP32-S3 controller, Home Assistant monitoring, and grow tracking from inoculation through harvest.

![Build Path — Part 1 Martha Tent (~$250) is a self-contained standalone build and branches into two optional tracks: the automation track (Part 2 DIY Controller ~$290–360 → Part 3 HA Monitoring) and the biology track (Part 4 Inoculating → Part 5 Growing). Amber marks the entry point; teal marks the biology track.](./docs/images/build-path.png)

Part 1 is a complete, standalone build. Parts 2–3 replace the off-the-shelf controllers with an ESP32-S3 board — same tent, same fogger, more sensors and a web dashboard. Parts 4–5 cover what to grow and how it went.

## What It Does

- **Automated fruiting chamber** — CO2 ventilation, external fogger reservoir, cross-flow airflow, anti-mold water treatment
- **DIY ESP32-S3 controller** — SCD30 CO2, per-shelf SHT45 humidity, substrate DS18B20 temps, spectral light sensor, 8-channel relay, hardware failsafe panel
- **Firmware with web dashboard** — live sensor cards, sparklines, REST API, OTA updates, NVS config persistence
- **Home Assistant integration** — 16 sensors, 9 binary sensors, alerting automations for CO2, humidity, water level, and controller offline
- **Strain guide** — 25 species across three difficulty tiers, filterable, with vendor buy links

## Quick Start

The tent build stands on its own — start there. The controller is an optional upgrade.

```bash
# If you're building the controller firmware
cd controller-build/firmware
pio test -e native        # run tests (no hardware needed)
pio run -e esp32s3        # build for flash

# Regenerate wiring schematic SVG
cd controller-build/hardware/schematic
python generate.py
```

## How It Works

The Martha tent is a greenhouse shelving unit with automated environmental control. A fogger maintains humidity, exhaust and intake fans handle CO2 exchange, and a UVC light keeps mold in check. The off-the-shelf version uses standalone controllers with plug-in sensors — the DIY version replaces them with a single ESP32-S3 board running control loops with hysteresis deadbands.

![Controller Data Flow — three sensors on the left (SCD30 for CO2 ppm, SHT45 for RH + temperature, DS18B20 for substrate temperature) feed an ESP32-S3 running hysteresis control loops, which drives four actuators (Fogger at RH < 85%, FAE Fans at CO2 > 950 ppm, UVC on timer, Pump by water level) and publishes to two services (Dashboard at martha.local, Home Assistant over REST). Amber marks the orchestrator; teal marks the service endpoints.](./docs/images/control-flow.png)

## Documentation

| Guide | Description |
|:------|:------------|
| [Martha Tent Build Guide](martha-tent-build/martha-tent-build-guide.md) | Part 1 — complete tent assembly, wiring, and setup |
| [Martha Tent Shopping List](https://aes87.github.io/mushroom-growing-automation/martha-tent-build/martha-tent-shopping-list.html) | Interactive checklist with cost tracking |
| [DIY Controller Build Guide](controller-build/hardware/diy-controller-build-guide.md) | Part 2A — ESP32-S3 hardware build |
| [DIY Controller Shopping List](https://aes87.github.io/mushroom-growing-automation/controller-build/hardware/diy-controller-shopping-list.html) | Interactive checklist with cost tracking |
| [Construction Guide](controller-build/hardware/construction-guide/chapters/00-overview.md) | 13-chapter step-by-step controller assembly |
| [Firmware README](controller-build/firmware/README.md) | Part 2B — firmware quickstart, architecture, API reference |
| [Mushroom Strain Guide](https://aes87.github.io/mushroom-growing-automation/growing/strain-guide.html) | 25 species, filterable by difficulty, with vendor links |

> [!WARNING]
> Parts 2A and 2B are an experiment in AI-assisted hardware and firmware design. The controller guide and firmware were written collaboratively with an AI assistant and have not been physically built or tested. Review the hardware, wiring, and firmware carefully before trusting them — or wait for the author to report back.

> [!NOTE]
> Parts 3B, 4B, and 5 are placeholders. Grafana dashboards, substrate prep workflows, and grow logs are coming once the hardware is running and the first grows are underway.

## Project Structure

```
mushroom-growing-automation/
├── martha-tent-build/           # Part 1 — tent guide + shopping list HTML
├── controller-build/
│   ├── hardware/                # Part 2A — build guide, wiring SVG, construction guide
│   │   ├── construction-guide/  #   13-chapter step-by-step assembly
│   │   └── schematic/           #   Python/schemdraw SVG generator
│   └── firmware/                # Part 2B — ESP32-S3 PlatformIO firmware
├── monitoring/                  # Part 3 — home-assistant/ and grafana/
├── inoculating/                 # Part 4 — strain catalog, substrate prep
│   ├── setup/
│   └── strains/
└── growing/                     # Part 5 — fruiting, yield tracking, strain-guide.html
```

## Credits

- **Martha tent build** — u/dccrens, r/MushroomGrowers ([original post](https://www.reddit.com/r/MushroomGrowers/comments/sbnlib/))
- **DIY controller hardware reference** — u/mettalmag, r/MushroomGrowers ([original post](https://www.reddit.com/r/MushroomGrowers/comments/1rao1ms/))
