---
layout: post
title: Did you forget to set the alarm?
categories: [Home Automation]
tags: [automation, home assistant, home automation, smart home]
comments: true
---

After [setting up my security system]({% post_url 2020-04-25-setting-up-a-security-system-with-home-assistant %}) in [Home Assistant](https://www.home-assistant.io/){:target="_blank"}, a common problem my wife and I ran into was remembering to arm it when we left the house. A security system is only effective when you use it, and we just could not consistently remember to set it.

In general, we want to arm the alarm when we're not home. However, instead of doing it automatically, we preferred that a notification be sent to our phones reminding us to set it. This is because historically our presence detection has been a bit spotty, and there are also some scenarios where we might want to leave the alarm unarmed when we leave, for instance if we have guests and we need to run out to pick up take-out or something.

The individual components of this automation all seem pretty straightforward, but the state management gets a bit hairy. So let's walk through it piece by piece.

First the trigger and condition. We want the automation to run when we leave the house and forget to arm the alarm. Note that I have a group set up containing all `person` entities, and each `person` has an associated device tracker (via the [Mobile App](https://www.home-assistant.io/integrations/mobile_app/#mobile-app-documentation){:target="_blank"}). This is fairly straightforward:

```yml
  trigger:
    platform: state
    entity_id: group.all_people
    to: "not_home"
  condition:
    condition: state
    entity_id: alarm_control_panel.alarm
    state: disarmed
```

Next I wanted to just send a simple notification. I wanted the notification to come to both me and my wife's phones, so I created a [notify group](https://www.home-assistant.io/integrations/notify.group/){:target="_blank"} containing both in `configuration.yml`:

```yml
notify:
  - name: all_phones
    platform: group
    services:
      - service: mobile_app_person1_phone
      - service: mobile_app_person2_phone
```

Now back in the automation, the group notification becomes pretty straightforward, including configuring it so that if you click on it, it brings you to the second lovelace tab (index 1) which is where my alarm panel card is:

```yml
  action:
    - alias: Send notification
      service: notify.all_phones
      data:
        title: Home Alarm
        message: Did you forget to set the alarm?
        data:
          channel: "home-alarm-reminder"
          tag: "home-alarm-reminder"
          importance: max
          priority: high
          ttl: 0
          vibrationPattern: 0, 250, 250, 250
          clickAction: /lovelace/1
```

Next I wanted to make the notification [actionable](https://companion.home-assistant.io/docs/notifications/actionable-notifications/){:target="_blank"} so that the alarm could be armed directly rather than having to navigate to HA and use the alarm panel card. This is where things get slightly more complex. Showing additional actions in the notification isn't too much extra, but handling it and having it perform an action requires a bit of fanciness.

In particular, the [`wait_for_trigger`](https://www.home-assistant.io/docs/scripts/#wait-for-trigger){:target="_blank"} action is used to wait for an event corresponding to the notification action ([`mobile_app_notification_action`](https://companion.home-assistant.io/docs/notifications/actionable-notifications?_highlight=mobile_app_notification_action#building-notification-action-scripts){:target="_blank"}). This event also has a name associated with it, and a unique one is preferred so that it doesn't collide with other notification actions. This can be done by dynamically defining a variable which uses the automation's `context.id` as a way to guarantee uniqueness.

```yml
  action:
    - alias: Set up variables
      variables:
        # Including an id in the action allows us to identify this script run
        # and not accidentally trigger for other notification actions
        action_arm: "{{ "{{ 'arm_' ~ context.id " }}}}"
    - alias: Send notification
      service: notify.all_phones
      data:
        title: Home Alarm
        message: Did you forget to set the alarm?
        data:
          channel: "home-alarm-reminder"
          tag: "home-alarm-reminder"
          importance: max
          priority: high
          ttl: 0
          vibrationPattern: 0, 250, 250, 250
          clickAction: /lovelace/1
          actions:
            - action: "{{ "{{ action_arm " }}}}"
              title: Arm Alarm
            - action: URI
              title: Open Alarm Panel
              uri: /lovelace/1
    - alias: Wait for a response
      wait_for_trigger:
        - platform: event
          event_type: mobile_app_notification_action
          event_data:
            action: "{{ "{{ action_arm " }}}}"
    - alias: Perform the action
      service: alarm_control_panel.alarm_arm_away
      target:
        entity_id: alarm_control_panel.alarm
```

So far so good. However, I wanted the notification to clear when the alarm was armed in some way (potentially by the other person), or we came back home. So this required some extra triggers in the `wait_for_trigger` to unblock the automation execution, a condition on the arming action, and a final action to execute for [clearing the notification](https://companion.home-assistant.io/docs/notifications/notifications-basic?_highlight=clear_notification#clearing){:target="_blank"}.

```yml
    - alias: Wait for a response
      wait_for_trigger:
        - platform: event
          event_type: mobile_app_notification_action
          event_data:
            action: "{{ "{{ action_arm " }}}}"
        # Stop waiting if it becomes moot (alarm gets set some other way or if we come home)
        - platform: state
          entity_id: alarm_control_panel.alarm
          from: disarmed
        - platform: state
          entity_id: group.all_people
          to: home
    - alias: Perform the action
      choose:
        # Handle the arm action
        - conditions: "{{ "{{ (wait is defined) and (wait.trigger is not none) and (wait.trigger.event.data.action == action_arm) " }}}}"
          sequence:
            - service: alarm_control_panel.alarm_arm_away
              target:
                entity_id: alarm_control_panel.alarm
    # Clear notifications once an action is taken *or* after it becomes moot
    - alias: Clear notifications
      service: notify.all_phones
      data:
        message: "clear_notification"
        data:
          tag: "home-alarm-reminder"
```

The `wait_for_trigger` now gets unblocked if the action is tapped, the alarm is armed, or we come home. This means the arming action needs to check that the wait trigger was specifically the action being tapped and not the other triggers. Finally, the notification is cleared from all phones, no matter which of the triggers unblocked execution.

Piecing everything together, this brings the full automation to the following:

```yml
# NOTE: This automation's primary purpose is simply to send a notification, but to manage the
# notification state, including clearing notifications once it's done, the automation doesn't
# finish running until either the alarm is set or someone returns home.
- alias: Alarm - Notification for disarmed alarm when no one is home
  id: alarm_forget_notification
  trigger:
    platform: state
    entity_id: group.all_people
    to: "not_home"
  condition:
    condition: state
    entity_id: alarm_control_panel.alarm
    state: disarmed
  action:
    - alias: Set up variables
      variables:
        # Including an id in the action allows us to identify this script run
        # and not accidentally trigger for other notification actions
        action_arm: "{{ "{{ 'arm_' ~ context.id " }}}}"
    - alias: Send notification
      service: notify.all_phones
      data:
        title: Home Alarm
        message: Did you forget to set the alarm?
        data:
          channel: "home-alarm-reminder"
          tag: "home-alarm-reminder"
          importance: max
          priority: high
          ttl: 0
          vibrationPattern: 0, 250, 250, 250
          clickAction: /lovelace/1
          actions:
            - action: "{{ "{{ action_arm " }}}}"
              title: Arm Alarm
            - action: URI
              title: Open Alarm Panel
              uri: /lovelace/1
    - alias: Wait for a response
      wait_for_trigger:
        - platform: event
          event_type: mobile_app_notification_action
          event_data:
            action: "{{ "{{ action_arm " }}}}"
        # Stop waiting if it becomes moot (alarm gets set some other way or if we come home)
        - platform: state
          entity_id: alarm_control_panel.alarm
          from: disarmed
        - platform: state
          entity_id: group.all_people
          to: home
    - alias: Perform the action
      choose:
        # Handle the arm action
        - conditions: "{{ "{{ (wait is defined) and (wait.trigger is not none) and (wait.trigger.event.data.action == action_arm) " }}}}"
          sequence:
            - service: alarm_control_panel.alarm_arm_away
              target:
                entity_id: alarm_control_panel.alarm
    # Clear notifications once an action is taken *or* after it becomes moot
    - alias: Clear notifications
      service: notify.all_phones
      data:
        message: "clear_notification"
        data:
          tag: "home-alarm-reminder"
```

I hope this helps you get the most out of your security system, namely by being reminded to actual use it!