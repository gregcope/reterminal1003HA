kiosk_mode:
  hide_sidebar: true
  hide_header: true
  admin_settings:
    ignore_kiosk_mode: true
views:
  - title: Main
    path: main
    type: panel
    theme: eink_mono
    cards:
      - type: grid
        square: false
        columns: 2
        cards:
          # Left Column: Clock, Five Key Sensors & Both Narrowed Line Charts
          - type: vertical-stack
            cards:
              - type: markdown
                content: |
                  {% set t = now() + timedelta(minutes=(60 - now().minute) if now().minute >= 45 else 0) %}
                  # {{ t.strftime('%H:00') if (t.minute <= 15) else t.strftime('%H:%M') }}
                  ### {{ t.strftime('%A, %d %B %Y') }}
              - type: glance
                show_state: true
                columns: 5
                entities:
                  - entity: sensor.i_9psl_temperature
                    name: Lounge
                  - entity: sensor.ruuvitag_0440_temperature
                    name: Outside
                  - entity: sensor.xiamo_6_temperature
                    name: Garage
                  - entity: sensor.ecodan_ashp_dhw_temperature
                    name: DHW
                  - entity: sensor.living_room_reterminal_e1003_battery_level
                    name: Screen
              - type: custom:apexcharts-card
                header:
                  show: true
                  title: Today's Power (W)
                  show_states: false
                graph_span: 24h
                span:
                  start: day
                yaxis:
                  - min: 0
                    max: auto
                apex_config:
                  chart:
                    type: line
                    width: '90%'
                    height: 175
                    animations:
                      enabled: false
                  stroke:
                    width: 2.5
                  grid:
                    show: false
                  xaxis:
                    labels:
                      style:
                        colors: '#000000'
                        fontSize: 10px
                  yaxis:
                    forceNiceScale: true
                    decimalsInFloat: 0
                    labels:
                      style:
                        colors: '#000000'
                        fontSize: 10px
                  legend:
                    show: true
                    position: bottom
                    fontSize: 10px
                    labels:
                      colors: '#000000'
                series:
                  - entity: sensor.grid_power
                    name: Grid
                    color: '#000000'
                    stroke_dash: 0
                    curve: stepline
                  - entity: sensor.solar_shellypmminig3_1_shellypmminig3_solar_power_positive
                    name: Solar
                    color: '#444444'
                    stroke_dash: 6
                    curve: smooth
                  - entity: sensor.heat_pump_power
                    name: ASHP
                    color: '#777777'
                    stroke_dash: 2
                    curve: stepline
              - type: custom:apexcharts-card
                header:
                  show: true
                  title: Temperature Trend (12h)
                  show_states: false
                graph_span: 12h
                yaxis:
                  - min: auto
                    max: auto
                apex_config:
                  chart:
                    type: line
                    width: '90%'
                    height: 175
                    animations:
                      enabled: false
                  stroke:
                    width: 2.5
                  grid:
                    borderColor: '#bbbbbb'
                    strokeDashArray: 3
                  xaxis:
                    labels:
                      style:
                        colors: '#000000'
                        fontSize: 10px
                  yaxis:
                    forceNiceScale: true
                    decimalsInFloat: 1
                    labels:
                      style:
                        colors: '#000000'
                        fontSize: 10px
                  legend:
                    show: true
                    position: bottom
                    fontSize: 10px
                    labels:
                      colors: '#000000'
                series:
                  - entity: sensor.i_9psl_temperature
                    name: Lounge
                    color: '#000000'
                    stroke_dash: 0
                    curve: smooth
                  - entity: sensor.ruuvitag_0440_temperature
                    name: Outside
                    color: '#555555'
                    stroke_dash: 5
                    curve: smooth

          # Right Column: Weather, Dense Two-Column Data Tables & Mode Status
          - type: vertical-stack
            cards:
              - type: weather-forecast
                entity: weather.forecast_home
                name: Weather Forecast
                show_current: true
                show_forecast: true
                forecast_type: daily
              - type: grid
                square: false
                columns: 2
                cards:
                  - type: markdown
                    content: |
                      ### Today's Energy & Tariff
                      | Metric | Value |
                      | :--- | :--- |
                      | Grid Used Today | **{{ states('sensor.garage_grid_daily_energy_produced') | float(0) | round(2) }} kWh** |
                      | Solar Generated | **{{ states('sensor.shellypmminig3_daily_energy_produced_pos') }} {{ state_attr('sensor.shellypmminig3_daily_energy_produced_pos', 'unit_of_measurement') or 'Wh' }}** |
                      | Today's CoP | **{{ states('sensor.ecodan_ashp_total_cop_today') | float(0) | round(2) }}** |
                      | Current Rate | **{{ (states('sensor.octopus_energy_electricity_19l3396601_2000016825050_current_rate') | float(0) * 100) | round(1) }}p** |
                  - type: markdown
                    content: |
                      ### MVHR
                      | Metric | Value |
                      | :--- | :--- |
                      | Supply Temp | **{{ states('sensor.loft_mvhr_monitor_supply_temperature') }} °C** |
                      | Extract Temp | **{{ states('sensor.loft_mvhr_monitor_extract_temperature') }} °C** |
                  - type: markdown
                    content: |
                      ### Storage & Solar Peak
                      | Metric | Value |
                      | :--- | :--- |
                      | EcoFlow Battery | **{{ states('sensor.ef_60037_battery_level') }}%** |
                      | Peak Solar Time | **{{ as_timestamp(states('sensor.power_highest_peak_time_today')) | timestamp_custom('%H:%M', true, 'N/A') }}** |
                  - type: markdown
                    content: |
                      ### Yesterday's Energy
                      {% set y_cost = states('sensor.octopus_energy_electricity_19l3396601_2000016825050_previous_accumulative_cost') %}
                      {% set y_unit = state_attr('sensor.octopus_energy_electricity_19l3396601_2000016825050_previous_accumulative_cost', 'unit_of_measurement') or '' %}
                      | Metric | Value |
                      | :--- | :--- |
                      | Usage | **{{ states('sensor.octopus_energy_electricity_19l3396601_2000016825050_previous_accumulative_consumption') | float(0) | round(2) }} kWh** |
                      | Cost | **{{ ('£' ~ (y_cost | float(0) | round(2))) if y_unit == 'GBP' else (y_cost ~ ' ' ~ y_unit) }}** |
              - type: conditional
                conditions:
                  - entity: input_select.e1003_time_of_day
                    state: morning
                card:
                  type: markdown
                  content: |
                    ### 🌅 Morning Mode (07:00 – 10:00)
                    Hourly e-ink refresh enabled.
              - type: conditional
                conditions:
                  - entity: input_select.e1003_time_of_day
                    state: day
                card:
                  type: markdown
                  content: |
                    ### ☀️ Daytime Mode (10:00 – 17:00)
                    Solar and heat pump monitoring active.
              - type: conditional
                conditions:
                  - entity: input_select.e1003_time_of_day
                    state: evenings
                card:
                  type: markdown
                  content: |
                    ### 🌆 Evening Mode (17:00 – 23:00)
                    Overnight deep sleep begins at 23:00.
              - type: conditional
                conditions:
                  - entity: input_select.e1003_time_of_day
                    state: night
                card:
                  type: markdown
                  content: |
                    ### 🌙 Night Mode (23:00 – 07:00)
                    Press right green button to wake.