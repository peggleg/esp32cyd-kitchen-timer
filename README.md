# Kitchen Timer - ESPHome

Kitchen timer on ESP32-2432S028 CYD (Cheap Yellow Display) with Home Assistant integration.

**Max duration:** 12 hours

## Hardware

- ESP32-2432S028 CYD (Cheap Yellow Display)

## Features

- Preset buttons: +1m, +10m, +30m, +1h
- Custom time entry with keypad (SEC/MIN/HRS)
- Pause/Resume
- Add time while running (+1 MIN, +5 MIN, +CUSTOM)
- Progress bar
- Flashing alarm (+ Automation to alarm on Sonos)
- Auto-silence after 10 minutes
- Home Assistant integration

## Setup

1. Add secrets to `secrets.yaml`:
```yaml
wifi_ssid: "Your SSID"
wifi_password: "Your password"
api_encryption_key: "generated_key"
ota: "ota_password"
```

2. Flash:
Flash using ESPHome

3. WiFi fallback (if needed):
   - SSID: `Kitchen-Timer-Fallback`
   - Password: `12345678`

## Home Assistant

Exposed entities:
- `sensor.timer_remaining` - Seconds remaining
- `binary_sensor.timer_alarm` - Alarm active
- `binary_sensor.timer_running` - Timer running
- `number.timer_set_duration` - Set duration (0-43200 sec)
- `switch.timer_control` - Start/pause

Example - start timer:
```yaml
service: switch.turn_on
target:
  entity_id: switch.timer_control
```

Example - set duration and start:
```yaml
service: number.set_value
target:
  entity_id: number.timer_set_duration
data:
  value: 600  # 10 minutes
```

## Display

- 320x240 pixels (rotated 270°)
- Roboto font (bold)
- Manual update via LVGL

Colors:
- Background: `0xF8F5EC`
- Text: `0x23211C`
- Buttons: `0xE0DED7`
- Accent: `0x00ABA3`
- Alert: `0xEF6856`

## Customization

**Max duration:** Change `43200` in lambda expressions

**Preset buttons:** Edit values in `page_select` (e.g., `+ 60` for 1 minute)

**Colors:** Update hex in `lvgl` section

**Touchscreen calibration:**
```yaml
touchscreen:
  calibration:
    x_min: 428
    x_max: 3538
    y_min: 193
    y_max: 3671
```

**Alarm timeout:** Change `delay: 5s` (buzz duration)

## Global State

- `sel_seconds` - Selected duration
- `remaining` - Time left
- `total` - Original duration
- `running` - Timer active
- `paused` - Paused
- `alarm_active` - Alarm sounding
- `alarm_secs` - Alarm duration
- `kbuf` - Keypad buffer
- `unit_idx` - Unit (0=SEC, 1=MIN, 2=HRS)
- `custom_from_running` - Adding time to running timer

## Home Assistant Alarm

```
alias: Timer End
description: ""
triggers:
  - trigger: state
    entity_id: binary_sensor.kitchen_timer_timer_alarm
    to: "on"
conditions: []
actions:
  - if:
      - condition: template
        value_template: "{{ was_playing }}"
    then:
      - action: sonos.snapshot
        target:
          entity_id: media_player.kitchen_sonos
        data:
          with_group: true
  - repeat:
      sequence:
        - action: media_player.play_media
          data:
            media:
              media_content_id: >-
                media-source://media_source/local/freesound_community-kitchen-timer-33043.mp3
              media_content_type: audio/mpeg
              metadata:
                title: freesound_community-kitchen-timer-33043.mp3
                thumbnail: null
                media_class: music
                children_media_class: null
                navigateIds:
                  - {}
                  - media_content_type: app
                    media_content_id: media-source://media_source
                browse_entity_id: media_player.kitchen_sonos
          target:
            device_id:
              - 3444d7dd23f163adb6d5b92e15e8f6f6
        - wait_for_trigger:
            - trigger: state
              entity_id: binary_sensor.kitchen_timer_timer_alarm
              to: "off"
          timeout: "00:00:32"
          continue_on_timeout: true
      while:
        - condition: state
          entity_id: binary_sensor.kitchen_timer_timer_alarm
          state: "on"
  - if:
      - condition: template
        value_template: "{{ was_playing }}"
    then:
      - action: sonos.restore
        target:
          entity_id: media_player.kitchen_sonos
        data:
          with_group: true
    else:
      - action: media_player.media_stop
        target:
          entity_id: media_player.kitchen_sonos
variables:
  was_playing: "{{ is_state('media_player.kitchen_sonos', 'playing') }}"
mode: restart
```
## Notes

- Updates every 1s
- Alarm flashes at 300ms
- Display buffer: 25% RAM
- Debug logging enabled
