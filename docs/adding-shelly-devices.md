# Adding New Shelly Devices to Home Assistant

_Home Claude · Home Claude Insights · Sigenergy Energy Centre · Climate_


## 1. What this guide covers

Every new Shelly goes through the same chain: the Shelly itself, then Home Assistant, then each dashboard that shows it. Skipping a link does not usually cause an error. It causes quiet problems instead: an orange **!** on a tile, a card that stops updating, or a device missing from a total. This guide walks the whole chain in order.

Not every dashboard needs touching. Only two of the four show Shelly devices:

| Dashboard | Shows Shellys? | What to do |
|---|---|---|
| Home Claude | Yes | Add the device in 4 places (section 6) |
| Home Claude Insights | Yes | Add the device in 2–3 places (section 7) |
| Sigenergy Energy Centre | No | Nothing (section 8) |
| Climate (AirTouch 5) | No | Nothing (section 8) |

> Sigenergy Energy Centre is built from the Sigenergy and analytics sensors, and Climate is built from the AirTouch `climate.*` and `cover.*` entities. Neither one references a Shelly entity, so a new Shelly will not show up or break anything there. If you ever want Shelly power or energy on the Energy Centre, that is a design change, not part of adding a device.


### The chain at a glance

1. **Shelly integration:** Home Assistant discovers and adopts the device (section 3).
2. **Entity IDs:** list the new device's entities so you can copy them exactly (section 4).
3. **Stats script on the Shelly:** counts on-time, starts and switch presses (section 5.1).
4. **HA package:** REST sensors read those stats plus the clock health (section 5.2).
5. **Home Claude** dashboard (section 6).
6. **Home Claude Insights** dashboard (section 7).
7. **Check everything** (section 9).


## 2. Before you start

**Back up every dashboard you will edit.** Open the dashboard → pencil → ⋮ → Raw configuration editor → select all → paste into a text file and save it with today's date. If anything looks wrong afterwards, paste the backup back in.

For each new device, decide and write down:

| Item | Example | Notes |
|---|---|---|
| Display name | Laundry Light | Must be unique. Tiles find devices by this name. |
| Key (slug) | `laundry` | Lowercase, underscores only. Used in sensor names and on the Shelly. |
| Room group | `laundry` | Must match an existing room's group, or you create a new room. |
| Icon | `mdi:washing-machine` | Optional. |
| Type | light or switch | Dimmer Gen4 = light. 1PM Mini Gen4 (fans) = switch. |

Throughout this guide, `XXXXXXXXXXXX` stands for the device's 12-character MAC suffix (for example `e8f60a7c4ef8`), `laundry` stands for your key, and **Laundry Light** for your display name.


## 3. Add the device to Home Assistant

1. Power the Shelly and join it to your Wi-Fi with the Shelly app or its own web page.
2. In HA go to **Settings → Devices & Services**. It should appear as **Discovered** under Shelly. Click **Configure**.
3. If the Shelly has a password set, HA will ask for it.
4. Give the device a friendly name and an area.
5. Update the firmware from the device page if an update is offered, before you go further.

> ⚠️ **Entity IDs: decide once.** Your dashboards use the raw IDs Shelly creates (for example `light.shellydimmerg4_e8f60a7c4ef8`). Keep it that way for consistency. If you rename entities, do it now, before any YAML references them. Renaming later silently breaks every card that uses the old ID.


## 4. Get the exact entity IDs

Typing IDs by hand is where most mistakes come from. Paste this into **Developer Tools → Template**. It lists only Shelly devices that are **not** already on your dashboards:

```jinja
{% set known = [
  'e8f60a7c01d4','e8f60a7c5ffc','acebe6f7211c','e83dc1a20f98',
  'e8f60a79a410','e8f60a7dc7b0','e8f60a7c0940','acebe6f60b34',
  'e8f60a7d87bc','e8f60a7cabbc','98c37700c828','e8f60a7c1d30',
  'd885aced2060','e8f60a7c1bec','e8f60a7c4ef8','98c37700ba58'] %}
{% for d in integration_entities('shelly') | map('device_id')
     | reject('none') | unique | list %}
{% set ents = device_entities(d) %}{% set joined = ents | join(' ') %}
{% if not (known | select('in', joined) | list) %}
== {{ device_attr(d,'name') }} | {{ device_attr(d,'model') }} | area: {{ area_name(d) }}
{% for e in ents %}  {{ e }}
{% endfor %}{% endif %}{% endfor %}
```

> **Keep the known list current.** After you finish adding a device, add its MAC suffix to the list above. Otherwise the next time you run it, the device will show up as "new" again.

From the output, note these entities for each device. A 1PM Mini Gen4 has `switch.` instead of `light.`:

| Role | Entity |
|---|---|
| entity | `light.shellydimmerg4_XXXXXXXXXXXX` |
| power | `sensor.shellydimmerg4_XXXXXXXXXXXX_power` |
| temp | `sensor.shellydimmerg4_XXXXXXXXXXXX_temperature` |
| wifi | `sensor.shellydimmerg4_XXXXXXXXXXXX_signal_strength` |

> Temperature and Wi-Fi sensors on Shellys are often **disabled by default** as diagnostic entities. If they are missing from the list, enable them on the device page (Entities → the entity → cog → Enabled), then wait a minute.


## 5. Stats and clock sensors

The dashboards show run hours, starts and switch presses, and they flag clock problems. None of this comes from the Shelly integration. It needs two extra pieces per device. Without them, the device still works but shows an orange **!** with "no stats" or "clock", and the health footer counts it as needing attention.


### 5.1 On the Shelly: the stats-only script

Install **shelly_stats_only.js** (kept in this repo under `shelly-scripts/`). It counts usage and saves it on the Shelly in the same format as your older devices, and it never switches the light or fan.

1. Open the Shelly's web page (HA device page → **Visit**).
2. **Scripts → Create script**, paste the code.
3. Change the three **EDIT** lines **before** the first start (table below).
4. Save, **Start**, and turn on **Run on startup**.
5. The log should show `KVS <key>_dimmer_stats` and **STATS SAVED**.

| EDIT line | Value | Example |
|---|---|---|
| `DEVICE_NAME` | Display name | `"Laundry Light"` |
| `DEVICE_KEY` | Key, all lowercase | `"laundry"` |
| `COMPONENT` | `"light"` for dimmers, `"switch"` for 1PM Mini | `"light"` |

> ⚠️ **Do not copy a sunset script from another light.** Your older devices run full sunset automations that turn the light on before sunset and off at sunrise. Copied onto a new device, it will start switching that device by itself every evening. On a 1PM Mini it fails outright, because it drives `light:0` and the Mini only has `switch:0`. Use the stats-only script unless you actually want the sunset behaviour.


### 5.2 In Home Assistant: the package

Your new-device sensors live in `/config/packages/shelly_new_devices.yaml`. Add one copy of each block below per device, under the matching heading in that file. The REST sensors look up the device's IP address from the Shelly integration, so nothing needs changing if its IP changes.

**a) Stats** (under `rest:`):

```yaml
  - resource_template: >-
      {{ device_attr(device_id('light.shellydimmerg4_XXXXXXXXXXXX'),
         'configuration_url') | string | regex_replace('/+$', '') }}/rpc/KVS.Get?key=laundry_dimmer_stats
    scan_interval: 60
    timeout: 8
    sensor:
      - name: Shelly Stats Laundry
        unique_id: shelly_stats_laundry
        value_template: "{{ value_json.get('value', {}).get('onCount', 0) }}"
        json_attributes_path: "$.value"
        json_attributes: [input0, input1, onCount, runtimeSeconds]
```

**b) Raw clock status** (also under `rest:`):

```yaml
  - resource_template: >-
      {{ device_attr(device_id('light.shellydimmerg4_XXXXXXXXXXXX'),
         'configuration_url') | string | regex_replace('/+$', '') }}/rpc/Sys.GetStatus
    scan_interval: 300
    timeout: 10
    sensor:
      - name: Shelly NTP Raw Laundry
        unique_id: shelly_ntp_raw_laundry
        value_template: "{{ value_json.last_sync_ts | default(0, true) | int(0) }}"
        json_attributes: [unixtime, time, uptime, mac]
```

**c) Clock health** (under `template:` → `- sensor:`):

```yaml
      - name: Shelly NTP Laundry Status
        unique_id: shelly_ntp_laundry_status
        icon: mdi:clock-check-outline
        state: >-
          {% set v = states('sensor.shelly_ntp_raw_laundry') %}
          {% set ts = v | int(0) %}
          {% set now_ts = as_timestamp(now()) %}
          {% if v in ['unknown', 'unavailable', 'none'] %} Offline
          {% elif ts <= 0 %} Unsynchronised
          {% elif ts > now_ts + 300 %} Clock error
          {% elif now_ts - ts > 14400 %} Stale
          {% else %} Healthy
          {% endif %}
```

**d) Clock fault** (under `template:` → `- binary_sensor:`), then add `'binary_sensor.shelly_ntp_laundry_fault'` to the names list in the **Shelly - New Device NTP Monitoring** automation at the bottom of the file:

```yaml
      - name: Shelly NTP Laundry Fault
        unique_id: shelly_ntp_fault_laundry
        device_class: problem
        delay_on: "00:10:00"
        state: >-
          {{ states('sensor.shelly_ntp_laundry_status') != 'Healthy' }}
```

> ⚠️ **Names matter.** HA builds the entity ID from the **name**, not the `unique_id`. "Shelly Stats Laundry" becomes `sensor.shelly_stats_laundry`, which is what the dashboards expect. If you wrote "Shelly Stats Laundry Light" you would get `sensor.shelly_stats_laundry_light` and the tile would show "no stats". So the words after "Shelly Stats" / "Shelly NTP" must turn into your key. Check the final IDs in Developer Tools → States before moving to section 6.

Then **Developer Tools → YAML → Check configuration**, and **Restart** Home Assistant. Within a minute or two you should see `sensor.shelly_stats_laundry` with an `onCount` attribute and `sensor.shelly_ntp_laundry_status` = `Healthy`.


## 6. Home Claude dashboard

Every card reads its devices from one list, but each device's entities also have to be listed in several `triggers_update` lists so the cards refresh. **Four places, every time.** Missing one means a card that silently stops updating.


### 6.1 The device list

In `button_card_templates → hm_base → variables → devices`, add:

```yaml
        - name: Laundry Light          # unique; tiles find devices by this
          group: laundry               # must match the room's group
          icon: mdi:washing-machine
          entity: light.shellydimmerg4_XXXXXXXXXXXX
          stats: sensor.shelly_stats_laundry
          clock: sensor.shelly_ntp_laundry_status
          power: sensor.shellydimmerg4_XXXXXXXXXXXX_power
          temp: sensor.shellydimmerg4_XXXXXXXXXXXX_temperature
          wifi: sensor.shellydimmerg4_XXXXXXXXXXXX_signal_strength
```


### 6.2 A tile in the room's section

```yaml
          - type: custom:button-card
            template: hm_tile
            variables:
              name: Laundry Light      # exactly as in the list
            triggers_update:
              - light.shellydimmerg4_XXXXXXXXXXXX
              - sensor.shelly_stats_laundry
              - sensor.shelly_ntp_laundry_status
              - sensor.shellydimmerg4_XXXXXXXXXXXX_power
              - sensor.shellydimmerg4_XXXXXXXXXXXX_temperature
              - sensor.shellydimmerg4_XXXXXXXXXXXX_signal_strength
            grid_options:
              columns: 6               # 6 = half width, 12 = full
              rows: auto
            hold_action:
              action: more-info
              entity: light.shellydimmerg4_XXXXXXXXXXXX
```


### 6.3 The room's hm_room and hm_alloff cards

Add the same **six** entities to the `triggers_update` list of both cards in that room's section. **New room?** Copy a whole existing section (room header, all-off button, tiles) and change its `group` to the new one.


### 6.4 The hm_health card at the bottom

Add the same six entities to its `triggers_update` list.

The hero card's "Lights x of y" count and the health total pick up the new device from the list automatically.

> **Two quirks.** Tiles shorten names by dropping a leading "Ensuite " or "Kitchen ", so "Kitchen Pantry" shows as "Pantry". And a wrong entity ID is partly forgiven: if one ID is wrong but another from the same device is right, the card still finds the device. That hides typos, so check the tile shows power, temperature and Wi-Fi, not just on/off.


## 7. Home Claude Insights dashboard

Insights has its own copy of the device list. It is **not** shared with Home Claude, so it must be updated separately and kept identical.


### 7.1 The device list

Same place (`button_card_templates → hm_base → variables → devices`), same entry, but **without the `clock` line**:

```yaml
        - name: Laundry Light
          group: laundry
          icon: mdi:washing-machine
          entity: light.shellydimmerg4_XXXXXXXXXXXX
          stats: sensor.shelly_stats_laundry
          power: sensor.shellydimmerg4_XXXXXXXXXXXX_power
          temp: sensor.shellydimmerg4_XXXXXXXXXXXX_temperature
          wifi: sensor.shellydimmerg4_XXXXXXXXXXXX_signal_strength
```


### 7.2 The hm_ins template's triggers_update

Insights uses one shared trigger list on the `hm_ins` template, not one per card. Add three lines, next to the matching lines of the other devices:

```yaml
      - light.shellydimmerg4_XXXXXXXXXXXX
      - sensor.shellydimmerg4_XXXXXXXXXXXX_power
      - sensor.shelly_stats_laundry
```


### 7.3 Only for a new room: the room glow map

Search for the array that starts with `['outside',` and add the new room as `[group, 'Label']`:

```js
... ['ensuite','Ensuite'],['main_bathroom','Main Bathroom'],['laundry','Laundry'],['theater','Theater']]
```

The CSS already stretches an odd last room across the full width, so an uneven count still lays out cleanly.

> **Expect this after adding:** a new device starts from zero in Habits and Hands, so it sits at the bottom of those lists until it builds up history. 1PM Minis run at 65–72 °C, which the Fleet panel colours as "hot" (70 °C or above). That is likely normal for that model, but the actual rated limit has not been confirmed.


## 8. Sigenergy Energy Centre and Climate

**No changes needed**, based on the versions of these dashboards built so far:

- **Sigenergy Energy Centre** uses the `sig_kpi`, `sig_card` and `sig_header` templates fed by `sensor.sigenergy_*` and `sigen_plant_*` entities. No Shelly entity appears in it.
- **Climate** is the AirTouch 5 dashboard, built from `climate.ac_0`, the five zone `climate.*` entities, their `cover.*_damper` entities and the airtouch package helpers. No Shelly entity appears in it.

Note that the **Insights** hero does use Sigenergy sensors (sun / battery / grid share of lighting power). That link runs from Insights to Sigenergy, so it needs no change when you add a Shelly either.


## 9. Final checks

| Check | Where | Expected |
|---|---|---|
| Shelly script running | Shelly web page → Scripts | Running, Run on startup on, log shows STATS SAVED |
| Stats sensor | Developer Tools → States | `sensor.shelly_stats_<key>` with `onCount` attribute |
| Clock sensor | Developer Tools → States | `sensor.shelly_ntp_<key>_status` = Healthy |
| Home Claude tile | Dashboard | Power, temperature and Wi-Fi shown; no orange ! |
| Home Claude total | Hero card | "Lights x of y" includes the new device |
| Health footer | Bottom of Home Claude | No "needs attention" for the new device |
| Insights | Fleet, Watch, Timeline | New device listed; updates when switched |
| Template list | Section 4 | New MAC suffix added to the known list |


### Troubleshooting

| Symptom | Likely cause |
|---|---|
| Orange ! with "no stats" | Script not running on the Shelly, wrong `DEVICE_KEY`, or the stats sensor ID doesn't match the dashboard |
| Orange ! with "clock …" | NTP status sensor missing, misnamed, or the Shelly can't reach a time server |
| Tile only shows on/off | Wrong or disabled power / temperature / Wi-Fi entity |
| Tile never updates | Entity missing from a `triggers_update` list (sections 6.2–6.4 or 7.2) |
| "Template not found" | Card pasted into a different dashboard from the one that defines its template |
| Raw editor won't save | Indentation off by a space, or a second `views:` line pasted in |
| Light turns on at sunset by itself | A sunset script was copied onto it instead of the stats-only script |


## 10. Keeping this in GitHub

Repo layout:

```text
claude-dashboards/
├── README.md
├── docs/
│   ├── adding-shelly-devices.pdf
│   └── adding-shelly-devices.md
├── dashboards/
│   ├── home_claude.yaml
│   ├── home_claude_insights.yaml
│   ├── sigenergy_energy_centre.yaml
│   └── climate.yaml
├── packages/
│   └── shelly_new_devices.yaml
└── shelly-scripts/
    └── shelly_stats_only.js
```

Workflow for each change: export the dashboard's raw config into `dashboards/`, commit with a clear message (for example "Add Laundry Light to Home Claude and Insights"), and you have a dated backup you can roll back to.

> ⚠️ **Check before you push.** If the repo is public, anyone can read it. Never commit `secrets.yaml`, tokens or passwords. Shelly scripts or packages that hold a device password should use placeholders. Older packages contain local IPs (10.2.2.x); those aren't reachable from outside your network, but decide whether you're comfortable publishing them.
