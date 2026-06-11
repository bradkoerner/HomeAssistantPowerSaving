# ComEd Hourly Pricing - Energy Automations for Home Assistant

Home Assistant automations for managing home energy load based on [ComEd hourly electricity pricing](https://www.comed.com/MyAccount/MyBills/Pages/CurrentHourlyPricing.aspx). Pricing is evaluated relative to a rolling mean rather than fixed thresholds, so the system adapts automatically to seasonal price changes without needing manual tuning.

---

## Automations

| File | Description |
|---|---|
| `automations/ac_mode_manager.yaml` | Multi-zone AC - sets price tier + time window helpers, calls Apply AC Settings script |
| `automations/ev_charging.yaml` | EV charger - four SOC-based charging tiers relative to rolling mean |
| `automations/dehumidifier.yaml` | Dehumidifier - runs only when price < 80% of mean and windows are closed |
| `automations/radiator.yaml` | Oil radiator - runs when price is zero or negative and it's cold outside |
| `scripts/apply_ac_settings.yaml` | AC zone implementation - called by AC Mode Manager |
| `packages/comed_ac_helpers.yaml` | All required helpers |

These are reference YAMLs intended to be adapted to your setup. Entity IDs are hardcoded to the author's devices - update them to match your own before use. Remove zones you don't have, add ones you do.

---

## How the Price Logic Works

All automations (except the radiator) use a **rolling mean** of the ComEd price sensor as a baseline rather than fixed cent thresholds. This means:

- No manual tuning as seasons change
- A single anomalous hour doesn't move the mean meaningfully
- The system naturally tightens restrictions as average prices rise in summer

The rolling mean is a [Statistics helper](https://www.home-assistant.io/integrations/statistics/) configured over a 30–60 day window.

The oil radiator uses an absolute threshold (≤ 0¢) because free/negative pricing is an absolute condition - it doesn't need context.

---

## AC Mode Manager

The centerpiece automation. Runs on every ComEd price change and at scheduled time transitions. Sets two helpers that the script reads:

- `input_select.ac_price_tier` - `normal` / `high` / `extreme`
- `input_select.ac_time_window` - `awake` / `night`

Then calls `script.apply_ac_settings` to implement zone-level logic.

### Price Tiers

| Tier | Threshold |
|---|---|
| normal | ≤ 120% of rolling mean |
| high | 120–150% of rolling mean |
| extreme | > 150% of rolling mean |

### Override Modes

`input_select.ac_override_mode` has three options that bypass normal price logic:

| Option | Description |
|---|---|
| `none` | Normal price-aware operation |
| `comfort` | Full comfort regardless of price - resets to `none` at midnight |
| `away` | Power saving for multi-day absences - must be turned off manually |

### Living Room AC Decision Table

| Time Window | Override | Price Tier | Temp | Fan | Preset |
|---|---|---|---|---|---|
| Awake | comfort | any | 72° | high | none |
| Awake | away | any | 78° | high | eco |
| Awake | none | extreme | 78° | high | eco |
| Awake | none | high | 74° | high | eco |
| Awake | none | normal | 72° | high | none |
| Night | away | any | 82° | low | eco |
| Night | none | any | 82° | low | eco |
| Night | comfort | any | 72° | high | none |

> **Note on eco mode + action ordering:** `set_preset_mode` must be called *before* `set_temperature`. The LG unit overrides the setpoint with its own eco default if eco is activated after a temperature is set.

### Primary Bedroom AC

| Override | Time Window | State |
|---|---|---|
| comfort | any | on |
| away | night | on |
| away | awake | off |
| none | any | on (except extreme pricing 5:30am–8pm) |

### Bedroom 3 AC

| Override | State |
|---|---|
| away | off |
| none/comfort | on 6pm–7am daily; on weekends 11:30am–3:30pm |

No price logic - serves as kitchen buffer zone (kitchen has no AC).

---

## EV Smart Charging

Four SOC-based tiers - the lower the battery, the more aggressively it charges:

| SOC | Charges when |
|---|---|
| < 30% | Always (emergency threshold) |
| 30–50% | Price < 90% of rolling mean |
| 50–70% | Price < 70% of rolling mean |
| > 70% | Price < 50% of rolling mean |

All SOC tiers include a staleness check - if the battery sensor hasn't updated in 30 minutes, that tier is skipped to avoid acting on stale data. `input_boolean.ev_charge_override` forces charging regardless of price.

---

## Dehumidifier

Runs when price is below 80% of the rolling mean **and** windows are closed. Triggers on both price changes and window state changes so it responds immediately to either condition.

---

## Oil Radiator

Turns on when ComEd price drops to zero or negative cents and outside temperature is below 60°F. Takes advantage of free/negative pricing to add passive heat rather than letting the grid pay you to do nothing.

---

## Requirements

### Integrations
- [ComEd Hourly Pricing](https://www.home-assistant.io/integrations/opower/) integration or equivalent providing a current hour average price sensor
- A [Statistics helper](https://www.home-assistant.io/integrations/statistics/) configured as a rolling mean of the price sensor (30–60 day window recommended)

### Entities

| Entity | Type | Description |
|---|---|---|
| `sensor.comed_current_hour_average_price` | sensor | Current hour price in cents |
| `sensor.comed_price_mean` | sensor | Statistics helper - rolling mean |
| `climate.dining_room_living_room_air_conditioner` | climate | LG smart AC unit (living room) |
| `switch.primary_bedroom_a_c` | switch | Smart plug, primary bedroom window AC |
| `switch.bedroom_3_a_c` | switch | Smart plug, bedroom 3 window AC |
| `switch.car_charger` | switch | EV charger |
| `sensor.2023_ioniq_5_ev_battery_level` | sensor | EV battery SOC % |
| `switch.dehumidifier_plug` | switch | Smart plug, dehumidifier |
| `switch.radiator_outlet` | switch | Smart plug, oil radiator |
| `weather.forecast_home` | weather | Outside temperature source |

### Helpers

| Helper | Type | Options / Notes |
|---|---|---|
| `input_select.ac_price_tier` | Dropdown | normal, high, extreme |
| `input_select.ac_time_window` | Dropdown | awake, night |
| `input_select.ac_override_mode` | Dropdown | none, comfort, away |
| `input_boolean.windows_open` | Toggle | On when windows are open |
| `input_boolean.ev_charge_override` | Toggle | Forces EV charging regardless of price |

---

## Installation

1. **Create helpers** - via Settings → Helpers, or drop `packages/comed_ac_helpers.yaml` into your `packages/` directory (requires packages enabled in `configuration.yaml`) and restart HA
2. **Create the Statistics helper** for the rolling price mean - source: ComEd price sensor, stat type: mean, max age: 720–1440 hours, sampling size: 5000
3. **Update entity IDs** in all YAML files to match your devices
4. **Paste the script** - Settings → Automations & Scenes → Scripts → Add Script → Edit in YAML, paste `scripts/apply_ac_settings.yaml`
5. **Paste each automation** - Settings → Automations & Scenes → Automations → Add Automation → Edit in YAML

---

## File Structure

```
ha-comed-energy-automations/
  README.md
  automations/
    ac_mode_manager.yaml      ← AC mode + time window logic, calls script
    ev_charging.yaml          ← EV SOC-based charging tiers
    dehumidifier.yaml         ← Dehumidifier price control
    radiator.yaml             ← Free/negative price radiator
  scripts/
    apply_ac_settings.yaml    ← AC zone implementation (called by ac_mode_manager)
  packages/
    comed_ac_helpers.yaml     ← All required helpers
```