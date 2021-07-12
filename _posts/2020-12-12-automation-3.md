---
layout: post
title:  "Home Automation Journal #3 - Zigbee2mqtt and IKEA TRÅDFRI (again)"
description: Revised notes on how to set up TRÅDFRI devices with zigbee2mqtt
date:   2020-12-12 12:00:00 +0200
tags: home-automation
---

This is a continuation of my first post in which I wrote about using Home Assistant together
with IKEA TRÅDFRI lights and zigbee2mqtt. The CC2531 based adapter I was using turned out
to be too weak to reliably handle a network with about 30 devices on it (but it did work
very well with smaller networks,) so I had to move all the lights back to the TRÅDFRI gateway.

Some months ago I bought a [zig-a-zig-ah!](https://electrolama.com/projects/zig-a-zig-ah/),a CC2652R based adapter, which can handle larger networks and I've transferred all the TRÅDFRI lights (and curtains) onto it. So far, it has been working well with no devices mysteriously dropping out like happened with the previous stick. 

Here are my revised notes on how to set up TRÅDFRI lights with their remote controls with zigbee2mqtt.

<!--more-->

## Pairing lights and remotes to zigbee2mqtt

Pair the lamp by first enabling pairing in zigbee2mqtt, then toggling the lamp on and off a few times quickly. The light will start pulsing to indicate it's in pairing mode. (Check the [supported devices page](https://www.zigbee2mqtt.io/information/supported_devices.html) for device specific instructions.)

Note: If there is more than one bulb in the fixture, unscrew the others and pair them one at a time. The process
does not seem to work reliably if you try to pair more than one device at a time.

You should see something like this in the MQTT topic `zigbee2mqtt/bridge/log`:

	{
		"type": "device_connected",
		"message": {
			"friendly_name": "0x14b457fffe9191cb"
		}
	}

	{
		"type": "pairing",
		"message": "interview_started",
		"meta": {
			"friendly_name": "0x14b457fffe9191cb"
		}
	}


	{
		"type": "device_announced",
		"message": "announce",
		"meta": {
			"friendly_name": "0x14b457fffe9191cb"
		}
	}

	{
		"type": "pairing",
		"message": "interview_successful",
		"meta": {
			"friendly_name": "0x14b457fffe9191cb",
			"model": "LED1732G11",
			"vendor": "IKEA",
			"description": "TRADFRI LED bulb E27 1000 lumen, dimmable, white spectrum, opal white",
			"supported": true
		}
	}

That final `interview_successful` message indicates the pairing completed successfully.

The default friendly name (the name used as the MQTT topic for the device) is the device's MAC address.
It's a good idea to rename it to something descriptive:

    Topic: zigbee2mqtt/bridge/config/rename
    
	Payload:
    {
        "old": "0x14b457fffe9191cb",
        "new": "ceiling_light"
    }

To use the light with a remote control, adding it to a [group](https://www.zigbee2mqtt.io/information/groups.html)
appears to be necessary.
Create a new group with:

    Topic: zigbee2mqtt/bridge/config/add_group
    
	Payload:
    {
        "friendly_name": "GROUP_FRIENDLY_NAME"
    }

Then add the light(s) to the group

	Topic: zigbee2mqtt/bridge/group/GROUP_FRIENDLY_NAME/add

	Payload:
    LIGHT_FRIENDLY_NAME
    
Pair a remote by opening the back cover and clicking the reset button quickly four times.
You should see something like this in the log topic:

    {
        "type": "pairing",
        "message": "interview_successful",
        "meta": {
            "friendly_name": "0xccccccfffe6cddd7",
            "model": "E1524/E1810",
            "vendor": "IKEA",
            "description": "TRADFRI remote control",
            "supported": true
        }
    }

For some reason, the remotes can be a bit tricky to pair, with the interview failing
multiple times before it goes through, especially if low on battery.

Rename remote the same was as the light above. I like to attach a little name sticker to the back cover to identify them.

Next, the remote must be [bound](https://www.zigbee2mqtt.io/information/binding.html) to the group. The IKEA remotes
are bound to a random group by by default, so we need to unbind it first for everything to work right:

    Topic: zigbee2mqtt/bridge/unbind/REMOTE_FRIENDLY_NAME
    
	Payload:
    default_bind_group

Note: Click remote button just before sending this message to ensure the remote
wakes up to receive it.

Now the remote can be bound the our own group:

    Topic: zigbee2mqtt/bridge/bind/REMOTE_FRIENDLY_NAME
    
	Payload:
	GROUP_FRIENDLY_NAME

The remote should now be able to control the light(s), even without the coordinator!

Pairing the FYRTUR curtains (and binding its remote) is done the same way as the lights.

## Light groups in Home Assistant

**Edit**: *As of Zigbee2mqtt version 1.20, this section is obsolete. Light groups now work without any manual configuration.*

A light group is a group of lights that act like a single device. Useful when you have a light fixture that holds multiple bulbs.

First of all, add this line to your HA `configuration.yaml` file so you can configure
the lights in a separate file. This will make the configuration file cleaner if you have lots.

	light: !include lights.yaml

Light groups can be defined in two different ways.

You can use the `group` platform to group together any light entities. This is (currently)
the only way of creating light groups when using the TRÅDFRI gateway, since it only exposes
individual lights:

	- platform: group
	  name: Ceiling light fixture
	  entities:
		- light.lamp1
		- light.lamp2

However, zigbee2mqtt exposes topics for directly controlling a zigbee group.
~~It does not publish an autodiscovery message for Home Assistant, so you will need to configure
it manually~~ (see also the [official documentation](https://www.zigbee2mqtt.io/integration/home_assistant.html))
by putting something like this in your lights.yaml file: 

	- platform: mqtt
	  schema: json
	  name: "HOME ASSISTANT NAME"
	  command_topic: "zigbee2mqtt/GROUP_NAME/set"
	  state_topic: "zigbee2mqtt/GROUP_NAME"
      availability_topic: "zigbee2mqtt/bridge/state"
	  color_temp: true
	  brightness: true

The `name:` key will set the name with which this group will appear in Home Assistant.
The group's zigbee2mqtt friendly-name is used in the topics.

Features supported by the bulbs must be specified: `color_temp`, `rgb` and `brightness`.

