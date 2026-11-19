alias: 'reTerminal: Time of Day and Pre-render'
description: >-
  Updates time-of-day helper and primes port 5000 PNG five minutes before
  scheduled E1003 wakes.
triggers:
  - trigger: time_pattern
    minutes: /5
actions:
  - variables:
      target_tod: >-
        {% set t = now().strftime('%H:%M:%S') %} {% if '05:45:00' <= t <
        '10:00:00' %}
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
        value_template: '{{ not is_state(''input_select.e1003_time_of_day'', target_tod) }}'
    then:
      - action: input_select.select_option
        target:
          entity_id: input_select.e1003_time_of_day
        data:
          option: '{{ target_tod }}'
  - if:
      - condition: template
        value_template: >-
          {% set raw = states('input_number.reterminal_update_interval') |
          int(60) %} {% set interval = [[raw, 5] | max, 120] | min %} {% set
          wake_min = (now().hour * 60 + now().minute + 5) % 1440 %} {% if
          wake_min % interval != 0 %}
            false
          {% elif not
          is_state('input_boolean.eink_reterminal_enable_activehours', 'on') %}
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
mode: single
