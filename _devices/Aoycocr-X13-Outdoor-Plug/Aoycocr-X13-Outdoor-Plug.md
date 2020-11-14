---
title: Aoycocr-X13 Outdoor Plug
date-published: 2020-05-09
type: plug
standard: us
---
The GPIO pinout was learned from [Blakadder Tasmota](https://templates.blakadder.com/aoycocr_X13.html) documentation.

1. TOC
{:toc}

## GPIO Pinout

| Pin     | Function                           |
|---------|------------------------------------|
| GPIO02  | Status LED - Blue (inverted)       |
| GPIO05  | cf_pin hlw8012                     |
| GPIO12  | Relay S2                           |
| GPIO13  | Button (inverted)                  |
| GPIO15  | Relay S1                           |

## Basic Configuration

```yaml
# Basic Config

substitutions:
  device_name: aoycocr_x13
  device_description: Aoycocr X13 Outdoor plug 
  friendly_name: Aoycocr X13
  
esphome:
  name: ${device_name}
  comment: ${device_description}
  platform: ESP8266
  board: esp01_1m

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password
  fast_connect: on

  # Enable fallback hotspot (captive portal) in case wifi connection fails
  ap:
    ssid: "${friendly_name} Hotspot"
    password: !secret ap_password

captive_portal:

# Enable logging
logger:
  baud_rate: 0 #disable UART logging

# Enable Home Assistant API
api:
  password: !secret api_password
  
# Enable OTA updates
ota:
  password: !secret ota_password
  
# Enable web server
web_server:
  port: 80

# Enable time component for use by daily power sensor
time:
  - platform: homeassistant
    id: homeassistant_time

binary_sensor:
# Reports when the button is pressed
- platform: gpio
  device_class: power
  pin:
    number: GPIO13
    inverted: True
  name: ${friendly_name} Button
  on_press:
    - switch.toggle: relay_s1
    - switch.toggle: relay_s2

# Reports if this device is Connected or not
- platform: status
  name: ${friendly_name} Status

sensor:
    # Reports the WiFi signal strength
  - platform: wifi_signal
    name: ${friendly_name} WiFi Strength
    update_interval: 60s

  # Uptime in seconds
  - platform: uptime
    id: uptime_sec
    internal: true

text_sensor:
    # Reports the ESPHome Version with compile date
  - platform: version
    name: ${friendly_name} ESPHome Version
    # Reports IP address
  - platform: wifi_info
    ip_address:
      name: "${friendly_name} IP Address"
    #Reports uptime in days hours minutes
  - platform: template
    name: "${friendly_name} Uptime"
    lambda: |-
      int seconds = (id(uptime_sec).state);
      int days = seconds / (24 * 3600);
      seconds = seconds % (24 * 3600); 
      int hours = seconds / 3600;
      seconds = seconds % 3600;
      int minutes = seconds /  60;
      seconds = seconds % 60;
      return { (String(days) +"d " + String(hours) +"h " + String(minutes) +"m "+ String(seconds) +"s").c_str() };
    icon: mdi:clock-start
    update_interval: 120s

switch:
#Plug S1
- platform: gpio
  name: ${friendly_name} S1
  pin: GPIO15
  id: relay_s1
  restore_mode: RESTORE_DEFAULT_OFF #Try to restore relay state after reboot/power-loss event.
  #RESTORE_DEFAULT_OFF (Default) - Attempt to restore state and default to OFF if not possible to restore. Uses flash write cycles.
  #RESTORE_DEFAULT_ON - Attempt to restore state and default to ON. Uses flash write cycles.
  #ALWAYS_OFF - Always initialize the pin as OFF on bootup. Does not use flash write cycles.
  #ALWAYS_ON - Always initialize the pin as ON on bootup. Does not use flash write cycles.
  on_turn_on:
    - light.turn_on:
        id: blue_led
        brightness: 100%
  on_turn_off:
    - light.turn_off: blue_led
#Plug S2
- platform: gpio
  name: ${friendly_name} S2
  pin: GPIO12
  id: relay_s2
  restore_mode: RESTORE_DEFAULT_OFF #Try to restore relay state after reboot/power-loss event.
  #RESTORE_DEFAULT_OFF (Default) - Attempt to restore state and default to OFF if not possible to restore. Uses flash write cycles.
  #RESTORE_DEFAULT_ON - Attempt to restore state and default to ON. Uses flash write cycles.
  #ALWAYS_OFF - Always initialize the pin as OFF on bootup. Does not use flash write cycles.
  #ALWAYS_ON - Always initialize the pin as ON on bootup. Does not use flash write cycles.
  on_turn_on:
    - light.turn_on:
        id: blue_led
        brightness: 100%
  on_turn_off:
    - light.turn_off: blue_led

output:
- platform: esp8266_pwm
  id: blue_output
  pin: GPIO2
  inverted: True

light:
- platform: monochromatic
  name: ${friendly_name} Blue LED
  output: blue_output
  id: blue_led
  restore_mode: ALWAYS_OFF #Start with light off after reboot/power-loss event.
  #RESTORE_DEFAULT_OFF (Default) - Attempt to restore state and default to OFF if not possible to restore. Uses flash write cycles.
  #RESTORE_DEFAULT_ON - Attempt to restore state and default to ON. Uses flash write cycles.
  #ALWAYS_OFF - Always initialize the pin as OFF on bootup. Does not use flash write cycles.
  #ALWAYS_ON - Always initialize the pin as ON on bootup. Does not use flash write cycles.
  effects:
    - strobe:
    - flicker:
        alpha: 50% #The percentage that the last color value should affect the light. More or less the “forget-factor” of an exponential moving average. Defaults to 95%.
        intensity: 50% #The intensity of the flickering, basically the maximum amplitude of the random offsets. Defaults to 1.5%.
    - lambda:
        name: Throb
        update_interval: 1s
        lambda: |-
          static int state = 0;
          auto call = id(blue_led).turn_on();
          // Transtion of 1000ms = 1s
          call.set_transition_length(1000);
          if (state == 0) {
            call.set_brightness(1.0);
          } else {
            call.set_brightness(0.01);
          }
          call.perform();
          state += 1;
          if (state == 2)
            state = 0;

# Blink the blue light if we aren't connected to WiFi. Could use https://esphome.io/components/status_led.html instead but then we couldn't use the blue light for other things as well.
interval:
  - interval: 500ms
    then:
      - if:
          condition:
            not:
              wifi.connected:
          then:
            - light.turn_on:
                id: blue_led
                brightness: 100%
                transition_length: 0s
            - delay: 250ms
            - light.turn_off:
                id: blue_led
                transition_length: 250ms
