# FoxESS Export Limit by Electricity Price — Home Assistant Blueprint

A [Home Assistant](https://www.home-assistant.io/) blueprint that automatically controls the export power limit on a **FoxESS H3 Smart** inverter based on the current electricity spot price — via the [foxess_modbus](https://github.com/nathanmarlor/foxess_modbus) integration.

[![Import Blueprint](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2FPlanckus%2Fha-blueprint-foxess-export%2Fmain%2Fblueprints%2Ffoxess_export_by_price.yaml)

---

## What it does

Every 30 seconds the automation reads your electricity price sensor and writes the appropriate export limit directly to the inverter over Modbus:

| Spot price | Export limit |
|---|---|
| **Above threshold** (e.g. > 0.1 kr/kWh) | ✅ Full export — configurable max watts |
| **At or below threshold** | 🚫 Export blocked — 0 W |

The limit is enforced continuously, so it always reflects the current price even after a Home Assistant restart.

### Why this approach?

The blueprint writes the native Modbus **Export Power Limit** register (46616/46617) directly — the same register the FoxESS app uses. This means:

- No work mode changes
- No extra scripts, helpers, or input numbers needed
- One automation does everything
- Survives HA restarts (re-applies the correct limit on startup)

---

## Supported inverters

| Inverter | Register |
|---|---|
| **FoxESS H3 Smart** (H3-5.0-Smart, H3-6.0-Smart, H3-10.0-Smart, etc.) | 46616 / 46617 |

> Other FoxESS series (H1, KH, H3-Pro) use different registers and are not covered by this blueprint.

---

## Requirements

- [Home Assistant](https://www.home-assistant.io/)
- [foxess_modbus](https://github.com/nathanmarlor/foxess_modbus) integration installed via HACS (v1.15.0 or later)
- [Strømligning](https://github.com/MTrab/stromligning) integration installed via HACS — provides the electricity spot price sensor
- FoxESS H3 Smart inverter connected via Modbus TCP

---

## Installation

### Option 1 — One-click import (recommended)

[![Import Blueprint](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2FPlanckus%2Fha-blueprint-foxess-export%2Fmain%2Fblueprints%2Ffoxess_export_by_price.yaml)

### Option 2 — Manual import via URL

1. In Home Assistant, go to **Settings → Automations & Scenes → Blueprints**
2. Click **Import Blueprint** (bottom right)
3. Paste this URL:
   ```
   https://raw.githubusercontent.com/Planckus/ha-blueprint-foxess-export/main/blueprints/foxess_export_by_price.yaml
   ```
4. Click **Preview** → **Import Blueprint**

### Option 3 — Manual file install

1. Download [`foxess_export_by_price.yaml`](blueprints/foxess_export_by_price.yaml)
2. Place it in your HA config folder:
   ```
   config/blueprints/automation/custom/foxess_export_by_price.yaml
   ```
3. Restart Home Assistant or reload blueprints

---

## Configuration

Once imported, create a new automation from the blueprint:

1. Go to **Settings → Automations & Scenes → Blueprints**
2. Find **"FoxESS - Export Limit by Electricity Price"** and click **Create Automation**
3. Fill in the fields:

| Field | Example | Description |
|---|---|---|
| **Electricity Price Sensor** | `sensor.stromligning_spotprice_vat` | Any numeric price sensor |
| **Price Threshold** | `0.1` | Export allowed above this value (kr/kWh or your unit) |
| **FoxESS Inverter** | *(select from dropdown)* | Your inverter as registered by foxess_modbus |
| **Maximum Export Power** | `10000` | Watts to allow when price is high (match your inverter rating) |

4. Give the automation a name and click **Save**

---

## How the logic works

```
Every 30 seconds:
  if spot_price > threshold:
      write 10000 W  →  inverter register 46616/46617
  else:
      write 0 W      →  inverter register 46616/46617
```

The automation also fires once on every Home Assistant startup (after a short delay for sensors to settle), so the inverter is always in the correct state.

---

## Troubleshooting

**Export is still happening even though price is low**
→ Check the automation is enabled and that the inverter device is correctly selected. You can trigger it manually from **Developer Tools → Automations**.

**"Service foxess_modbus.write_registers not found"**
→ Make sure the foxess_modbus integration is installed via HACS and your inverter is connected and showing entities.

**Nothing changes at all**
→ Go to **Developer Tools → MQTT** and verify the Modbus connection is alive. Check the foxess_modbus integration logs under **Settings → System → Logs**.

**I want different thresholds for on and off (hysteresis)**
→ This blueprint uses a single threshold for simplicity. For hysteresis, create two automations using `numeric_state` triggers with different thresholds for the on/off transitions.

---

## Technical details

The FoxESS H3 Smart inverter exposes the export power limit as a signed 32-bit integer (I32) split across two 16-bit Modbus holding registers:

| Register | Description | Access |
|---|---|---|
| 46616 | Export Power Limit — high word (I32) | RW |
| 46617 | Export Power Limit — low word (I32) | RW |

For values below 65,536 W the high word is always 0. The blueprint handles the 32-bit split automatically.

Reference: *FoxESS H3 Smart Modbus Protocol V1.05.04.00*

---

## License

MIT — free to use, share, and modify.

---

*Made with ❤️ and [Claude](https://claude.ai)*
