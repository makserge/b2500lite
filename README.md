# B2500 Lite for Home Assistant

A minimal Home Assistant package that shows the live status of **two Marstek B2500
batteries** over local MQTT. It is a reduced spin-off of
[b2500pkg](https://github.com/Martin0475/b2500pkg): read-only, no helpers, no update
script, no control functions.

What it consists of:

| Part | What it does |
|---|---|
| 1 automation (`B2500 Lite bridge`) | Every 30 s asks both batteries for their status (`cd=01`) and republishes each answer as JSON on `b2500lite/b1/cd01` and `b2500lite/b2/cd01`. |
| 28 MQTT sensors | 14 per battery, grouped into two devices: **B2500 Lite 1** and **B2500 Lite 2**. |
| 1 dashboard card | Standard entities card (`dashboard_card.yaml`) with one section per battery. |

The card is a single list titled *B2500* with two sections, *Battery 1* and
*Battery 2*, each showing the 14 values of that battery:

```
B2500
Battery 1
  SOC                 67 %
  Temperature min     21 °C
  Temperature max     24 °C
  Battery             Charging
  PV1 in state        Charging
  PV1 in power        210 W
  ...
Battery 2
  SOC                 54 %
  ...
```

## Sensors

`X` is `1` or `2`. All values come from the battery's CD01 answer.

| Entity | Meaning | CD01 field |
|---|---|---|
| `sensor.b2500lite_bX_soc` | State of charge, % | `pe` |
| `sensor.b2500lite_bX_temp_min` | Lowest internal temperature, °C | `tl` |
| `sensor.b2500lite_bX_temp_max` | Highest internal temperature, °C | `th` |
| `sensor.b2500lite_bX_pv1_state` | PV1 input: `Off` / `Charging` / `Pass-through` | `p1` |
| `sensor.b2500lite_bX_pv2_state` | PV2 input: `Off` / `Charging` / `Pass-through` | `p2` |
| `sensor.b2500lite_bX_pv1_power` | PV1 input power, W | `w1` |
| `sensor.b2500lite_bX_pv2_power` | PV2 input power, W | `w2` |
| `sensor.b2500lite_bX_out1_state` | Output 1: `Off` / `Discharging` | `o1` |
| `sensor.b2500lite_bX_out2_state` | Output 2: `Off` / `Discharging` | `o2` |
| `sensor.b2500lite_bX_out1_power` | Output 1 power, W | `g1` |
| `sensor.b2500lite_bX_out2_power` | Output 2 power, W | `g2` |
| `sensor.b2500lite_bX_battery_state` | `Charging` / `Discharging` / `Idle` | `l0` bit 1 / bit 0 |
| `sensor.b2500lite_bX_charge_power` | Power going into the battery, W | `w1 + w2 − g1 − g2` when positive |
| `sensor.b2500lite_bX_discharge_power` | Power coming out of the battery, W | `g1 + g2 − w1 − w2` when positive |

Charge and discharge power are estimates: CD01 has no direct battery-power value, so
they are calculated as solar input minus AC output, without conversion losses.

Every sensor is `unavailable` until its battery's first answer arrives, and turns
`unavailable` again when the battery has not answered for 5 minutes.

## Requirements

- Home Assistant **2025.10 or newer** (the sensors use `default_entity_id`; see
  [Older Home Assistant versions](#older-home-assistant-versions) otherwise).
- The **MQTT integration** set up in Home Assistant, connected to a broker
  (for example the Mosquitto add-on).
- Both batteries connected to that broker via their local MQTT setting.
  If you use more than one battery, give **each battery its own MQTT user**
  on the broker (for Mosquitto: add a login per battery in the add-on configuration).

## Installation

### 1. Find each battery's topic prefix, device type and MAC

Each battery answers on a topic of this form:

```
<prefix>/<device type>/device/<mac>/ctrl
```

for example `hame_energy/HMA-1/device/0123456789ab/ctrl`.

The easiest way to find yours: open an MQTT client such as
[MQTT Explorer](https://mqtt-explorer.com/), connect it to your broker and look for
topics starting with `hame_energy/` or `marstek_energy/`. Note for each battery:

- **base** = `<prefix>/<device type>`, e.g. `hame_energy/HMA-1`
- **mac** = the part between `device/` and `/ctrl`, exactly as written in the topic

If nothing shows up, publish `cd=01` to `<prefix>/<device type>/App/<mac>/ctrl` with
the values you expect; the battery should answer on the `device` topic within a few seconds.

### 2. Enable packages (once)

If your `configuration.yaml` does not already load packages, add:

```yaml
homeassistant:
  packages: !include_dir_named packages
```

and create a folder named `packages` next to `configuration.yaml`.
If you already have a `homeassistant:` section, add only the `packages:` line to it.

### 3. Copy the package

Copy `packages/b2500lite.yaml` from this archive to `/config/packages/b2500lite.yaml`
(for example with the *File editor* or *Studio Code Server* add-on, or Samba).

### 4. Set the constants

Open `/config/packages/b2500lite.yaml` and edit the **CONSTANTS** block near the top:

```yaml
    trigger_variables:
      b1_base: "hame_energy/HMA-1"     # battery 1: <prefix>/<device type>
      b1_mac: "000000000001"           # battery 1: MAC/ID as in the topic
      b2_base: "hame_energy/HMA-1"     # battery 2
      b2_mac: "000000000002"
```

These four values are the only thing you need to change.

### 5. Check and restart

1. **Settings → Tools → YAML → Check configuration**: it must report no errors.
   (Before Home Assistant 2026.8 this page is called *Developer tools*.)
2. **Settings → System → Restart**. This is the simplest way to load the new
   automation and the 28 MQTT sensors in one go. Alternatively reload
   **Automations** and **Manually configured MQTT entities** on the same YAML page.

Note: the configuration check does not validate the individual MQTT sensors. If one of
them is invalid, the check still passes and the problem only shows up after the
restart, under **Settings → System → Repairs** and in the log.

### 6. Verify

- **Settings → Devices & services → MQTT**: two devices, *B2500 Lite 1* and
  *B2500 Lite 2*, with 14 entities each.
- Within about 30 seconds the sensors show values. If they stay `unavailable`,
  see [Troubleshooting](#troubleshooting).

### 7. Add the card

1. Open a dashboard you created yourself. If you have none, create one:
   **Settings → Dashboards → Add dashboard → New dashboard from scratch**.
   (The built-in *Home* dashboard does not accept manual cards.)
2. Click **Edit** (pencil). On an auto-generated dashboard, choose
   ⋮ → **Take control** first.
3. Click **+** inside a section (or **Add card** in a masonry view).
4. Switch to the **By card** tab (the dialog opens on *By entity*), scroll down and
   choose **Manual**.
5. Replace the content with the content of `dashboard_card.yaml` from this archive,
   then click **Save**.

The card works as is if you did not rename any sensors. The card's title and section
labels are set in `dashboard_card.yaml`.

## Customization

| What | Where |
|---|---|
| Poll interval | `seconds: "/30"` in the automation's `time_pattern` trigger |
| Timeout before values go `unavailable` | `expire_after: 300` on each sensor |
| Device names in HA | `name: B2500 Lite 1` / `name: B2500 Lite 2` under `device:` |
| Card title and section labels | `title:` and the `label:` lines of the `section` entries in `dashboard_card.yaml` |

Restart Home Assistant after editing the package.

## Uninstall

1. **Remove the card**: open the dashboard → **Edit** → card menu (⋮) → **Delete**.
2. **Delete the package file** `/config/packages/b2500lite.yaml`.
3. If B2500 Lite was your only package, you may also remove the
   `packages: !include_dir_named packages` line (and the empty `packages` folder).
   Keep it if other packages use it.
4. **Restart Home Assistant** (Settings → System → Restart).
5. **Remove leftover entities**: Settings → Devices & services → **Entities**, search
   for `b2500lite`. Entities still listed are shown as *Not provided*. Enter selection
   mode, select them all, then in the ⋮ menu of the selection bar choose
   **Delete selected**. If the devices *B2500 Lite 1/2* are still listed under
   **MQTT**, open each and choose **Delete**.
6. **Optional, remove history**: **Settings → Tools → Statistics** (before 2026.8:
   *Developer tools → Statistics*), search for `b2500lite`. The removed sensors show the
   issue *There is no state available for this entity*. For each one choose
   **Fix issue**; in the dialog *Entity has no state* choose **Delete**.

Nothing is left on the MQTT broker: the bridge publishes without the *retain* flag,
and the batteries' own MQTT settings are not touched.

## Troubleshooting

| Symptom | Check |
|---|---|
| No `b2500lite` sensors exist after the restart, and **Settings → System → Repairs** (or the log) shows *Invalid config found for MQTT sensor item* with `extra keys not allowed @ data['default_entity_id']` | Home Assistant is older than 2025.10; the configuration check does not catch this. See below. |
| Sensors are `unavailable` from the start and never show a value | In MQTT Explorer, is anything published on `b2500lite/b1/cd01`? If not, check the battery answers on `<base>/device/<mac>/ctrl`; base and mac must match the topic **exactly** (case included). |
| Only one battery works | Each battery needs its own MQTT user on the broker; two devices sharing one login keep disconnecting each other. |
| Values turn `unavailable` after they had worked | The battery stopped answering for 5 minutes (Wi-Fi, broker connection). |
| Automation trace is empty | Traces are disabled (`stored_traces: 0`) to keep the database small; set it to e.g. `5` while debugging. |

### Older Home Assistant versions

Before 2025.10, MQTT entities used `object_id` instead of `default_entity_id`.
Replace every

```yaml
      default_entity_id: sensor.b2500lite_b1_soc
```

with

```yaml
      object_id: b2500lite_b1_soc
```

(drop the `sensor.` prefix), then check the configuration and restart.

## Running next to the full b2500pkg

Both can be installed at the same time. Nothing B2500 Lite creates overlaps with
B2500PKG:

| | B2500 Lite | B2500PKG |
|---|---|---|
| Entity IDs | `sensor.b2500lite_b1_*`, `sensor.b2500lite_b2_*` | `sensor.b2500_01_*` … `sensor.b2500_09_*` and others |
| Unique IDs | `b2500lite_*` | `b2500_*`, `b2500pkg_*` |
| Devices | *B2500 Lite 1*, *B2500 Lite 2* (`b2500lite_b1/_b2`) | *B01* … *B09* (`b2500_01` … `b2500_09`) |
| Entity names | e.g. *B2500 Lite 1 SOC* | e.g. *B01 Battery SOC* |
| Automation | `automation.b2500_lite_bridge` | `automation.b2500pkg_*` |
| Own MQTT topics | `b2500lite/...` (not retained) | `B2500PKG/...`, `homeassistant/...` |
| Package file | `packages/b2500lite.yaml` | `packages/b2500pkg_update.yaml`, `packages/b2500pkg/` |

B2500 Lite only reads the batteries' answers; it does not use or change any B2500PKG
helper or setting. With both installed, each battery receives one extra `cd=01`
request every 30 seconds.

## Copyright and license

B2500 Lite is derived from **B2500PKG**
© [Martin0475](https://github.com/Martin0475) –
<https://github.com/Martin0475/b2500pkg> – licensed under the
GNU General Public License v2.0.

B2500 Lite reuses B2500PKG's MQTT request/answer handling (topic layout, `cd=01`
polling, answer parsing and the CD01 field meanings) in reduced and modified form
(changed 2026-10-26). It is distributed under the same license,
**GPL-2.0**; see the `LICENSE` file. There is no warranty, as stated in the license.
