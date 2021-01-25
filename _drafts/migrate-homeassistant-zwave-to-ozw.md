---
layout: post
title: Migrate from Home Assistant ZWave to OpenZWave Beta
categories: [Home Automation]
tags: [automation, home assistant, home automation, smart home, z-wave]
comments: true
---

## TODO

Explain difference between addon and integration

### My setup

### Take a backup

### Copy current entities
* Developer Tools
* Filter attributes by "node_id"

![Filtering by node_id in Developer Tools](/assets/zwave-nodeid-devtools.PNG){: .center }

* Copy and paste into Excel.

Hint: click into the entity table, ctrl+a, in excel ctrl+v, delete rows 2 and 3 (empty row and filter input row), Select "Data -> Filter", sort attributes (Note 1, 10, 11, 20, etc)

* Cross check entity count

Mine shows 107 entities, Excel shows 108 rows (1 extra for header)

![Node id backups in Excel](/assets/zwave-nodeid-excel.PNG){: .center }

![Existing Z-Wave integration](/assets/zwave-integration-old.PNG){: .center }

* Save current Z-Wave configuration

```yaml
zwave:
  usb_path: /dev/ttyACM0
  network_key: !secret zwave_network_key
  # Attempt to mitigate switch update delays
  polling_interval: 30000
  device_config_domain:
    switch:
      polling_intensity: 1
  device_config_glob:
    sensor.*_sourcenodeid:
      ignored: true
```

Or just comment out.

* Uninstall existing Z-Wave integration

Integrations page - Delete. Restart HA.

* Install OpenZWave add-on

![Mosquitto and OpenZWave in the add-on store](/assets/zwave-addons-store.PNG){: .center }

Mosquitto broker addon

Docs: https://github.com/home-assistant/addons/blob/master/mosquitto/DOCS.md

Install and enable watchdog. Default configuration is fine. Start the addon

OpenZWave add-on

Docs: https://github.com/home-assistant/addons/blob/master/zwave/DOCS.md

Install and enable watchdog.

Add device USB path and network key to configuration. Paste the actual network key, not the secret name.

![OpenZWave Configuration](/assets/zwave-addon-config.PNG){: .center }

Save and start the add-on.

* Add OpenZWave Integration

Docs: https://www.home-assistant.io/integrations/ozw/

![Discovered MQTT and OpenZWave integrations](/assets/zwave-integrations-new.PNG){: .center }

Configure both the MQTT and OpenZWave integration. Neither have any noteworthy settings for the initial configuration.

![Discovered MQTT and OpenZWave integrations](/assets/zwave-integrations-new-configured.PNG){: .center }

You may notice that in the image above only 59 entities are present instead of the original 107. Entities may be different. No `zwave.*` entities. Added unavailable and disabled entities, for example all light switches now have an entity ending in `_basic`, `_basic_duration`, and `_basic_target`. Device count is more accurate. Also replaced some entities with others.

Device count is also missing some, 30 vs 16. Battery-powered Z-Wave devices may not initially be present until they "check in" with the controller (your Z-Wave stick).

* Rename entity id and display names

Tedious

Rename devices, then entities.

![Rename entity prompt](/assets/zwave-device-nodeid.PNG){: .center }

![Rename entity prompt](/assets/zwave-device-rename.PNG){: .center }

Messed up?
`Garage Door: Access Control: Door/Window Open`
`binary_sensor.garage_door_access_control_door_window_open`

* Z-Wave Node Configuration

Readonly in the UI.

Settable using `ozw.set_config_parameter` service.

Docs: https://www.home-assistant.io/integrations/ozw/#service-ozwset_config_parameter