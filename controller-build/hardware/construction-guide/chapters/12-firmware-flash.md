# Chapter 12 — Firmware Flash

**What you'll do:** Install PlatformIO, clone the firmware repository, flash the
ESP32-S3, verify sensor readings in the serial output, access the web dashboard,
and configure OTA for future updates.

**Prerequisites:** Chapter 11 complete (all 5 power-on tests passed). Computer with
internet access, USB-C cable.

---

## Step 1 — Install PlatformIO

**Option A — VS Code extension (recommended):**

1. Install [Visual Studio Code](https://code.visualstudio.com/) if needed.
2. Open Extensions (Ctrl+Shift+X / Cmd+Shift+X), search "PlatformIO IDE", install.
3. Restart VS Code. PlatformIO installs its core components on first launch.

**Option B — CLI:**
```bash
pip install platformio
```

<details>
<summary><strong>[?] PlatformIO vs Arduino IDE:</strong></summary>

PlatformIO uses a `platformio.ini` file that
specifies the exact board, framework, and library versions. When you clone the
firmware repo, PlatformIO automatically downloads the correct versions of
everything — no manual library management.

</details>

---

## Step 2 — Clone the Firmware Repository

```bash
git clone https://github.com/harteWired/mushroom-growing-automation.git
cd mushroom-growing-automation/controller-build/firmware
```

Open the `controller-build/firmware/` folder in VS Code. PlatformIO will detect `platformio.ini`
and prompt to install dependencies. Accept, or run:

```bash
pio pkg install
```

---

## Step 3 — Understand the Build Environments

Open `platformio.ini`. Two environments are defined:

```ini
[env:esp32s3]
platform = espressif32
board = esp32-s3-devkitc-1
framework = arduino
build_flags = -DBOARD_S3
...

[env:esp32dev]
platform = espressif32
board = esp32dev
framework = arduino
...
```

<details>
<summary><strong>[?] `-DBOARD_S3` build flag:</strong></summary>

This tells the compiler to apply the S3 pin
remapping in `hal.h`. Without this flag, the firmware uses DevKit V1 pin numbers,
which are wrong for this build. Always use the `esp32s3` environment.

</details>

**Always use the `esp32s3` environment.**

---

## Step 4 — First Flash

1. Connect the ESP32-S3 via USB-C.
2. Confirm the COM port is present (Device Manager on Windows; `ls /dev/tty*` on
   Linux/macOS).

**VS Code:** Click the Upload arrow (↑) in the PlatformIO toolbar.

**CLI:**
```bash
pio run -e esp32s3 --target upload
```

PlatformIO will compile (2–5 minutes first time), detect the port, flash, and reset.

**Expected output:**
```
Compiling...
Linking...
Uploading...
Writing at 0x00001000... (2%)
...
Writing at 0x000c0000... (100%)
Hash of data verified.
Hard resetting via RTS pin...
```

**If upload fails:**

*No port found:* Try a different USB-C cable (many are charge-only). On Linux:
`sudo usermod -a -G dialout $USER`

*Board won't enter programming mode automatically:* Hold BOOT, press and release
RESET, release BOOT to manually enter download mode.

*Wrong environment used:* If you see errors about missing pins or wrong chip, verify
you're building with `-e esp32s3`.

---

## Step 5 — Serial Monitor: First Boot Log

Immediately after flashing, open the serial monitor:

```bash
pio device monitor --baud 115200 --environment esp32s3
```

**Expected boot log sequence:**
```
Martha Tent Controller v0.1.0
Board: ESP32-S3 DevKitC-1
Boot #1 | Reset reason: Power-on

[HAL] FOGGER=38 TUB_FAN=39 EXHAUST=18 INTAKE=19 UVC=40 LIGHTS=41 PUMP=42 SPARE=47
[HAL] I2C SDA=21 SCL=9 | 1-Wire GPIO=4 | ADC water GPIO=7

[RELAY] Boot lock active — all relays OFF for 5000ms
[RELAY] UVC extra guard — OFF for additional 5000ms (10s total)

[I2C] Found: 0x39 (AS7341)
[I2C] Found: 0x61 (SCD30)
[I2C] Found: 0x70 (TCA9548A)
[TCA] ch0: SHT45 at 0x44
[TCA] ch1: SHT45 at 0x44
[TCA] ch2: SHT45 at 0x44

[1W] 5 devices found
[ADC] Water level: 412 mV

[WiFi] Connecting...
[WiFi] Connected. IP: 192.168.1.xxx
[mDNS] Hostname: martha.local

[RELAY] Boot lock expired
Martha ready.
```

**What to verify:**
- `Board: ESP32-S3 DevKitC-1` — confirms `BOARD_S3` flag is active
- Pin table shows S3 GPIO numbers (38/39/40/41/42/47, SCL=9, ADC=7)
- All 6 sensors detected (AS7341, SCD30, TCA9548A, 3×SHT45 via mux)
- 5 DS18B20 ROM addresses found
- IP address assigned

**If the log shows V1 pin numbers** (GPIO 16/17, SCL=22): the firmware was built with
the wrong environment. Rebuild and reflash with `pio run -e esp32s3 --target upload`.

---

## Step 6 — Sensor Reading Verification

After a minute, the serial output should show live sensor readings:

```
[SHT45-0] T: 22.4°C  RH: 54.2%  VPD: 1.26 kPa  (top shelf)
[SHT45-1] T: 22.1°C  RH: 55.0%  VPD: 1.22 kPa  (mid shelf)
[SHT45-2] T: 21.8°C  RH: 56.1%  VPD: 1.18 kPa  (bottom shelf)
[SCD30]   CO2: 412 ppm  T: 22.3°C  RH: 54.8%
[DS18B20] S1: 22.1°C  S2: 22.0°C  S3: 21.9°C  S4: 21.8°C  S5: 21.7°C
[AS7341]  Clear: 1240  F1(415nm): 210  F2(445nm): 380  ...
[WATER]   Level: 68%  Raw: 2188 mV
```

**Bench sanity checks (sensors not yet in tent):**
- Temperature: near room temperature (± 1°C)
- CO2: 400–1000 ppm (outdoor to indoor ambient)
- DS18B20: all 5 reading similar values (all at room temp)
- Water level: a raw mV value — calibration in the web UI

If any sensor shows an error, refer to Chapter 11 Test 3 troubleshooting.

---

## Step 7 — Web Dashboard

With WiFi connected:

1. Open a browser on the same local network: `http://martha.local`

   If mDNS doesn't resolve, use the IP from the serial log:
   `http://192.168.1.xxx`

2. The dashboard shows CO2, temp/RH per shelf, VPD[^1], substrate temps, water level,
   relay status, and light spectrum.

3. Go to **Settings** to:
   - Set water level sensor MIN/MAX calibration (mV at empty and full reservoir,
     measured in-situ)
   - Configure WiFi credentials
   - Adjust humidity setpoint and CO2 thresholds if needed (firmware defaults:
     RH 85%, CO2 ON at 950 ppm)

> [!WARNING]
> Do not expose the web dashboard to the internet. This controller has physical
> relay outputs — remote access from outside your local network is a fire and safety
> risk. Do not set up port forwarding to this device.

---

## Step 8 — Configure OTA

OTA (Over-The-Air) updates let you flash new firmware without opening the enclosure
or connecting a cable.

1. In Settings, enable OTA and set an OTA password.
2. Push future updates over the network:

```bash
pio run -e esp32s3 --target upload --upload-port martha.local
```

Enter the OTA password when prompted.

With OTA configured, the enclosure should not need to be reopened for firmware
updates — each time you open a sealed IP65 enclosure you risk disturbing wiring and
displacing the gasket.

---

## Chapter 12 Checkpoint

- [ ] PlatformIO installed
- [ ] Firmware repo cloned; dependencies installed
- [ ] `esp32s3` environment confirmed in `platformio.ini`
- [ ] Firmware compiled without errors
- [ ] Firmware flashed to ESP32-S3
- [ ] Boot log shows `Board: ESP32-S3 DevKitC-1` and correct S3 pin table
- [ ] All sensors detected in boot log
- [ ] Live sensor readings visible in serial output
- [ ] Web dashboard accessible at `http://martha.local` or IP
- [ ] Water level calibration MIN/MAX set in Settings
- [ ] OTA configured for future updates

---

## You're Done

The controller is assembled, tested, and running firmware. Remaining steps before
a grow:

1. Mount sensors in the tent (see the setup documentation for placement positions)
2. Connect load cables to tent equipment
3. Do a test run: verify control loops trigger at correct setpoints
4. Calibrate the water level sensor with the reservoir filled and empty
5. Set auto-refill thresholds in Settings (if using the optional pump)

[^1]: VPD (vapour pressure deficit) is the difference between the maximum water vapour the air can hold at a given temperature and how much it actually holds. It drives evaporation from mushroom surfaces and correlates with pinning success. Target range: 0.8–1.2 kPa for pinning, 1.2–2.0 kPa for fruiting bodies. The firmware calculates VPD from each SHT45 reading using the Magnus formula.

---

[← Ch 11 — Initial Power-On Test](11-power-on-test.md)
