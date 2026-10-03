# Local Hive Heating Build Guide

Oct 1, 2026 · @Derek McAuley

Hive heating moved off the Hive cloud and runs entirely in Home Assistant via Zigbee2MQTT, with a schedule, boost and a failsafe if HA goes down. This is the step-by-step for the video.

## At a glance

&#91;embedded content: local heating · control chain\]

The schedule sets a target, Home Heating compares it with the average temperature, and the Boiler switch tells the receiver over Zigbee2MQTT. The Mini stays paired to the receiver as a manual fallback.

## What you need

No new hardware: the existing Hive kit plus a Zigbee coordinator and three pieces of software.

| Item | What it does | Notes |
| --- | --- | --- |
| Hive Receiver (SLR2-family, dual channel) | Switches the boiler (heating + hot water) | Two buttons: tap = hot water, flame = heating |
| Hive Thermostat Mini (SLT6) | Wall dial + hall temperature | Battery powered; only reports battery to Z2M |
| Sonoff ZBDongle-P | Zigbee coordinator | Passed through to the HA VM |
| Zigbee2MQTT | Talks to the Hive devices locally | Replaces the Hive Hub 360 + cloud |
| Home Assistant helpers | Boiler switch, Home Heating thermostat, average temperature | All built-in, created in the UI |
| Climate Scheduler (kneave, HACS) | Graph-based weekly schedule | Integration + dashboard card |
| Room temperature sensors | Living room, bedroom, office | Averaged with the hall reading |

## Step 1: Pair the Hive Receiver to Zigbee2MQTT

The receiver leaves the Hive hub and joins your own Zigbee network. Do it on a mild day; it takes about 20 minutes.

1. Unplug the Hive Hub 360 so it can't grab the receiver back.
2. Lift the Hive Mini off its mount and pull one battery.
3. Hold the **flame** (central heating) button until the light turns **pink** — stand-alone mode.
4. Hold it again until the light **double-flashes amber** — pairing mode.
5. In Zigbee2MQTT, click **Permit join (All)**. The receiver joins.
6. Rename it `Hive Receiver` and tick **Update Home Assistant entity ID**.

If the light won't go pink or amber, power-cycle the receiver at the heating fused spur (10 seconds off) and repeat steps 3–4. Safety: the backplate terminals are 230 V — switch off the spur before touching the receiver.

You now get `climate.hive_receiver_heat` and `climate.hive_receiver_water` in HA. Blur the install code sticker on camera.

## Step 2: Pair the Hive Mini through the receiver

The Mini pairs to the receiver, with both on Zigbee2MQTT, so the wall dial still works even if HA is down.

1. Put the battery back in the Mini and let it boot.
2. Hold **Menu + Back** until the countdown finishes and the Hive logo shows — factory reset, now in pairing mode.
3. In Zigbee2MQTT, open the dropdown next to Permit join and pick **Hive Receiver**, so the Mini joins through it.
4. The receiver light goes **amber → green** and the Mini runs its setup wizard.
5. Rename it `Hive Thermostat`.

In Z2M the Mini only shows a battery entity. Its temperature appears on the receiver as `sensor.hive_receiver_local_temperature_heat`. Pairing often takes two or three tries — reset the Mini and repeat if it sticks on "searching".

No dropdown in your Z2M version? Run this from **Developer tools → Actions**:

```yaml
action: mqtt.publish
data:
  topic: zigbee2mqtt/bridge/request/permit_join
  payload: '{"time": 254, "device": "Hive Receiver"}'
```

## Step 3: The gotcha — why HA doesn't drive the receiver directly

The receiver ignores normal HA thermostat commands. Sending `climate.set_temperature` with heat at 7°C switched it to heat but dropped the setpoint to 1°C.

It only behaves when the mode, a "hold" flag and the setpoint arrive together in one MQTT message:

```json
{"system_mode_heat":"heat","temperature_setpoint_hold_heat":1,"occupied_heating_setpoint_heat":20}
```

So most heating add-ons and blueprints misbehave if pointed straight at `climate.hive_receiver_heat`. The fix: HA becomes the thermostat, and the receiver becomes a simple boiler on/off switch (Steps 4–5). Good moment for the video — show the failed command, then the fix.

## Step 4: The Boiler switch

`switch.boiler` turns the receiver into a simple on/off boiler switch. On = heat with a 23°C cap; off = heat at 7°C (frost protection), never fully off.

Create it in **Settings → Devices & services → Helpers → Create helper → Template → Switch**, name `Boiler`, icon `mdi:water-boiler`.

**Value template**

```jinja
{{ is_state('climate.hive_receiver_heat', 'heat') and (state_attr('climate.hive_receiver_heat', 'temperature') | float(0)) > 10 }}
```

**Actions on turn on**

```yaml
action: mqtt.publish
data:
  topic: zigbee2mqtt/Hive Receiver/set
  payload: '{"system_mode_heat":"heat","temperature_setpoint_hold_heat":1,"occupied_heating_setpoint_heat":23}'
```

**Actions on turn off**

```yaml
action: mqtt.publish
data:
  topic: zigbee2mqtt/Hive Receiver/set
  payload: '{"system_mode_heat":"heat","temperature_setpoint_hold_heat":1,"occupied_heating_setpoint_heat":7}'
```

The 23°C and 7°C values are the failsafe: the receiver regulates to them itself using the Mini's reading, so if HA stops mid-cycle the house can't overheat or freeze.

## Step 5: The Home Heating thermostat

`climate.home_heating` is the thermostat everything else talks to. It reads the average house temperature and flips the Boiler switch.

**5a. Average temperature** — Helpers → Create helper → **Combine the state of several sensors** (min/max), name `Heating Average Temperature`, type **Mean**, 1 decimal place:

- `sensor.living_room_living_room_temperature_sensor_temperature`
- `sensor.bedroom_bedroom_temperature_sensor_temperature`
- `sensor.timmerflotte_temp_hmd_sensor_temperature` (office)
- `sensor.hive_receiver_local_temperature_heat` (hall, from the Mini)

**5b. Thermostat** — Helpers → Create helper → **Generic thermostat**:

| Setting | Value |
| --- | --- |
| Name | Home Heating |
| Heater | `switch.boiler` |
| Temperature sensor | `sensor.heating_average_temperature` |
| Cold / hot tolerance | 0.3°C / 0.3°C |
| Minimum cycle duration | 5 minutes |
| Min / max temperature | 7°C / 25°C |
| Presets (°C) | Home 20, Comfort 21, Eco 18, Sleep 17, Away 16 |

The tolerances and 5-minute minimum stop the boiler short-cycling. Unlike the receiver, this entity accepts normal climate commands, so any scheduler or blueprint works with it.

## Step 6: Schedule with Climate Scheduler

Climate Scheduler gives a Hive-style schedule as a graph you drag. It must schedule `climate.home_heating`, never the receiver.

1. HACS → install **Climate Scheduler** (kneave) → restart HA.
2. Settings → Devices & services → Add integration → Climate Scheduler. It asks nothing; it just creates two summary sensors (coldest / warmest entity).
3. Hard-refresh the browser, then add the **Climate Scheduler card** to a dashboard (already on the Heating view).
4. In the card, tick **Home Heating** so it moves to Active. Leave Hive Receiver Heat and the old Hive Thermostat unticked.
5. Click Home Heating, pick weekday/weekend, drag the points.

Starter schedule:

| Time | Weekdays | Weekends |
| --- | --- | --- |
| 06:30–08:30 | 20°C | 17°C |
| 08:30–17:00 | 17°C | 20°C from 08:30 |
| 17:00–22:30 | 21°C | 21°C |
| 22:30–06:30 | 17°C | 17°C |

Tip for filming: turn on the card's **Panel** mode for a full-screen graph.

## Step 7: On / Off / Boost and the Heating dashboard

Three scripts give one-tap control; all target `climate.home_heating`.

**Heating On** (`script.heating_on`) — keeps the current target or schedule:

```yaml
alias: Heating On
icon: mdi:radiator
sequence:
  - action: climate.set_hvac_mode
    target:
      entity_id: climate.home_heating
    data:
      hvac_mode: heat
```

**Heating Off** (`script.heating_off`) — receiver drops to 7°C frost protection:

```yaml
alias: Heating Off
icon: mdi:radiator-off
sequence:
  - action: climate.set_hvac_mode
    target:
      entity_id: climate.home_heating
    data:
      hvac_mode: "off"
```

**I'm Cold** (`script.heating_boost_1_hour`) — 22°C for an hour, then back to what it was doing:

```yaml
alias: I'm Cold
icon: mdi:fire
mode: restart
sequence:
  - variables:
      prev_mode: "{{ states('climate.home_heating') }}"
      prev_temp: "{{ state_attr('climate.home_heating', 'temperature') | float(18) }}"
  - action: climate.set_temperature
    target:
      entity_id: climate.home_heating
    data:
      hvac_mode: heat
      temperature: 22
  - delay:
      hours: 1
  - action: climate.set_temperature
    target:
      entity_id: climate.home_heating
    data:
      hvac_mode: "{{ prev_mode if prev_mode in ['heat', 'off'] else 'heat' }}"
      temperature: "{{ prev_temp }}"
```

**Heating view** (main dashboard, second tab, sections layout):

- Badges: outside temperature, Home Heating state, Boiler on/off.
- Central Heating: thermostat dial for Home Heating with preset buttons (Home, Comfort, Eco, Sleep, Away), plus On / Off / Boost 1h tiles.
- Schedule: the Climate Scheduler card, full width.
- Rooms: a tile per room (living room, kitchen, office, extension, bathroom, hall) with a 24-hour trend line.
- Last 24 hours: one history graph of every room.

## Step 8: Testing and the failsafe demo

Test every path on camera before calling it done.

- [ ] Set Home Heating above the average temperature → Boiler badge turns on, receiver LED reacts, boiler fires.
- [ ] Drop the target below the average → Boiler turns off within the 0.3°C tolerance.
- [ ] Tap **Boost 1h** → target jumps to 22°C; check it restores an hour later.
- [ ] Tap **Off** → receiver shows heat at 7°C (frost protection).
- [ ] Move a point on the Climate Scheduler graph to now → target follows.
- [ ] Toggle hot water from the `climate.hive_receiver_water` entity and listen for the valve.
- [ ] **Failsafe A:** heating on, stop HA → receiver keeps heating but caps at 23°C.
- [ ] **Failsafe B:** heating off, stop HA → house holds at 7°C.
- [ ] Press the flame button on the receiver → manual override still works without HA.

## Clean-up and the TRV follow-up

Once the new setup has run for a few days, retire the Hive cloud:

- [ ] Remove the old `climate.thermostat` card from the Private view.
- [ ] Delete the Hive integration in HA.
- [ ] Unplug and retire the Hive Hub 360 (keep it until you're confident — it's the rollback path).

Rollback if anything goes wrong: factory-reset the receiver and pair it back to the Hive hub.

**Next video — smart valves:** SONOFF TRV Gen2 (Z2M 2.12.1+) on the rooms you use. Each TRV calls for heat; HA fires the Boiler switch when any room asks. Leave one radiator without a TRV as the bypass. Advanced Heating Control (blueprint) or Better Thermostat fit there, not in this video.

## Sources

- [Zigbee2MQTT: Hive SLR2 receiver](https://www.zigbee2mqtt.io/devices/SLR2.html)
- [Zigbee2MQTT: Hive SLT6 thermostat](https://www.zigbee2mqtt.io/devices/SLT6.html)
- [Zigbee2MQTT issue #27328: SLT6 stuck searching](https://github.com/Koenkk/zigbee2mqtt/issues/27328)
- [Climate Scheduler (kneave)](https://github.com/kneave/climate-scheduler)
- [Zigbee2MQTT: SONOFF TRV-ZBT](https://www.zigbee2mqtt.io/devices/TRV-ZBT.html)
- [Advanced Heating Control blueprint](https://community.home-assistant.io/t/advanced-heating-control/469873)
