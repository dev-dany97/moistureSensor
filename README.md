# Moisture Sensor

- A self-built soil moisture sensor for plants
- ESPHome and Home Assistant
- 3D printed case

<div align="center">

<img src="docs/images/v1-closed-in-pot.jpg" width="420" alt="The finished sensor in a plant pot">

</div>

## Moisture Sensor

### Version 1: D1 mini in a 3D printed case

The first version is made for cable operation. It uses an ESP8266 (Wemos D1 mini) with a capacitive soil moisture sensor, which doesn't corrode in the soil, and is powered over USB. I used the case from [Cults3D](https://cults3d.com/:127333), since I didn't design it myself.

<img src="docs/images/v1-closed-in-pot.jpg" width="320" alt="Version 1, closed, in a plant pot">

#### Parts

| Part | Notes |
|---|---|
| [Wemos D1 mini (Lite)](https://www.wemos.cc/en/latest/d1/d1_mini_lite.html) | ESP8266, Wi-Fi |
| Capacitive Soil Moisture Sensor v1.2 | Capacitive, so it doesn't corrode in the soil like resistive sensors |
| USB cable and power supply | 5 V |
| 3D printed case | From [Cults3D](https://cults3d.com/:127333) |

#### Wiring

| From | To |
|---|---|
| Soil sensor VCC | 3.3V on the D1 mini |
| Soil sensor GND | GND on the D1 mini |
| Soil sensor AOUT | A0 on the D1 mini |
| USB power supply | USB port on the D1 mini |

#### ESPHome and Home Assistant

Version 1 runs without deep sleep, because it is powered by cable. It reads the soil sensor on A0 every 60 seconds and sends the values over MQTT:

| Topic | Value |
|---|---|
| `pflanzensensor/sensor/bodenfeuchtigkeit/state` | Soil moisture in % |
| `pflanzensensor/sensor/soil_moisture_sensor_voltage/state` | Raw sensor voltage |

Home Assistant picks the values up through its MQTT integration. Calibrate the two values in `calibrate_linear` for your own sensor: the voltage in water is 100 %, the voltage in dry air is 0 %. If Windows doesn't recognise the D1 mini, install the [CH340 driver](https://www.wemos.cc/en/latest/ch340_driver.html).

<details>
<summary>Show <code>plantsensor-v1.yaml</code></summary>

```yaml
esphome:
  name: plantsensor-v1
  friendly_name: Plantsensor_V1
  min_version: 2025.11.0
  name_add_mac_suffix: false

esp8266:
  board: d1_mini

logger:

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password

ota:
  - platform: esphome

captive_portal:

# --------------------
# MQTT
# Topic: pflanzensensor/sensor/bodenfeuchtigkeit/state
# --------------------
mqtt:
  broker: !secret mqtt_broker
  username: !secret mqtt_username
  password: !secret mqtt_password
  topic_prefix: pflanzensensor

# --------------------
# Sensoren
# --------------------
sensor:

  # Bodenfeuchte Spannung (A0)
  # Der D1 mini hat einen Spannungsteiler am A0, daher x 3.3
  - platform: adc
    pin: A0
    name: "Soil Moisture Sensor Voltage"
    id: soil_moisture_voltage
    unit_of_measurement: "V"
    accuracy_decimals: 3
    update_interval: 60s
    filters:
      - multiply: 3.3

  # Bodenfeuchte %
  # Kalibrierung: Wert im Wasser -> 100, Wert in trockener Luft -> 0
  - platform: template
    name: "Bodenfeuchtigkeit"
    unit_of_measurement: "%"
    device_class: moisture
    accuracy_decimals: 0
    lambda: |-
      return id(soil_moisture_voltage).state;
    update_interval: 60s
    filters:
      - calibrate_linear:
          - 1.29 -> 100
          - 2.80 -> 0
      - lambda: |-
          if (x > 100) return 100;
          if (x < 0) return 0;
          return x;
```

</details>

## License

[MIT](LICENSE)

Thanks to the maker and ESPHome community for the inspiration.
