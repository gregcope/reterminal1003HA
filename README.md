# Home Assistant E-Ink Dashboard for Seeed Studio reTerminal E1003

A low-power, monochrome Home Assistant dashboard for the **Seeed Studio reTerminal E1003** (`1872x1404` 10.3-inch IT8951 e-paper display). Home Assistant pre-renders a Lovelace view to PNG via **Lovelace Kindle Screensaver**, while ESPHome wakes the display on the hour during active hours (`07:00` to `23:00`), fetches the image, reports battery state and returns to deep sleep.

## Repository Structure

```text
├── .gitignore
├── README.md
├── esphome-reterminal-e1003.yaml
├── reterminal-e1003-dashboard.yaml
├── reterminal_e1003_package.yaml
└── secrets.yaml.example
```

## Prerequisites

1. **ESPHome `>= 2026.7.0`** (for native `it8951` `seeed-reterminal-e1003` display support).
2. **Lovelace Kindle Screensaver** Home Assistant add-on running on port `5000`, configured for `1872x1404` resolution and pointed at `/reterminal-e1003/main`.
3. **HACS Frontend Cards:**
   - `kiosk-mode` (hides the Home Assistant sidebar and header on the rendered view; bypass with `?disable_km` when editing).
   - `apexcharts-card` (renders the 24-hour multi-line power chart and 12-hour temperature trend).

## Installation

### Step One: Add Home Assistant Helpers, REST Command and Automation

1. Enable packages in your Home Assistant `configuration.yaml` if not already enabled:
   ```yaml
   homeassistant:
     packages: !include_dir_named packages
   ```
2. Copy `reterminal_e1003_package.yaml` into `/config/packages/reterminal_e1003_package.yaml`.
3. Update `192.168.86.52:5000` in `rest_command.trigger_reterminal_renderer` if your Home Assistant host uses a different IPv4 address.
4. Restart Home Assistant (or reload YAML configurations under **Developer Tools > YAML**).

### Step Two: Create the Lovelace Dashboard

1. In Home Assistant, go to **Settings > Dashboards > Add Dashboard**, set the title to **reTerminal E1003** and set the URL path to `reterminal-e1003`.
2. Open `http://<YOUR_HA_IP>:8123/reterminal-e1003/main?disable_km`, click the pencil **Edit** icon, open the three-dot menu in the top-right corner and select **Raw configuration editor**.
3. Paste the full contents of `reterminal-e1003-dashboard.yaml`, substitute any sensor entity IDs specific to your home and click **Save**.

### Step Three: Create `secrets.yaml` from the Template

1. Copy `secrets.yaml.example` to `secrets.yaml` (or open **Secrets** in the top-right corner of **ESPHome Builder** in Home Assistant).
2. Replace the four placeholder values with your own details:
   ```yaml
   wifi_ssid: "YOUR_WIFI_SSID"
   wifi_password: "YOUR_WIFI_PASSWORD"
   ha_ip: "192.168.1.100"
   api_key: "YOUR_32_BYTE_BASE64_API_KEY"
   ```
   - `ha_ip` must be the numeric IPv4 address of the host running **Lovelace Kindle Screensaver** on port `5000`.
   - `secrets.yaml` is excluded in `.gitignore` so credentials are never committed to GitHub.

### Step Four: Flash the ESPHome Firmware

1. Create or open your `reterminal-e1003` device configuration in **ESPHome Builder** and paste the full contents of `esphome-reterminal-e1003.yaml`.
2. Click **Validate** to verify that all secrets resolve cleanly.
3. Flash the initial build over USB-C; subsequent updates can be flashed wirelessly over OTA by switching `input_boolean.eink_reterminal_enable_deepsleep` **off** first.

## Operation and Controls

- **Hourly Schedule (`07:00` – `23:00`):** At minute `55` of each hour, Home Assistant updates `input_select.e1003_time_of_day` and triggers port `5000` to render a fresh PNG. At the top of the hour, the E1003 wakes, reports battery level, downloads the image, repaints the screen only if the image changed and sleeps until the next hour.
- **Near-Hour Clock Snapping:** Any render captured from minute `45` onward snaps forward to `xx:00` so pre-rendered screenshots never display `:55` or `:59` timestamps.
- **Manual Wake (`KEY0`):** Pressing the right-hand green button (`GPIO3`) wakes the board immediately, triggers a fresh render on port `5000`, waits `40s` and forces an e-ink repaint.
