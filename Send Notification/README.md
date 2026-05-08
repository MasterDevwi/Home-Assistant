# Send Notification script
Home Assistant script template for sending a notification to Home Assistant mobile apps or HTML5 endpoints.

## Overview
I use notifications all of the time, and they’re very powerful. But they can also be very hard to use. Every time I create a new notification I have to look up the YAML for specifying a tag or group, and more advanced features like critical notifications, badging, or including an image are even more complex. Features vary by platform, and you often have to carefully read through the [developer docs](https://companion.home-assistant.io/docs/notifications/notifications-basic) multiple times to understand all of the capabilities.

To make this easier, I created a script with as many of the notification features as possible. You can send and update notifications, remove notifications, set badge counts, and much more. The notifications also support actions, attachments, and even triggering Siri Shortcuts. The script has quite a few fields (I wish I could dynamically populate the script based on your selection), but I still think it’s far easier to use since you only have to set cross-platform features once, even if the syntax is different per-platform.

Note: I primarily use iOS, so I haven’t yet added some of the Android-only features like notification LED colors, alert once, etc. But it includes most other features so far. It also supports both mobile (iOS/Android) and HTML5 notifications.

> ⚠️ **Set up this script before May 2026?** See [Migrating from a previous version](#️-migrating-from-a-previous-version) for breaking changes and required migration steps.

## Pre-requisites / setup instructions

1. You must already have [mobile app](https://www.home-assistant.io/integrations/mobile_app/) and/or [HTML5 notifications](https://www.home-assistant.io/integrations/html5) set up.
2. Copy the code for the script ([send_notification.yaml](send_notification.yaml)) into a new script in Home Assistant.
3. **Make sure the ID for each `notify.*` entity matches its underlying service name** (see "Entity vs. service naming" below). This is the single most important setup step — if entities are renamed away from the convention, the script will fail.
4. **Create notify groups** so the script has a single target per recipient (see "Creating notify groups" below).
5. In the script's `send_to` field, open the Selector Options and edit the list to match the groups and individual entities you want to expose. The `value` of each option must equal the entity's ID (i.e. everything after `notify.`). For example, if your group is `notify.family`, the value is `family`.
6. That's it — the script handles everything else (HTML5 vs. mobile_app routing, payload assembly, dismissals, etc.) automatically based on the integration each entity belongs to.

### Entity vs. service naming

Home Assistant exposes notify targets in two parallel forms:

| Form | Example | Used by |
|---|---|---|
| Modern entity | `notify.williams_iphone` | UI group helpers, `html5.send_message`, `expand()` |
| Legacy service | `notify.mobile_app_williams_iphone` | The actual `action:` call this script makes for non-HTML5 targets |

For **mobile_app** entities, the legacy service is always `notify.mobile_app_<entity_slug>` — the script auto-prepends `mobile_app_`. **The entity slug must match the slug HA assigned at registration time.** If you rename the entity in the UI (e.g. `notify.william_s_iphone` → something else), the legacy service slug stays locked to the original and dispatch will fail with `Action notify.<name> not found`. The fix is to rename the entity back so its slug matches the registered service name (check Developer Tools → Actions for the actual `notify.mobile_app_*` service name and rename the entity to match). 

Note: Eventually this script will migrate to use the new modern entities for sending the notification. See below for more details.

For **HTML5** endpoints, the entity_id can be anything — the script identifies them by integration (`integration_entities('html5')`) and uses the modern `html5.send_message` action with `target.entity_id`, so naming doesn't matter.

For **TVs and other notify integrations**, the entity_id IS the service name (no prefix), so naming should already match by default.

### Creating notify groups

The recommended path is **UI group helpers**:

1. Settings → Devices & Services → Helpers → Create Helper → **Group** → **Notify group**.
2. Name it (e.g. `Family`) — the resulting entity will be `notify.family`.
3. Add the member `notify.*` entities (mobile_app entities, HTML5 endpoints, TV notify entities — mix and match freely).

The script uses `expand()` on the chosen group, so any combination of mobile, web, and TV targets in one group works correctly — each member is dispatched via the right action automatically.

(YAML notify groups in `configuration.yaml` also work; they expand the same way.)

### Other things worth knowing

- **HA version**: Targets HA 2026.5+ (uses `integration_entities()`, modern `html5.send_message` / `html5.dismiss_message` entity-based actions, and `notify.*` entity registration from mobile_app). Older versions may need adjustments.
- **Why the legacy `notify.<service>` call is still used for mobile**: HA's modern `notify.send_message` action only accepts `message` and `title` — no `data:` block. Until HA expands the entity schema to accept rich payloads, the per-service legacy call is the only way to deliver images, actions, sounds, badges, critical alerts, etc. to mobile_app and TV targets. HTML5 has its own dedicated rich entity schema, so it uses the modern action.
- **Script mode**: `parallel` with `max: 100`. You can fire many notifications in rapid succession without queueing.
- **Single target per call**: Each invocation targets exactly one `send_to` group/entity. To notify multiple groups, call the script multiple times (or wrap it in a parent script that fans out).
- **Failures are non-fatal**: The non-web branch uses `continue_on_error: true` per target, so one offline phone won't stop the rest of the group from receiving the notification.

---

## Using the script

The script is invoked like any other script — from an automation, another script, the Developer Tools → Actions panel, a dashboard button, etc. Below are the most common patterns.

### Minimal example (text only)

```yaml
action: script.send_notification
data:
  send_to: family
  message: Dinner is ready!
```

### Title + message + click action

```yaml
action: script.send_notification
data:
  send_to: william
  title: Garage door
  message: Left open for 10 minutes
  url: /lovelace/garage   # opens this dashboard view when tapped
```

### Critical alert (overrides Do Not Disturb / Focus)

```yaml
action: script.send_notification
data:
  send_to: parents
  title: Water leak detected
  message: Kitchen sensor triggered
  interruption_level: critical   # iOS: full critical alert; Android: high priority
  critical_alert_volume: 1       # 0.0–1.0 (iOS); ignored on Android unless 1
  critical_alert_tts_text: "Warning: water leak in the kitchen"   # Android TTS
```

### Notification with actions (buttons)

Each action is defined by a numbered set of fields (`action_1_*` through `action_10_*`). Only `action_N_id` and `action_N_title` are required per action; the rest are optional.

```yaml
action: script.send_notification
data:
  send_to: william
  title: Front door
  message: Someone is at the door
  action_1_id: UNLOCK_DOOR
  action_1_title: Unlock
  action_1_authentication_required: true   # iOS PIN/FaceID
  action_2_id: IGNORE
  action_2_title: Ignore
  action_2_use_red_text: true              # iOS destructive style
```

Then in your automation, listen for the `mobile_app_notification_action` event with `event_data.action: UNLOCK_DOOR` (mobile) or the `html5_notification.clicked` event (web) to handle the user's choice.

### Image / camera snapshot

```yaml
action: script.send_notification
data:
  send_to: family
  title: Motion detected
  message: Front yard camera
  image: /api/camera_proxy/camera.front_yard
  camera: camera.front_yard   # iOS only: live feed when notification is expanded
```

### Updating or replacing a notification (tag)

Send a notification with a `tag`, then send another with the same `tag` to replace it:

```yaml
action: script.send_notification
data:
  send_to: william
  tag: laundry_status
  title: Laundry
  message: Washer started
```

Later:

```yaml
action: script.send_notification
data:
  send_to: william
  tag: laundry_status
  title: Laundry
  message: Wash complete — switching to dryer
```

For browsers, also enable `renotify: true` if you want the second notification to actually re-alert the user instead of silently updating in place.

### Clearing a notification

Use `notification_type: clear_notification` with the same `tag`:

```yaml
action: script.send_notification
data:
  send_to: william
  notification_type: clear_notification
  tag: laundry_status
```

### Targeting only mobile or only web

By default (`device_type: all`), the script sends to every member of the group. To restrict:

```yaml
data:
  send_to: family
  device_type: mobile   # phones, tablets, TVs — anything except browsers
  message: ...
```

```yaml
data:
  send_to: family
  device_type: web      # browsers (HTML5) only
  message: ...
```

### Browser-specific options

These only affect HTML5 endpoints (silently ignored by mobile):

| Field | Effect |
|---|---|
| `silent: true` | Deliver without sound or vibration |
| `require_interaction: true` | Don't auto-dismiss; user must click or close |
| `renotify: true` | Re-alert on tag-update instead of silent replace |
| `urgency: high` | Push service delivery hint (`very-low` / `low` / `normal` / `high`) |
| `badge` | URL to a small monochrome badge icon (browser tray) |

### iOS Shortcut trigger

Run a Siri Shortcut on the phone when the notification is opened:

```yaml
data:
  send_to: william
  title: Bedtime
  message: Tap to start your nightly routine
  ios_shortcut_name: "Goodnight"
  ios_shortcut_key_1_name: source
  ios_shortcut_key_1_value: ha_notification
```

### Map preview (iOS)

```yaml
data:
  send_to: william
  title: Package out for delivery
  message: ETA 2:30 PM
  map_latitude: "37.7749"
  map_longitude: "-122.4194"
  map_shows_traffic: true
```

### Tips

- **Use groups, not individual entities, in `send_to`** — easier to maintain and the script transparently handles mixed mobile/web members.
- **Set a `tag` for any notification you might want to update or clear later.** Without a tag, you can't address that specific notification afterwards.
- **Use `interruption_level: passive`** for low-priority informational notifications so they don't wake the screen on iOS.
- **Test from Developer Tools → Actions** with `send_to: <yourself>` before wiring into automations.
- **Check the trace** of the script run if a delivery fails — the error message usually points directly at a malformed entity_id, missing service, or schema rejection.

---

## ⚠️ Migrating from a previous version

If you set up this script **before May 2026**, a few things changed that may need a one-time fix-up.

### What's different now

- **Browser (HTML5) notifications are picked up automatically.** You no longer need to maintain a list of which browsers belong to which person inside the script. If a browser is in the notify group you send to, it gets the notification.
- **The script understands [HA's new notify groups](https://www.home-assistant.io/blog/2026/05/06/release-20265) directly.** It looks at the group's members and sends each one through the right channel (phone, browser, TV, etc.) — you don't have to think about it.
- **A few new browser-only options** are available: `silent`, `require_interaction`, `renotify`, and `urgency`. Old calls keep working without them.

### Migration steps

You have two options:

**Option 1 — Migrate to the new HA groups (more effort, but recommended for future-proofing)**
1. Create a new notify group for each person (Settings → Devices & Services → Helpers → Create Helper → **Group** → **Notify group**) which includes both their mobile apps and their HMTL5 browsers. You can also create nested groups like `Parents` which includes multiple people, or `Family` which includes `Parents` and other people.
2. Update the script's **Send to** selector options so each `value` matches the new group's entity ID (e.g. `family`, `parents`, `william`) — no `notify_` prefix.
3. Update your existing notifications: replace `send_to: notify_family` with `send_to: family`, etc. (this is the part that will require the most effort)
4. Leave any entries that point directly at a single notify entity (e.g. an LG TV's `notify.lg_living_room`) as-is — only group names need updating.

**Option 2 — Keep your existing groups and `send_to` values (least effort for mobile compatibility, but uses old notify platform and "breaks" browsers)**
1. Open the new script in Home Assistant and copy your old script's `send_to` selector options over (or click the **Send to** field's selector → **Edit options** and re-add each option with `value` set to your existing group names, e.g. `notify_family`, `notify_parents`, `notify_william`).
2. **Browsers won't be reached automatically anymore.** The old script had a built-in list of which browsers belonged to each person; the new one doesn't. To send to browsers, you can create legacy notify service groups in notify.yaml. Or you can just add each browser separately in the script's **Send to** selector.

---

## Migrating from raw `notify.notify_*` calls

If you have existing automations and scripts that call the legacy `notify.notify_<group>` services directly (with `data: { ... }` payloads), this section is the field-by-field mapping to convert them to `script.send_notification`.

### Top-level call

| Before | After |
|---|---|
| `action: notify.notify_<group>` (or `service: notify.notify_<group>`) | `action: script.send_notification` |
| top-level `data:` block | flatten — fields go directly under `data:` (no nested `data.data:`) |

### Target / recipient

| Before | After |
|---|---|
| `notify.notify_william` | `send_to: william` (no `notify_` prefix) |
| Group sent to mixed mobile + web targets | (no extra config — the script auto-classifies group members by integration) |
| `device_type` not set | defaults to `mobile`. Set `device_type: all` to also reach browsers, or `device_type: web` for browsers only |

### Standard payload fields (rename / pass-through)

| Old (under `data.data:` or `data:`) | New (under `data:`) | Notes |
|---|---|---|
| `tag` | `tag` | Same — used for replace/clear |
| `group` | `group` | Same — iOS notification grouping |
| `channel` (Android, equal to `group`) | `group` | Auto-merged |
| `channel` (Android, **different** from `group`, e.g. `alarm_stream`) | `alarm_stream:` AND `alarm_stream_max:` | Set both to the channel value; the script picks one based on `critical` + `critical_alert_volume == 1` |
| `subtitle` | `subtitle` | |
| `image` | `image` | |
| `video` | `video` | |
| `audio` | `audio` | |
| `ttl` | `ttl` | |
| `subject` | `subject` | |
| `url` | `url` | |
| `clickAction` | `url` | Rename |
| `icon` / `icon_url` | `icon` | |
| `entity_id: camera.<x>` (deprecated) | `camera: camera.<x>` | |
| `attachment.hide-thumbnail: true` | `hide_media_thumbnail: true` | |
| `presentation_options: [badge, sound]` | `hide_notification_when_open: true` | iOS-only; suppresses banner when app is foregrounded |
| `tts_text` (Android critical TTS) | `critical_alert_tts_text` | |

### `push:` block (iOS payload)

| Before | After |
|---|---|
| `push.sound.name: <custom.wav>` | `notification_sound: <custom.wav>` |
| `push.sound.critical: 1` | `interruption_level: critical` |
| `push.sound.volume: 0.75` | `critical_alert_volume: 0.75` (0.0–1.0; **not rescaled**) |
| `push.badge: 5` | `badge: 5` |
| `push.interruption-level: passive` | `interruption_level: passive` |
| `push.category: camera` / `map` | drop — auto-derived from `camera:` / `map_latitude:` |
| top-level `priority: high` (Android) | drop — auto-derived from `interruption_level` |
| `media_stream` | drop — auto-derived from critical/volume |

### Magic-string messages

Some legacy calls use the `message:` field as a control signal rather than user-visible text. These now have a dedicated `notification_type:` field:

| Old `message:` value | New `notification_type:` |
|---|---|
| `clear_notification` | `clear_notification` |
| `delete_alert` | `delete_alert` |
| `remove_channel` | `remove_channel` |
| `request_location_update` | _(leave as `message:` for now — script passes it through; not yet a first-class option)_ |

For all other calls, set `notification_type: notify` (or omit it — that's the default).

### Action buttons (`actions:` list)

The legacy nested list:

```yaml
data:
  actions:
    - action: UNLOCK_DOOR
      title: Unlock
      activationMode: foreground
      authenticationRequired: true
    - action: IGNORE
      title: Ignore
      destructive: true
```

becomes flat numbered fields (up to 10 actions):

```yaml
data:
  action_1_id: UNLOCK_DOOR
  action_1_title: Unlock
  action_1_launch_app: true
  action_1_authentication_required: true
  action_2_id: IGNORE
  action_2_title: Ignore
  action_2_use_red_text: true
```

Per-action mapping:

| Old (under `actions[i]`) | New |
|---|---|
| `action` | `action_N_id` (templated values like `'{{ action_close_garage_door_now }}'` work fine) |
| `title` | `action_N_title` |
| `icon` | `action_N_icon` |
| `activationMode: foreground` | `action_N_launch_app: true` |
| `authenticationRequired: true` | `action_N_authentication_required: true` |
| `destructive: true` | `action_N_use_red_text: true` |
| `uri: <deep-link>` | `action_N_launch_uri: <deep-link>` |
| `behavior: textInput` | `action_N_accept_text_input: true` |

> ⚠️ **10-action limit.** The script exposes `action_1_*` through `action_10_*`. If you have more than 10 buttons on a single notification, you'll need to either trim, split into multiple notifications, or extend the script.

### iOS Shortcut trigger

The legacy nested `shortcut:` block:

```yaml
data:
  shortcut:
    name: Sleep mode
    source: ha_notification
    intensity: 50
```

becomes flat numbered key/value pairs (up to 5):

```yaml
data:
  ios_shortcut_name: Sleep mode
  ios_shortcut_key_1_name: source
  ios_shortcut_key_1_value: ha_notification
  ios_shortcut_key_2_name: intensity
  ios_shortcut_key_2_value: 50
```

### Quick before/after example

**Before:**

```yaml
- alias: Notify family that the garage door is open
  data:
    title: Garage door
    message: Open for 10 minutes
    data:
      tag: garage_door
      group: garage
      url: /lovelace-home/home
      push:
        sound:
          name: default
          critical: 1
          volume: 1
        interruption-level: critical
      actions:
        - action: '{{ action_close_garage }}'
          title: Close now
          activationMode: foreground
        - action: IGNORE
          title: Ignore
          destructive: true
  action: notify.notify_family
```

**After:**

```yaml
- alias: Notify family that the garage door is open
  action: script.send_notification
  data:
    notification_type: notify
    send_to: family
    device_type: all
    title: Garage door
    message: Open for 10 minutes
    tag: garage_door
    group: garage
    url: /lovelace-home/home
    interruption_level: critical
    critical_alert_volume: 1.0
    action_1_id: '{{ action_close_garage }}'
    action_1_title: Close now
    action_1_launch_app: true
    action_2_id: IGNORE
    action_2_title: Ignore
    action_2_use_red_text: true
```

---

I hope this helps a few folks! Please let me know if you have any feedback.
