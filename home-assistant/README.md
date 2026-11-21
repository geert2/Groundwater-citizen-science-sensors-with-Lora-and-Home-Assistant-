# Home Assistant

* `dashboard-card.yaml` - Lovelace card for Well 1 and Well 2 (depth and groundwater-temperature gauges, air temperature and humidity, node health, 7-day graphs).

The **gateway** calculates *Water Table Depth Below Ground* itself and exposes an editable **Sensor Depth Below Ground** number per well. If you re-lower a probe, change that number in Home Assistant (Settings > Devices > Groundwater Gateway); no reflashing is needed.

Entities per well:

| Entity | Meaning |
| :-- | :-- |
| `Water Table Depth Below Ground` | metres below ground level (calculated on the gateway) |
| `Water Column Height` | water above the TL-136 |
| `Groundwater Temperature` | DS18B20 at the probe |
| `Air Temperature`, `Air Humidity` | BME280 in the radiation shield next to the well |
| `Battery Voltage` | node battery (diagnostic) |
| `Node Online` | false after 2.5 h without a packet |
| `Water Below Sensor` | water level at or below the probe |

Useful automations: "Node Online turns off for 10 minutes", "Battery Voltage below 3.5 V".
