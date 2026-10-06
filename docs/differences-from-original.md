# Differences from the original blueprint

For anyone moving over from
[ImSorryButWho's Ring Keypad v2 blueprint](https://github.com/ImSorryButWho/HomeAssistantNotes)
(`keypad_blueprint.yaml`, as of `375e1d0`, 2025-08-09). Everything here is
already covered from a user's point of view in the [README](../README.md); this
page only records what changed and why.

## Switching over

The two blueprints live at different paths, so they can be installed side by
side. Import this one, create an automation from it with the same keypad and
Alarmo entity, then disable or delete the old automation. Running both at once
makes the keypad answer every event twice.

## Pull requests on the original repo

| PR | Title | Taken? | Notes |
|---|---|---|---|
| [#43](https://github.com/ImSorryButWho/HomeAssistantNotes/pull/43) | Scope `alarmo_failed_to_arm` to the selected panel | **As-is** | Without it, any Alarmo area failing to arm beeps your keypad. Needs Alarmo v1.10.9 or newer, which added `entity_id` to the event. |
| [#36](https://github.com/ImSorryButWho/HomeAssistantNotes/pull/36) | Volume per alarm mode + night mode | **Ideas only** | `armed_night` / `armed_vacation` handling is in. Its indicator `value: 100` is out of range (max 99) and was not taken. |
| [#49](https://github.com/ImSorryButWho/HomeAssistantNotes/pull/49) | Multi-keypad sync blueprint | **Architecture only** | Adopted its approach of waiting for Alarmo's actual outcome after an arm command. The multi-keypad machinery itself is not needed for one keypad. For several keypads, that PR is the better starting point. |
| [#48](https://github.com/ImSorryButWho/HomeAssistantNotes/pull/48) | Fix the `property_key` docs | **Docs** | Folded into [`keypad-reference.md`](keypad-reference.md). |
| [#19](https://github.com/ImSorryButWho/HomeAssistantNotes/pull/19) | Align features with native Ring behaviour | **N/A** | Targets the v1 keypad. |

## Issues on the original repo

| Issue | Status here |
|---|---|
| [#23](https://github.com/ImSorryButWho/HomeAssistantNotes/issues/23) Bypass never completes | **Fixed.** The confirmation window was a fixed 5 s from sending the arm command and only matched a bare Enter. It now opens after Alarmo reports open sensors, lasts as long as you configure (default 30 s), and accepts Enter or Arm, with or without a code. |
| [#45](https://github.com/ImSorryButWho/HomeAssistantNotes/issues/45) No way to cancel a panic alert | **Fixed.** Disarm or Cancel while disarmed clears the alert and runs an optional action. Code-gated by default. |
| [#21](https://github.com/ImSorryButWho/HomeAssistantNotes/issues/21) Fire alarm peculiarities | **Partly fixed.** Clearing works as for #45. The 10–20 s lag afterwards is keypad firmware. |
| [#44](https://github.com/ImSorryButWho/HomeAssistantNotes/issues/44) Arming countdown stopped after an upgrade | **Mitigated.** The delay indicator format is selectable: modern `timeout` (default) or legacy property key 7. |
| [#46](https://github.com/ImSorryButWho/HomeAssistantNotes/issues/46) `property_key` docs outdated | **Fixed** in the docs. |
| [#35](https://github.com/ImSorryButWho/HomeAssistantNotes/issues/35) Volume low after a trigger | **Optional workaround**, off by default: re-assert announcement volume after triggered → disarmed. |
| [#32](https://github.com/ImSorryButWho/HomeAssistantNotes/issues/32) Keypad missing from the device picker | **Mitigated.** The picker filters on `zwave_js` only, not on manufacturer. |
| [#27](https://github.com/ImSorryButWho/HomeAssistantNotes/issues/27) Map a button to night mode | The keypad has no night key. Night armed from HA is shown on the Home LED. |
| [#37](https://github.com/ImSorryButWho/HomeAssistantNotes/issues/37), [#38](https://github.com/ImSorryButWho/HomeAssistantNotes/issues/38), [#42](https://github.com/ImSorryButWho/HomeAssistantNotes/issues/42), [#47](https://github.com/ImSorryButWho/HomeAssistantNotes/issues/47) | Not blueprint bugs (YAML pasting, v1 docs, Z-Wave interview, RF). |

## Other changes

- Modern `triggers:` / `actions:` / `action:` syntax; inputs in collapsible
  sections.
- The **Cancel** key (`event_type: 25`) is handled.
- Enter while disarmed no longer sends a disarm, which used to earn an error
  tone.
- `not_allowed` rejections get the error tone too.
- `armed_vacation` → Away LED; `armed_night` and `armed_custom_bypass` → Home
  LED.
- `not_from: [unknown, unavailable]` on every state trigger, so the keypad stays
  quiet through a restart.
- Arm modes can be switched from the keypad (Home → Away by default).
- The mode LED is re-asserted after a restart, and optionally every N hours.
- A zero-length delay no longer sends a `0m0s` countdown.
