# =============================================================================
# reTerminal E1003 - Home Assistant Dashboard & Deep-Sleep Firmware
# Requires ESPHome >= 2026.7.0 for the it8951 display platform.
# =============================================================================
substitutions:
  # Device naming
  device_name: reterminal-e1003
  friendly_name: "reTerminal_E1003"

  # Home Assistant helper entity IDs
  ha_enable_deepsleep: input_boolean.eink_reterminal_enable_deepsleep
  ha_enable_activehours: input_boolean.eink_reterminal_enable_activehours
  ha_activehours_start: input_datetime.eink_reterminal_enable_activehours_start
  ha_activehours_end: input_datetime.eink_reterminal_enable_activehours_end

  # Safety values used if Home Assistant or the clock is unavailable
  deep_sleep_fallback_duration: "1h"
  deep_sleep_run_duration: "3min"
  dashboard_refresh_interval: "10min"

  # Lovelace Kindle Screensaver server (must be a numeric IPv4 address)
  dashboard_renderer_host: "192.168.86.52"
  dashboard_renderer_port: "5000"
  dashboard_renderer_request_path: "/?clearCache=1&refresh=1"
  dashboard_renderer_wait: "40s"

esphome:
  name: ${device_name}
  friendly_name: ${friendly_name}
  comment: "Home Assistant dashboard display for reTerminal E1003"
  includes:
    - <driver/gpio.h>
    - <driver/rtc_io.h>
    - <lwip/sockets.h>
    - <lwip/inet.h>
    - <unistd.h>

  on_boot:
    # Release non-display peripheral pins latched before deep sleep
    - priority: 900
      then:
        - lambda: |-
            gpio_deep_sleep_hold_dis();
            gpio_hold_dis(GPIO_NUM_16);
            gpio_hold_dis(GPIO_NUM_38);
            gpio_hold_dis(GPIO_NUM_39);
            gpio_hold_dis(GPIO_NUM_45);
            gpio_hold_dis(GPIO_NUM_48);

            // Detect whether GPIO3 (green button) woke the ESP from deep sleep
            const auto wake_cause = esp_sleep_get_wakeup_cause();
            const uint64_t ext1_status = esp_sleep_get_ext1_wakeup_status();
            id(green_button_wakeup) =
              wake_cause == ESP_SLEEP_WAKEUP_EXT1 ||
              (ext1_status & (1ULL << GPIO_NUM_3)) != 0;
            ESP_LOGI("wake", "Wake cause %d; GPIO3 EXT1 status: %s",
                     static_cast<int>(wake_cause),
                     (ext1_status & (1ULL << GPIO_NUM_3)) != 0 ? "active" : "inactive");
            id(suppress_green_button_press) = id(green_button_wakeup);
            id(boot_routing_started) = false;
            id(update_cycle_complete) = false;
            id(refresh_in_progress) = false;
            id(force_display_after_download) = false;
            id(renderer_trigger_sent) = false;

    # Read hardware RTC and sample battery voltage
    - priority: 600
      then:
        - pcf8563.read_time:
        - script.execute: sample_battery

    # Read onboard SHT4x temperature and humidity
    - priority: -100
      then:
        - component.update: sht4x_sensor

  on_shutdown:
    # Turn off battery voltage divider before sleep
    - priority: 200
      then:
        - output.turn_off: bsp_battery_enable

    # Latch unused peripheral pins low (display pins GPIO11/GPIO21 excluded so IT8951 wakes cleanly)
    - priority: 50
      then:
        - lambda: |-
            // Disable I2C lines during sleep
            gpio_set_direction(GPIO_NUM_19, GPIO_MODE_DISABLE);
            gpio_set_direction(GPIO_NUM_20, GPIO_MODE_DISABLE);

            // Latch microphone, SD card, buzzer and touch reset low
            gpio_set_direction(GPIO_NUM_38, GPIO_MODE_OUTPUT);
            gpio_set_level(GPIO_NUM_38, 0);
            gpio_hold_en(GPIO_NUM_38);

            gpio_set_direction(GPIO_NUM_39, GPIO_MODE_OUTPUT);
            gpio_set_level(GPIO_NUM_39, 0);
            gpio_hold_en(GPIO_NUM_39);

            gpio_set_direction(GPIO_NUM_45, GPIO_MODE_OUTPUT);
            gpio_set_level(GPIO_NUM_45, 0);
            gpio_hold_en(GPIO_NUM_45);

            gpio_set_direction(GPIO_NUM_48, GPIO_MODE_OUTPUT);
            gpio_set_level(GPIO_NUM_48, 0);
            gpio_hold_en(GPIO_NUM_48);

            gpio_deep_sleep_hold_en();

            // Keep RTC peripheral powered so active-low GPIO3 can wake via EXT1
            esp_sleep_pd_config(ESP_PD_DOMAIN_RTC_PERIPH, ESP_PD_OPTION_ON);
            rtc_gpio_pullup_en(GPIO_NUM_3);
            rtc_gpio_pulldown_dis(GPIO_NUM_3);

            ESP_LOGI("power", "Peripheral pins prepared for low-leakage sleep");

esp32:
  board: seeed_xiao_esp32s3
  framework:
    type: esp-idf

# OPI PSRAM stores the 16-level grayscale framebuffer (1872x1404)
psram:
  mode: octal

# Serial logging via onboard CH340K USB-to-UART bridge on UART0
logger:
  hardware_uart: UART0
  baud_rate: 115200
  level: INFO

api:
  encryption:
    key: !secret api_key
  on_client_connected:
    then:
      # Re-sample and publish battery percentage once Home Assistant connects
      - script.execute: sample_battery

ota:
  - platform: esphome
    password: api_key

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password
  power_save_mode: HIGH
  fast_connect: true
  ap:
    ssid: "reTerminal-E1003"
    password: "ChangeMe123"

  on_connect:
    then:
      - delay: 5s
      - lambda: |-
          id(boot_routing_started) = true;
          id(update_cycle_complete) = false;
      - if:
          condition:
            lambda: |-
              return id(green_button_wakeup);
          then:
            - logger.log:
                level: INFO
                format: "Green button wake; forcing dashboard render"
            - if:
                condition:
                  lambda: |-
                    return !id(refresh_in_progress);
                then:
                  - script.execute: force_dashboard_refresh
                else:
                  - logger.log:
                      level: DEBUG
                      format: "Refresh already active; ignoring duplicate wake request"
            - lambda: |-
                id(green_button_wakeup) = false;
                id(suppress_green_button_press) = false;
          else:
            - if:
                condition:
                  lambda: |-
                    if (!id(ha_enable_activehours).has_state() ||
                        !id(ha_enable_activehours).state) {
                      return true;
                    }

                    auto now = id(rtc_time).now();
                    if (!now.is_valid()) now = id(ha_time).now();
                    if (!now.is_valid()) return true;

                    const int current = now.hour * 60 + now.minute;
                    const int start = id(activehours_start_minutes);
                    const int end = id(activehours_end_minutes);
                    if (start == end) return true;
                    if (start < end) return current >= start && current < end;
                    return current >= start || current < end;
                then:
                  - logger.log:
                      level: INFO
                      format: "Wi-Fi connected; checking dashboard image"
                  - component.update: ha_dashboard_image
                else:
                  - logger.log:
                      level: INFO
                      format: "Outside active hours; skipping dashboard update"
                  - lambda: |-
                      id(update_cycle_complete) = true;
                  - script.execute: sleep_if_enabled

captive_portal:

http_request:
  timeout: 60s
  verify_ssl: false

font:
  - file: "gfonts://Inter@700"
    id: font_small
    size: 20
  - file: "gfonts://Inter@700"
    id: font_medium
    size: 32
  - file: "gfonts://Inter@700"
    id: font_large
    size: 48

# SPI — shared by ePaper display and SD card
spi:
  id: epaper_spi_bus
  clk_pin: GPIO7
  mosi_pin: GPIO9
  miso_pin: GPIO8

# I2C — shared by SHT4x sensor and PCF8563 RTC
i2c:
  scl: GPIO20
  sda: GPIO19
  scan: true
  frequency: 100kHz

output:
  # Battery measurement voltage divider enable
  - platform: gpio
    pin: GPIO40
    id: bsp_battery_enable

  # Buzzer PWM output
  - platform: ledc
    pin: GPIO45
    id: buzzer_pwm
    frequency: 1000Hz

light:
  - platform: monochromatic
    output: buzzer_pwm
    name: "Buzzer"
    id: buzzer
    default_transition_length: 0s

globals:
  # Tracks whether wifi.on_connect has routed the current boot
  - id: boot_routing_started
    type: bool
    restore_value: false
    initial_value: "false"

  # Forces an e-ink repaint after a manual green-button refresh
  - id: force_display_after_download
    type: bool
    restore_value: false
    initial_value: "false"

  # True when GPIO3 woke the ESP from deep sleep
  - id: green_button_wakeup
    type: bool
    restore_value: false
    initial_value: "false"

  # Prevents duplicate refresh from a held wake button
  - id: suppress_green_button_press
    type: bool
    restore_value: false
    initial_value: "false"

  # Guards against overlapping render requests
  - id: refresh_in_progress
    type: bool
    restore_value: false
    initial_value: "false"

  # Confirms the TCP trigger packet reached port 5000
  - id: renderer_trigger_sent
    type: bool
    restore_value: false
    initial_value: "false"

  # Active-hours window in minutes past midnight
  - id: activehours_start_minutes
    type: int
    restore_value: false
    initial_value: "420"  # 07:00

  - id: activehours_end_minutes
    type: int
    restore_value: false
    initial_value: "1380"  # 23:00

  # Marks whether the current wake cycle has finished downloading
  - id: update_cycle_complete
    type: bool
    restore_value: false
    initial_value: "false"

sensor:
  # Battery voltage (ADC on GPIO1 with one-to-two divider compensation)
  - platform: adc
    pin: GPIO1
    name: "Battery Voltage"
    id: battery_voltage
    update_interval: never
    attenuation: 12db
    accuracy_decimals: 2
    device_class: voltage
    state_class: measurement
    filters:
      - multiply: 2.0

  # Battery level percentage (datapoints sorted in ascending order)
  - platform: template
    name: "Battery Level"
    id: battery_level
    unit_of_measurement: "%"
    icon: "mdi:battery"
    device_class: battery
    state_class: measurement
    accuracy_decimals: 1
    lambda: 'return id(battery_voltage).state;'
    update_interval: never
    filters:
      - calibrate_linear:
          method: exact
          datapoints:
            - 3.27 -> 0.0
            - 3.30 -> 5.0
            - 3.41 -> 10.0
            - 3.49 -> 20.0
            - 3.58 -> 30.0
            - 3.68 -> 40.0
            - 3.75 -> 50.0
            - 3.80 -> 60.0
            - 3.85 -> 70.0
            - 3.91 -> 80.0
            - 3.96 -> 90.0
            - 4.15 -> 100.0
      - clamp:
          min_value: 0
          max_value: 100

  # Onboard SHT4x temperature and humidity
  - platform: sht4x
    id: sht4x_sensor
    address: 0x44
    update_interval: never
    temperature:
      name: "Temperature"
      id: temp_sensor
      accuracy_decimals: 1
    humidity:
      name: "Relative Humidity"
      id: hum_sensor
      accuracy_decimals: 1

time:
  # Onboard PCF8563 RTC
  - platform: pcf8563
    id: rtc_time
    address: 0x51
    update_interval: never

  # Home Assistant time sync writes directly to the hardware RTC
  - platform: homeassistant
    id: ha_time
    on_time_sync:
      then:
        - pcf8563.write_time:
            id: rtc_time

text_sensor:
  # Active-hours start helper
  - platform: homeassistant
    id: ha_activehours_start_time
    entity_id: ${ha_activehours_start}
    on_value:
      then:
        - lambda: |-
            int hour = 0, minute = 0, second = 0;
            if (sscanf(x.c_str(), "%d:%d:%d", &hour, &minute, &second) >= 2 &&
                hour >= 0 && hour <= 23 && minute >= 0 && minute <= 59) {
              id(activehours_start_minutes) = hour * 60 + minute;
              ESP_LOGI("active_hours", "Active hours start: %02d:%02d", hour, minute);
            } else {
              ESP_LOGW("active_hours", "Invalid start helper value: %s", x.c_str());
            }

  # Active-hours end helper
  - platform: homeassistant
    id: ha_activehours_end_time
    entity_id: ${ha_activehours_end}
    on_value:
      then:
        - lambda: |-
            int hour = 0, minute = 0, second = 0;
            if (sscanf(x.c_str(), "%d:%d:%d", &hour, &minute, &second) >= 2 &&
                hour >= 0 && hour <= 23 && minute >= 0 && minute <= 59) {
              id(activehours_end_minutes) = hour * 60 + minute;
              ESP_LOGI("active_hours", "Active hours end: %02d:%02d", hour, minute);
            } else {
              ESP_LOGW("active_hours", "Invalid end helper value: %s", x.c_str());
            }

  # Diagnostic timestamp when PNG was last fetched
  - platform: template
    id: dashboard_last_checked
    name: "Dashboard Last Checked"
    icon: mdi:cloud-check-outline
    device_class: timestamp
    entity_category: diagnostic
    update_interval: never

  # Diagnostic timestamp when e-ink display finished repainting
  - platform: template
    id: display_last_updated
    name: "Display Last Updated"
    icon: mdi:monitor-dashboard
    device_class: timestamp
    entity_category: diagnostic
    update_interval: never

binary_sensor:
  # Deep-sleep enable toggle imported from Home Assistant
  - platform: homeassistant
    id: ha_enable_deepsleep
    entity_id: ${ha_enable_deepsleep}
    trigger_on_initial_state: true
    on_state:
      then:
        - if:
            condition:
              binary_sensor.is_on: ha_enable_deepsleep
            then:
              - logger.log:
                  level: INFO
                  format: "Home Assistant enabled deep sleep"
              - deep_sleep.allow: deep_sleep_control
              - if:
                  condition:
                    lambda: |-
                      return id(boot_routing_started) &&
                             id(update_cycle_complete);
                  then:
                    - script.execute: sleep_if_enabled
            else:
              - logger.log:
                  level: INFO
                  format: "Home Assistant disabled deep sleep"
              - deep_sleep.prevent: deep_sleep_control

  # Active-hours enable toggle imported from Home Assistant
  - platform: homeassistant
    id: ha_enable_activehours
    entity_id: ${ha_enable_activehours}

  # KEY0 Right Green Button (GPIO3 — wakes device from deep sleep and forces refresh)
  - platform: gpio
    use_interrupt: false
    pin:
      number: GPIO3
      allow_other_uses: true
      mode: INPUT_PULLUP
      inverted: true
    id: button_1
    name: "KEY0 Right Green Button"
    icon: mdi:refresh
    filters:
      - delayed_on: 20ms
      - delayed_off: 20ms
    on_press:
      then:
        - if:
            condition:
              lambda: |-
                return !id(suppress_green_button_press) &&
                       !id(refresh_in_progress);
            then:
              - script.execute: force_dashboard_refresh
            else:
              - logger.log:
                  level: DEBUG
                  format: "Ignoring duplicate green-button press"

  # KEY1 Middle Button (GPIO4)
  - platform: gpio
    pin:
      number: GPIO4
      mode: INPUT_PULLUP
      inverted: true
    id: button_2
    name: "KEY1 Middle Button"
    filters:
      - delayed_on: 20ms
      - delayed_off: 20ms
    on_press:
      then:
        - logger.log: "KEY1 (GPIO4) pressed"

  # KEY2 Left Button (GPIO5)
  - platform: gpio
    pin:
      number: GPIO5
      mode: INPUT_PULLUP
      inverted: true
    id: button_3
    name: "KEY2 Left Button"
    filters:
      - delayed_on: 20ms
      - delayed_off: 20ms
    on_press:
      then:
        - logger.log: "KEY2 (GPIO5) pressed"

script:
  # Enables voltage divider briefly to read battery ADC
  - id: sample_battery
    mode: restart
    then:
      - output.turn_on: bsp_battery_enable
      - delay: 200ms
      - component.update: battery_voltage
      - component.update: battery_level
      - output.turn_off: bsp_battery_enable

  # Sends non-blocking TCP trigger to port 5000, waits 40s, then downloads PNG
  - id: force_dashboard_refresh
    mode: single
    then:
      - lambda: |-
          id(refresh_in_progress) = true;
          id(renderer_trigger_sent) = false;
      - logger.log:
          level: INFO
          format: "Requesting a new renderer screenshot"
      - lambda: |-
          const int sock = ::socket(AF_INET, SOCK_STREAM, IPPROTO_TCP);
          if (sock < 0) {
            ESP_LOGE("renderer", "Could not create renderer trigger socket");
            return;
          }

          struct timeval timeout;
          timeout.tv_sec = 2;
          timeout.tv_usec = 0;
          ::setsockopt(sock, SOL_SOCKET, SO_SNDTIMEO, &timeout, sizeof(timeout));
          ::setsockopt(sock, SOL_SOCKET, SO_RCVTIMEO, &timeout, sizeof(timeout));

          struct sockaddr_in address = {};
          address.sin_family = AF_INET;
          address.sin_port = ::htons(${dashboard_renderer_port});
          if (::inet_pton(AF_INET, "${dashboard_renderer_host}",
                          &address.sin_addr) != 1 ||
              ::connect(sock, reinterpret_cast<struct sockaddr *>(&address),
                        sizeof(address)) != 0) {
            ESP_LOGE("renderer", "Could not connect to dashboard renderer");
            ::close(sock);
            return;
          }

          const char request[] =
            "GET ${dashboard_renderer_request_path} HTTP/1.1\r\n"
            "Host: ${dashboard_renderer_host}:${dashboard_renderer_port}\r\n"
            "Connection: close\r\n\r\n";
          size_t sent = 0;
          while (sent < sizeof(request) - 1) {
            const int result = ::send(sock, request + sent,
                                      sizeof(request) - 1 - sent, 0);
            if (result <= 0) {
              break;
            }
            sent += static_cast<size_t>(result);
          }
          ::shutdown(sock, SHUT_RDWR);
          ::close(sock);
          id(renderer_trigger_sent) = sent == sizeof(request) - 1;
      - if:
          condition:
            lambda: |-
              return id(renderer_trigger_sent);
          then:
            - logger.log:
                level: INFO
                format: "Renderer triggered; waiting for background render"
            - delay: ${dashboard_renderer_wait}
            - logger.log:
                level: INFO
                format: "Renderer wait complete; downloading dashboard image"
            - lambda: |-
                id(force_display_after_download) = true;
            - component.update: ha_dashboard_image
          else:
            - lambda: |-
                id(force_display_after_download) = false;
                id(update_cycle_complete) = true;
                id(refresh_in_progress) = false;
            - logger.log:
                level: ERROR
                format: "Could not trigger dashboard renderer"
            - script.execute: sleep_if_enabled

  # Calculates seconds until the next top-of-hour inside active hours and sleeps
  - id: sleep_if_enabled
    mode: single
    then:
      - if:
          condition:
            lambda: |-
              return id(ha_enable_deepsleep).has_state() &&
                     id(ha_enable_deepsleep).state;
          then:
            - delay: 2s
            - logger.log:
                level: INFO
                format: "Update cycle complete; entering deep sleep"
            - deep_sleep.enter:
                id: deep_sleep_control
                sleep_duration: !lambda |-
                  auto now = id(rtc_time).now();
                  if (!now.is_valid()) now = id(ha_time).now();
                  if (!now.is_valid()) {
                    ESP_LOGW("sleep_schedule", "Clock unavailable; using one-hour fallback");
                    return 3600000UL;
                  }

                  const uint32_t seconds_now =
                    now.hour * 3600UL + now.minute * 60UL + now.second;
                  uint32_t seconds_to_hour = 3600UL - (seconds_now % 3600UL);
                  // Roll forward if within three minutes of the hour to avoid :59 drift double-wakes
                  if (seconds_to_hour <= 180UL) seconds_to_hour += 3600UL;
                  seconds_to_hour += 10UL;

                  uint32_t sleep_seconds = seconds_to_hour;
                  const bool use_active_hours =
                    id(ha_enable_activehours).has_state() &&
                    id(ha_enable_activehours).state;

                  if (use_active_hours) {
                    const int start_minutes = id(activehours_start_minutes);
                    const int end_minutes = id(activehours_end_minutes);

                    if (start_minutes != end_minutes) {
                      uint32_t next_allowed_hour = 0;
                      for (int i = 0; i < 48; i++) {
                        const uint32_t candidate = seconds_to_hour + i * 3600UL;
                        const int candidate_minutes =
                          ((seconds_now + candidate) % 86400UL) / 60UL;
                        const bool allowed = start_minutes < end_minutes
                          ? candidate_minutes >= start_minutes && candidate_minutes < end_minutes
                          : candidate_minutes >= start_minutes || candidate_minutes < end_minutes;
                        if (allowed) {
                          next_allowed_hour = candidate;
                          break;
                        }
                      }

                      const uint32_t start_seconds = start_minutes * 60UL;
                      uint32_t seconds_to_start =
                        start_seconds > seconds_now
                          ? start_seconds - seconds_now
                          : 86400UL - seconds_now + start_seconds;
                      seconds_to_start += 10UL;

                      if (next_allowed_hour == 0 || seconds_to_start < next_allowed_hour) {
                        sleep_seconds = seconds_to_start;
                      } else {
                        sleep_seconds = next_allowed_hour;
                      }
                    }
                  }

                  ESP_LOGI(
                    "sleep_schedule",
                    "Sleeping %lu seconds; next wake at %02u:%02u",
                    (unsigned long) sleep_seconds,
                    (unsigned) ((seconds_now + sleep_seconds) % 86400UL / 3600UL),
                    (unsigned) (((seconds_now + sleep_seconds) % 3600UL) / 60UL)
                  );
                  return sleep_seconds * 1000UL;

image:
  # Downloads pre-rendered 1872x1404 PNG from Lovelace Kindle Screensaver
  - platform: online_image
    id: ha_dashboard_image
    url: "http://${dashboard_renderer_host}:${dashboard_renderer_port}/"
    format: PNG
    type: GRAYSCALE
    resize: 1872x1404
    update_interval: never
    buffer_size: 65536

    on_download_finished:
      then:
        - lambda: |-
            id(update_cycle_complete) = true;
            id(refresh_in_progress) = false;
            auto now = id(ha_time).utcnow();
            if (now.is_valid()) {
              id(dashboard_last_checked).publish_state(
                now.strftime("%Y-%m-%dT%H:%M:%SZ")
              );
            }
        - if:
            condition:
              lambda: |-
                return !cached || id(force_display_after_download);
            then:
              - logger.log:
                  level: INFO
                  format: "Refreshing dashboard display"
              - it8951.update:
                  id: epaper_display
                  mode: GC16
              - lambda: |-
                  id(force_display_after_download) = false;
                  auto now = id(ha_time).utcnow();
                  if (now.is_valid()) {
                    id(display_last_updated).publish_state(
                      now.strftime("%Y-%m-%dT%H:%M:%SZ")
                    );
                  }
            else:
              - logger.log:
                  level: DEBUG
                  format: "Dashboard image unchanged; skipping e-ink refresh"
        - script.execute: sleep_if_enabled

    on_error:
      then:
        - lambda: |-
            id(force_display_after_download) = false;
            id(update_cycle_complete) = true;
            id(refresh_in_progress) = false;
        - logger.log:
            level: ERROR
            format: "Dashboard image download or decoding failed"
        - script.execute: sleep_if_enabled

# ePaper display (10.3" 16-level grayscale, 1872x1404, IT8951)
display:
  - platform: it8951
    id: epaper_display
    spi_id: epaper_spi_bus
    model: seeed-reterminal-e1003
    sleep_when_done: true
    update_interval: never
    full_update_every: 30
    grayscale: true
    dithering: false
    update_mode: GC16
    transform:
      mirror_x: true
      mirror_y: false
    lambda: |-
      it.fill(Color::WHITE);
      if (id(ha_dashboard_image).is_ready()) {
        it.image(0, 0, id(ha_dashboard_image));
      } else {
        it.printf(
          it.get_width() / 2,
          it.get_height() / 2,
          id(font_large),
          Color::BLACK,
          TextAlign::CENTER,
          "Waiting for Home Assistant dashboard..."
        );
      }

# Polling fallback when deep sleep is disabled in Home Assistant
interval:
  - interval: ${dashboard_refresh_interval}
    then:
      - if:
          condition:
            lambda: |-
              return !id(ha_enable_deepsleep).has_state() ||
                     !id(ha_enable_deepsleep).state;
          then:
            - logger.log:
                level: DEBUG
                format: "Checking for an updated dashboard image"
            - component.update: ha_dashboard_image

  - interval: 10min
    then:
      - if:
          condition:
            lambda: |-
              return !id(ha_enable_deepsleep).has_state() ||
                     !id(ha_enable_deepsleep).state;
          then:
            - component.update: sht4x_sensor
            - script.execute: sample_battery

deep_sleep:
  id: deep_sleep_control
  run_duration: ${deep_sleep_run_duration}
  sleep_duration: ${deep_sleep_fallback_duration}
  esp32_ext1_wakeup:
    pins:
      - number: GPIO3
        allow_other_uses: true
    mode: ANY_LOW

button:
  - platform: template
    name: "Check for dashboard update"
    id: check_dashboard_update
    icon: mdi:download
    entity_category: diagnostic
    on_press:
      then:
        - component.update: ha_dashboard_image

  - platform: template
    name: "Force display repaint"
    id: force_display_repaint
    icon: mdi:monitor
    entity_category: diagnostic
    on_press:
      then:
        - it8951.update:
            id: epaper_display
            mode: GC16
        - lambda: |-
            auto now = id(ha_time).utcnow();
            if (now.is_valid()) {
              id(display_last_updated).publish_state(
                now.strftime("%Y-%m-%dT%H:%M:%SZ")
              );
            }

  - platform: restart
    name: "Restart"
    entity_category: diagnostic
# === END OF RETERMINAL E1003 CONFIGURATION ===
