---
layout: post
title: Migrate from Home Assistant legacy Z-Wave to Z-Wave JS
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

Hint: click into the entity table, `ctrl+a`, in Excel ctrl+v, delete rows 2 and 3 (empty row and filter input row), Select "Data -> Filter", sort attributes (Note 1, 10, 11, 20, etc)

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

Integrations page - Delete.

* Restart HA.

* Install Z-Wave JS add-on

![Z-Wave JS in the add-on store](/assets/zwave-addons-store.PNG){: .center }

Docs: https://github.com/home-assistant/addons/blob/master/zwave_js/DOCS.md

Wait... Could take a couple minutes.

Add device USB path and network key to configuration. Paste the actual network key, not the secret name.

Note: had to "Edit in YAML", device in dropdown was not recognized.

![Z-Wave JS Configuration](/assets/zwave-addon-config.PNG){: .center }

Paste auto-formatting, 0x format.

Save the configuration, enable watchdog, (optional) auto-update, start the add-on.

* Add Z-Wave JS Integration

Docs: https://www.home-assistant.io/integrations/zwave_js/

![Adding the Z-Wave JS integrations](/assets/zwave-integration-add.PNG){: .center }

Configure the integration.

![Configuring the Z-Wave JS integration](/assets/zwave-integration-configuring.PNG){: .center }

Submit and click through to finish. We'll configure and rename each device later.

![Configured Z-Wave JS integration](/assets/zwave-integration-done.PNG){: .center }

You may notice that in the image above only 26 devices of the original 30 are shown. The device count is more accurate with the new integration. With the old integration, I had several dead nodes which showed up as devices with no entities in Home Assistant.

You may also notice that battery-powered Z-Wave devices may not initially be properly recognized or populated with entities until they "wake up" and check in with the controller (your Z-Wave stick).

![An asleep Z-Wave device](/assets/zwave-device-asleep.PNG){: .center }

Most devices will wake up on some time interval, or you can look up how to manually wake up a device by reading the manual for that specific device, which usually involes pressing a physical button on the device.

The overall status of the Z-Wave network, including how many nodes are ready, but clicking on "Configure" for the integration.

![Configure the Z-Wave JS integration](/assets/zwave-integration-configure.PNG){: .center }

Because the integration uses a completely different back-end, entities may be different too. All the old `zwave.*` entities are gone, and the are some added disabled entities, for example all my light switches now have an entity ending in `_basic`. Beyond some additions and substrations, some entities will just be different.

* Rename entity id and display names

Tedious

Rename devices, then entities.

![Getting the node id from a device](/assets/zwave-device-nodeid.PNG){: .center }

![Rename entity prompt](/assets/zwave-device-rename.PNG){: .center }

* Z-Wave Node Configuration not yet available.

> Configuration of Z-Wave nodes and/or configuration with the Home Assistant UI is currently not yet implemented.