# Testing the blueprint

A run-through that exercises every branch in the automation. Roughly 30–40
minutes if you do all of it. Groups A–C are the boring sanity checks; **group D
is the one worth your attention**, because the sensor-bypass flow is the part
that was broken upstream and is the most intricate logic here.

Before you start:

- Put a temporary action on **Panic button actions → Police / Fire / Medical**
  and on **Panic alert cleared**, so you can see them fire. A
  `notify.persistent_notification` action with a distinct message each is ideal.
- Note your Alarmo exit and entry delays per mode
  (**Settings → Alarmo → your area**), you will compare countdowns against them.
- Open **Developer tools → Events** in a second tab and start listening — see
  [Where to look](#where-to-look) below.
- If you have anyone else in the house, warn them before group F. The siren is
  loud and the Fire alert is very loud.

Legend: 🔊 means the keypad should speak or play a sound.

---

## A. Keypad → Alarmo, the basics

| # | Do this | Expect |
|---|---|---|
| A1 | Code + **Away** | 🔊 "Exit delay started", bar counts up, then 🔊 "Away and armed", Away LED on motion |
| A2 | Code + **Home** | 🔊 "Exit delay started", then 🔊 "Home and armed", Home LED |
| A3 | While armed: code + **Disarm** | 🔊 "Disarmed", Disarmed LED |
| A4 | While armed: **wrong** code + **Disarm** | 🔊 soft error tone, stays armed |
| A5 | While armed: bare **Enter**, no code | 🔊 soft error tone (Alarmo reports no code as an invalid code) |
| A6 | While disarmed: bare **Enter**, no code | **Nothing at all.** Upstream answered this with a spurious error tone; the disarm branch is now gated on the alarm not already being disarmed |

If A4 or A5 stay silent, check in Developer tools → Events whether Alarmo is
actually firing `alarmo_failed_to_arm` for a failed *disarm* on your version —
the event name says "arm", but Alarmo's source fires it for any command with a
bad or missing code. If your version does not, those two tones simply cannot
work and that is Alarmo's side, not the blueprint's.

## B. Alarmo → keypad, from the HA UI

Arm and disarm from a dashboard card, not the keypad. The keypad should follow
every time.

| # | Do this | Expect |
|---|---|---|
| B1 | Arm **Away** in HA | 🔊 exit delay, then 🔊 "Away and armed" |
| B2 | Arm **Home** in HA | 🔊 "Home and armed" |
| B3 | Arm **Night** in HA | 🔊 "Home and armed", **Home** LED — the keypad has no night LED, so night rides on Home by design |
| B4 | Arm **Vacation** in HA | 🔊 "Away and armed", **Away** LED |
| B5 | Disarm in HA | 🔊 "Disarmed" |

B3 and B4 are the only test of those two branches, since you cannot reach those
modes from the keypad's three mode keys.

## C. Countdowns

| # | Do this | Expect |
|---|---|---|
| C1 | Arm Away, watch the light bar | Countdown length matches your Alarmo **exit** delay for that mode |
| C2 | Armed Away, open an entry door | Alarmo → `pending`, 🔊 "Entry delay started", bar counts **down**, length matches your **entry** delay |
| C3 | Set that mode's exit delay to **0** in Alarmo, then arm | Arms instantly with **no** exit-delay announcement |

C3 checks a small fix of ours: a zero delay used to be sent as a `0m0s`
countdown, which makes the keypad announce a countdown that does not exist.

While C1/C2 run, watch **Developer tools → States** for your
`alarm_control_panel.*` entity: the `delay` attribute is exactly what the
blueprint reads to size the countdown. If the announcement fires but the bar
does nothing, that is the `property_key` format problem — see
[If the countdown does not work](#if-the-countdown-does-not-work).

## D. Sensor bypass — the important one

Open a door or window that Alarmo uses for arming, so arming will be refused.

| # | Do this | Expect |
|---|---|---|
| D1 | Code + **Away** with the door open | Does **not** arm. 🔊 "Sensors require bypass", **Enter** button blinks |
| D2 | Then press **Enter** within 30 s | Force-arms anyway → 🔊 "Away and armed" |
| D3 | Repeat D1, then code + **Enter** | Same result. Upstream only accepted a *bare* Enter, so retyping your code could never work — this is the actual bug from upstream issue #23 |
| D4 | Repeat D1, then press **Away** again | Also confirms — Arm counts as confirmation too |
| D5 | Repeat D1, then press **Disarm** or **Cancel** | Abandons it. Stays disarmed, 🔊 "Disarmed" |
| D6 | Repeat D1, then wait out the window | Nothing arms, nothing announced. Confirm the window length matches your **Bypass confirmation window** setting |

If D2 works but D3 does not, the confirmation wait is matching on `event_data`
somewhere it should not. If none of D2–D4 work, check in Developer tools →
Events that Alarmo fires `alarmo_failed_to_arm` with `reason: open_sensors`
**and** an `entity_id` matching your panel — the arm branch waits on exactly
that event, and `entity_id` only appears in Alarmo v1.10.9 and later.

## E. Panic buttons and cancel

| # | Do this | Expect |
|---|---|---|
| E1 | Hold **Medical** until all three lights go out | Your Medical action runs; Medical button lights, bar flashes, no sound |
| E2 | Then code + **Disarm** | Alert clears, 🔊 "Disarmed", and your **Panic alert cleared** action runs |
| E3 | Repeat E1, then a bare **Disarm** with no code | **Nothing** — clearing is code-gated by default. Turn **Require a code to clear a panic alert** off and retry to confirm the other behaviour |
| E4 | Hold **Police**, then code + **Cancel** | Alert clears the same way. Cancel is treated exactly like Disarm |
| E5 | During an exit delay: code + **Cancel** | Countdown stops, Alarmo disarms |
| E6 | Hold **Fire** (loud), then code + **Disarm** | Alert clears |

E1–E4 are entirely new — upstream had no way to silence a manual panic alert
at all, which is open issues #45 and #21.

If the panic buttons do not fire at all, they need a genuine long hold. The
threshold is Z-Wave config **parameter 7** (2–5 s, default 3). Lower it to 2 if
three seconds feels wrong, or raise it to 5 to make accidents less likely.

## F. Siren

| # | Do this | Expect |
|---|---|---|
| F1 | Arm Away, open a door, let the entry delay run out | Alarmo → `triggered`, 🔊 siren at full volume, bar flashes |
| F2 | Code + **Disarm** | Siren stops, 🔊 "Disarmed" |
| F3 | Right after F2, press a few keys and arm again | Key tones and announcements should be at **normal** volume |

F3 is the check for upstream issue #35. If everything sounds quiet afterwards,
look at Z-Wave config **parameter 4** (Announcement Audio Volume) on the
keypad — if it has been changed out from under you, turn on **Re-assert
announcement volume after an alarm** in the blueprint and set the value to
match your normal setting.

## G. Robustness

| # | Do this | Expect |
|---|---|---|
| G1 | Arm Away, then restart Home Assistant | Nothing at the moment HA comes up — it must not replay "Away and armed" as the panel's state is restored. The mode re-sync then fires ~60 s later; see group H |
| G2 | Press **Away** twice quickly | No errors in the log; the automation runs in parallel mode and tolerates it |
| G3 | Only if you run a second Alarmo area: make *that* area fail to arm | Your keypad stays silent |

G1 is the `not_from: [unknown, unavailable]` guard on every state trigger — HA
publishes `unknown → armed_away` for the panel during startup, and without the
guard the keypad announces itself on every restart. What you are checking is
that nothing happens *at* startup; the deliberate re-sync a minute later is
group H. G3 is upstream PR #43.

## H. Mode re-sync

The keypad remembers its own mode LED, and the blueprint normally writes it
only when Alarmo changes state. These checks cover the branch that re-states it
anyway.

| # | Do this | Expect |
|---|---|---|
| H1 | With Alarmo disarmed, restart Home Assistant and wait ~90 s | The Disarmed key is lit again, in silence (**Re-sync silently** is on by default). With that option off it also 🔊 says "Disarmed" |
| H2 | Arm Away, restart HA, wait ~90 s | Away is re-asserted, not Disarmed |
| H3 | Start an exit delay, restart HA mid-countdown | The re-sync **skips** it — no mode is written over the countdown, because `arming` is not a settled state |
| H4 | Set **Also re-sync every N hours** to 1 and wait for the top of the hour | The mode is re-asserted. Leave it at 1 only for the test; 6 is the sensible standing value |

### Testing the silent form by hand

Do this from **Developer tools → Actions** in YAML mode, standing next to the
keypad, somewhere quiet. The point of step 1 is to put the keypad in a
*visibly wrong* state first — firing the silent call while it already shows
Disarmed tells you nothing, because "worked silently" and "command rejected"
look identical.

Make sure Alarmo is **disarmed** before you start. These calls only write to the
keypad; they do not change Alarmo, and they do not trigger the blueprint.

**Step 1 — set a known-wrong mode, audibly.** This also proves your device id
and call shape are right:

```yaml
action: zwave_js.set_value
target:
  device_id: YOUR_KEYPAD_DEVICE_ID
data:
  command_class: "135"
  endpoint: "0"
  property: "11"          # Armed Away
  property_key: "1"
  value: 99
```

Expect the Away LED and 🔊 "Away and armed".

**Step 2 — now the silent Disarmed call.** Watch the LED and listen:

```yaml
action: zwave_js.set_value
target:
  device_id: YOUR_KEYPAD_DEVICE_ID
data:
  command_class: "135"
  endpoint: "0"
  property: "2"           # Disarmed
  property_key: "9"       # voice volume instead of LED brightness
  value: 0
```

**Step 3 — read the result:**

| What happens | Meaning | What to do |
|---|---|---|
| LED switches to Disarmed, **no voice** | Silent re-sync works | Turn on **Re-sync silently**; the N-hourly timer is now reasonable to enable |
| LED switches **and** it says "Disarmed" | Command fine, volume 0 not honoured | Leave the option off, or try `value: 1` — the documented minimum — which may be quiet enough |
| **Nothing at all** | The value was rejected | Retry with `value: 1`. If that also does nothing, this firmware has no silent form |

**Step 4 — put it back**, so the keypad matches Alarmo again:

```yaml
action: zwave_js.set_value
target:
  device_id: YOUR_KEYPAD_DEVICE_ID
data:
  command_class: "135"
  endpoint: "0"
  property: "2"
  property_key: "1"
  value: 99
```

If step 2 produced nothing, also check **Settings → System → Logs** — a value
Z-Wave JS refuses outright usually says so there, which separates "rejected by
the driver" from "accepted but ignored by the keypad".

### If there is no silent form

There is a guaranteed fallback that uses only documented config parameters:
drop **parameter 4** (Announcement Audio Volume) to 0, send the normal
`property_key: "1"` indicator, then put parameter 4 back. Upstream PR #36 uses
exactly this sandwich for its night mode. It costs two extra Z-Wave writes per
re-sync and leaves a brief window where an unrelated announcement would also be
muted, which is why it is not the default — but it works on any firmware.

H1 is deliberately in tension with [G1](#g-robustness): G1 checks that the
keypad stays quiet through a restart, which is what the `not_from` guards on
the state triggers ensure. The re-sync is the *deliberate* exception, running a
minute later and only to fix the LED. With silent re-sync working, both hold at
once.

---

## Where to look

### 1. Developer tools → Events (start here)

This is the best instrument by a wide margin, and it stores nothing. Under
**Listen to events**, subscribe to:

- **`zwave_js_notification`** — one event per keypress. Check `event_type`
  against the table in [`keypad-reference.md`](keypad-reference.md): 2 = Enter,
  3 = Disarm, 5 = Away, 6 = Home, 25 = Cancel, 16/17/19 = Fire/Police/Medical.
  `event_data` is the code you typed, or `null`. Also confirm `device_id`
  matches the keypad you selected in the blueprint.
- **`alarmo_failed_to_arm`** — should carry `entity_id` for your panel and a
  `reason` of `invalid_code`, `open_sensors` or `not_allowed`.

If a keypress produces no event here, the problem is Z-Wave or pairing and the
blueprint never even ran. If the event appears but nothing happens, the problem
is in the automation.

### 2. Developer tools → States

Watch your `alarm_control_panel.*` entity through a whole arm cycle: the
`state` and, during `arming`/`pending`, the `delay` attribute. Those two values
are the entire input to the countdown logic.

### 3. Settings → System → Logs

Use **Load full logs** and search for `zwave_js` or the automation's name.
This is where a failing `zwave_js.set_value` call surfaces — a rejected
`property_key`, an out-of-range value, or a device that did not acknowledge.
Template errors would appear here too.

### 4. Settings → Automations → the automation itself

Check **Last triggered** to confirm it is firing at all, and that HA has not
disabled it after repeated errors.

### 5. Z-Wave JS debug logging (last resort)

**Settings → Devices & services → Z-Wave JS → ⋮ → Enable debug logging**, then
reproduce and download. Use this only when the events look right, the log shows
no errors, and the keypad still does nothing — i.e. the command is leaving HA
but the device is not acting on it. Turn it back off; it is very noisy.

### Automation traces are off on purpose

`stored_traces: 0` in the blueprint means there are no traces to inspect. That
is deliberate: a trace records the template variables, which include the PIN
typed on the keypad, in clear text in `.storage`.

If you really need a trace, do it deliberately and clean up:

1. Create a **throwaway Alarmo user code** and use only that for the test.
2. Edit `stored_traces: 0` to `stored_traces: 5` in the blueprint file, then
   **Developer tools → YAML → Reload blueprints**.
3. Reproduce the problem once. Read the trace.
4. Set it back to `0`, reload, and delete the throwaway code.

The Events tab answers almost everything a trace would, without any of that.

---

## If the countdown does not work

The single most likely incompatibility. If arming announces itself but the
light bar never counts, switch **Entry/exit delay indicator format** to
*Legacy: property key 7*, then retest C1 and C2.

To tell which one your keypad wants, look at its entities in Z-Wave JS. A
keypad that exposes `135-0-18-7` wants the legacy numeric key; one that exposes
only `135-0-18-0/1/9` wants `timeout`. This is upstream issues #42 and #44.

## What to report if something fails

For any failing row above, the useful evidence is:

- the row number and what happened instead
- the raw `zwave_js_notification` event from the Events tab
- the panel's `state` and `delay` at that moment
- any matching lines from the full log
- your HA version, Alarmo version, and Z-Wave JS driver version
