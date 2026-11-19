# Home Assistant E-Ink Dashboard for Seeed Studio reTerminal E1003

A low-power, monochrome Home Assistant dashboard for the **Seeed Studio reTerminal E1003** (`1872x1404` 10.3-inch IT8951 e-paper display). Home Assistant pre-renders a Lovelace view to PNG via **Lovelace Kindle Screensaver**, while ESPHome wakes the display on a configurable daytime interval (`5` to `120` minutes) during active hours (`07:00` to `23:00`), fetches the image, reports battery state and sleeps overnight.

## Repository Structure

```text
├── .gitignore
├── README.md
├── esphomeCode.md
├── home_assistant_package_pre_render_automation.md
├── lovelace_monochrome_dashboard.md
└── secrets.yaml.example
```

## Prerequisites

1. **ESPHome `>= 2026.7.0`** (for native `it8951` `seeed-reterminal-e1003` display support).
2. **Lovelace Kindle Screensaver** Home Assistant add-on running on port `5000`, configured for `1872x1404` resolution and pointed at `/reterminal-e1003/main`.
3. **HACS Frontend Cards:**
   - `kiosk-mode` (hides the Home Assistant sidebar and header on the rendered view; bypass with `?disable_km` when editing).
   - `apexcharts-card` (renders the 24-hour multi-line power chart and 12-hour temperature trend).
4. If on a Mac you will need the CH34X USB-to-Serial MacOS Driver See https://wiki.seeedstudio.com/getting_started_with_reterminal_e1003

## Installation

### Step One: Add Home Assistant Helpers, REST Command and Automation

1. Enable packages in your Home Assistant `configuration.yaml` if not already enabled:
   ```yaml
   homeassistant:
     packages: !include_dir_named packages
   ```
2. Copy the YAML block from `home_assistant_package_pre_render_automation.md` into `/config/packages/reterminal_e1003_package.yaml` (or create the helpers and automation via **Settings > Devices & services > Helpers** and **Settings > Automations & scenes**).
3. Update `192.168.86.52:5000` in `rest_command.trigger_reterminal_renderer` if your Home Assistant host uses a different IPv4 address.
4. Restart Home Assistant (or reload YAML configurations under **Developer Tools > YAML**).

### Step Two: Create the Lovelace Dashboard

1. In Home Assistant, go to **Settings > Dashboards > Add Dashboard**, set the title to **reTerminal E1003** and set the URL path to `reterminal-e1003`.
2. Open `http://<YOUR_HA_IP>:8123/reterminal-e1003/main?disable_km`, click the pencil **Edit** icon, open the three-dot menu in the top-right corner and select **Raw configuration editor**.
3. Paste the YAML block from `lovelace_monochrome_dashboard.md`, substitute any sensor entity IDs specific to your home and click **Save**.

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

1. Create or open your `reterminal-e1003` device configuration in **ESPHome Builder** and paste the YAML block from `esphomeCode.md`.
2. Click **Validate** to verify that all secrets resolve cleanly.
3. Flash the initial build over USB-C; subsequent updates can be flashed wirelessly over OTA by switching `input_boolean.eink_reterminal_enable_deepsleep` **off** first.

## Operation and Controls

- **Configurable Daytime Schedule (`07:00` – `23:00`):** Adjust `input_number.reterminal_update_interval` (`5` to `120` minutes in five-minute steps) to set the daytime refresh cadence. Five minutes before each scheduled interval boundary during active hours, Home Assistant updates `input_select.e1003_time_of_day` and triggers port `5000` to render a fresh PNG. At the scheduled boundary, the E1003 wakes, reports battery level, downloads the image, repaints the screen only if the image changed and sleeps until the next interval (or until `07:00` overnight).
- **Five-Minute Clock Rounding:** Renders captured on or one minute after a five-minute boundary snap forward to the upcoming wake mark (`xx:00`, `xx:05`, `xx:10` and so on) so pre-rendered screenshots display the actual wake time rather than the five-minute pre-render timestamp.
- **Manual Wake (`KEY0`):** Pressing the right-hand green button (`GPIO3`) wakes the board immediately, triggers a fresh render on port `5000`, waits `40s` and forces an e-ink repaint.
