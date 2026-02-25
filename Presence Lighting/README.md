# Presence Lighting Blueprint

A Home Assistant automation blueprint that turns lights on and off based on room presence. Designed for setups where each room has an `input_select` entity tracking presence state (e.g. `input_select.presence_garage`).

## Table of Contents

- [Prerequisites](#prerequisites)
  - [Required Global Helpers](#required-global-helpers)
  - [Optional Global Helpers](#optional-global-helpers)
- [Installation](#installation)
- [Automation Mode](#automation-mode)
- [Inputs Reference](#inputs-reference)
  - [Room Setup](#room-setup)
  - [Turn-On](#turn-on)
  - [Turn-Off](#turn-off)
- [How It Works](#how-it-works)
  - [Triggers](#triggers)
  - [Global Conditions](#global-conditions)
  - [Variables](#variables)
  - [Turn-On Logic Flow](#turn-on-logic-flow)
  - [Turn-Off Logic Flow](#turn-off-logic-flow)

## Prerequisites

### Required Global Helpers

These must exist in your HA instance. Every automation created from this blueprint checks all three on every trigger:

| Helper | Purpose |
|---|---|
| `input_boolean.presence_enable_automations` | Must be **on** — master toggle for all presence automations |
| `input_boolean.enable_lighting_automations` | Must be **on** — master toggle for all lighting automations |
| `input_boolean.vacation_mode` | Must be **off** — disables presence lighting when on vacation |

### Optional Global Helpers

Only needed if you enable their corresponding flags in the blueprint UI:

| Helper | Used when |
|---|---|
| `input_boolean.suppress_lights_for_sleeping_child` | `Confirm lighting isn't suppressed for a sleeping child` is enabled |
| `timer.briefly_minimize_upstairs_lighting` | `Confirm upstairs lighting isn't minimized` is enabled |

## Installation

1. Place the YAML file in `config/blueprints/automation/MasterDevwi/presence_lighting.yaml` on your HA instance.
2. Go to **Settings → Automations & Scenes → + Create Automation → Use a Blueprint**.
3. Select **Presence Lighting**.
4. Fill in the inputs and save.

## Automation Mode

All automations created from this blueprint run in **`mode: restart`** with **`max_exceeded: silent`**:

- **Restart** means if the automation is already running (e.g. waiting through a turn-off delay) and a new trigger fires, the running instance is cancelled and a fresh run starts. This is critical for the turn-off delay — if someone re-enters the room during the delay, the delay is cancelled and the turn-on branch fires instead.
- **Silent** means if a trigger fires while the automation is restarting, no warning is logged.

---

## Inputs Reference

### Room Setup

#### Room presence entity *(required)*

The `input_select` that tracks presence for this room. Expected states include: `Occupied`, `Just Arrived`, `Extended Away`, `Empty`, `Unknown`, `Going to Sleep`, `Sleeping`, `Waking Up`.

The blueprint classifies these states into two groups:

| Group | States | Meaning |
|---|---|---|
| **Unoccupied** | `Unknown`, `Extended Away`, `Empty` | Nobody is in the room |
| **Sleeping** | `Going to Sleep`, `Sleeping`, `Waking Up` | Someone is sleeping — lights should not turn on |

All other states (e.g. `Occupied`, `Just Arrived`) are considered occupied but not sleeping.

#### Use room presence for lighting *(default: Occupancy and vacancy)*

Controls whether changes to the room presence entity trigger the turn-on branch, the turn-off branch, both, or neither. This is a 4-option select:

| Mode | Turn-on trigger | Turn-off trigger | Use case |
|---|---|---|---|
| **Occupancy and vacancy** | Yes | Yes | Most rooms — room presence drives everything |
| **Occupancy only** | Yes | No | Rooms where a separate sensor handles clearing (e.g. an area presence sensor that stays on longer than room presence) |
| **Vacancy only** | No | Yes | Rooms like the Hallway where a motion sensor detects presence but room presence handles the off transition |
| **Ignore room presence** | No | No | Rooms like Stairs or Front Room Lamps where only additional sensor entities drive the lights. The room presence entity is still used as a *condition* to check occupancy state, just not as a trigger. |

#### Primary light *(default: empty)*

A `light.*` entity. When set:
- **Turn-on** only fires if this light is currently **off**
- **Turn-off** only fires if this light is currently **on**

Leave empty to skip the state check. This is useful for rooms with multiple independent lights where the on/off logic is handled in custom actions or conditions.

---

### Turn-On

#### Additional turn-on entities *(default: sun.sun)*

A multi-select entity picker. These entities trigger the **turn-on** branch when they transition from `off` to `on`. Intended for motion sensors, door contacts, or area presence binary sensors.

**How the default works:** `sun.sun` uses states `above_horizon` / `below_horizon`, never `on` or `off`. Since the trigger requires a `from: "off"` → `to: "on"` transition, `sun.sun` never matches — it's a safe no-op placeholder that satisfies HA's requirement for at least one entity in the trigger.

**Shared entities:** If a single entity should trigger both turn-on AND turn-off (e.g. a motion sensor), add it to both this list and the "Additional turn-off entities" list.

#### Additional turn-on conditions *(default: empty)*

A condition editor input. All conditions listed here are evaluated with AND logic — every condition must pass for the turn-on branch to proceed.

These are evaluated **after** all built-in checks (see [Turn-On Logic Flow](#turn-on-logic-flow) below). Use this for room-specific logic like theater mode, dinner party mode, time-of-day restrictions, or checking nearby room lights.

#### Confirm room is unoccupied *(default: true)*

Controls what happens when an additional turn-on entity fires while the room is already occupied:

- **Enabled (default):** The turn-on entity only triggers turn-on if the room presence is in an unoccupied state (`Empty`, `Extended Away`, `Unknown`). This prevents sensors from re-running turn-on actions (and potentially adjusting brightness or re-triggering scripts) when someone is already in the room.
- **Disabled:** The turn-on entity triggers turn-on regardless of room presence state. Useful for entities like a door contact that should always trigger lights when opened, even if someone is already inside.

> **Note:** This setting only applies to additional turn-on entities, not to the room presence entity itself. Room presence always requires a transition from unoccupied → occupied.

#### Confirm upstairs lighting isn't minimized *(default: false)*

When enabled, the turn-on branch is blocked if `timer.briefly_minimize_upstairs_lighting` is currently `active`. Use this for upstairs rooms where lights should stay dim or off while the minimize timer is running.

This is a **blocking condition** — it completely prevents the turn-on actions from executing.

#### Confirm lighting isn't suppressed for a sleeping child *(default: false)*

When enabled, the turn-on branch is blocked if `input_boolean.suppress_lights_for_sleeping_child` is `on`. Use this for rooms a sleeping child may pass through when returning home.

#### Turn-on actions *(required)*

The actions to execute when the light should turn on. Can be as simple as calling a script (e.g. `script.lighting_garage_on`) or as complex as a multi-step sequence with choose/if blocks for brightness levels, theater mode, etc.

---

### Turn-Off

#### Additional turn-off entities *(default: sun.sun)*

A multi-select entity picker. These entities trigger the **turn-off** branch when they transition from `on` to `off`. Intended for area presence sensors, occupancy sensors, or override toggles.

The `sun.sun` default is a no-op placeholder (same mechanism as additional turn-on entities — `sun.sun` never transitions `on` → `off`).

#### Additional turn-off conditions *(default: empty)*

A condition editor input. All conditions listed here are evaluated with AND logic. These are checked **after** the built-in checks (see [Turn-Off Logic Flow](#turn-off-logic-flow) below).

#### Delay before off *(default: 0 minutes)*

Minutes to wait after the turn-off branch is entered before executing the turn-off actions. Accepts values in 0.1-minute increments (e.g. 0.5 = 30 seconds, 1.5 = 90 seconds). Range: 0–60 minutes.

Because the automation uses `mode: restart`, if someone re-enters the room during this delay, the entire automation restarts and the turn-on branch fires — the pending turn-off actions are cancelled.

#### Alternate delay condition *(default: empty)*

A condition editor input. When **all** conditions in this list are met (AND logic), the alternate delay is used instead of the normal delay.

An empty list (the default) evaluates to true, but since the alternate delay defaults to 0 and must be explicitly configured, this has no practical effect unless both an alternate delay value and conditions are set.

Examples: guest mode is on, sleep mode is on, time is after 11 PM.

#### Alternate delay *(default: 0 minutes)*

The delay value used when the alternate delay condition is met. Same 0.1-minute step as the normal delay. Set to 0 for instant off.

#### Turn-off actions *(required)*

The actions to execute when the light should turn off (after the delay has elapsed). Can include script calls, fan turn-offs, inline `light.turn_off`, etc.

---

## How It Works

### Triggers

Every automation created from this blueprint has exactly three triggers:

| Trigger ID | Entity source | Fires when | Purpose |
|---|---|---|---|
| `room_presence_changes` | Room presence entity | Any state change | Detects occupancy transitions |
| `sensor_detected` | Additional turn-on entities | `off` → `on` | Motion sensors, door contacts activating |
| `sensor_cleared` | Additional turn-off entities | `on` → `off` | Area presence sensors, occupancy sensors clearing |

All three triggers are always present. The `sun.sun` defaults ensure the detect/clear triggers exist but never fire unless you replace them with real entities.

### Global Conditions

Before any logic runs, these three conditions must all pass:

1. `input_boolean.presence_enable_automations` is **on**
2. `input_boolean.enable_lighting_automations` is **on**
3. `input_boolean.vacation_mode` is **off**

If any fail, the automation exits immediately.

### Variables

After the global conditions pass, the automation computes several variables used throughout the logic:

| Variable | Value |
|---|---|
| `room_presence_state` | Current state of the room presence entity |
| `triggered_by_a_room` | `true` if the trigger entity starts with `input_select.presence_` |
| `is_room_state_trigger_empty_to_occupied` | See below |

**`is_room_state_trigger_empty_to_occupied`** is the key variable for the turn-on branch. Its value depends on what triggered the automation:

- **If triggered by a room presence entity:** Checks whether the state transitioned from an unoccupied state to an occupied state. Both the previous state and current state are examined from the trigger object. Returns `true` only if `from_state` was in [`Unknown`, `Extended Away`, `Empty`] and `to_state` is not in that list.
- **If triggered by a sensor (not a room presence entity):**
  - If `Confirm room is unoccupied` is enabled: Returns `true` only if the room presence entity is currently in an unoccupied state.
  - If `Confirm room is unoccupied` is disabled: Always returns `true`.

### Turn-On Logic Flow

The turn-on branch fires when **all** of these conditions pass (evaluated in order):

1. **Trigger routing:** The trigger must be either `sensor_detected`, or `room_presence_changes` with the trigger mode set to `Occupancy and vacancy` or `Occupancy only`.
2. **Empty-to-occupied transition:** `is_room_state_trigger_empty_to_occupied` must be `true` (see variable logic above).
3. **Not sleeping:** The room presence entity must NOT be in a sleeping state (`Going to Sleep`, `Sleeping`, `Waking Up`). This is always enforced and cannot be disabled — sleeping rooms never get lights turned on.
4. **Light is off:** If a primary light entity is configured, it must be in the `off` state. If no light entity is set, this check is skipped.
5. **Sleeping child:** If `Confirm lighting isn't suppressed for a sleeping child` is enabled, `input_boolean.suppress_lights_for_sleeping_child` must be `off`.
6. **Upstairs minimized:** If `Confirm upstairs lighting isn't minimized` is enabled, `timer.briefly_minimize_upstairs_lighting` must not be `active`.
7. **Additional conditions:** All conditions from the `Additional turn-on conditions` input must pass (AND logic).

If all conditions pass, the **turn-on actions** execute.

> **Why sleeping rooms never get lights:** The turn-on branch requires an unoccupied → occupied transition. Sleeping states (`Going to Sleep`, `Sleeping`, `Waking Up`) are not classified as unoccupied, so a transition from one sleeping state to another (or from sleeping to occupied) never satisfies the empty-to-occupied check. Additionally, step 3 explicitly blocks any trigger that fires while the room is in a sleeping state. Together, these two checks make it impossible for lights to turn on in a sleeping room.

### Turn-Off Logic Flow

The turn-off branch fires when **all** of these conditions pass:

1. **Trigger routing:** The trigger must be either `sensor_cleared`, or `room_presence_changes` with the trigger mode set to `Occupancy and vacancy` or `Vacancy only`.
2. **Room is unoccupied:** The room presence entity must be in an unoccupied state (`Unknown`, `Extended Away`, `Empty`).
3. **Light is on:** If a primary light entity is configured, it must be in the `on` state. If no light entity is set, this check is skipped.
4. **Additional conditions:** All conditions from the `Additional turn-off conditions` input must pass (AND logic).

If all conditions pass, the automation proceeds to the delay:

5. **Delay selection:** If the alternate delay condition is met (all conditions in the list are true — AND logic), the alternate delay is used. Otherwise, the normal delay is used.
6. **Wait:** The automation waits for the selected delay duration.
7. **Turn-off actions:** After the delay, the turn-off actions execute.

> **Restart cancellation:** Because the automation uses `mode: restart`, if any trigger fires during the delay (step 6), the entire automation restarts from the beginning. If the new trigger routes to the turn-on branch, the lights stay on. If it routes to the turn-off branch again, the delay restarts from zero.
