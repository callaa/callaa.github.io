---
layout: post
title:  "Home Automation Journal #2 - DIY doorbell"
description: Building a DIY smart doorbell using NodeMCU and MQTT
date:   2020-06-13 11:11:00 +0300
tags: home-automation
---

Previously, I wrote about getting started with home automation using TRÅDFRI lights and Home Assistant. The cool thing about Home Assistant is that it lets you mix and match devices from different manufacturers, even things you build yourself.

The easiest way (IMO) to integrate a custom device into Home Assistant is to use the MQTT protocol. MQTT is a simple and lightweight pub-sub protocol especially useful for IoT devices. Home Assistant has first class support for it and can run the Mosquitto MQTT broker as an add-on.

The goal of this project is to retrofit an existing battery operated doorbell into a smart IoT doorbell that integrates with Home Assistant. This will make it possible to trigger automations when the doorbell is pressed, such as sending a message to my phone if I'm not home.

<!--more-->

## Building the doorbell

<aside class="picture">
<img src="/assets/images/smart-doorbell-v1.jpeg" width="400" height="600" alt="A DIY smart doorbell in a 3D printed case"><br>
</aside>

I decided to reuse my old doorbell, a simple battery powered chime. When the button is pressed, the battery is connected to the solenoid, causing the plunger to hit the chime and make the "ding-dong" noise. The physical button can be replaced with a transistor to connect power to the solenoid.

The IoT part is handled by a NodeMCU, programmed using the Arduino IDE. The NodeMCU connects to my Wifi and the MQTT broker running on the Raspberry Pi. Presently, all settings are hardcoded into the firmware, which means I will need to reflash it when I change my Wifi password. This is something that could be improved later.

An alternative would be to use [ESPHome](https://esphome.io/), which provides handy features out of the box, such as OTA updates and a web configuration interface. I might give this a try in the future.

You can find the source code and a more detailed description of the hardware at my [smart-doorbell repository](https://github.com/callaa/smart-doorbell).

To better hold the extra components, I also 3D printed a new case for the doorbell.

## MQTT and Home Assistant

The interesting part of the smart doorbell is how it uses the MQTT protocol to integrate with Home Assistant.

A quick summary of salient MQTT features:

 * MQTT clients connect to a *broker*
 * Clients can publish messages which will be received by other connected clients. A message has a *topic* and a *payload*
 * A message can be tagged as *retained*, in which case the broker will keep a copy and automatically send it to all clients who subscribe to its topic. (Only the latest retained message per topic is kept)
 * Clients can subscribe to specific topics to receive messages they're interested in
 * A client can register a "last will and testament" message that the broker will publish on its behalf upon disconnection

Basically, the smart doorbell needs to do just one thing: publish a message on some topic when the button is pressed (and also ring the bell.)
Any topic can be used, but its a good idea to give it some structure, like:

    diy/frontdoor/button

The `diy` part is what I'll use as a namespace for all my home-made devices. The second part (`frontdoor`) identifies the device. On a mass produced product, this might be the MAC address or some other unique identifier. The final part (`button`) indicates which feature of the device the message is related to. The doorbell isn't just a doorbell: it also has a reed switch for detecting if the door is open, so I need to distinguish between `button` and `door` messages. (In Home Assistant terms, the doorbell *device* has two *entities*: the button and the door sensor.)

When the button is pressed, a message with the payload "press" is published on the above topic. Also, when the state of the door sensor changes,either "open" or "closed" is published at `diy/frontdoor/door`.

Now, this is enough. It's possible to create a device in Home Assistant's configuration to get the status of the doorbell. The button press event can then be used in automations to do things like send a text notification. However, there's a better way. A device can publish special messages for Home Assistant to automatically discover it.

## MQTT Discovery

To enable automatic discovery of MQTT devices, we can publish messages that contain the same info that would be written in the configuration file. The [MQTT Discovery](https://www.home-assistant.io/docs/mqtt/discovery/) page lists the types of entities supported by Home Assistant.

My doorbell exposes two entities: the button and the door sensor.

The button is exposed as a *Device Trigger*. This makes it show up in the Automations editor.

	Topic: homeassistant/device_automation/frontdoor/button_short_press/config
	Payload:
	{
		"device": {
			"identifiers": "frontdoor",
			"manufacturer": "DIY",
			"model": "Smart Doorbell",
			"name": "Doorbell"
		},
		"availability_topic": "diy/doorbell/available",
		"topic": "diy/frontdoor/button",
		"automation_type": "trigger",
		"type": "button_short_press",
		"subtype": "button1",
		"payload": "press"
	}

The `device` object makes the physical device show up in Home Assistant's Devices list.
The `availability_topic` is where the state (online or offline) of the device is published. More on that below.
The `topic` is the MQTT topic where the button press will be announced.
The `payload` field contains the value that identifies this device automation. Multiple types of
device trigger could be published, like `button_double_press` or `button_long_press`.

The door sensor is published using the *Binary Sensor* entity type:

	Topic: homeassistant/binary_sensor/frontdoor/door/config
	Payload:
	{
		"device": {
			"identifiers": "frontdoor",
			"manufacturer": "DIY",
			"model": "Smart Doorbell",
			"name": "Doorbell"
		},
		"availability_topic": "diy/doorbell/available",
		"topic": "diy/frontdoor/door",
		"device_class": "door",
		"name": "Door",
		"unique_id": "frontdoor_door",
		"payload_on": "open",
		"payload_off": "closed"
	}

The `device_class` tells Home Assistant what icon to use for this entity.
A binary sensor has an on and off state, so we need to tell it which payload string means which.

These configuration messages are published with the *retained* flag set, so Home Assistant will receive
them even if it is (re)started after the device has connected to the broker.

To let Home Assistant know the device is online, we'll publish the retained message `online` at the `availability_topic`.
We also register the last-will message `offline` to be published at the same topic. This way, when the doorbell
goes offline for any reason, Home Assistant will know about it.

## Ringing the bell remotely

As a final extra feature, I decided to make it possible to ring the bell via MQTT command.

The doorbell listens on the topic `diy/doorbell/bell`. When it receives a message containing a number from 1-9,
it will ring the bell the given number of times.

There is no entity type for autodiscovering this feature, but we can still use it an automation by using the `mqtt.publish` service:

	service: mqtt.publish
	data:
	  topic: diy/frontdoor/bell
	  payload: "1"

MQTT messages can also be published via Node-RED.

