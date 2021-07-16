---
layout: post
title: "Home automation journal #5 - A DIY floor lamp light with Shelly RGBW2"
description: Building a floor lamp using a Shelly RGBW2 LED controller with a custom firmware
date:   2021-07-16 16:30:00 +0300
tags: home-automation
---

I needed (wanted) a floor lamp for my bedside that provides a bright indirect light for late night reading. There's a rather nice looking one in the Philips Hue line but it's rather expensive and (as of this writing) out of stock everywhere. So, I decided to make my own.

## Hardware

<aside class="picture">
<img src="/assets/images/diy-lamp-1.jpeg" width="600" height="800" alt="A heavy base for the LED strip floor lamp"><br>
<img src="/assets/images/diy-lamp-2.jpeg" width="600" height="800" alt="The base with a 3D printed cover"><br>
</aside>

The design is as simple as it gets: a heavy base with an aluminum LED profile sticking up.

I made the base from a piece of ⌀90mm steel round stock I had lying around and screwed in a Paulmann Delta aluminum LED profile. The LED strips (2x 1.3 meter strips) are 24V 19 W/meter adjustable color temperature strips. To control it, I used a Shelly RGBW2 module.

To make it look pretty, I 3D printed a case to hide the steel base and to hold the Shelly. I'm not sure if the metal part was completely necessary; I might have gotten away with just a 3D print but this at least feels a bit sturdier and less likely to tip over.

I'm currently using a 36W power supply, which is not enough. The maximum consumption of the two strips is ~50W, but (theoretically) only half of the LEDs should be at full power at any given time, so a 36 Watt transformer *should* be enough. In practice, it's not. When near the midway point between the two color temperature extremes, the strips consume just enough power to trip the overcurrent limit.


## Software

Now, the interesting part. Shellies come with rather nice stock firmware with comprehensive support for local control using HTTP, COAP and MQTT. However, the RGBW2's default firmware only supports two types of lights: an RGBW lamp or four individual lamps. Since I'm controlling this lamp with Home Assistant, I could have just exported the warm-white and cold-white channels as individual lights and wrapped them in a template light, but I thought it would be much neater if the device would export a single temperature adjustable light entity.

Enter [ESPHome](https://esphome.io/).

ESPHome is a framework for generating custom firmwares for ESP8266/ESP32 based devices, which includes most of the cheap Wifi operated smart lights and plugs on the market at the moment.

So, let's write down all the features we want:

 * Wifi with captive portal for setup
 * Over-the-air updates
 * Adjustable color temperature LED strip control
 * Button for toggling the light on/off (in the end, I didn't bother adding this, since I always just use the HA app or Google Assistant.)

An ESPHome configuration for the Shelly RGBW2 might look like this:

```yaml
esphome:
  name: my-rgbw2-lamp
  platform: ESP8266
  board: esp01_1m

# Enable logging
logger:

# Enable Home Assistant API
api:
  password: ""

ota:
  password: ""

wifi:
  ssid: "MyWifi"
  password: "MyWifiPassword"

  # Enable fallback hotspot (captive portal) in case wifi connection fails
  ap:
    ssid: "Fallback Hotspot"
    password: ""

captive_portal:

status_led:
  pin:
    number: 2
    inverted: true

output:
  - platform: esp8266_pwm
    pin: 4
    frequency: 1000 Hz
    id: pwm_w
  - platform: esp8266_pwm
    pin: 14
    frequency: 1000 Hz
    id: pwm_b

light:
  - platform: cwww
    name: "My RGBW2 Light"
    id: cwww_light
    cold_white: pwm_b
    warm_white: pwm_w
    cold_white_color_temperature: 6500 K
    warm_white_color_temperature: 2300 K
    constant_brightness: true

# TODO this is the power meter. It reads the voltage difference from a shunt
# resistor. Knowing the input voltage, we can calculate the power from this.
sensor:
  - platform: adc
    id: voltage_analog_reading
    pin: A0
    name: Voltage 
    unit_of_measurement: V 
    icon: mdi:flash-circle
    update_interval: 10s

binary_sensor:
  # This is the external switch input
  - platform: gpio
    pin: 5
    internal: true
    id: button
    filters:
      - delayed_on: 100ms
    on_press:
      - light.toggle: cwww_light
  # This is the built-in reset button.
  - platform: gpio
    pin: 13
    internal: true
    id: resetbtn
    filters:
      - delayed_on: 100ms
    on_press:
      - light.turn_off: cwww_light
```
    
## Flashing the custom firmware

Flashing the firmware can be done in two ways: by connecting to the exposed serial port header on the device itself or over the air. Since the header on the RGBW2 is not the usual 2.54mm pitch header and I don't happen to have a suitable connector, I decided to try the OTA method first.

This is done by first flashing Tasmota, using [https://github.com/yaourdt/mgos-to-tasmota](mgos-to-tasmota). Afterwards, Tasmota's own OTA update method can be used to install the ESPHome firmware.

Unfortunately, for whatever reason I was unable to get this method to work. The mgos-to-tasmota got onto the shelly, it even connected to the Wifi, but just appeared to sit there. In the end, I had to resort to using the serial port.

The pin header is too small for regular dupont jumper wires to fit ([some workaround ideas here](https://koen.vervloesem.eu/blog/flashing-a-shelly-rgbw2-with-crocodile-clips-and-cut-resistor-leads/)), but I managed to get them in by filing the jumper pins down a bit. Then it was simply a matter of flashing Esphome in the usual manner. Afterwards, the firmware can be updated over-the-air. This can be done easily with the Esphome dashboard, which can be installed as a Home Assistant add-on. You can modify the firmware and observe logging output of your devices straight from HA!

