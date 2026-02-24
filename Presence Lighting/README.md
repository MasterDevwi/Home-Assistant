# Presence Lighting Blueprint

A Home Assistant automation blueprint that turns lights on and off based on room presence. Designed for setups where each room has an `input_select` entity tracking presence state (e.g. `input_select.presence_garage`).

## Prerequisites

These global helpers must exist in your HA instance:

| Helper | Purpose |
|---|---|
| `input_boolean.presence_enable_automations` | Master toggle for all presence automations |
| `input_boolean.enable_lighting_automations` | Master toggle for all lighting automations |
| `input_boolean.vacation_mode` | Disables presence lighting when on |

The following are only needed if you enable their corresponding flags:

| Helper | Used by flag |
|---|---|
| `input_boolean.suppress_lights_for_sleeping_child` | `check_suppress_sleeping_child` |
| `timer.briefly_minimize_upstairs_lighting` | `check_upstairs_timer` |

## Installation

1. Import the blueprint into Home Assistant (from GitHub URL or by placing the YAML in `config/blueprints/automation/`).
2. Go to **Settings → Automations & Scenes → + Create Automation → Use a Blueprint**.
3. Select **Presence Lighting**.
4. Fill in the inputs (see below) and save.

## Inputs Reference

### Required

| Input | Description |
|---|---|
| **Room presence entity** | The `input_select` that tracks presence for this room (e.g. `input_select.presence_garage`). Expected states: `Occupied`, `Just Arrived`, `Extended Away`, `Empty`, `Unknown`, `Going to Sleep`, `Sleeping`, `Waking Up`, etc. |
| **Delay before off** | Minutes to wait after the room becomes unoccupied before turning off. Default: `5`. |
| **Turn-on actions** | What to do when someone enters the room. Can be a script call, `light.turn_on`, or any action sequence. |
| **Turn-off actions** | What to do when the room has been empty for the delay period. |

### Optional

| Input | Default | Description |
|---|---|---|
| **Primary light entity to check** | *(empty)* | A `light.*` entity. When set, turn-on only fires if this light is off, and turn-off only fires if it's on. Leave empty to skip the check (useful for rooms with multiple independent lights). |
| **Alternate delay** | `0` | A second delay value (minutes) used when the alternate delay condition is true. |
| **Alternate delay condition** | `{{ false }}` | Jinja2 template. When it evaluates to true, the alternate delay is used instead of the normal delay. |
| **Check suppress sleeping child** | `false` | When on, blocks turn-on if `input_boolean.suppress_lights_for_sleeping_child` is on. |
| **Check upstairs lighting timer** | `false` | When on, sets the variable `is_upstairs_timer_active` (true/false) which you can reference in your turn-on actions to lower brightness. |
| **Block turn-on when room is sleeping** | `true` | Prevents lights from turning on when the room is in a sleeping state (`Going to Sleep`, `Sleeping`, `Waking Up`). Turn this **off** for rooms like Kitchen that should still light up at reduced brightness during sleep. |
| **Additional turn-on condition** | `true` | Jinja2 template that must be true for the light to turn on. Use for theater mode, dinner party mode, etc. |
| **Additional turn-off condition** | `true` | Jinja2 template that must be true for the light to turn off. |

## How It Works

The automation has a single built-in trigger: the room presence entity changing state. When it fires:

1. **Global conditions** are checked — presence automations enabled, lighting automations enabled, vacation mode off.
2. **Variables** are computed — room state, whether the transition was from empty to occupied, whether the upstairs timer is active, etc.
3. A **choose** block picks one of two branches:

### Turn-on branch
Fires when **all** of these are true:
- Trigger ID is `room_presence_changes` or `sensor_detected`
- Room transitioned from an unoccupied state (`Unknown`, `Extended Away`, `Empty`) to an occupied state
- Room is not in a sleeping state (if `check_room_not_sleeping` is on)
- Primary light is off (if a light entity was provided)
- Sleeping child suppression is not active (if `check_suppress_sleeping_child` is on)
- Additional turn-on condition is true

Then executes your **turn-on actions**.

### Turn-off branch
Fires when **all** of these are true:
- Trigger ID is `room_presence_changes` or `sensor_cleared`
- Room is in an unoccupied state
- Primary light is on (if a light entity was provided)
- Additional turn-off condition is true

Then **waits** for the configured delay (normal or alternate) and executes your **turn-off actions**. Because the automation runs in `mode: restart`, if someone re-enters the room during the delay, the automation restarts and the turn-on branch fires instead — the turn-off actions never execute.

## Adding Extra Triggers

After creating an automation from this blueprint, open it in the automation editor and add triggers manually. Use these trigger IDs so the choose logic routes them correctly:

| Trigger ID | Purpose |
|---|---|
| `sensor_detected` | Additional turn-on trigger (e.g. a motion sensor or door contact) |
| `sensor_cleared` | Additional turn-off trigger (e.g. a motion sensor clearing) |

The blueprint's choose block already handles both IDs alongside the built-in `room_presence_changes` trigger.

## Examples

### Simple room (Garage)

```yaml
room_presence_entity: input_select.presence_garage
light_entity_to_check: light.garage_light
delay_before_off: 5
turn_on_actions:
  - action: script.lighting_garage_on
turn_off_actions:
  - action: script.lighting_garage_off
```

All optional inputs left at defaults. Lights turn on when presence goes from empty → occupied, turn off 5 minutes after the room empties.

### Room with guest delay (Living Room)

```yaml
room_presence_entity: input_select.presence_living_room
light_entity_to_check: light.living_room_light
delay_before_off: 10
alternate_delay: 30
alternate_delay_condition: "{{ is_state('input_boolean.guest_mode', 'on') }}"
turn_on_actions:
  - action: script.lighting_living_room_on
turn_off_actions:
  - action: script.lighting_living_room_off
```

Lights stay on 30 minutes instead of 10 when guest mode is active.

### Upstairs room with sleeping child check

```yaml
room_presence_entity: input_select.presence_nursery
light_entity_to_check: light.nursery_light
delay_before_off: 3
check_suppress_sleeping_child: true
check_upstairs_timer: true
turn_on_actions:
  - choose:
    - conditions:
      - condition: template
        value_template: "{{ is_upstairs_timer_active }}"
      sequence:
        - action: light.turn_on
          target:
            entity_id: light.nursery_light
          data:
            brightness_pct: 20
    default:
      - action: script.lighting_nursery_on
turn_off_actions:
  - action: script.lighting_nursery_off
```

Won't turn on if a child is sleeping. When the upstairs timer is active, turns on at 20% brightness instead of full.

### Kitchen (sleep mode = low brightness, not blocked)

```yaml
room_presence_entity: input_select.presence_kitchen
light_entity_to_check: light.kitchen_light
delay_before_off: 5
check_room_not_sleeping: false
turn_on_actions:
  - choose:
    - conditions:
      - condition: state
        entity_id: input_boolean.sleep_mode
        state: "on"
      sequence:
        - action: light.turn_on
          target:
            entity_id: light.kitchen_light
          data:
            brightness_pct: 10
    default:
      - action: script.lighting_kitchen_on
turn_off_actions:
  - action: script.lighting_kitchen_off
additional_on_condition: "{{ is_state('input_boolean.theater_mode', 'off') }}"
```

With `check_room_not_sleeping: false`, lights still turn on during sleep states but at 10% brightness via the choose block in turn-on actions. Theater mode blocks turn-on entirely.
