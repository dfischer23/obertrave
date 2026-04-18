# homeclaude

Working directory for managing a home Home Assistant installation.

## Access

- HA web UI: http://homeassistant:8123/config/dashboard
- SSH: `ssh homeassistant` (root, works from this Mac)
- Config lives on Pi at `/config/`

## Stack

- Home Assistant OS on Raspberry Pi (aarch64, kernel `haos-raspi`)
- IKEA Dirigera hub — custom HACS component: `dirigera_platform` (`/config/custom_components/dirigera_platform`)
- Apple Home bridging via HA built-in HomeKit integration
- ESPHome for custom ESP32 devices
- MQTT used for state publishing

## Devices / Integrations

### ESPHome — `esphome-web-097c1c` ("POT ENC BTN")
ESP32 with rotary encoder (GPIO32/33), potentiometer (GPIO35), push button (GPIO25), red LED (GPIO12), green LED (GPIO14).
Currently: button toggles KaiserIdell light.

### Kitchen Radio (`radio.area:5050`)
Self-built internet radio with HTTP API. `/play` and `/pause` endpoints.
Triggered by amp power sensor (`sensor.steckdose_verstarker_switch_0_power`): turns on above 20W, off below 15W.

### KaiserIdell light
MQTT publishes state/brightness on every state change (topics: `kaiseridell/state`, `kaiseridell/brightness`). Kept for research/monitoring purposes.

## Add-on / firmware versions (as of 2026-04-18)

| Component | Version |
|---|---|
| Home Assistant OS | 2026.4.3 |
| Matter Server | 8.4.0 |
| File Editor | 6.0.0 |
| Mosquitto Broker | 7.0.1 |
| Terminal & SSH | 10.1.0 |
| ESPHome | 2026.4.0 |
| dirigera_platform (HACS) | 0.2.12 |

## Long-lived API token

Stored in HA profile. Use `Bearer` auth against `http://homeassistant:8123/api/`.
(Token not stored here — retrieve from HA UI if needed: Profile → Security → Long-Lived Access Tokens)

## Areas / Rooms

Current areas in HA: Anton, Bathroom, Bedroom, Büro, Dach (Spitzboden), Kabuff, Küche, Living Room, Meta's Zimmer, Vorratsraum.

### Open TODOs

- **Fix amp socket IP** — `shellyplusplugs-e86beae84674` moved from `192.168.2.138` to `192.168.3.138` after VLAN change; currently in `setup_retry`
- **Treppenhaus** → missing area; hosts: wall switches, Raspberry Pi, Dirigera hub
- **3uAirPlayer-DESKTOP-I3DGF3E** → spurious auto-created area, delete it
- **Bedroom** and **Living Room** → empty areas, verify during walk-through
- **ShellyHT 4** (`sensor.shellyht_3d36c8`) → assigned to Büro but likely wrong room; locate during walk-through
- **ShellyHT naming conflict** → sensors in Meta's Zimmer and Vorratsraum both named "ShellyHT 2"; fix after walk-through
- **`media_player.dachgeschoss`** → unassigned, unclear what device this is
- **iBeacon device_trackers** → several unassigned; clarify if intentional or cruft

### Known issues (to fix during walk-through)
- ~~**Anton + Anton's Room** → merge into one area~~ ✓ done
- **Treppenhaus** → missing area; hosts: wall switches, Raspberry Pi, Dirigera hub
- **3uAirPlayer-DESKTOP-I3DGF3E** → spurious auto-created area, delete it
- **Bedroom** and **Living Room** → empty, check if devices need assigning or areas should be removed
- **ShellyHT 4** (`sensor.shellyht_3d36c8`) → assigned to Büro but likely wrong room; locate during walk-through
- **ShellyHT naming conflict** → sensors in Meta's Zimmer and Vorratsraum both named "ShellyHT 2"; fix after walk-through confirms physical locations
- **`media_player.dachgeschoss`** → unassigned, unclear what device this is
- **iBeacon device_trackers** → several unassigned; clarify if intentional tracking or cruft

### Sensors per room (confirmed or assumed)
| Room | Temp/Humidity sensor |
|---|---|
| Bathroom | ShellyHTG3 Sensor 1 |
| Küche | ShellyHTG3 Sensor 2 |
| Meta's Zimmer | ShellyHT (shellyht_3cc34d) |
| Vorratsraum | ShellyHT (shellyht_3d1a7d) |
| Dach (Spitzboden) | ShellyHT (shellyht_3be05e) |
| Kabuff | ShellyHT Unmarked (shellyht_cc1c33) — assumed correct |
| Büro | ShellyHT 4 (shellyht_3d36c8) — **location unconfirmed, likely wrong** |

## light.esstisch — note

IKEA bulb named "Esstisch" in the IKEA app. Currently in storage, not installed anywhere.
Hidden from Apple Home. Name cannot be changed in HA without changing it in IKEA app first.

## Unterbauleuchten (Küche) — needs consolidation

Three entities from dirigera_platform, all exposed to Apple Home causing duplication:
- `light.unterbauleuchten` — IKEA Device Set (group), controls both sides
- `light.76b21e98_0f7a_4a0a_86e0_40b9d3fc685e_1` — TRADFRI Driver 30W, Herdseite (device name still raw UUID)
- `light.2671482c_3634_48ff_84a1_03798c69a2e1_1` — TRADFRI Driver 30W, Spülenseite (device name still raw UUID)

TODO:
- ~~Decide control setup~~ → group only ✓
- ~~Tune Apple Home exposure~~ → individuals excluded ✓
- Rename device names in HA registry (currently UUID strings) — low priority

## Apple Home integration

- **HASS Bridge** (port 21064) — HA exposes entities to Apple Home
- **DIRIGERA HomeKit controller** — ignored (using dirigera_platform instead)
- Included domains: `light`, `switch`, `climate`, `cover`, `fan`, `humidifier`, `lock`, `media_player`, `vacuum`, `water_heater`, `binary_sensor`, `automation`, `scene`, `script`, `sensor`, `person`
- Excluded: diagnostic binary sensors, ESPHome dev board, internal automations, reboot buttons, power/energy sensors, iBeacon trackers, unavailable devices. See `exclude_entities` in HomeKit config entry options.

## Anton's Zimmer — device notes

- **Schreibtischlampe** (`light.anton_dimmer_hinter_lichtscha`) — Shelly Dimmer 2 (SHDM-2) at `192.168.3.192`
  - Button type: `edge` (set via web UI — API rejects changes, likely locked by pulse_mode)
  - `edge` mode required for mixed wall-switch + HA/Apple Home control: prevents double-toggle sync issue
  - `btn_type: toggle` causes out-of-sync when HA turns light on without physical switch movement
  - Automation: turns on at 100% (day) or 40% (night/sun below horizon); only fires when `context.user_id is none` (wall switch), not for Apple Home / HA user actions
- **Nachttischlampe** (`light.tretakt_1`) — switch_as_x wrapper over `switch.tretakt_1`
- **Deckenlampe** (`light.shellybulbduo_08f9e0705f24`) — ShellyBulbDuo

## Meta's Zimmer — device notes

- **Bettlicht** (`light.meta_dimmer_hinter_lichtschalt`) — Shelly dimmer, corner/bed light
- **Deckenlampe** (`light.shellydimmer2_ec64c9c2eacf`) — Shelly Dimmer 2, 5-spotlight ceiling lamp (`shellydimmer2-EC64C9C2EACF` at `192.168.3.69`)
- **Schreibtischlampe** (`light.schreibtischlampe`) — Shelly Color Bulb (`shellycolorbulb-08F9E07024DD`), RGB
- Wall switch (`binary_sensor.meta_dimmer_hinter_lichtschalt_channel_1_input`) — enabled, used by "Meta Dimmerschalter an Deckenlampe" automation (hidden from Apple Home)
- ShellyHT temp/humidity sensor (`shellyht_3cc34d`) — battery entity has wrong name "ShellyHT 2 battery" (naming conflict with Vorratsraum sensor — fix during walk-through)

## Apple Home per-room access control

Apple Home has no per-room permissions for shared users — access is all-or-nothing (full control or view-only for the whole home).
Options if needed: separate HomeKit bridges per zone, or HA app with user-level entity restrictions instead of Apple Home.

## Apple Home room assignment quirks

Apple Home does not reliably inherit room assignments from HA areas. New devices often land in the wrong room (typically Treppenhaus). Fix manually in the Apple Home app: long-press tile → room → correct room.
This has to be done in Apple Home directly — there is no API to push room assignments from HA to Apple Home.

## Shelly btn_type notes

For Shelly dimmers used alongside HA/Apple Home control, use `edge` mode (not `toggle`).
`toggle` mode maps switch physical position to on/off state — breaks when HA changes light state without moving the switch.
`edge` mode toggles on every input state change regardless of direction — no sync dependency.
The SHDM-2 API (`/settings/lights/0?btn_type=edge`) silently ignores changes; must be set via web UI at the device IP.

## Automations summary

| Alias | Trigger | Action |
|---|---|---|
| Küchenlicht an Schalter | Wall switch state change | Toggle kitchen area lights |
| Flood Alarm | Flood sensor moist | Push notify `mobile_app_dphone` |
| Meta Dimmerschalter an Deckenlampe | Dimmer switch ch1 | Toggle Deckenlampe (hidden from Apple Home) |
| Turn on Radio when Amp is turned on | Amp power > 20W | `shell_command.turn_on_kitchen_radio` |
| Turn off Radio when Amp is turned off | Amp power < 15W | `shell_command.turn_off_kitchen_radio` |
| BTN -> KaiserIdell | ESPHome button | Toggle KaiserIdell light |
| Publish KaiserIdell state to MQTT | KaiserIdell state change | Publish state + brightness to MQTT |
| Anton Schreibtischlampe Helligkeit | Schreibtischlampe off→on, no user context | 100% if sun up, 40% if sun down |
