# Home Assistant E-Ink Dashboard for Seeed Studio reTerminal E1003

A low-power, monochrome Home Assistant dashboard for the **Seeed Studio reTerminal E1003** (`1872x1404` 10.3-inch IT8951 e-paper display)[cite: 2]. Home Assistant pre-renders a Lovelace view to PNG via **Lovelace Kindle Screensaver**, while ESPHome wakes the display on a configurable daytime interval (`5` to `120` minutes) during active hours (`07:00` to `23:00`), fetches the image, reports battery state and sleeps overnight[cite: 2].

## Repository Structure

```text
├── .gitignore
├── README.md
├── esphomeCode.md
├── home_assistant_package_pre_render_automation.md
├── lovelace_monochrome_dashboard.md
└── secrets.yaml.example
```
[cite: 2]

## Prerequisites

1. **ESPHome `>= 2026.7.0`** (for native `it8951` `seeed-reterminal-e1003` display support)[cite: 2].
2. **Lovelace Kindle Screensaver** Home Assistant add-on running on port `5000`, configured for `1872x1404` resolution and pointed at `/reterminal-e1003/main`[cite: 2].
3. **HACS Frontend Cards:**
   - `kiosk-mode` (hides the Home Assistant sidebar and header on the rendered view; bypass with `?disable_km` when editing)[cite: 2].
   - `apexcharts-card` (renders the 24-hour multi-line power chart and 12-hour temperature trend)[cite: 2].
4. **macOS USB Driver:** If you are on a Mac, install the CH34X USB-to-Serial macOS driver (see the [Seeed Studio reTerminal E1003 wiki](https://wiki.seeedstudio.com/getting_started_with_reterminal_e1003))[cite: 2].

## Installation

### Step One: Add Home Assistant Helpers, REST Command and Automation

1. Add the REST command (with `timeout: 60` so Home Assistant waits for the screenshot render) to your Home Assistant `configuration.yaml`[cite: 2]:
   ```yaml
   rest_command:
     trigger_reterminal_renderer:
       url: "[http://homeassistant.local:5000/?clearCache=1&refresh=1](http://homeassistant.local:5000/?clearCache=1&refresh=1)"
       method: GET
       timeout: 60
   ```
2. Copy the YAML block from `home_assistant_package_pre_render_automation.md` into `/config/packages/reterminal_e1003_package.yaml` (or create the helpers — including `input_number.reterminal_update_interval` — and automation via **Settings > Devices & services > Helpers** and **Settings > Automations & scenes**)[cite: 2].
3. Update `homeassistant.local:5000` in `rest_command.trigger_reterminal_renderer` if your renderer runs on a different host or IPv4 address[cite: 2].
4. Restart Home Assistant (or reload YAML configurations under **Developer Tools > YAML**)[cite: 2].

### Step Two: Create the Lovelace Dashboard

1. In Home Assistant, go to **Settings > Dashboards > Add Dashboard**, set the title to **reTerminal E1003** and set the URL path to `reterminal-e1003`[cite: 2].
2. Open `http://homeassistant.local:8123/reterminal-e1003/main?disable_km` (replacing `homeassistant.local` with your Home Assistant IPv4 address if required), click the pencil **Edit** icon, open the three-dot menu in the top-right corner and select **Raw configuration editor**[cite: 2].
3. Paste the YAML block from `lovelace_monochrome_dashboard.md`, substitute any sensor entity IDs specific to your home and click **Save**[cite: 2].

### Step Three: Create `secrets.yaml` from the Template

1. Copy `secrets.yaml.example` to `secrets.yaml` (or open **Secrets** in the top-right corner of **ESPHome Builder** in Home Assistant)[cite: 2].
2. Replace the four placeholder values with your own details:
   ```yaml
   wifi_ssid: "YOUR_WIFI_SSID"
   wifi_password: "YOUR_WIFI_PASSWORD"
   ha_ip: "192.168.1.100"
   api_key: "YOUR_32_BYTE_BASE64_API_KEY"
   ota_password: "YOUR_OTA_PASSWORD"
   ```
   - `ha_ip` must be the numeric IPv4 address of the host running **Lovelace Kindle Screensaver** on port `5000`[cite: 2].
   - `secrets.yaml` is excluded in `.gitignore` so credentials are never committed to GitHub[cite: 2].

### Step Four: Flash the ESPHome Firmware

1. Create or open your `reterminal-e1003` device configuration in **ESPHome Builder** and paste the YAML block from `esphomeCode.md`[cite: 2].
2. Click **Validate** to verify that all secrets resolve cleanly[cite: 2].
3. Flash the initial build over USB-C; subsequent updates can be flashed wirelessly over OTA by switching `input_boolean.eink_reterminal_enable_deepsleep` **off** first[cite: 2].

## Operation and Controls

- **Configurable Daytime Schedule (`07:00` – `23:00`):** Adjust `input_number.reterminal_update_interval` (`5` to `120` minutes in five-minute steps, default `60`) to set the daytime refresh cadence[cite: 2]. Five minutes before each scheduled interval boundary during active hours, Home Assistant updates `input_select.e1003_time_of_day` and triggers port `5000` to render a fresh PNG[cite: 2]. At the scheduled boundary, the E1003 wakes, reports battery level, downloads the image, repaints the screen only if the image changed and sleeps until the next interval (or until `07:00` overnight)[cite: 2].
- **Five-Minute Clock Rounding:** Renders captured on or one minute after a five-minute boundary snap forward to the upcoming wake mark (`xx:00`, `xx:05`, `xx:10` and so on) so pre-rendered screenshots display the actual wake time rather than the five-minute pre-render timestamp[cite: 2].
- **Manual Wake (`KEY0`):** Pressing the right-hand green button (`GPIO3`) wakes the board immediately, triggers a fresh render on port `5000`, waits `40s` and forces an e-ink repaint[cite: 2].
