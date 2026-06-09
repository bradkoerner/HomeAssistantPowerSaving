# ComEd Hourly Pricing — AC Energy Automations

Home Assistant automations for managing a multi-zone AC system based on [ComEd hourly electricity pricing](https://www.comed.com/MyAccount/MyBills/Pages/CurrentHourlyPricing.aspx). Pricing is evaluated relative to a rolling mean rather than fixed thresholds, so the system adapts automatically to seasonal price changes.

---

## How It Works

A single automation (`AC Mode Manager`) runs on every ComEd price change and at scheduled time transitions. It:

1. Sets a **price tier** helper (`normal` / `high` / `extreme`) based on the current price vs. the rolling mean
2. Sets a **time window** helper (`awake` / `night`) based on time of day
3. Calls the **Apply AC Settings** script, which reads those helpers and applies the appropriate settings to each AC zone

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

Eco mode is used during high/extreme pricing and at night — the LG unit cuts the compressor when the setpoint is reached and polls every 3 minutes to check if more cooling is needed, reducing energy draw.

**Note on action ordering:** `set_preset_mode` must be called *before* `set_temperature`. Setting temperature after eco mode is activated causes the unit to correctly hold the target temp. Setting eco *after* temperature causes the unit to override the setpoint with its own eco default.

### Primary Bedroom AC (dumb unit on smart plug)

- On from 8pm daily
- Off during extreme pricing between 5:30am–8pm
- Always on when comfort override is active

### Bedroom 3 AC (dumb unit on smart plug)

- On 6pm–7am daily
- On weekends 11:30am–3:30pm
- No price logic (serves as kitchen buffer zone)

### Comfort Override

`input_boolean.ac_comfort_override` bypasses all price logic:
- Living room runs at 72° / high fan / no eco
- Primary bedroom stays on regardless of price
- Resets automatically at midnight

---

## Requirements

### Integrations
- [ComEd Hourly Pricing](https://www.home-assistant.io/integrations/opower/) or equivalent integration providing a current hour average price sensor
- A [Statistics helper](https://www.home-assistant.io/integrations/statistics/) configured as a rolling mean of the price sensor (30–60 day window recommended)

### Entities
| Entity | Type | Description |
|---|---|---|
| `sensor.comed_current_hour_average_price` | sensor | Current hour price in cents |
| `sensor.comed_price_mean` | sensor | Statistics helper — rolling mean |
| `climate.dining_room_living_room_air_conditioner` | climate | LG smart AC unit |
| `switch.primary_bedroom_a_c` | switch | Smart plug, primary bedroom window AC |
| `switch.bedroom_3_a_c` | switch | Smart plug, bedroom 3 window AC |

Entity IDs in the script are hardcoded to the above. Update them to match your own before installing.

---

## Installation

### Step 1 — Helpers

**Option A: Package file (recommended)**

If you have packages enabled in `configuration.yaml`:
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

### Step 3 — Script

Go to Settings → Automations & Scenes → Scripts → Add Script → Edit in YAML. Paste the contents of `scripts/apply_ac_settings.yaml`.

Update entity IDs to match your setup before saving.

### Step 4 — Automation Blueprint

[![Import Blueprint](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https://github.com/YOUR_USERNAME/ha-comed-energy-automations/blob/main/blueprints/automation/comed_ac_mode_manager.yaml)

Or manually: Settings → Automations & Scenes → Blueprints → Import Blueprint, paste the raw URL of `blueprints/automation/comed_ac_mode_manager.yaml`.

When creating the automation from the blueprint, select your price sensor and rolling mean sensor.

---

## Statistics Helper Setup Detail

The rolling mean sensor is what makes thresholds adaptive. A 30–60 day window captures enough history to reflect seasonal pricing without being skewed by short spikes. In practice this means:

- Summer peak pricing (~6–8¢/hr) won't trigger extreme tier during mild spring conditions
- A single anomalous hour doesn't move the mean meaningfully
- The system naturally tightens restrictions as average prices rise seasonally

---

## File Structure

```
ha-comed-energy-automations/
  README.md
  blueprints/
    automation/
      comed_ac_mode_manager.yaml    ← import this as a blueprint
  scripts/
    apply_ac_settings.yaml          ← paste into Scripts editor
  packages/
    comed_ac_helpers.yaml           ← drop into packages/ directory
```
