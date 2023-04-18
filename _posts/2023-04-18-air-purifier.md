---
layout: post
title:  "Smart Vacuum + Dumb Air Purifier"
subtitle: "Enhancing my robot vacuum experience with a little automation"
date:   2023-04-18 10:00:00 -0600
img: roborock-and-air-purifier.jpg

tags: [HomeAssistant, Roborock, vacuum, zigbee] # add tag
---
My Roborock S7 (named "Roxy") has been quietly docked in the corner of our dining room throughout the day for the past year. This peaceful coexistence with my toddler has been due primarily to his fear of this little demon that only comes alive at night. As he reminds us when he gets close, "Roxy sooo scary."

His younger brothers have learned to crawl but did not learn his fear of the robot, and Roxy had to be re-homed to our laundry room for its daytime rest to remain undisturbed. Having the Roborock auto empty dock in such a small room has developed that "just vacuumed" dusty smell when the bag gets over half full, a problem we never noticed when it was docked in the larger rooms.

The solution was easy: a smart outlet/plug, a dumb air purifier, and a little automation. After the vacuum finishes and returns to the dock to empty itself, I use the smart outlet to turn on an air purifier for a couple of hours to clear the air in the little room.

The [SONOFF S31 Lite Zigbee](https://www.amazon.com/gp/product/B082PSKRSP) plug has worked great. Or grab a ZWave or WiFi plug if that fits your setup better.

Next up is getting an air purifier. My first attempt didn't work and was returned to Amazon. It had this nice touch panel for the controls, but when the power was cutoff and restored by the smart plug, it reverted to an off stage and had to be manually touched to turn on. This was replaced with an even "dumber" [unit](https://www.amazon.com/dp/B07DFVYZCR) that had a manual dial that can be left in the on position.

## Implementation

Pair your chosen smart plug with Home Assistant.

There are many ways the automation could work in Home Assistant. It would be slightly simpler to use the "delay" or "timer" features available. However, those are in memory based and do not survive a restart or partial reload for parts of the configuration. That's likely not an issue considering this happens overnight when I'm rarely working on Home Assistant and doing restarts, but a better solution was only slightly more complex.

Two automations were added. The first automation turns on the outlet for the purifier when the vacuum returns to the dock. The second turns off the humidifier after 2 hours.

To control that 2 hour window, and additional helper input was created to save the time that the second automation should trigger.

Under: Settings > Devices & Services > Helpers > Create Helper, Select "Date and/or Time"

Give the helper a name and click the radio for "Date and Time"

Here's the first automation. When the vacuum returns to the dock, the smart outlet is turned on, and the value of our date/time helper is updated to 2 hours from the current time.

{% raw %}
```yaml
alias: Start Air Purifier after vacuuming
trigger:
  - platform: state
    entity_id:
      - vacuum.roborock_vacuum_a15
    attribute: status
    from: Cleaning
    to: Docked
  - platform: state
    entity_id:
      - vacuum.roborock_vacuum_a15
    attribute: status
    from: Returning to dock
    to: Docked
condition: []
action:
  - service: input_datetime.set_datetime
    data:
      timestamp: "{{ (now() + timedelta( hours = 2 )).timestamp() }}"
    target:
      entity_id: input_datetime.air_purifier_off
  - type: turn_on
    entity_id: switch.sonoff_s31_lite_zb_switch
    domain: switch
mode: single
```
{% endraw %}

The second automation is triggered by the date/time helper and simply turns the smart outlet off:

```yaml
alias: Stop Air Purifier after set time
description: ""
trigger:
  - platform: time
    at: input_datetime.air_purifier_off
condition: []
action:
  - type: turn_off
    entity_id: switch.sonoff_s31_lite_zb_switch
    domain: switch
mode: single
```
