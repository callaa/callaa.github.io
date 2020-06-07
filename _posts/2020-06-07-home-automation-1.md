---
layout: post
title:  "Home Automation Journal #1 - IKEA and Zigbee"
description: Getting started with home automation using IKEA TRÅDFRI, Home Assistant and Zigbee2mqtt.
date:   2020-06-07 18:58:47 +0300
tags: home-automation
---

A couple of months ago, I (re)discovered home automation, by way of IKEA automated blinds. With summer approaching and the sun rising earlier and earlier, the IKEA FYRTUR blackout blinds were just what I needed for my bedroom. Naturally, this quickly led me to the other IKEA smart home stuff: their Zigbee controlled smart lights. In short order, I replaced all the lamps in my home with IKEA smart lamps and now, two months and a few learning experiences later, I'm all the more excited about smart home technology.

My experience with the IKEA system is mostly positive. The lamps are much cheaper than Philips Hue and the system has one very important feature (or from another perspective, *lack of misfeature*): **No cloud dependency**.

The lights work completely locally. You don't even need the hub for the remotes to work: clicking on a light switch sends the message directly to the bulb without looping it through a server in China (or Sweden.)

Another important factor was that the TRÅDFRI hub is supported by [Home Assistant], a Free/Libre home automation hub that can run on a Raspberry Pi. The IKEA smart home app is fairly barebones by itself, but with Home Assistant, you can do pretty much anything.

An extra bonus I discovered while researching the TRÅDFRI ecosystem was that the devices are also supported by [Zigbee2mqtt], which means the official gateway is not strictly needed even.

The devices make use of Zigbee groups, which allows the remotes to send commands directly to the lamps and blinds without the need for a hub at all. This is awesome, since it means my remote light switches continue to function even if the gateway and Home Assistant are down. (For example, the remotes continued to work right after a brief power outage in the middle of the night rebooted everything in the house and turned all the lights on.)

<!-- more -->

## First step: playing around with TRÅDFRI the way IKEA intended

<aside class="picture">
<img src="/assets/images/tradfri_gateway.jpeg" width="800" height="600" alt="TRÅDFRI gateway"><br>
</aside>

I started out with just a few lights, the FYRTUR blinds and the TRÅDFRI gateway. While neat, there isn't much you can do with this setup alone. Scheduling the blinds to open in the morning and close in the evening is great, but that's pretty much it. The app does support Alexa and Google Assistant integrations, which probably opens up more features, but I have not tried it.

Pairing the devices can be frustrating at times. The app will tell you how to do it, but sometimes a remote fails to pair with the bridge and sometimes a random lamp gets paired with a remote instead of the one you intended.

The way pairing is done is that you first pair a remote with the gateway, and then use the remote to add more devices into the network. (The remotes utilize Touchlink to reset nearby devices and put them into pairing mode.) Pairing a lamp with the remote adds it to the remote's group, but a lamp (or other controllable device) can be moved to another group in the app.

Confusingly, the app calls remote groups "rooms". If you have two lamps in the same room that you want to control with separate remotes, you'll need to add two "rooms" in the app. If you want to add two remotes to the same group (e.g. a motion detector + remote) this is... supposedly possible. As of writing this, it can't be done directly via the app and isn't documented anywhere, but apparently it's possible to pair the remotes to each other somehow. (Behind the user friendly abstraction, what's happening is that both remotes are bound to the same Zigbee group.)


## The next step: I'll do it my way!

<aside class="picture">
<img src="/assets/images/raspberry_pi_ha.jpeg" width="535" height="800" alt="Raspberry Pi in a 3D printed Home Assistant case">
</aside>

The TRÅDFRI gateway does not expose all the features of the devices connected to it. The biggest omission is related to the remotes: you can't receive events when remote buttons are clicked, nor can you receive notifications when a motion detector detects (or stops detecting) motion. You can, however, detect the state of a device the remote (or motion detector) is controlling and use it as a proxy, but that's terrible.

A solution to this is to use Zigbee2mqtt instead of the official gateway. When my CC2531 USB-Zigbee adapter came in the mail, I turned off the IKEA bridge and paired all my devices with the USB stick. (There are also alternatives such as ConBee, but the software is closed source.)

The Zigbee stick is plugged into a Raspberry Pi that also runs Home Assistant. (I'm using zigbee2mqtt installed as a home assistant add-on.) In the future, I will probably replace the Pi with something more reliable, since Pis are prone to corrupting their SD cards when the not shut down cleanly.

The pairing process with Zigbee2mqtt is somehow both easier and more complicated. Typically, it goes like this:

1. Turn on "permit join" in zigbee2mqtt (can be done with a MQTT message.)
2. Reset the device to be paired. (Per device instructions can be found in zigbee2mqtt documentation.)
3. Look at zigbee2mqtt log output and watch for an "interview complete" line.

After the device has been paired, it's a good idea to give it a descriptive "friendly name". Initially, the name will be the device's MAC address, but changing it to something like `bathroom_light` will make debugging a whole lot easier.

Next, if the device is a remote or something to be controlled by a remote, it should be added to a Zigbee group. This way, it will work even without the gateway and without having to create automations for button presses.

I'm not sure if this is the correct or best way to do it, but this seemed to work for me:

1. Add a new group with random group number to the group configuration file.
2. Bind all the lights (or other devices) to the group using an MQTT command to zigbee2mqtt
3. Unbind the remote from its default group. (Since the remotes are battery powered, they need to be woken up first. Clicking on a button then immediately sending the (un)bind command worked fairly reliably for me.)
4. Bind the remote to the new group

I also tried adding devices to the group the IKEA way by using the remote to trigger pairing, however this did not work reliably when using Zigbee2mqtt.

With this method I managed to add all my devices into the network:

 - Various IKEA smart bulbs
 - FYRTUR blinds
 - Remotes
 - An IKEA smart plug
 - Xiaomi temperature/humidity sensors (these are not supported by the IKEA gateway)

The devices seemed to work fine, with just two features missing:

 - The FYRTUR did not report back its position correctly. (Not sure whether this is/was a problem in zigbee2mqtt or Home Assistant)
 - The remote arrow buttons did not change lighting scenes. (These might need support from the gateway)

But I did gain some features:

 - Zigbee2mqtt does send events when remote buttons are pressed, enabling them to be used in automations
 - The motion detector can now be used in automations without hacks. (Sadly, it's not very customizable. When it detects motion, it sends the "occupied" signal for three minutes before sensing again and this timeout cannot be changed.)
 - Support for other Zigbee devices, like the Xiaomi temperature sensors!

After a couple of weeks of using the CC2531 adapter, problems started cropping up. Sometimes, devices would fall off the network. Trying to add them back might cause other devices to disappear. Sometimes, a command to turn on a light would apparently sit in a buffer (no idea what was actually happening) for up to 10 seconds before being executed. From what I've read, these are symptoms of the little USB stick not being powerful enough for the network. At this point, I had over 30 devices connected to it.

In the end, I decided to go back to using the TRÅDFRI gateway.


## A hybrid approach

<aside class="picture">
<img src="/assets/images/tradfri_remote.jpeg" width="300" height="300" alt="TRÅDFRI remote">
</aside>

Because the network was apparently too big for my little USB Zigbee adapter, I moved all the lights and remotes back to the IKEA hub. The Xiaomi temperature sensors and the IKEA motion detector I left on the CC2531 network, however.

This meant losing remote button press events, but in practice I never actually used those anyway.

With the TRÅDFRI hub taking care of the lights and the USB stick receiving sensor events, everything has worked flawlessly since. Having the motion detector handled using Home Assistant (or Node-RED) automation lets me make better use of it. Presently, I have it set up so that motion will turn the hallway light on and off, unless already turned on via a remote bound directly to the lamp.

The IKEA gateway does not have much in the way of smarts, but this is actually good when used together with Home Assistant. Between HA's simple automations, Node-RED and AppDaemon (Python code,) there isn't much you *can't* do.

A few of my favorite HA automations:

 * [Circadian Lighting] adjusts the brightness and color temperature of your bulbs depending on the time of day. This, in my opinion, is *the* killer feature of adjustable temperature bulbs.
 * Holiday sensor: opens the blinds in the morning on work days, but not on weekends or holidays.
 * Sunset/sunrise: closes the blinds at sunset (or manually set time, whichever comes first)
 * When the Chromecast is playing, living room lights automatically switch to movie watching mode. Pausing or stopping restores them to the way they were. (I wrote a short AppDaemon script for this.)

## Summary

The IKEA TRÅDFRI smart home system is a great, inexpensive way to get into home automation. I consider its lack of Cloud dependency a feature, since it preserves my privacy and lets me use Home Assistant locally, even when the Internet connection is down.

TRÅDFRI devices can be used with Zigbee2mqtt and a cheap USB Zigbee adapter, but in my experience this only works reliably when only few devices are connected to the network. However, the two can be used together to complement each other.

[Home Assistant] is an excellent Free Software home automation hub that integrates with almost anything and lets you create however complicated automations you wish by using Node-RED or even Python programming.


[Home Assistant]: https://www.home-assistant.io/
[Zigbee2mqtt]: https://www.zigbee2mqtt.io/
[Circadian Lighting]: https://github.com/claytonjn/hass-circadian_lighting
