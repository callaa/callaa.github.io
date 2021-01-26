---
layout: post
title: "Home automation journal #4 - Liberating a Nedis SmartLife Wifi socket"
description: Reflashing a Nedis Smartlife Wifi outdoor socket with Tasmota
date:   2021-01-26 20:00:00 +0200
tags: home-automation
---

<aside class="picture">
<img src="/assets/images/nedis-smartplug-1.jpeg" width="467" height="600" alt="Nedis SmartLife Wifi outdoor socket"><br>
</aside>

With the weather getting cold, I thought it would be nice to have a remote controlled outdoor plug so I could start my car heater from the comfort of the warm indoors.

A Nedis SmartLife Wifi outdoor socket found at a local department store looked just right for the task. Like most Wifi smart plugs, it's made by Tuya. Using the stock firmware with its cloud dependency was not acceptable to me, but luckily Tuya devices are easy to flash with open source firmware, like [Tasmota](https://tasmota.github.io/docs/) or [ESPHome](https://esphome.io/)... Or so it seemed. In any case, the device was cheap enough that I was willing to risk bricking it just for the experience.

First, I tried [Tuya-Convert](https://github.com/ct-Open-Source/tuya-convert), a program that emulates the manufacturer's cloud and lets you slip in your own firmware as an OTA update. I installed the software on a Raspberry Pi 2 with a Wifi stick and followed the instructions. Pressing down the button on the plug for a few seconds until the LED starts blinking puts it into "quick setup" mode where it connects to the access point created by Tuya-Convert.

So far so good, I could see a connection attempt in the `smarthack-wifi.log` file. However, the firmware update would not proceed. Looking at `smarthack-psk.log` file revealed a [problem](https://github.com/ct-Open-Source/tuya-convert/wiki/Collaboration-document-for-PSK-Identity-02). The smart plug uses a new version of the firmware with better security so it can't be intercepted by Tuya-convert (yet, anyway.) The Nedis plug is actually included in the list of devices known to have this problem, so I should have done better research before buying. Oh well.

If OTA update was not an option, I could still do it the old fashioned way by opening the case and directly connecting a serial cable.

<aside class="picture">
<img src="/assets/images/nedis-smartplug-2.jpeg" width="800" height="591" alt="Nedis smart socket with case open"><br>
</aside>

Next problem: tri-wing screws. The manufacturer really doesn't want users tampering with the device so instead of usual philips or torx screws, it has tri-wing. And not only that, they are pretty deep, so a regular hardware store security bit set won't reach. Luckily, I found a set of extra-long tri-wing bits from Amazon that wasn't too expensive. (The correct size appears to be tri-wing 1. Tri-wing 0 is often used in mobile phones and game consoles.)

With the screws removed, the case could be twisted open with some force.

Inside, the Wifi module is a [TYWE3S](https://tasmota.github.io/docs/devices/TYWE3S/). Luckily, the pins needed for flashing are not only already broken out, but labeled! I assume this is for ease of programming at the factory using a pogo pin jig. I could probably 3D print a jig, but since I only need to flash one device, simply soldering jumper wires onto the pads is easier.

<aside class="picture">
<img src="/assets/images/nedis-smartplug-3.jpeg" width="800" height="591" alt="Serial port wires soldered"><br>
</aside>

I used a common USB-serial adapter to connect to the board. GND to GND, 3.3V to 3.3V (had to be extra careful here, my adapter has both 3.3V and 5V power pins.), RX to TX and TX to RX. Note that the Wifi board in the plug receives all the power it needs from the USB adapter. It should *not* be plugged into the wall when doing this!

IO0 should be connected to GND before the board is powered up to put the bootloader into flashing mode. (There is no need to solder a wire to IO0, pressing the onboard button does the same thing.)

I downloaded [Tasmotizer](https://github.com/tasmota/tasmotizer), plugged the USB adapter in and hit the flash button. And, surprise, it just worked!

After restarting the device once, it appeared as a new Wifi access point. I connected my phone and visited the default IP (192.168.4.1) and filled in my home Wifi details.

Unlike ESPHome, where you write a configuration file and generate a custom firmware, Tasmota is a "one size fits all" firmware that is configured at runtime by assigning components to the GPIO pins. Luckily, there is already a [Tasmota template for this device](https://github.com/tasmota/tasmotizer), so configuring it was just a matter of copy&pasting the template in.

For reference, the following components are connected to the Wifi board:

 * GPIO0: Button
 * GPIO04: HLW8012 CF
 * GPIO05: HLW8012 CF1
 * GPIO12: HLW8012 select (inverted)
 * GPIO13: Status LED (inverted)
 * GPIO14: Relay

The HLW8012 is a power meter chip. At least for me, it appears to work well enough without any calibration. (Tip: the command `TelePeriod` controls how often measurement data is pushed to MQTT. The default value is 300 seconds.)

After setting the template, the plug appeared to be fully operational: clicking on the toggle button on the admin UI switched the relay on and off, as did pushing the physical button.

Finally, I just needed to link it into Home Assistant. This turned out to be very easy: I just had to set my MQTT server's address and password. Home Assistant automatically detected the new device and offered to activate the Tasmota integration. With the integration enabled, the plug appeared as a device with entities for power metering and the relay itself. Very nice.

In summary, I did get the Nedis outdoor wifi plug reflashed with Tasmota, but it wasn't as easy as I had hoped. Was it worth it? As an experience, sure. Soldering wires to reflash a device makes one feel like a real hacker. However, in the future I'll stick with companies like Shelly that don't try to lock you out of your own hardware.

