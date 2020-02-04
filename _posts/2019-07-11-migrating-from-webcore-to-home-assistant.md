---
layout: post
title: Migrating from WebCORE to Home Assistant
date: 2019-07-11 20:43
categories: [Home Automation]
tags: [alexa, automation, home assistant, home automation, smart home, webcore, z-wave]
---
About a month ago I decided to migrate all my home automation from [WebCORE](https://www.webcore.co/){:target="_blank"} to [Home Assistant](https://home-assistant.io){:target="_blank"}. I liked the idea of all my home automation being local and inside my house, both from a security and reliability perspective. Furthermore it just seemed like a fun project to tinker with, and in general seemed like it would add a bunch of flexibility to my home automation. In this post I'll describe my experience with the migration, the good and the bad.

However I do want to start with saying that I think the migration was worth it for me, but it had quite a few challenges and I have quite a few criticisms for Home Assistant. It was a significant time investment and required a lot of tinkering and trial and error, so if you're considering doing this yourself just be aware of that.

## Initial state
First to describe my initial setup. For equipment I have:
* SmartThings hub
* Hue hub
* Three Alexa dots
* Ten Z-Wave light switches
* Ten Hue lights
* Two Z-Wave door locks
* Motion sensor

Additionally, I have a handful of pistons configured with WebCORE (that's what it calls its automations) and presence sensors using the SmartThings app on my wife's phone and my phone to trigger the "Home" and "Away" home states.

## Installation of Home Assistant
I wanted to start with the easiest and most hand-holding Home Assistant configuration possible, so I bought a [Raspberry Pi starter kit](https://www.amazon.com/gp/product/B07BCC8PK7){:target="_blank"} ($80). I knew that I could get it cheaper if I got the parts separately, but as this was my first experience with a Raspberry Pi, I decided to go with a kit. Assembling the Raspberry Pi was definitely a smooth and easy process.

I then proceeded to [install Hass.io](https://www.home-assistant.io/getting-started/#installing-hassio){:target="_blank"}. The instructions are easy enough to follow, but on first boot I ended up getting stuck. It might have been a patience issue, but eventually after a combination of restarts of the Pi, trying to access via IP address, and flushing my computer's DNS lead to successfully being able to access the web UI.

## Add-ons
Next I wanted to set up some of the basic add-ons. I wanted remote access (especially from my phone), so I installed the DuckDNS addon and it was one of the smoother steps in this process. It just worked, although I did change my mind on the DuckDNS subdomain to use and Hass.io seemed to get confused by that. Uninstalling and reinstalling the add-on fixed everything up.

I also wanted to more easily edit my configuration files in a text-editor so installed the Samba add-on. The config UI was unclear to me (a common criticism during this whole process) and so I had a little trouble with the configuration, compounded by an extremely unclear error message (another common criticism), but eventually after going to the documentation for the [Samba](https://www.home-assistant.io/addons/samba/){:target="_blank"} add-on, the configuration became more clear and was easy to set up from there. As a side note, personally I'm just not a fan of yaml. I know it's the go-to for configuration, but I just find it hard to read and edit.

## Integrations and Components
At this point I wanted to actually start setting up my smart devices and controlling them from Home Assistant. Most my devices were managed through SmartThings, so I started there. I have to say that this was probably the most challenging part of this process. The first step in setting up the integration is to provide a SmartThings PAT, which I did, but ended up with the error: "_The `base_url` for the `http` component must be configured and start with `https://`._" Looking back, now that I understand Home Assistant better, I can parse it better, but to a newbie I was lost. Eventually I found that I needed to add some [configuration](https://www.home-assistant.io/components/http#base_url){:target="_blank"} to my configuration.yaml. The next problem I ran into was adding the SmartApp in SmartThings. Every time I tried adding it, it told me it could not connect. Eventually I turned off DNS66 (Host-based adblocker for Android) and it seemed to make progress. At that point I could control my SmartThings devices from Home Assistant, but for whatever reason I could not get Home Assistant to update the status when I turned lights on or off outside of Home Assistant (manually, via Alexa, or via SmartThings). I saw errors about unregistered webhooks, so I suspect SmartThings was attempting to update Home Assistant, but Home Assistant lost some configuration about the webhook SmartThings was using. I eventually just removed and re-added the integration and things started working from there.

Phillips Hue was much more straightforward and after SmartThings this integration was extremely easy to set up.

Next I tried setting up presence, called device trackers in Home Assistant. The Getting Started docs were fairly hard to follow, but I eventually successfully enabled OwnTracks after lots of guesswork. After a few days I noticed that OwnTracks is pretty inaccurate, so using a fairly large radius for Zones is important. Don't expect to walk to your neighbors' and expect it to detect you as away. I looked into the Google Maps Location Sharing component instead, but the setup is pretty hacky (involves setting up a dummy Google account) and at the end of the day I just couldn't get it working at all. After tweaking the OwnTracks settings over a week or two and learning its limitations, I feel like it's now in a usable state. I've learned that reliable and accurate presence using your phone's GPS is just a hard problem to solve (unless you want to just kill your phone's battery), so in the future I may look into layering in other presence signals like Wifi connection to my router. This is possible with the fairly recent addition of the [Person](https://www.home-assistant.io/components/person/){:target="_blank"} component.

Related to presence, I set up [Zones](https://www.home-assistant.io/components/zone/){:target="_blank"} for Home, Work, and my in-laws' house. It would be nice to have a better UI, for example just typing in an address and it giving you the lat/long, but you can do that through Google/Bing maps as well. It just would be nice to do it within Home Assistant directly. Besides that and the earlier mentioned need to have a larger-than-expected radius, Zones are pretty easy to work with and I find it quite useful.

As this process took a month, my wife and I ended up impulse purchasing an [ecobee3 lite](https://www.ecobee.com/ecobee3-lite/){:target="_blank"} from Costco since it was on sale. The setup of the thermostat itself and the integration into Home Assistant were both very easy and I have no complaints.

## Automations
Next I wanted to to what the title of this post suggests, actually migrate my automations from WebCORE to Home Assistant. I found that the UI for this really sucks and found myself very quickly just dealing with the yaml directly. I did find the documentation very hard to dig through, but eventually figured out the basic syntax. Still, the logging and debugging for automations is not great and I think that's one advantage WebCORE has over Home Assistant.

Global state is also a little more awkward to manage in Home Assistant than WebCORE. In WebCORE there is a notion of global variables directly, but in Home Assistant you have to fake it using an input one would typically use in the UI to control you devices.

I also found that Home Assistant's templating leaves a bit to be desired. It's not documented in a very helpful way and I found myself needing to do lots of trial and error in the Developer Tools Templates page. Specifically, I have an automation to cap the brightness level for my dimable lights to 80% (ie. if their brightness changes to > 80%, set it to 80%). In WebCORE this was fairly straightforward, but in Home Assistant it's something more like...

{% raw  %}
```yml
- alias: Max light brightness
  hide_entity: true
  initial_state: "true"
  trigger:
    - platform: state
      entity_id: ... long list of lights ...
  condition:
    condition: template
    value_template: "{{ (100 * trigger.to_state.attributes.brightness|float / 255) > states.input_number.max_brightness.state|float }}"
  action:
    - service: light.turn_on
      data_template:
        brightness_pct: "{{ states.input_number.max_brightness.state | int }}"
        entity_id: "{{ trigger.entity_id }}"
```
{% endraw %}

It might just be my inexperience with Home Assistant, but I found it very difficult to simply check if the brightness of the triggering device was above some configured max and if so set it to that configured max.

## Lovelace UI
I haven't had a chance to heavily customize the UI, but from what I have done so far, and from what the demos show me I can do, it looks extremely powerful and flexible. This alone adds a ton of value to Home Assistant as it opens up so many scenarios, such as mounting a tablet running a browser in kiosk mode as a control panel for your home. Another scenario being to flash the WyzeCams I currently have (which are currently not really connected to the rest of my smart home in any way) with the firmware which supports RTSP and adding them to the Home Assistant UI. There are a ton of possibilities here and I'm pretty excited to explore this are more in the future.

## Hass.io Updates
I also noticed over the past month that Home Assistant updates are a little destabilizing. It's really great that updates are frequent and as a software developer myself I appreciate the quick updates and even the need to deprecate features in favor of better ways to do things, but I did find that the updates did break things for me from time to time. Perhaps if some migration scripts would run automatically, or there were clear instructions on the blog of how to migrate it wouldn't be so bad, but I found that it was more like "this is deprecated in favor of that, figure it out yourself". Specifically in my case I believe my OwnTracks device trackers were duplicated (suffixed with "_2") and the originals were just stuck in their last state. This led to presence detection just being broken until I went in, deleted the originals, and renamed the dupes. Little things like that could use a little more polish I think.

## Next Steps
My initial goal was to migrate from WebCore to Home Assistant, which I succeeded in, but I realized that I do need to take it a bit further to really get what I wanted out of it. For example, my devices are still primarily controlled through SmartThings, and Home Assistant talks to SmartThings via the SmartThings cloud (as opposed to directly talking to the hub on my LAN...), so my dream of everything being 100% local isn't quite realized. Automations can still be delayed as SmartThings devices signal to my hub to update the cloud to webhook back to my house in Home Assistant and finally trigger the automation, which take action on a SmartThings device which has to make another trip through the internet.

The full list of my future plans include:
1.  Migrate from SmartThings to a local Z-Wave controller (USB stick plugged into my Hass.io RPi, likely the [Aeotech Z-Stick](https://www.amazon.com/Aeotec-Z-Stick-Z-Wave-create-gateway/dp/B00X0AWA6E){:target="_blank"}) and managed completely locally my Home Assistant.
2.  Alexa integration with Home Assistant (via [Haaska](https://github.com/auchter/haaska){:target="_blank"}), a likely pre-req for moving to the local Z-Wave controller.
3.  Migrate my ADT wireless-but-not-smart security system to Home Assistant. Both so I have more control (ADT really doesn't want me fiddling with the devices **I own**), better expandability (integration with my existing smart locks), and because ADT just charges way too much for what they provide ($66/month for basic burglar monitoring, no smart devices, no cameras, no fire/smoke). I'll likely move to [Noonlight](https://noonlight.com/){:target="_blank"} which I read costs only $3/month and they provide integrations like IFTTT that I can trigger however I want to get them to contact the police for me.
4.  Integrate my WyzeCams into my Home Assistant by flashing them with the RTSP-enabled firmware
5.  Alexa TTS (if possible) to use Alexa as a general-purpose speaker to announce things like people in my home arriving.
6.  Generally explore more in the home automation space
But those are for a future blog post!
