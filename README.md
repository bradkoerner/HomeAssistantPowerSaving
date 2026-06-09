# ComEd Hourly Pricing — Energy Automations for Home Assistant

Home Assistant automations for managing home energy load based on [ComEd hourly electricity pricing](https://www.comed.com/MyAccount/MyBills/Pages/CurrentHourlyPricing.aspx). Pricing is evaluated relative to a rolling mean rather than fixed thresholds, so the system adapts automatically to seasonal price changes without needing manual tuning.

---

## Automations

| File | Description |
|---|---|
| `comed_ac_mode_manager.yaml` | Multi-zone AC — sets price tier + time window helpers, calls Apply AC Settings script |
| `comed_ev_charging.yaml` | EV charger — four SOC-based charging tiers relative to rolling mean |
| `comed_dehumidifier.yaml` | Dehumidifier — runs only when price < 80% of mean and windows are closed |
| `comed_radiator.yaml` | Oil radiator — runs when price is zero or negative and it's cold outside |

---

## How the Price Logic Works

All automations (except the radiator) use a **rolling mean** of the ComEd price sensor as a baseline rather than fixed cent thresholds. This means:

- No manual tuning as seasons change
- A single anomalous hour doesn't move the mean meaningfully
- The system naturally tightens restrictions as average prices rise in summer

The rolling mean is a [Statistics helper](https://www.home-assistant.io/integrations/statistics/) configured over a 30–60 day window.

The oil radiator uses an absolute threshold (≤ 0¢) because free/negative pricing is an absolute condition — it doesn't need context.

---

## AC Mode Manager

The centerpiece automation. Runs on every ComEd price change and at scheduled time transitions. Sets two helpers that the script reads:

- `input_select.ac_price_tier` — `normal` / `high` / `extreme`
- `input_select.ac_time_window` — `awake` / `night`

Then calls `script.apply_ac_settings` to implement zone-level logic.

### Price Tiers

| Tier | Threshold |
|---|---|
| normal | ≤ 120% of rolling mean |
| high | 120–150% of rolling mean |
| extreme | > 150% of rolling mean |

### Living Room AC (LG Smart Unit — full climate control)

| Time Window | Price Tier | Preset | Temp | Fan |
|---|---|---|---|---|
| Night | any | eco | 82° | low |
| Awake | extreme | eco | 78° | high |
| Awake | high | eco | 74° | high |
| Awake | normal | none | 72° | high |

Eco mode is used during high/extreme pricing and at night — the LG unit cuts the compressor when the setpoint is reached and polls every 3 minutes to check if more cooling is needed, reducing energy draw without sacrificing comfort entirely.

> **Note on action ordering:** `set_preset_mode` must be called *before* `set_temperature`. Setting eco after temperature causes the unit to override the setpoint with its own eco default.

### Primary Bedroom AC (dumb unit on smart plug)

- On from 8pm daily
- Off during extreme pricing between 5:30am–8pm
- Always on when comfort override is active

### Bedroom 3 AC (dumb unit on smart plug)

- On 6pm–7am daily
- On weekends 11:30am–3:30pm
- No price logic (serves as kitchen buffer zone — kitchen has no AC)

### Comfort Override

`input_boolean.ac_comfort_override` bypasses all price logic for living room and primary bedroom:
- Living room runs at 72° / high fan / no eco
- Primary bedroom stays on regardless of price
- Resets automatically at midnight

---

## EV Smart Charging

Four SOC-based tiers — the lower the battery, the more aggressively it charges:

| SOC | Charges when |
|---|---|
| < 30% | Always (emergency threshold) |
| 30–50% | Price < 90% of rolling mean |
| 50–70% | Price < 70% of rolling mean |
| > 70% | Price < 50% of rolling mean |

All SOC tiers include a staleness check — if the battery sensor hasn't updated in 30 minutes, that tier is skipped to avoid acting on stale data. `input_boolean.ev_charge_override` forces charging regardless of price.

---

## Dehumidifier

Runs when price is below 80% of the rolling mean **and** windows are closed. Triggers on both price changes and window state changes so it responds immediately to either condition.

---

## Oil Radiator

Turns on when ComEd price drops to zero or negative cents and outside temperature is below a configurable threshold (default 60°F). Takes advantage of free/negative pricing to add passive heat rather than letting the grid pay you to do nothing.

---

## Requirements

### Integrations
- [ComEd Hourly Pricing](https://www.home-assistant.io/integrations/opower/) integration or equivalent providing a current hour average price sensor
- A [Statistics helper](https://www.home-assistant.io/integrations/statistics/) configured as a rolling mean of the price sensor (30–60 day window recommended)

### Entities (AC automations)

| Entity | Type | Description |
|---|---|---|
| `sensor.comed_current_hour_average_price` | sensor | Current hour price in cents |
| `sensor.comed_price_mean` | sensor | Statistics helper — rolling mean |
| `climate.dining_room_living_room_air_conditioner` | climate | LG smart AC unit |
| `switch.primary_bedroom_a_c` | switch | Smart plug, primary bedroom window AC |
| `switch.bedroom_3_a_c` | switch | Smart plug, bedroom 3 window AC |

Entity IDs in the script are hardcoded. Update them to match your setup before installing.

---

## Installation

### Step 1 — Helpers

**Option A: Package file (recommended)**

If packages are enabled in `configuration.yaml`:
```yaml
homeassistant:
  packages: !include_dir_named packages
```
Copy `packages/comed_ac_helpers.yaml` into your `packages/` directory and restart HA.

**Option B: Manual**

Create each helper via Settings → Helpers:

| Helper | Type | Options |
|---|---|---|
| `input_select.ac_price_tier` | Dropdown | normal, high, extreme |
| `input_select.ac_time_window` | Dropdown | awake, night |
| `input_boolean.ac_comfort_override` | Toggle | — |
| `input_boolean.windows_open` | Toggle | — |
| `input_boolean.ev_charge_override` | Toggle | — |

### Step 2 — Statistics Helper

Create a Statistics helper for the rolling price mean:
- **Source:** your ComEd price sensor
- **Stat type:** mean
- **Max age:** 720–1440 hours (30–60 days)
- **Sampling size:** 5000

### Step 3 — Apply AC Settings Script

Go to Settings → Automations & Scenes → Scripts → Add Script → Edit in YAML. Paste the contents of `scripts/apply_ac_settings.yaml`. Update entity IDs to match your setup before saving.

### Step 4 — Import Blueprints

Import whichever automations apply to your setup:

**AC Mode Manager**
[![Import Blueprint](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https://github.com/YOUR_USERNAME/ha-comed-energy-automations/blob/main/blueprints/automation/comed_ac_mode_manager.yaml)

**EV Smart Charging**
[![Import Blueprint](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https://github.com/YOUR_USERNAME/ha-comed-energy-automations/blob/main/blueprints/automation/comed_ev_charging.yaml)

**Dehumidifier**
[![Import Blueprint](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https://github.com/YOUR_USERNAME/ha-comed-energy-automations/blob/main/blueprints/automation/comed_dehumidifier.yaml)

**Oil Radiator**
[![Import Blueprint](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https://github.com/YOUR_USERNAME/ha-comed-energy-automations/blob/main/blueprints/automation/comed_radiator.yaml)

Or manually: Settings → Automations & Scenes → Blueprints → Import Blueprint, paste the raw GitHub URL of the blueprint file.

---

## File Structure

```
ha-comed-energy-automations/
  README.md
  blueprints/
    automation/
      comed_ac_mode_manager.yaml    ← AC mode + time window logic
      comed_ev_charging.yaml        ← EV SOC-based charging tiers
      comed_dehumidifier.yaml       ← Dehumidifier price control
      comed_radiator.yaml           ← Free/negative price radiator
  scripts/
    apply_ac_settings.yaml          ← AC zone implementation (paste into Scripts editor)
  packages/
    comed_ac_helpers.yaml           ← All required helpers (drop into packages/ directory)
```
