# 🛁 Hot Tub Chemistry Manager for Home Assistant

A complete Home Assistant dashboard and automation package for managing hot tub water chemistry — dosing suggestions, adaptive learning, history graphs, and notifications. Built for a 1200-liter hot tub using a cheap WiFi water quality sensor instead of expensive proprietary solutions like Ondilo ICO or Blue Connect.

![Home Assistant](https://img.shields.io/badge/Home%20Assistant-2026.4%2B-blue)
![License](https://img.shields.io/badge/License-MIT-green)

---

## 📸 Screenshots

**Home Assistant Dashboard**

<img width="843" height="900" alt="Home Assistant Dashboard" src="https://github.com/user-attachments/assets/138c13c0-e16e-4739-9cf0-6ed308ac4520" />

**Smartlife / Tuya App**

<img width="419" height="918" alt="Smartlife Tuya App" src="https://github.com/user-attachments/assets/ea99f78b-471d-4717-bbc7-da6cbaefa7fb" />

**GIDIGI Device**

<img width="470" height="531" alt="GIDIGI Water Quality Sensor" src="https://github.com/user-attachments/assets/e132d48d-3001-4c86-bec8-9a0218ab8a62" />

---

## ✨ Features

| Feature | Description |
|---|---|
| 📊 Live sensor readings | pH, ORP, free chlorine estimate, temperature, TDS, EC / salinity |
| 💊 Dosing suggestions | Calculated for your exact water volume and chemical products |
| 🧠 Adaptive learning | The system remembers how your tub reacts and adjusts suggestions over time |
| 📈 History graphs | 24-hour graphs for all sensor values |
| 🔔 Notifications | pH and ORP alerts with smart suppression after chemical additions |
| ✏️ Dose logging | Every chemical addition saved as a persistent notification with full context |

---

## 🛒 Hardware

| Item | Details |
|---|---|
| Sensor | GIDIGI 8-in-1 WiFi Water Quality Monitor (pH, ORP, TDS, EC, salinity, temperature) |
| Integration | Tuya / Smart Life → Home Assistant via Tuya integration |
| Hot tub volume | 1200 liters |

> 💡 Any WiFi water quality sensor that exposes pH and ORP to Home Assistant will work. Just adjust the entity IDs in the configuration files.

---

## 🧪 Chemicals

| Chemical | Product | Active content |
|---|---|---|
| Chlorine | HTH Shock (calcium hypochlorite) | 75% |
| pH reducer | BAYZID® pH Minus liquid | 14.9% |

> The dosing formulas are calibrated for these products and 1200 L. See the **Adaptation** section below to adjust for your setup.

---

## 📋 Requirements

- Home Assistant 2026.4 or newer
- A water quality sensor providing pH and ORP values to HA
- Extended or Local Tuya integration — with the standard Tuya integration you only get TDS and temperature from the GIDIGI device
- The following HA integrations enabled: `input_number`, `input_boolean`, `template`

---

## 📁 File structure

```
whirlpool-chemistry-ha/
├── README.md
├── LICENSE
├── input_number.yaml       # Input numbers (dose entries, learning factors)
├── input_boolean.yaml      # Input booleans (confirm dose, enable learning)
├── sensor_template.yaml    # Template sensors (calculations, suggestions, status)
├── automations.yaml        # All automations (alerts, logging, learning)
└── dashboard.yaml          # Lovelace dashboard YAML
```

---

## 🚀 Installation

### Step 1 — Sensor entity IDs

Check your sensor entity IDs in HA: *Developer Tools → States → search for your sensor name.*

This package uses the following entity IDs (replace with yours throughout all files):

| Sensor | Entity ID in this package |
|---|---|
| pH | `sensor.wpsensor_ph` |
| ORP | `sensor.wpsensor_orp` |
| Temperature | `sensor.wpsensor_temperature` |
| TDS | `sensor.wpsensor_tds` |
| EC / Salinity | `sensor.wpsensor_salinity` |

### Step 2 — Add input_number entries

Add the contents of `input_number.yaml` to your existing `input_number:` block in `configuration.yaml`, or include it via:

```yaml
input_number: !include input_number.yaml
```

### Step 3 — Add input_boolean entries

Same as above for `input_boolean.yaml`:

```yaml
input_boolean: !include input_boolean.yaml
```

### Step 4 — Add template sensors

Add the contents of `sensor_template.yaml` to your template configuration. If you use a separate file:

```yaml
template: !include_dir_merge_list templates/
```

Or add directly to `configuration.yaml` under `template:`.

### Step 5 — Add automations

Append the contents of `automations.yaml` to your `automations.yaml` file.

### Step 6 — Restart Home Assistant

*Settings → System → Restart*

After restart, verify all entities are available:
*Developer Tools → States → search for* `whirlpool`

### Step 7 — Create the dashboard

Go to *Settings → Dashboards → Add Dashboard*

Name: `Hot Tub Chemistry` | Icon: `mdi:hot-tub`

Open the dashboard → top right ⋮ → Edit → Raw configuration editor → paste the contents of `dashboard.yaml` → Save.

---

## ⚙️ Adapting to your setup

### Different water volume

In `sensor_template.yaml`, find and update these values:

```yaml
{# Chlorine: HTH 75%: 2.4g raises 1200L by 1 mg/L #}
{% set g_per_mgl = 2.4 %}

{# pH-: BAYZID 14.9%: 18ml lowers 1200L by 0.1 pH #}
{% set ml_per_0_1_ph = 18 %}
```

Formula for chlorine (g per 1 mg/L increase):

```
g_per_mgl = volume_liters / 1000 × (1 / active_chlorine_fraction) × 1.5
```

Example — 1500 L with HTH 75%: 1.5 × (1 / 0.75) × 1.5 = **3.0 g per mg/L**

Formula for pH minus (ml per 0.1 pH reduction):

```
ml_per_0_1_ph = volume_liters × 0.015
```

Example — 1500 L: 1500 × 0.015 = **22.5 ml per 0.1 pH**

### Different target values

In `sensor_template.yaml`:

```yaml
{% set fc_target = 3.0 %}   # Free chlorine target (mg/L) – hot tubs need more than pools
{% set ph_target = 7.3 %}   # pH target
```

In `automations.yaml`, adjust alert thresholds as needed.

---

## 🧠 How adaptive learning works

Every time you confirm a dose addition, the system compares what it suggested vs. what you actually added, then adjusts the learning factor using exponential smoothing:

```
new_factor = old_factor × 0.7 + (actual / suggested) × 0.3
```

| Factor | Meaning |
|---|---|
| = 1.0 | Formulas are well calibrated |
| < 1.0 | System was overestimating (you add less than suggested) |
| > 1.0 | System was underestimating (you add more than suggested) |

Range is clamped between 0.5 and 2.0.

To reset: set `input_number.whirlpool_learning_factor_chlorine` and `input_number.whirlpool_learning_factor_ph` back to `1.0`.

---

## 🔔 Notifications

| Trigger | Level | Action |
|---|---|---|
| pH > 7.6 for 10 min | 🟡 Warning | Notify with pH- suggestion |
| pH > 7.9 for 5 min | 🔴 Critical | Critical notification |
| ORP < 620 mV for 15 min | 🟡 Warning | Notify with chlorine suggestion |
| ORP < 500 mV for 10 min | 🔴 Critical | Shock chlorination recommended |
| Daily at 08:00 | ℹ️ Info | Morning status report |

> ⚠️ pH alerts are suppressed for **60 minutes** after a dose is confirmed — calcium hypochlorite temporarily raises pH when added.

---

## 💊 Dosing quick reference (1200 L)

### HTH Shock 75% chlorine

- Raise FC by 1 mg/L = **2.4 g**
- Shock chlorination (10 mg/L) = **24 g**
- Always pre-dissolve in a bucket of water before adding

### BAYZID® pH Minus 14.9% liquid

- Lower pH by 0.1 = **~18 ml**
- Maximum single dose: 150 ml → wait 30–45 min, then re-test
- Always add with jets running

> ⚠️ **Correct pH first, then add chlorine!**
> At pH > 7.6, chlorine effectiveness drops to only 10–30%!

### Wait times after dosing

- After chlorine: 15 min with pump running, then test
- After pH minus: 30–45 min, then test
- After shock chlorination: wait until ORP > 650 mV

---

## 📊 Free chlorine estimation

Your ORP sensor does not directly measure free chlorine (FC). This package uses a linear approximation:

```
FC (mg/L) ≈ (ORP - 600) / 75
```

> This is a rough estimate. For accurate FC readings, use test strips or a drop test kit and enter the value manually in the dashboard under "Free chlorine (manual)".

---

## 🤝 Contributing

Pull requests and issues welcome! If you adapt this for a different sensor or chemical product, feel free to share your configuration.

---

## 📄 License

MIT License — see `LICENSE` file.

---

## 🙏 Credits

Built with the help of Claude (Anthropic) 🤖
