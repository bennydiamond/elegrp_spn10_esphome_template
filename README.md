# elegrp_spn10_esphome_template
ESPHome device package for ELEGRP SPN10 switch device.

These units use a [Tuya Wifi Module CB2S](https://docs.libretiny.eu/boards/cb2s/).
There is no TuyaMCU. Buttons, LED and relay are wired to GPIOs of CB2S module.
There is no power monitoring capability.

Notable functionnality of this pacakage available:
- Relay control with paddle up and paddle down switches. Paddle Up turns ON the relay.
- Single-click, Double-click and long press events for both paddle buttons.
- Very long press triggers device reboot(paddle down) or reboot into safe mode(paddle up).
- White status LED follows relay output status, inverted. LED is ON when relay is OFF.
- White status LED brightness control. Brightness value can be modified without turning ON the LED.
- If disconnected from API for 10 minutes straight, reset White LED indicator to a preset value
- Ability to disable device's paddle switches from directly controlling the relay. Useful to turn this switch into a scene controller.
- Ability to enable local relay control fallback (switch paddles controls relay) on connectivity issues with Home Assistant
- Ability to configure relay state on boot without reprogramming

This is a leaner variant over the main template with a few components removed or modified:
- Removed API encryption
- Removed OTA password
- Removed Web Server
- Removed fallback Hotspot and captive portal


# Initial device programming
My units bought in November 2024 came with Wifi module firmware version 2.2.4. 

I was able to use [tuya cloudcutter](https://github.com/tuya-cloudcutter/tuya-cloudcutter) without a specific device profile. There is a ready-made profile for a "elegrp" device available.

Activating AP mode is not intuitive. You activate AP mode by holding either paddles on the switch for 6 to 7 seconds. The white status LED does not blink slowly. You will need to verify if a "Smartlife" AP appears.

Be warned that I was notified of a firmware update when I provisionned a unit with the Tuya Android App. I suspect the updated version is not exploitable through cloudcutter.
It is likely the manufacturer will switch to this updated firmware version in future production.

## Serial flashing
If your device is programmed with a version incompatible with tuya cloudcutter, you can still relatively easily partially dissassemble the unit to flash the unit through serial.

Information on how to program is located on [Elektroda's forums](https://www.elektroda.com/rtvforum/viewtopic.php?p=21344277).



# How to use
Include this package and populate the necessary variables.

Example of a device YAML file:
```yaml
substitutions:
  device_name: "my-spn10-switch"
  device_friendly_name: "Switch"
  api_key: "supersecretapikey"
  ota_password: "otapsswd"
  hotspot_name: "SPN10-switch-AP"
  hotspot_password: !secret fallback_hotspot_password
  log_level: "INFO" # Optional, default value is "INFO"
  indicator_brightness_when_offline_percent: '100' # Optional, default value is "100"
  multiclick_min_length: "40ms" # Optional, default value is "40ms". Should be above 30ms
  multiclick_max_length: "350ms" # Optional, default value is "350ms"
  longpress_min_length: "1000ms" # Optional, default value is "1000ms"
  # Relay state on boot in case value cannot be restored from flash. 
  # You should set to false if not needed as the device will momentarily de-energize the relay on reboots (hardware limitation)
  # If set to true, on reboot, relay will momentarily turn off then will turn back on again.
  default_relay_on_boot_is_active: "false"  # Optional, default value is "false"

packages:
  remote_package_files:
    url: https://github.com/bennydiamond/elegrp_spn10_esphome_template
    files: [.base.elegrp.spn10_template.yaml]  # optional; if not specified, all files will be included
    ref: master  # optional
    refresh: 1d  # optional
```

## Customization

Refer to ESPHome's documentation on [packages](https://esphome.io/components/packages) to customize your device.

### Example of customization. 
To remove the Fallback Wifi Hotspot component from your device, you would just need to add the following at the bottom of your device's yaml file.

```yaml
wifi:
  ap: !remove
```

To set static IP.
```yaml
wifi:
  manual_ip:
    static_ip: 192.168.1.10
    gateway: 192.168.1.1
    subnet: 255.255.255.0
```

To remove double-click and longpress handling (which removes any perceptible delay between paddle presses and relay toggling)
```yaml
binary_sensor:
  - id: !extend button_up
    on_multi_click: 
      - timing: 
          - ON for at least ${multiclick_min_length}
        then:
          - if:
              condition:
                or:
                  # If local control is enabled
                  - lambda: return id(relay_local_control_active) == true;
                  # Or if fallback control is enabled and API or WIFI are not connected
                  - and:
                      - lambda: return id(relay_local_control_fallback_active) == true;
                      - or:
                          - not:
                              - api.connected:
                          - not:
                              - wifi.connected:
              then:
                - switch.turn_on: output_relay
          - event.trigger:
              id: button_up_events
              event_type: singleclick

  - id: !extend button_down
    on_multi_click: 
      - timing: 
          - ON for at least ${multiclick_min_length}
        then:
          - if:
              condition:
                or:
                  # If local control is enabled
                  - lambda: return id(relay_local_control_active) == true;
                  # Or if fallback control is enabled and API or WIFI are not connected
                  - and:
                      - lambda: return id(relay_local_control_fallback_active) == true;
                      - or:
                          - not:
                              - api.connected:
                          - not:
                              - wifi.connected:
              then:
                - switch.turn_off: output_relay
          - event.trigger:
              id: button_down_events
              event_type: singleclick

```
