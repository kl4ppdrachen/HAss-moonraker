# Moonraker (Klipper) automation for Home Assistant

It provides a RESTful sensor for Moonraker states, RESTful commands, some automation for safety shutdown, emergency shutdown, notification when finished, control and automation that does not flood the Home Assistant logs when Moonraker is offline.

You can also use the HACS-Integration "Moonraker" from marcolivierarsenault: https://github.com/marcolivierarsenault/moonraker-home-assistant 

## What do you need?

- Home Assistant
- Device with Klipper (in my case RPi with Ratrig OS)
- A controlable switch (power plug) for the printer
- Optional: A Zigbee button or similar for control

## How to use?

In your Home Assistant `configuration.yaml` you need to at following lines:

```yaml
homeassistant:
  packages: !include_dir_named packages
```

and create a folder named `packages` and place the `moonraker.yaml` into it. Or your download and copy it from github.

It must be look like this:

```
config/
├─ configuration.yaml
├─ packages/
│ ├─ moonraker.yaml
```

In `moonraker.yaml` you need to change (CTRL+F and replace) `YOUR_IP` to the **_IP_** from **_your_** printer. Done.

---

To use the automation you need a power plug for the printer. In my case I use a BlitzWolf BW-SHP6 (16A) called `switch.3dprinter`.

---

To get notified, you need to change the `notify.devices` service to your device.

## Optional

### Camera

Best way is to use the [MJPEG Intergration](https://www.home-assistant.io/integrations/mjpeg) from Home Assistant.

- MJPEG-URL: `http://YOUR_IP/webcam/?action=stream`
- Still Image URL: `http://YOUR_IP/webcam/?action=snapshot`

### Shortcut button

I use a TRADFRI shortcut button to control the printer when I am standing in front. I can turn on and of the printer or do a emergency stop.

```yaml
alias: 3D Printer - Toggle [Shortcut button]
description: ""
trigger:
  - platform: state
    entity_id:
      - sensor.3dprinter_button
    from: "on"
condition: []
action:
  - choose:
      - conditions:
          - condition: device
            type: is_off
            entity_id: switch.3dprinter
            domain: switch
        sequence:
          - service: switch.turn_on
            data: {}
            target:
              entity_id:
                - switch.3dprinter
      - conditions:
          - condition: and
            conditions:
              - condition: not
                conditions:
                  - condition: template
                    value_template: "{{ is_state('sensor.moonraker_status', 'printing') }}"
              - condition: device
                type: is_on
                entity_id: switch.3dprinter
                domain: switch
        sequence:
          - service: script.safe_shutdown
            data: {}
      - conditions:
          - condition: template
            value_template: "{{ is_state('sensor.moonraker_status', 'printing') }}"
        sequence:
          - service: rest_command.moonraker_emergency_stop
            data: {}
    default: []
mode: single
```

## Notification when PAUSED or ERROR

```yaml
alias: 3D Drucker - Benachrichtigung [Error, Paused]
description: ""
trigger:
  - platform: state
    entity_id:
      - sensor.moonraker_status
    for:
      hours: 0
      minutes: 0
      seconds: 0
    to:
      - error
      - paused
condition: []
action:
  - service: notify.devices
    data:
      message: '{{ (states("sensor.moonraker_state")|string)|upper }}'
      title: 3D Printer
  - if:
      - condition: state
        entity_id: person.admin
        state: home
    then:
      - service: tts.google_translate_say
        data:
          entity_id: media_player.nestmini
          message: 3D Printer {{ (states("sensor.moonraker_state")|string)|upper }}
mode: single
```
