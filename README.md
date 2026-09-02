# Ring Keypad v2 + Alarmo — personal blueprint

A maintained fork of the Ring Keypad v2 blueprint from
[ImSorryButWho/HomeAssistantNotes](https://github.com/ImSorryButWho/HomeAssistantNotes),
which has not had a commit since **2025-08-09** (`375e1d0`) and has 5 open pull
requests and 12 open issues.

Blueprint: [`blueprints/automation/ring_keypad_v2/ring_keypad_v2_alarmo.yaml`](blueprints/automation/ring_keypad_v2/ring_keypad_v2_alarmo.yaml)

Reference tables for the keypad's Z-Wave indicators and config parameters:
[`docs/keypad-reference.md`](docs/keypad-reference.md)

Scope of this fork: **one keypad**, **Alarmo**, Away + Home as the buttons you
actually press. Night / Vacation / Custom-bypass are handled too, for when you
arm them from Home Assistant instead of the keypad.

Requires **Home Assistant 2024.10 or newer** (modern `triggers:` / `actions:` /
`action:` syntax, the `sequence:` grouping action and blueprint input sections).

---

## Install

1. Copy `blueprints/automation/ring_keypad_v2/ring_keypad_v2_alarmo.yaml` into
   your HA config at the same relative path:
   `config/blueprints/automation/ring_keypad_v2/ring_keypad_v2_alarmo.yaml`
2. **Developer tools → YAML → Reload blueprints** (or restart HA).
3. **Settings → Automations & scenes → Blueprints** → *Ring Keypad v2 + Alarmo*
   → **Create automation**.
4. Pick your keypad device and your Alarmo entity. Everything else has a
   working default.

Or use the one-click import link under [Pushing to GitHub](#pushing-to-github)
once the repo is pushed.

### What is deliberately *not* in the blueprint

Keypad **volume**, **LED brightness** and **voice language** are Z-Wave device
config parameters, not automation logic. Set them once in
**Settings → Devices & services → Z-Wave JS → your keypad → Configure device**:

| Parameter | What it does | Range |
|---|---|---|
| 4 | Announcement audio volume | 0–10 (default 7) |
| 5 | Key tone volume | 0–10 (default 6) |
| 6 | Siren volume | 0–10 (default 10) |
| 12 | Brightness: security mode LEDs | 0–100 % |
| 13 | Brightness: key backlight | 0–100 % |
| 18 | Keypad language (English / French / Spanish) | — |

(You remembered right: the language *is* settable from Z-Wave JS, and so are
the volumes and brightness.) The full list is in
[`docs/keypad-reference.md`](docs/keypad-reference.md).

---

## Verdict on every open upstream pull request

| PR | Title | Taken? | Why |
|---|---|---|---|
| [#43](https://github.com/ImSorryButWho/HomeAssistantNotes/pull/43) | Scope `alarmo_failed_to_arm` to the selected panel | **Taken as-is** | Correct and important. Upstream reacts to *any* Alarmo area's failure, so with more than one Alarmo area your keypad beeps for a panel it has nothing to do with. Verified against Alarmo's source: `event.py` puts `entity_id` in the event payload, added 2025-05-25 and shipped in **Alarmo v1.10.9**. If you are on something older than that, this filter would silently stop the error tones — check your Alarmo version. |
| [#36](https://github.com/ImSorryButWho/HomeAssistantNotes/pull/36) | Volume per alarm mode + night mode | **Ideas taken, code not** | Its `armed_night` / `armed_vacation` handling is right and is in this fork. Its volume mechanism (writing Z-Wave config parameter 4 around the mode change) is also sound — parameter 4 really is "Announcement Audio Volume", 0–10. But the PR has two defects: it sets the Armed-Home indicator to `value: 100`, and the valid range is 0–99 (upstream fixed exactly this in commit `7973a1e`, "Max value 100 -> 99"), and it reuses the same trigger `id` for two different triggers, which works but makes the intent unreadable. You also said you do not use night mode, so the quiet-night volume dance would be dead code. |
| [#49](https://github.com/ImSorryButWho/HomeAssistantNotes/pull/49) | Multi-keypad sync blueprint | **Architecture borrowed, not the file** | This is the best-engineered PR in the repo. Its key insight — after asking Alarmo to arm, *wait to find out which* of success / invalid code / open sensors happened, instead of blindly waiting for a keypress — is adopted here. Its actual file is a separate multi-keypad blueprint (device filtering moved from the trigger into a template condition, `repeat` loops over every keypad); you have one keypad, so that machinery is pure overhead and it also means the automation wakes up for **every** Z-Wave entry-control event in your house. If you ever add a second keypad, this is the PR to revisit. |
| [#48](https://github.com/ImSorryButWho/HomeAssistantNotes/pull/48) | Fix the `property_key` docs | **Taken (docs)** | Correct: time-taking indicators use `property_key: timeout` with a `1m30s`-style duration, not `property_key: 7` in seconds. Folded into `docs/keypad-reference.md`. Fixes open issue #46. |
| [#19](https://github.com/ImSorryButWho/HomeAssistantNotes/pull/19) | Align features with native Ring behaviour | **Not applicable** | Touches `keypad-blueprint-v1.yaml` only — the **v1** keypad, a different device. Irrelevant to you. |

## Verdict on every open upstream issue

| Issue | What it reports | Handled here? |
|---|---|---|
| [#23](https://github.com/ImSorryButWho/HomeAssistantNotes/issues/23) Bypass never completes | Arming with an open sensor prompts for bypass; entering the code again does nothing | **Fixed.** Two real bugs upstream: the confirmation window is hard-coded to **5 seconds** starting the instant the arm command is sent — too short for a human to hear "Sensors require bypass" and react — and the wait only matches `event_data: null`, i.e. a *bare* Enter, so entering your code again never matches. Here the arm branch waits for Alarmo's `open_sensors` rejection, *then* opens a window you control (default 30 s) that accepts Enter **or** Arm, with or without a code, and force-arms with whichever code was given. Disarm or Cancel walks away instead. |
| [#45](https://github.com/ImSorryButWho/HomeAssistantNotes/issues/45) No way to cancel a panic alert | A Fire/Police/Medical alert "doesn't seem to shut up" | **Fixed.** Pressing Disarm or Cancel while the alarm is already disarmed re-asserts the Disarmed indicator (which is what actually silences the keypad) and runs a new optional **Panic alert cleared** action so you can stop sirens and notifications too. Code-gated by default, as the reporter argued — an intruder should not be able to silence it. |
| [#21](https://github.com/ImSorryButWho/HomeAssistantNotes/issues/21) Fire alarm peculiarities | Fire alert cannot be turned off; keypad then lags 10–20 s | **Partly fixed.** Same root cause as #45 and handled by the same path. The 10–20 s lag afterwards is keypad firmware/Z-Wave behaviour, nothing a blueprint can reach. |
| [#44](https://github.com/ImSorryButWho/HomeAssistantNotes/issues/44) Arming countdown stopped after an upgrade | Reporter blames commit `d9cebab` | **Mitigated, with a switch.** `d9cebab` moved the delay indicator from `property_key: 7` (seconds) to `property_key: timeout` (`0m30s`). That is right for current Z-Wave JS but breaks anyone whose keypad still exposes the numeric key — see issue #42, where a keypad exposes `135-0-18-0/1/9` but not `-18-7`. So the format is a blueprint option: **Entry/exit delay indicator format**, modern by default, legacy one click away. Also added: a zero-length delay no longer sends a pointless `0m0s` countdown. |
| [#46](https://github.com/ImSorryButWho/HomeAssistantNotes/issues/46) `property_key` docs outdated | — | **Fixed in docs** (same fix as PR #48). |
| [#35](https://github.com/ImSorryButWho/HomeAssistantNotes/issues/35) Volume low after a trigger | Keypad volume stays low after a siren; only re-pairing fixes it | **Optional workaround, off by default.** Nobody has ever diagnosed this and it may be firmware. Turn on **Re-assert announcement volume after an alarm** and the blueprint writes config parameter 4 back to your chosen value once the alarm goes triggered → disarmed. Off unless you actually see the problem. |
| [#32](https://github.com/ImSorryButWho/HomeAssistantNotes/issues/32) Keypad missing from the device picker | Device selector comes up empty | **Mitigated.** Upstream filters the picker on `manufacturer: Ring`, which fails when Z-Wave JS reports the device without a usable manufacturer string. This fork filters on `integration: zwave_js` only, so your keypad always appears (at the cost of listing your other Z-Wave devices too). |
| [#37](https://github.com/ImSorryButWho/HomeAssistantNotes/issues/37) `extra keys not allowed @ data['automation']` | — | **Not a blueprint bug.** The reporter pasted a `configuration.yaml`-style `automation:` block into the UI editor, which expects the mapping without that key. Using the blueprint avoids the whole problem. |
| [#38](https://github.com/ImSorryButWho/HomeAssistantNotes/issues/38) Broken v1 import link | — | **Not applicable** (v1 keypad docs typo). |
| [#27](https://github.com/ImSorryButWho/HomeAssistantNotes/issues/27) How to map a button to night mode | — | **Answered by design.** The keypad has three mode keys (Disarm / Away / Home) and no night key; the hardware cannot send a fourth mode. This fork instead *reflects* `armed_night` on the Home LED, so arming night from HA still shows correctly on the keypad. |
| [#42](https://github.com/ImSorryButWho/HomeAssistantNotes/issues/42) Keypad not exposing all topics | One of two keypads lacks `135-0-18-7` | **Not a blueprint bug** — a Z-Wave interview problem. It is, however, the evidence behind the delay-format switch above. |
| [#47](https://github.com/ImSorryButWho/HomeAssistantNotes/issues/47) Connectivity unstable | — | **Not a blueprint bug** — Z-Wave RF / mesh. |

None of these 12 issues has a single reply from the maintainer, which is the
clearest signal that forking was the right call.

---

## Everything this fork changes

Beyond the PR and issue fixes above:

- **Modernised syntax** — `triggers:` / `actions:`, `- trigger: event`,
  `action:` instead of the deprecated `service:`. This is what upstream PR #36
  started; here it is applied consistently.
- **Cancel key (`event_type: 25`) is handled.** Upstream ignores it entirely.
  It now behaves like Ring's own keypad: cancel a running countdown, or clear a
  panic alert.
- **`not_from: [unknown, unavailable]` on every state trigger** instead of a
  global "not from unknown" condition, so the keypad stays quiet on HA restart
  and the intent is local to each trigger. Also covers `unavailable`, which
  upstream missed.
- **Pressing Enter when nothing is armed no longer fires a disarm.** Upstream
  sent `alarmo.disarm` on every Enter press; with no code that makes Alarmo
  emit `invalid_code`, so the keypad answers a harmless keypress with an error
  tone. The disarm branch is now gated on the alarm not already being disarmed.
- **`not_allowed` rejections get the error tone too.** Alarmo's third rejection
  reason (arming a mode that is not enabled) was silent upstream.
- **`armed_vacation` → Away LED, `armed_night` and `armed_custom_bypass` → Home
  LED.** The keypad has no LED for these, so they ride on the closest one.
- **Inputs are grouped into collapsible sections** (Devices / Behaviour /
  Volume recovery / Panic actions), so the two things you must set are the only
  two things you see.
- **Traces stay disabled** (`stored_traces: 0`), and now with a comment saying
  why: a trace of this automation would write your PIN in clear text into
  `.storage`. Raise it temporarily if you need to debug, then put it back.

### Known trade-offs

- The **Panic alert cleared** action also runs if you press Disarm + code while
  already disarmed and no alert is active. Keep that action idempotent
  (e.g. "turn the siren off"), which is the normal shape for it anyway.
- Arming from the keypad still needs a Disarm/Cancel press to abandon a bypass
  prompt, otherwise the confirmation window just times out silently.

---

## Pushing to GitHub

The repo lives at
<https://github.com/evancauteren/ring-keypad-alarmo-blueprint> and `source_url`
in the blueprint already points at it, so the one-click import below works as
soon as this is pushed.

```bash
cd /Users/evancauteren/Documents/Repos/Personales/ring-keypad-alarmo-blueprint
git init -b main
git add .
git commit -m "Ring Keypad v2 + Alarmo blueprint, forked from ImSorryButWho/HomeAssistantNotes"
git remote add origin https://github.com/evancauteren/ring-keypad-alarmo-blueprint.git
git push -u origin main
```

### One-click import into Home Assistant

```
https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2Fevancauteren%2Fring-keypad-alarmo-blueprint%2Fblob%2Fmain%2Fblueprints%2Fautomation%2Fring_keypad_v2%2Fring_keypad_v2_alarmo.yaml
```

If you later move or rename the file, update `source_url` in the blueprint to
match — Home Assistant compares the two and refuses the import if they differ.

### Tracking upstream

Add the original as a second remote so you can see if it ever moves again:

```bash
git remote add upstream https://github.com/ImSorryButWho/HomeAssistantNotes.git
git fetch upstream
git log --oneline HEAD..upstream/main -- keypad_blueprint.yaml
```

To see new pull requests without the GitHub UI:

```bash
git fetch upstream '+refs/pull/*/head:refs/remotes/upstream/pr/*'
git log --oneline upstream/main..upstream/pr/50
```

---

## Validation

The blueprint was validated against **Home Assistant 2026.2.3** using HA's own
schemas, not by eye:

- `blueprint.schemas.BLUEPRINT_SCHEMA` — blueprint metadata, sections, selectors
- input substitution via `BlueprintInputs.async_substitute()`
- `cv.TRIGGER_SCHEMA` (20 triggers), `cv.SCRIPT_SCHEMA`, `cv.SCRIPT_VARIABLES_SCHEMA`
- every Jinja template compiled through HA's own template engine
- cross-checks: no orphan trigger ids, no branch referring to a nonexistent
  trigger, no declared-but-unused inputs, no undeclared `!input`
- the three computed variables (`entered_code`, `panic_cancel_requested`,
  `delay_indicator_value`) rendered under 13 scenarios — numeric PIN, PIN with
  a leading zero, bare keypress, both delay formats, a missing `delay`
  attribute, and the post-siren volume-restore path

The one thing no static check can do is press the buttons. Worth trying in
this order after installing: arm away, arm home, disarm with a code, disarm
with a wrong code (error tone), open a door and arm (bypass prompt → Enter),
hold Medical then Disarm with a code (alert clears).

## Credit

Original work and all the Z-Wave reverse-engineering:
[ImSorryButWho](https://github.com/ImSorryButWho/HomeAssistantNotes).
Ideas taken from PRs by [alimac87](https://github.com/ImSorryButWho/HomeAssistantNotes/pull/43),
[olibos](https://github.com/ImSorryButWho/HomeAssistantNotes/pull/36),
[bharvey88](https://github.com/ImSorryButWho/HomeAssistantNotes/pull/49) and
[jsnallen](https://github.com/ImSorryButWho/HomeAssistantNotes/pull/48).
