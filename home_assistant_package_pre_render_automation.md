# Home Assistant Package: reTerminal E1003 Helpers, REST Renderer & Hourly Automation
# Save to /config/packages/reterminal_e1003_package.yaml

input_select:
  e1003_time_of_day:
    name: reTerminal E1003 Time of Day
    options:
      - morning
      - day
      - evenings
      - night
    initial: day
    icon: mdi:clock-outline

input_boolean:
  eink_reterminal_enable_deepsleep:
    name: reTerminal Enable Deep Sleep
    initial: true
    icon: mdi:sleep

  eink_reterminal_enable_activehours:
    name: reTerminal Enable Active Hours
    initial: true
    icon: mdi:calendar-clock

input_datetime:
  eink_reterminal_enable_activehours_start:
    name: reTerminal Active Hours Start
    has_date: false
    has_time: true
    initial: "07:00:00"

  eink_reterminal_enable_activehours_end:
    name: reTerminal Active Hours End
    has_date: false
    has_time: true
    initial: "23:00:00"

rest_command:
  trigger_reterminal_renderer:
    url: "http://192.168.86.52:5000/?clearCache=1&refresh=1"
    method: GET
    timeout: 60

automation:
  - id: reterminal_time_of_day_and_prerender
    alias: "reTerminal: Time of Day and Pre-render"
    description: "Updates time-of-day helper and primes port 5000 PNG ahead of the hourly E1003 wake."
    mode: single
    triggers:
      - trigger: time_pattern
        minutes: 55
    actions:
      - choose:
          - conditions:
              - condition: time
                after: "05:45:00"
                before: "10:00:00"
            sequence:
              - action: input_select.select_option
                target:
                  entity_id: input_select.e1003_time_of_day
                data:
                  option: morning
          - conditions:
              - condition: time
                after: "10:00:00"
                before: "17:00:00"
            sequence:
              - action: input_select.select_option
                target:
                  entity_id: input_select.e1003_time_of_day
                data:
                  option: day
          - conditions:
              - condition: time
                after: "17:00:00"
                before: "23:00:00"
            sequence:
              - action: input_select.select_option
                target:
                  entity_id: input_select.e1003_time_of_day
                data:
                  option: evenings
        default:
          - action: input_select.select_option
            target:
              entity_id: input_select.e1003_time_of_day
            data:
              option: night
      - action: rest_command.trigger_reterminal_renderer