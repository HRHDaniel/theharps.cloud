---
layout: post
title:  "Ecobee integration to Home Assistant"
subtitle: 
date:   2023-03-17 10:00:00 -0600
img: ecobee3lite.png
tags: [Ecobee, HomeAssistant] # add tag
---
Integrating my Ecobee lite thermostat into Home Assistant is ultimately trivial, if you watch out for some key issues.

The first "issue" is how to integrate. Home Assistant does provide an Ecobee integration. It may even self discover this and prompt you to integrate. However, that integrates through the Ecobee cloud. If you would rather have direct/local integration, do not accept this self discovery. Instead we will integrate through HomeKit.

You can follow Ecobee's documentation for setup and connecting to your WiFi. The biggest issue here is if you are using VLANS, your best option is to pace the Ecobee on the same VLAN as Home Assistant. HomeKit uses a discovery protocol called Bonjour, which is an implementation mDNS. This likely works great on the average consumer grade router. If you're using VLANs or more extensive router/firewall rules between devices on your LAN, you will have to ensure these mDNS broadcasts are routable between the Ecobee and Home Assistant.

In Home Assistant, select Add Integration, search for "Apple" (or "HomeKit").  You'll see two entries listed with HomeKit, and you want the one that says "HomeKit Controller".
![HomeKit Controller](/imgs/HomeKitController.png)

The **"HomeKit"** integration allows broadcasting your Home Assistant entities to a HomeKit hub so that your Home Assistant can be controlled through HomeKit.

The **"HomeKit Controller"** integration has Home Assistant acting as the HomeKit hub so that it can control HomeKit devices. In my case, I want Home Assistant to control the Ecobee thermostat, so it needs to act as the hub.

On the Ecobee enable HomeKit pairing. It will display a pairing key to enter in Home Asstistan when adding the "HomeKit Controller" integration.

On my Dashboard, I display a card grid with the following configuration:
```yaml
      - type: grid
        square: false
        columns: 1
        cards:
          - type: entities
            entities:
              - entity: sensor.living_room_thermostat_current_humidity
                name: Thermostat Current Humidity
              - entity: sensor.living_room_thermostat_current_temperature
                name: Thermostat Current Temperature
              - entity: button.living_room_thermostat_clear_hold
                name: Thermostat Clear Hold
              - entity: select.living_room_thermostat_current_mode
                name: Thermostat Current Mode
            title: Living Room
          - type: thermostat
            entity: climate.living_room_thermostat
```

![DashBoard](/imgs/ecobeeHomeAssistantDashboard.png)
