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

I hope this helps a few folks! Please let me know if you have any feedback.

