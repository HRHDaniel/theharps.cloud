---
layout: post
title:  "Home Assistant Database Tuning"
subtitle: "How to keep the database from quickly filling your entire disk or destroying your SD Card"
date:   2023-03-15 10:00:00 -0600
img: HADBTuning.png
tags: [HomeAssistant] # add tag
---
The "recorder" component of Home Assistant is responsible for saving the history of all states and events. Unfortunately, it is also responsible for crashing many Home Assistant installations by producing too many I/O cycles on a SD card causing the card to fail or by completely filling up the storage to the extent that Home Assistant can no longer function. Both of these events happened to me before I learned more about configuring the recorder (and moving off a Raspberry Pi and SD Card installation).

If you haven't already set up [automated backups]({% post_url 2023-03-10-home-assistant3 %}), do that first. This post may help you save yourself from one possible and common failure, but backups help regardless of the failure.

## Selecting tracked entities
By default, the Home Assistant recorder tracks everything for 10 days to a local SQLite database. Some of this history may be very interesting to you (I particularly like having the data to graph the temperature and humidity sensors I have throughout the house), but most of this history I rarely or never looked at when it was collected.

The first step is to determine what history to capture. To get a list of your options, log in to Home Assistant, go to Developer Tools, and under Template enter the following:

{% raw %}
```yaml
{% for state in states %}
{{ state.entity_id -}} : {{ state.state -}}
{% endfor %}
```
{% endraw %}

Copy and paste that to Excel or any other editor and sort through the entities that you would like to track. You will have the option to do this by only including selected elements, including everything except excluded elements, or a combination. Additionally, elements can be selected by domain (all "light" or "sensor" entities) or with pattern matching ("sensor.weather_*")

Edit your Home Assistant `configuration.yaml` and update (or add) the recorder configuration as desired.

**Example 1:** Only includes (all lights, temperature, humidity, and a keyfob battery). All other entities will not be tracked.
```yaml
recorder:
  include:
    domains:
      - light
    entity_globs:    
      - sensor.*_temperature
      - sensor.*_humidity
    entities:
      - sensor.keyfob_battery_level
```

**Example 2:** Only excludes (includes everything except the excluded entities)
```yaml
recorder:
  exclude:
    domains:
      - automation
    entity_globs:    
      - sensor.washer_*
      - sensor.dryer_*
    entities:
      - light.master_bathroom_light
```

**Example 3:** Combining includes and excludes. The following will only record all lights, except the master bathroom light will be excluded.
```yaml
recorder:
  include:
    domains:
      - light
  exclude:
    entities:
      - light.master_bathroom_light
```

Two more settings to tune at this point:
- purge_keep_days: specifies how many days of history to keep
- commit_interval: specifies in seconds how often to commit events to the database.

purge_keep_days defaults to 10 days. Updating this will depend on your preference of what seems likely useful and how much disk space you have available for the storage.

commit_interval defaults to 1 second. This can cause a lot of I/O, particularly with SQLite on an SD card.  Increasing this value can significantly reduce the I/O on the database, but does come at a cost of delaying the availability of history to display in your dashboards or logbook. I set this to 30 seconds and have never noticed this "lag". I likely could set this to several minutes without noticing, but at 30 seconds I've neither noticed lag requiring this to be lowered or performance issues requiring it to be raised, so that is where it has stayed.

## External Database
As noted, Home Assistant uses a local SQLite database by default. I found many tutorials related to switching to MariaDB or other databases, but most of these were through integrations that ran MariaDB locally as well. When running on a Raspberry Pi I found it better to host my database on my NAS (QNAP), but this could be installed on any host.

After installing MariaDB:
- Create a new user for remote access: `CREATE USER homeassistant IDENTIFIED BY '***PASSWORD***';`
- Enable remote access: `GRANT ALL PRIVILEGES ON \*.\* TO 'homeassistant'@'%' IDENTIFIED BY '***PASSWORD***';FLUSH PRIVILEGES;`
- Home Assistant needs the InnoDB engine: `SET GLOBAL default_storage_engine = 'InnoDB';`
- Create the database: `CREATE DATABASE homeassistant_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;`

Now we update the recorder to use this external database:
```yaml
recorder:
  db_url: mysql://homeassisant:***password***@192.168.0.100:3307/homeassistant_db?charset=utf8mb4 
  purge_keep_days: 30
  commit_interval: 20
...
```