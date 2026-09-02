# Ring Alarm Keypad v2 — Z-Wave reference

Condensed and corrected from
[ImSorryButWho's RingKeypadV2.md](https://github.com/ImSorryButWho/HomeAssistantNotes/blob/main/RingKeypadV2.md),
with the `property_key` correction from upstream PR #48 / issue #46 and the
config-parameter table taken from the
[Z-Wave JS device file](https://github.com/zwave-js/zwave-js/blob/master/packages/config/config/devices/0x0346/keypad_v2.json)
(manufacturer `0x0346`, label `4AK1SZ`).

## Pairing

Full functionality requires **S2** security, and the negotiation times out
quickly — use **Smart Start** and scan the QR code rather than classic
inclusion. The keypad will not pair on battery: plug it in first.

## Button events — `zwave_js_notification`, `command_class: 111`

| `event_type` | Meaning |
|---|---|
| 0 | Started entering a code |
| 1 | Entered a code but never pressed a button (timeout) |
| 2 | Enter |
| 3 | Disarm |
| 5 | Arm Away |
| 6 | Arm Stay / Home |
| 16 | Fire — only sent if held until all three lights go out |
| 17 | Police — only sent if held until all three lights go out |
| 19 | Medical — only sent if held until all three lights go out |
| 25 | Cancel |

`event_data` carries the digits typed before the button press, or `null`.

The blueprint uses 2, 3, 5, 6, 16, 17, 19 and 25. Events 0 and 1 are noise.

## Indicators — `zwave_js.set_value`, `command_class: 135`, `endpoint: 0`

### Modes, alarms and messages — `property_key: "1"`, `value` 0–99

`value` is the brightness of the corresponding LED while switching modes.
**99 is the maximum — 100 is out of range** (upstream fixed this in `7973a1e`;
open PR #36 reintroduces the bug).

| `property` | Effect |
|---|---|
| 2 | **Disarmed.** Says "Disarmed"; Disarmed LED lights on motion. Also the way to silence a running alert. |
| 9 | **Code not accepted.** Soft error tone. |
| 10 | **Armed Stay.** Says "Home and armed". |
| 11 | **Armed Away.** Says "Away and armed". |
| 12 | Generic alarm. Plays alarm, flashes until another mode is set. Ignores duration. |
| 13 | Burglar alarm. Identical to 12. |
| 14 | Smoke alarm. Ignores duration. |
| 15 | Carbon monoxide alarm — intermittent beeping. Ignores duration. |
| 16 | Says "Sensors require bypass"; Enter button blinks. |
| 19 | Medical alert. Medical button lights, bar flashes, no sound. Ignores duration. |

### Entry / exit delays — `property_key: "timeout"`

**This is the part the upstream docs get wrong.** Current Z-Wave JS wants
`property_key: "timeout"` and a duration string like `"1m30s"` — not
`property_key: 7` with a number of seconds. Some keypads/older Z-Wave JS
versions still expose the numeric key instead (issue #42), which is why the
blueprint makes the format selectable.

| `property` | Effect |
|---|---|
| 17 | **Entry delay.** Says "Entry delay started", accelerating tone, bar counts down. |
| 18 | **Exit delay.** Says "Exit delay started", accelerating tone, bar counts up. |

### Notification sounds — `property_key: "9"` (volume 0–99)

| `property` | Sound |
|---|---|
| 96 | Electronic double beep |
| 97 | Guitar riff |
| 98 | Wind chimes |
| 99 | Echoey bing-bong |
| 100 | Ring doorbell chime |

Property 13 with `property_key: "9"` is how the blueprint fires the siren at
full volume on `triggered` (upstream PR #40).

Combining property keys — a volume *and* a duration in one call — is not
possible through `zwave_js.set_value`; it would need `zwave_js.invoke_cc_api`.

## Device config parameters

Set these in **Settings → Devices & services → Z-Wave JS → your keypad →
Configure device**. They are persistent device settings, which is why the
blueprint does not manage them.

| # | Label | Range | Default |
|---|---|---|---|
| 1 | Heartbeat interval | 1–70 min | 70 |
| 2 | Message retry attempt limit | 0–5 | 1 |
| 3 | Delay between retry attempts | 1–60 s | 5 |
| **4** | **Announcement audio volume** | 0–10 | 7 |
| **5** | **Key tone volume** | 0–10 | 6 |
| **6** | **Siren volume** | 0–10 | 10 |
| 7 | Long press duration: emergency buttons | 2–5 s | 3 |
| 8 | Long press duration: number pad | 2–5 s | 3 |
| 9 | Timeout: proximity display | 0–30 s | 5 |
| 10 | Timeout: display on button press | 0–30 s | 5 |
| 11 | Timeout: display on status change | 1–30 s | 5 |
| **12** | **Brightness: security mode** | 0–100 % | 100 |
| **13** | **Brightness: key backlight** | 0–100 % | 100 |
| 14 | Key backlight ambient light sensor level | 0–100 % | 20 |
| 15 | Proximity detection | disable / enable | enable |
| 16 | LED ramp time | 0–255 s | 50 |
| 17 | Battery low threshold | 0–100 % | 15 |
| **18** | **Keypad language** | English / French / Spanish | 30 |
| 19 | Battery warning threshold | 0–100 % | 5 |
| 20 | Security mode blink duration | 1–60 | 2 |
| 21 | Supervision report timeout | 500–30000 ms | 10000 |
| 22 | Security mode display | always off / always on | always off |
| 23 | Languages supported (read-only bitmask) | — | 37 |
| 24 | Calibrate speaker | disable / enable | disable |
| 26 | Motion sensor timeout (firmware ≥ 1.18) | 0–60 s | 3 |

Parameter 7 is worth knowing about: it is the hold time required before the
Fire / Police / Medical buttons actually send their event. Shorten it to 2 s if
three seconds of holding feels long, lengthen it to 5 s to make accidental
panic alerts harder.

## Alarmo side

The blueprint reads Alarmo's `delay` attribute on the panel entity to size the
countdown, and listens for `alarmo_failed_to_arm` with `reason` of
`invalid_code`, `open_sensors` or `not_allowed`. `entity_id` in that event
payload has been present since Alarmo **v1.10.9** (2025-05-25); the filter on it
is what stops a second Alarmo area from beeping your keypad.
