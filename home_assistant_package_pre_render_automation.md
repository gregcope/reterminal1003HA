```yaml
# Home Assistant Package: reTerminal E1003 Helpers, REST Renderer & Dynamic Schedule
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

input_number:
  reterminal_update_interval:
    name: reTerminal Update Interval
    min: 5
    max: 120
    step: 5
    initial: 60
    unit_of_measurement: min
    mode: slider
    icon: mdi:timer-refresh-outline

rest_command:
  trigger_reterminal_renderer:
    url: "[http://192.168.86.52:5000/?clearCache=1&refresh=1](http://192.168.86.52:5000/?clearCache=1&refresh=1)"
    method: GET
    timeout: 60

automation:
  - id: reterminal_time_of_day_and_prerender
    alias: "reTerminal: Time of Day and Pre-render"
    description: "Updates time-of-day helper and primes port 5000 PNG five minutes before scheduled E1003 wakes."
    mode: single
    triggers:
      - trigger: time_pattern
        minutes: "/5"
    actions:
      - variables:
          target_tod: >-
            {% set t = now().strftime('%H:%M:%S') %}
            {% if '05:45:00' <= t < '10:00:00' %}
              morning
            {% elif '10:00:00' <= t < '17:00:00' %}
              day
            {% elif '17:00:00' <= t < '23:00:00' %}
              evenings
            {% else %}
              night
            {% endif %}
      - if:
          - condition: template
            value_template: "{{ not is_state('input_select.e1003_time_of_day', target_tod) }}"
        then:
          - action: input_select.select_option
            target:
              entity_id: input_select.e1003_time_of_day
            data:
              option: "{{ target_tod }}"
      - if:
          - condition: template
            value_template: >-
              {% set raw = states('input_number.reterminal_update_interval') | int(60) %}
              {% set interval = [[raw, 5] | max, 120] | min %}
              {% set wake_min = (now().hour * 60 + now().minute + 5) % 1440 %}
              {% if wake_min % interval != 0 %}
                false
              {% elif not is_state('input_boolean.eink_reterminal_enable_activehours', 'on') %}
                true
              {% else %}
                {% set s = states('input_datetime.eink_reterminal_enable_activehours_start').split(':') %}
                {% set e = states('input_datetime.eink_reterminal_enable_activehours_end').split(':') %}
                {% if s | length < 2 or e | length < 2 %}
                  true
                {% else %}
                  {% set start_min = (s[0] | int(7)) * 60 + (s[1] | int(0)) %}
                  {% set end_min = (e[0] | int(23)) * 60 + (e[1] | int(0)) %}
                  {% if start_min == end_min %}
                    true
                  {% elif start_min < end_min %}
                    {{ wake_min >= start_min and wake_min < end_min }}
                  {% else %}
                    {{ wake_min >= start_min or wake_min < end_min }}
                  {% endif %}
                {% endif %}
              {% endif %}
        then:
          - action: rest_command.trigger_reterminal_renderer
```
