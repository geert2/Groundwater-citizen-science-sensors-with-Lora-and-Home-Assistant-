# Wiring Notes (LITE): Depth Channel, Power, EC, BME280 and Pinout

*`[R#]` tags refer to [REFERENCES.md](REFERENCES.md).*

> Not yet tested on hardware. Build one node on the bench and check every voltage before connecting the transmitter in a well.

## Overview: whole node

![Field node wiring overview](img/node-wiring.svg)

*Red = switched 3.3 V (`Vext`), black = GND. Crossing wires are not connected unless a dot is drawn. Diagram sources are plain SVG files in `docs/img/`.*

## 1. Depth channel (TL-136, 4-20 mA loop)

![Depth channel with the ESP32 ADC](img/depth-frontend-esp.svg)


* Voltage across the shunt: 4 mA = 0.40 V (0 m), 20 mA = 2.00 V (5 m). This stays inside the ESP32-S3 ADC range, unlike 160 Ω (3.2 V).
* The **1 kΩ + zener** protect `GPIO5`. If the transmitter fails short onto the 18 V rail, the zener clamps the pin near 3.6 V and the 1 kΩ limits the current (about 15 mA). Without them an ESP32-S3 pin would see up to 18 V.
* Check the zener's leakage near 2 V (should be negligible). If the reading at 20 mA is more than about 10 mV low, use a TVS or a BAT54S Schottky clamp (one diode to `GND`, one to `3V3`) instead.
* The firmware converts the shunt voltage to current with `shunt_ohm` (default 100.0). Measure your actual resistor and set `shunt_ohm: "100.3"` (for example) in the node file.
* **Valid range:** 3.5 mA to 22 mA. Outside it the node reports *unavailable* (broken wire, short, boost fault).

### Accuracy note
The ESP32-S3 ADC reads up to about 3.1 V at 12 dB attenuation [R17] and its reference voltage varies between 1000 and 1200 mV from chip to chip [R16]. In practice that means tens of mV of error, roughly a few cm of water. The zero and span calibration in the field guide removes most of it. If you need better, the `main` branch has an ADS1115 option that uses the same shunt.

### Using a 160 Ω shunt instead
It works if your water column stays below about 4.8 m (the ESP32-S3 ADC saturates near 3.1 V [R17][R3], which is about 19.5 mA). Set `shunt_ohm: "160.0"` in the node file. Trade-offs: the top of the range is less linear, a fault current above about 19.5 mA reads as the maximum instead of "unavailable", and the 3.6 V zener sits close to the 3.2 V full-scale signal, so prefer a **BAT54S Schottky clamp** (one diode to `GND`, one to `3V3`). 

## 2. Power

```text
18650 --> Heltec battery connector --> 3.3 V regulator --> Vext switch (GPIO36 LOW = ON)
                                                              |
                          +-----------------------------------+--------------+--------------+
                          |                                   |              |              |
                  [220-470 uF] + MT3608 IN                 BME280       EC module      DS18B20 (+4.7 k pull-up to GPIO4)
                          |
                      MT3608 OUT = 18.0 V  --> TL-136 V+
```

* The 220-470 µF capacitor at the boost input is a precaution against the start-up current spike browning out the ESP32. The MT3608 has internal soft start [R20], so this is a cheap safeguard rather than a fix for a known fault.
* The TL-136 is listed with a 12 to 32 V supply [R21]; at 18 V a 160 Ω shunt (3.2 V at 20 mA) still leaves about 14.8 V. Check the label of your unit.
* Set the MT3608 to 18.0 V **before** connecting the TL-136, and re-check after transport (the trimmer drifts).
* `Vext` is switched on very early at boot (before the sensors are set up), because the BME280 and DS18B20 are detected once during start-up and must already have power.
* Active energy is dominated by the boost converter. Keep the `warmup` time as short as the transmitter allows (measure how fast the reading settles).
* **Optional improvement (not in the firmware):** switch the boost converter with its own MOSFET so the EC and temperature probes can be read while the 18 V loop is off. This removes possible stray current in the water and saves energy.

## 3. Pin summary

| Function | Heltec V3 pin | Notes |
| :--- | :--- | :--- |
| Vext enable | `GPIO36` | LOW = sensor rail ON |
| Depth signal | `GPIO5` (ADC1) | via 1 kΩ, zener, 100 nF |
| EC signal | `GPIO6` (ADC1) | analog output of the DFRobot EC/TDS module (0-2.3 V) |
| DS18B20 data | `GPIO4` | 4.7 kΩ to `Vext` |
| BME280 SDA / SCL | `GPIO41` / `GPIO42` | separate I²C bus from the onboard OLED, 50 kHz; powered from `Vext` |
| Battery ADC | `GPIO1` | divider 390 k / 100 k (factor 4.9) |
| Battery divider enable | `GPIO37` | V3 / V3.1: **LOW** connects the divider; **V3.2: HIGH** [R11][R13]. Set `vbat_ctrl_active_low` |
| Status LED | `GPIO35` | flashes once at every wake-up |
| LoRa SX1262 | `GPIO8/9/10/11/12/13/14` | onboard; `tcxo_voltage: 1_8V`, `rf_switch: true` |

Check the battery reading against a multimeter once. If it is off, adjust `vbat_scale` in the node file (default 4.9; a 390 k / 100 k divider [R12], while ESPHome's device page uses 2.0 [R10]).

### Board revision check (do this once per board)
The polarity of two pins differs between Heltec V3 revisions, and the firmware has two settings for it:

| Setting in the node file | V3.0 / V3.1 | V3.2 | Source |
| :-- | :-- | :-- | :-- |
| `vbat_ctrl_active_low` (GPIO37) | `"true"` | `"false"` (default) | [R11] [R13] |
| `vext_active_low` (GPIO36) | `"true"` (default) | **not confirmed**, check | [R10] [R11] [R14] |

1. Read the revision printed on the board (V3.2 is the current one).
2. Battery: with the battery connected and the node awake, *Battery Voltage* should match a multimeter within a few percent after adjusting `vbat_scale`. A reading near 0 V means the polarity is wrong (the firmware then ignores the reading and carries on measuring).
3. Vext: measure between the `Ve` pin and `GND` while the node is awake (it switches `Vext` on in early boot). About 3.3 V means the setting is right. If it reads 0 V, flip `vext_active_low`. Compare with the V3.2 schematic [R14] if in doubt.
4. A cleaner test: flash a minimal config that only toggles `GPIO36` and `GPIO37` from two buttons or switches, and watch the voltages.


## 4. Enclosure
* The TL-136 vent tube must reach **atmospheric pressure**. A sealed box warms in the sun and changes its internal pressure (1 hPa is about 1 cm of water). Fit a **membrane vent plug** to the enclosure and keep the vent tube open inside it, together with a desiccant packet.
* Never put the antenna's U.FL connector in contact with water; **never power the board without the antenna attached**.

## 5. EC sensor (DFRobot Gravity analog TDS/EC)
* The **module** (`+` / `-` / `A`) stays in the dry enclosure: `+` to `Vext`, `-` to `GND`, `A` to `GPIO6`. Only the two-wire **probe** goes into the well.
* The module runs on 3.3-5.5 V and outputs 0-2.3 V [R19], well inside the ADC range at 12 dB attenuation. It is powered from `Vext`, so it only excites the probe for the second or so of a reading. The module drives the probe with AC, which avoids electrolysis of the electrodes.
* Extend the probe cable with a **two-core twisted pair** (or screened cable); the probe signal is high-impedance and picks up noise. Waterproof the splice as in section 1 of the field guide.
* Hang the probe at the same depth as the DS18B20, which provides the temperature compensation (2 %/°C to 25 °C), and keep the electrodes **a few cm clear of the TL-136 body**.
* The firmware converts the voltage with DFRobot's cubic curve and then applies `ec_gain` / `ec_offset` from the two-point calibration (field guide, section 3.2). Readings are only meaningful up to about 2000 µS/cm.

## 6. BME280 (air temperature + humidity)
* Wire `VIN` to `Vext`, `GND` to `GND`, `SDA` to `GPIO41`, `SCL` to `GPIO42`. It is powered only during a reading, which also avoids self-heating.
* Mount it **outside the enclosure in a radiation shield** (a small louvred Stevenson-type shield, or a stack of white plant saucers), ideally 1.5 m above ground and on the shaded north side of the post. Without a shield, direct sun makes the air temperature read several degrees too high.
* Use a 4-core cable of **at most about 1 m** through its own cable gland. I²C is meant for short distances; the firmware runs the bus at 50 kHz to tolerate the extra cable capacitance. Most breakouts already have 10 kΩ pull-ups on SDA/SCL; if readings drop out with a longer cable, add 4.7 kΩ pull-ups from SDA and SCL to `Vext` at the Heltec side.
* Seal the cable entry of the shield loosely (drip loop downwards); the sensor itself must stay in free air, not in a sealed box, or the humidity reading follows the box instead of the weather.
* Many cheap modules sold as "BME280" are really BMP280 (no humidity). If *Air Humidity* stays unavailable, check the chip marking.
* If the module's address pin is wired high the address is `0x77`; set `bme280_address: "0x77"` in the node file.
* Accuracy is about ±1 °C and ±3 % RH [R24]. Long exposure to near-100 % RH (fog, dew) can cause a temporary offset that recovers in dry weather.
* Because the BME280 is outside, the lite version has no humidity reading inside the box: replace the silica gel on a fixed schedule (spring and autumn).
