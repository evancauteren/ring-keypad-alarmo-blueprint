# Contributing

Thanks for helping out. A few things to know before you open an issue or a
pull request.

## Scope

This blueprint does one thing: one **Ring Keypad v2**, kept in sync with one
**Alarmo** panel. Changes that stay within that scope are welcome. Support for
other alarm integrations, the v1 keypad, or multi-keypad orchestration is out
of scope, because each of these would make the blueprint harder to use for the
common case.

## Reporting a bug

Run the relevant group in [`docs/testing.md`](docs/testing.md) first. It tells
you where to look for each kind of failure. Then include:

- the test row number and what happened instead
- the raw `zwave_js_notification` event from **Developer tools → Events**
- the panel's `state` and `delay` attribute at that moment
- any matching lines from **Settings → System → Logs** (full logs)
- your Home Assistant, Alarmo and Z-Wave JS driver versions

**Never post your PIN.** If you share a trace or an event, check it for the
code you typed and redact it.

## Making a change

The whole blueprint is one file:
[`blueprints/automation/ring_keypad_v2/ring_keypad_v2_alarmo.yaml`](blueprints/automation/ring_keypad_v2/ring_keypad_v2_alarmo.yaml).

- Keep `trace: stored_traces: 0`. A trace stores the entered PIN in plain
  text.
- Every template that reads `trigger.event` must survive triggers where
  `event` is missing or a plain string (`homeassistant`, `sun`, `calendar`). An
  error while rendering `variables:` aborts the whole run before any action
  executes, and the log is the only place it shows.
- New inputs need a default that keeps existing automations behaving as before.
- If you change the Z-Wave side, update
  [`docs/keypad-reference.md`](docs/keypad-reference.md). If you add or change a
  branch, add a row to [`docs/testing.md`](docs/testing.md).

## Validation

Releases are checked against Home Assistant's own schemas, not by eye (last run
on Home Assistant 2026.2.3):

- `blueprint.schemas.BLUEPRINT_SCHEMA`: blueprint metadata, sections, selectors
- input substitution via `BlueprintInputs.async_substitute()`
- `cv.TRIGGER_SCHEMA`, `cv.SCRIPT_SCHEMA`, `cv.SCRIPT_VARIABLES_SCHEMA`
- every Jinja template compiled through Home Assistant's template engine
- cross-checks: no orphan trigger ids, no branch referring to a nonexistent
  trigger, no declared-but-unused inputs, no undeclared `!input`
- the computed variables (`entered_code`, `panic_cancel_requested`,
  `delay_indicator_value`) rendered against a real `Event` object for every
  trigger platform the blueprint uses, including payloads without
  `event_data`, plus numeric PINs, PINs with a leading zero, bare keypresses,
  both delay formats and a missing `delay` attribute

No static check can press the buttons, so a change to the automation logic
also needs the relevant groups of [`docs/testing.md`](docs/testing.md) run on a
real keypad.

## Versioning and releases

Home Assistant has no version field for blueprints (`blueprint.version` is
rejected by the schema), and the blueprint list shows names only. So the
version goes in the **name**, `Ring Keypad v2 + Alarmo (vX.Y.Z)`, and in the
header comment of the YAML.

[Semantic versioning](https://semver.org/):

- **major**: an input is removed or renamed, or behaviour changes in a way that
  could weaken security or break a working setup, so re-importing needs the
  user to check their settings
- **minor**: new inputs or new behaviour, with defaults that leave existing
  automations working as before
- **patch**: fixes and documentation only

To release:

1. Bump the version in `blueprint.name` and in the header comment.
2. Commit and push to `main`. Re-import fetches from `source_url`, which points
   at `main`.
3. Tag `vX.Y.Z` and create a GitHub Release with the notes. Say clearly if
   users need to check their settings after re-importing.

Renaming the blueprint on each release is safe: Home Assistant identifies a
blueprint by its file path, so existing automations keep working and just show
the new name.
