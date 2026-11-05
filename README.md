# ESP32 Four-Level Water-Level Monitor

This package contains:

- `water_level_esp32/water_level_esp32.ino` — ESP32 firmware for four water probes, four BC547 LED stages, and ThingSpeak uploads.
- `index.html` — a small public dashboard that reads the latest four probe states from ThingSpeak.

## 1. Pin map used by the code

All sensor inputs are in the ESP32's ADC1 group, so they can be read while Wi-Fi is active.

| Function | GPIO | Notes |
|---|---:|---|
| Lowest probe / Level 1 | GPIO34 | Input only; external pull-down required |
| Level 2 probe | GPIO35 | Input only; external pull-down required |
| Level 3 probe | GPIO32 | ADC1 input |
| Highest probe / Level 4 | GPIO33 | ADC1 input |
| Common probe excitation | GPIO25 | Briefly driven HIGH during each measurement |
| LED 1 / BC547 Q1 base | GPIO19 | Through the existing 1k base resistor |
| LED 2 / BC547 Q2 base | GPIO16 | Through the existing 1k base resistor |
| LED 3 / BC547 Q3 base | GPIO21 | Through the existing 1k base resistor |
| LED 4 / BC547 Q4 base | GPIO23 | Through the existing 1k base resistor |

**The code assumes this pin map. Reconnect the wires to match it.** The latest schematic shows GPIO23 for the top LED stage; make sure the lowest LED's base resistor is connected to GPIO19, not VIN. Never use VIN as a GPIO signal.

## 2. Water-probe wiring

- Connect the common electrode to GPIO25 (not directly to 3V3) when using the pulsed-excitation code.
- Connect each level electrode to its corresponding sensor GPIO in the table.
- Keep one external pull-down resistor from each sensor GPIO to GND. The supplied code's starting thresholds assume the 10k pull-down resistors shown in the schematic.
- Tie ESP32 GND and the external 5V LED supply negative together.
- Do not connect external 5V to any ESP32 GPIO or to the 3V3 pin.

Pulsing the common electrode reduces the time DC is applied to the water; it does not eliminate electrode corrosion completely. Use corrosion-resistant probes and check them periodically.

## 3. LED/transistor wiring

For each BC547 stage:

- ESP32 LED GPIO -> 1k base resistor -> BC547 base.
- BC547 emitter -> common GND.
- 5V LED supply -> LED and its 1k series resistor -> BC547 collector (keep the LED polarity correct).
- The four collector branches remain separate; do not join a collector directly to GND.

The external 5V supply is for the LED load. The ESP32 can be powered by USB during programming. Avoid feeding VIN from an external 5V supply at the same time as USB unless the exact board's power-path design explicitly supports it.

## 4. Sensor threshold tuning

The sketch begins with:

- Wet threshold: 500 mV.
- Dry threshold: 350 mV.
- Eight averaged ADC readings per probe.
- A 15 ms probe excitation pulse every 500 ms.

Open Serial Monitor at 115200 baud and observe `Probe mV (L1 L2 L3 L4)` with the probes dry and then submerged. If wet values do not cross 500 mV, reduce `PROBE_ON_MV` in small steps. If dry values falsely trigger, increase it. Keep `PROBE_OFF_MV` lower than `PROBE_ON_MV` to preserve hysteresis.

These are starting thresholds, not a universal calibration; water conductivity, probe material, distance, and the actual resistor values affect the readings.

## 5. Create a separate ThingSpeak channel

Create a new channel named `ESP32 Water Level Detection` and enable these four fields:

1. `Level 1 Probe (0/1)` — lowest probe.
2. `Level 2 Probe (0/1)`.
3. `Level 3 Probe (0/1)`.
4. `Level 4 Probe (0/1)` — highest probe.

Make the channel **public** if the GitHub Pages dashboard should read it without a Read API Key. Copy the channel's **Write API Key** into `THINGSPEAK_WRITE_API_KEY` in `water_level_esp32/water_level_esp32.ino`. Do not place the Write API Key in `index.html` or commit it to GitHub.

The sketch uploads the four raw probe states every 20 seconds. ThingSpeak's free license currently permits a 15-second minimum update interval, so 20 seconds is within that limit. A free license currently supports up to four channels and 3 million messages per year; the existing diagnostics channel plus this new water-level channel fit within the channel limit if no other channels are in use.

## 6. Configure the dashboard

In `index.html`, set:

```js
const CHANNEL_ID = "YOUR_NEW_WATER_LEVEL_CHANNEL_ID";
```

Leave `READ_API_KEY` empty if the channel is public. The website only reads the latest entry and shows:

- Current level (0–4).
- Four individual probe states (wet/dry).
- Device online/offline based on whether the latest upload is within 90 seconds.
- Last ThingSpeak update time.

The dashboard intentionally has no large graphs or extra panels; it is designed for four-level detection only.

## 7. Arduino IDE settings

- Board: `ESP32 Dev Module`
- Port: your ESP32 CP210x COM port (COM4 in the current setup, if it remains assigned)
- Upload speed: 115200
- Flash mode: DIO
- Flash frequency: 40 MHz (the setting that worked on this board)
- Serial Monitor: 115200 baud

The sketch uses the ESP32 Arduino core libraries `WiFi`, `WiFiClientSecure`, and `HTTPClient`; no separate Arduino library installation is required.

The sketch uses `WiFiClientSecure::setInsecure()` to simplify this first HTTPS prototype. HTTPS encrypts the request, but the ESP32 does not verify the server certificate in this mode. Add certificate validation before treating the system as a security-hardened deployment.

## 8. Publish with GitHub Pages

1. Create a **public** repository named `esp32-water-level-dashboard` under your GitHub account.
2. Upload only `index.html` (do not upload the `.ino` file containing your Wi-Fi password and ThingSpeak Write API Key).
3. In the repository, open **Settings → Pages**.
4. Under Build and deployment, choose **Deploy from a branch**.
5. Select branch `main` and folder `/(root)`, then Save.
6. Wait for the Pages deployment to finish. The expected site URL for GitHub user `kyoseia` is `https://kyoseia.github.io/esp32-water-level-dashboard/`.

If the GitHub username or repository name differs, the URL will differ too.
