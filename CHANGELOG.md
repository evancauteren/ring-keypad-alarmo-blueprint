# Changelog

Home Assistant has no version field for blueprints, so the version lives in the
**blueprint name** — which is the only thing shown in
**Settings → Automations & scenes → Blueprints** — plus a comment at the top of
the YAML and a git tag here.

This project uses [semantic versioning](https://semver.org/):

- **major** — an input is removed or renamed, or behaviour changes in a way
  that could weaken security or break a working setup, so re-importing needs
  you to check your settings
- **minor** — new inputs or new behaviour, with defaults that leave existing
  automations working as before
- **patch** — fixes and documentation only

To update, push this repo, then **Settings → Automations & scenes → Blueprints
→ ⋮ → Re-import blueprint**. New inputs take their defaults automatically; your
existing choices are kept.

---

## 1.2.1

**Fixes a crash that made the restart re-sync from 1.1.0 never work.**

`entered_code` guarded its chain with `{% if 'event' in trigger %}`, which tests
only whether the key exists. On a `homeassistant` trigger the key is present but
holds the plain string `"start"` (confirmed in HA's own
`components/homeassistant/triggers/homeassistant.py`), so `"start".data` raised
`UndefinedError: 'str object' has no attribute 'data'`.

Rendering the `variables:` block is not recoverable: Home Assistant aborts the
run before any action executes. So every restart fired `resync_start`, the
render threw, and the mode LED was never re-asserted — the feature had not
worked since it shipped. `sun` and `calendar` triggers carry the same string
shape and would have hit it too.

The same unguarded chain also produced `Template variable warning: 'dict object'
has no attribute 'event_data'` on every `alarmo_failed_to_arm` event, whose data
has no `event_data` key. A warning rather than an error, but the same cause.

- `entered_code` now walks the chain defensively: `none` if there is no `event`
  key, `none` if `event` is a string, `none` if the data mapping lacks
  `event_data`, and the entered code otherwise.
- The force-arm branch's `code:` template got the same treatment. It was only
  reachable via a `zwave_js_notification` wait, so it was not failing in
  practice, but an unguarded chain there would abort a force-arm instead of
  falling back to the code originally entered.

Verified two ways: rendering both the old and new templates against realistic
payloads for every trigger platform the blueprint uses — the old one raises on
`homeassistant start`, `homeassistant shutdown` and `sun`, the new one on none of
them — and evaluating the new chain on a live HA 2026.9.2 instance.

The test harness that missed this built one hand-made trigger shape with a
`.data` mapping, so it only ever exercised the path that worked. It now uses a
real `Event` object per platform, including payloads with no `event_data` key.

Reported against v1.2.0 on HA Core 2026.9.2.

## 1.2.0

**Switching arm modes from the keypad.** Pressing **Away** while armed Home now
switches straight to Away and starts the exit delay, instead of doing nothing
until you disarm first.

Alarmo has always supported this — its arm handler is literally documented as
"arm the alarm or switch between arm modes", it accepts an arm command from any
`ARM_MODES` state, and it sets `_revert_state` to the previous mode so a failed
switch falls back to where you were rather than to disarmed. The upstream
blueprint simply gated its arm branch on `state: disarmed` and never exposed
any of it.

New **Switching arm modes from the keypad** input, three settings:

- `to_stricter` **(default)** — Home / Night / Custom-bypass → Away or
  Vacation. Cannot move to a looser mode from the keypad.
- `any` — any arm-mode switch, including Away → Home.
- `disarmed_only` — the previous behaviour, arm only from disarmed.

`arming`, `pending` and `triggered` are never valid starting points in any
setting; Alarmo refuses those itself and the blueprint no longer asks.

**This changes behaviour on re-import**, so it is called out here rather than
buried. It is a minor rather than a major release because the default can only
ever move the alarm to a *stricter* mode, it still requires whatever code
Alarmo requires for arming, and the previous behaviour is one dropdown away.
If you have turned off "code required for arming" while keeping it for
disarming, consider `to_stricter` or `disarmed_only` rather than `any`, since
`any` would otherwise let someone drop Away to Home without a code.

## 1.1.1

- The version now lives in the blueprint **name**
  (`Ring Keypad v2 + Alarmo (v1.1.1)`) rather than the description. The
  blueprint list in Home Assistant shows names only, so a version in the
  description was invisible exactly where you would look for it. This follows
  the convention other published blueprints use.

No behavioural change. Re-import to pick it up; your automation keeps working
either way, because Home Assistant identifies a blueprint by file path, not by
name.

## 1.1.0

**Mode re-sync.** The keypad remembers its own mode LED, and the blueprint only
wrote it when Alarmo *changed* state — so a keypad that forgot its mode (a
power blip, a spell off the mesh) while Alarmo sat in one state showed nothing
until the next arm or disarm. Symptom: you walk up in the morning, the keys
backlight white, but the Disarmed key is not blue.

- New **State re-sync** section with three inputs: re-sync after a Home
  Assistant restart (on by default, ~60 s after start so Z-Wave JS is ready),
  an optional every-N-hours re-sync (off by default), and **Re-sync silently**
  (on by default).
- Silent re-sync sets the indicator with `property_key: "9"`, `value: 0` —
  voice volume zero — so the LED updates without the keypad announcing itself.
  Verified working on a Ring Keypad v2; if a firmware rejects it the mode LED
  simply never updates after a restart, and turning the option off falls back
  to the universally accepted audible form.
- The re-sync only ever writes to the keypad. It never arms, disarms or touches
  Alarmo, and it skips `arming`, `pending` and `triggered` so it cannot wipe a
  running countdown or re-fire the siren.
- The N-hourly timer is off by default because any N-hourly cycle includes
  midnight, which on a keypad that ignores the silent form would mean an
  announcement in the middle of the night.

**Metadata.**

- Added `homeassistant: min_version: 2024.10.0`. Home Assistant now refuses the
  import on older versions with "Requires at least Home Assistant 2024.10.0"
  instead of failing obscurely on `triggers:` or `sequence:` later.
- Added `author`, and a version string in the file header.

**Docs.**

- `docs/testing.md` — new group H for the re-sync, including a three-step
  procedure for testing the silent form that can tell "worked silently" apart
  from "command rejected". Group G1 reworded, since the keypad now deliberately
  re-syncs about a minute after a restart.
- `docs/keypad-reference.md` — explained config parameter 22 (System Security
  Mode Display) and the three display timeouts, which is why the mode LED is
  dark until something wakes it. Corrected several parameter descriptions
  against Ring's own Z-Wave manual.

## 1.0.0

Initial fork of
[ImSorryButWho/HomeAssistantNotes](https://github.com/ImSorryButWho/HomeAssistantNotes)
`keypad_blueprint.yaml` at upstream main `375e1d0` (2025-08-09).

Merged from upstream pull requests:

- **#43** — `alarmo_failed_to_arm` triggers now filter on `entity_id`, so a
  second Alarmo area cannot beep this keypad.
- **#49** (architecture only) — after asking Alarmo to arm, wait to learn
  *which* outcome happened rather than blindly waiting for a keypress.
- **#36** (ideas only) — `armed_night` and `armed_vacation` handling. Its
  `value: 100` indicator is out of range (max is 99) and was not taken.
- **#48** — the `property_key: timeout` documentation fix, folded into
  `docs/keypad-reference.md`.

Fixed from upstream issues:

- **#23** — sensor bypass never completed. The confirmation window was
  hard-coded to 5 seconds from the moment the arm command was sent, and only
  matched a *bare* Enter, so retyping your code could never work. Now the arm
  branch waits for Alarmo's `open_sensors` rejection, then opens a configurable
  window (default 30 s) that accepts Enter or Arm, with or without a code.
- **#45**, **#21** — no way to silence a manually raised panic alert. Disarm or
  Cancel while already disarmed now clears it and runs an optional action.
  Code-gated by default.
- **#44** — the entry/exit delay indicator format is now selectable, modern
  (`timeout`) or legacy (property key 7), for keypads that expose one and not
  the other. A zero-length delay no longer sends a pointless `0m0s` countdown.
- **#35** — optional, off by default: re-assert announcement volume (config
  parameter 4) after the alarm goes triggered → disarmed.
- **#32** — the device picker no longer filters on `manufacturer: Ring`, which
  hid keypads reported without a usable manufacturer string.

Other changes:

- Handles the **Cancel** key (`event_type: 25`), which upstream ignored.
- Pressing Enter while nothing is armed no longer fires a disarm, which
  upstream answered with a spurious error tone.
- `not_allowed` rejections now get the error tone.
- `armed_vacation` → Away LED; `armed_night` and `armed_custom_bypass` → Home
  LED.
- `not_from: [unknown, unavailable]` on every state trigger, so the keypad
  stays quiet through a Home Assistant restart.
- Modernised to `triggers:` / `actions:` / `action:` syntax; inputs grouped
  into collapsible sections.
- Traces remain disabled, now with a comment saying why: a trace would store
  the PIN in clear text under `.storage`.
