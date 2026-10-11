Smart FloodGuard

A low-cost, multi-sensor flood monitoring and early warning system using ESP32, LoRa and ESP32-CAM.

Built for the **Open Hardware Make-A-Thon 2026** by students of the Department of EEE, Easwari Engineering College, Chennai.

**Team:** Abi J, Ajayvarman V, Allanso Griffith M, Dibika A, Hemala R

> Smart FloodGuard is a prototype for controlled testing. It is not a replacement for official flood-warning services.

INTRODUCTION:

Most simple flood alarms trigger only when the water crosses one fixed height. Smart FloodGuard asks three questions instead of one: **how high is the water, how fast is it rising, and is it raining?**

An ESP32 reads a water-level sensor and an LM393 rain sensor, calculates the rate of rise, combines the three readings with a simple scoring scheme, and classifies the situation as **Safe, Warning, Alert or Critical**. The status is shown locally and sent over a long-range LoRa link to a central warning unit, so the main warning path works without Internet. An ESP32-CAM gives a visual check that supports the sensor readings.

The design uses common modules, a 5 V supply and rule-based threshold logic (no machine learning), so it is easy to reproduce and calibrate.

 Features

- Multi-sensor risk model using water level, rate of rise and rainfall
- Four risk levels: Safe, Warning, Alert, Critical
- Local warning with 3 LEDs, an active buzzer and a 0.96-inch SSD1306 OLED display
- Internet-independent communication using two LoRa RA-02 (SX1278) modules
- Visual flood confirmation using an ESP32-CAM
- Moving-average filtering and multi-sample confirmation to reduce false alarms
- Open design with a documented bill of materials, wiring, code structure and test procedure

System architecture

```
SENSOR NODE (near the water)                     CENTRAL WARNING UNIT
+-----------------------------+                  +----------------------------+
| Water sensor --+            |                  |                            |
| Rain sensor  --+--> ESP32   |  -- LoRa link -> | LoRa RA-02 --> ESP32       |
|                |    DevKit  |   (RA-02 x 2)    |                |           |
|                |    V1      |                  |     OLED + LEDs + Buzzer   |
|                v            |                  |                            |
|  OLED + 3 LEDs + Buzzer     |                  | ESP32-CAM (visual check)   |
+-----------------------------+                  +----------------------------+
```

Add your block diagram here:

```
![Block diagram](docs/block-diagram.png)
```

 Hardware

| No. | Component | Function |
|---|---|---|
| 1 | ESP32 DevKit V1 (x2) | Main controller, one at the node and one at the central unit |
| 2 | Water sensor | Water-level detection (analog, read by ESP32 ADC) |
| 3 | Rain sensor (LM393) | Rainfall detection (analog and digital outputs) |
| 4 | LoRa RA-02 SX1278 (x2) | Long-range link, one transmitter and one receiver (SPI) |
| 5 | ESP32-CAM | Visual flood detection |
| 6 | 0.96" OLED (SSD1306) | Shows level, rain, rise rate and risk level (I2C) |
| 7 | 3 LEDs (green, yellow, red) | Status indication |
| 8 | Active buzzer | Warning alarm with different beep patterns |
| 9 | 220 ohm resistors | LED protection |
| 10 | 5 V power supply / USB adapter | Power |
| 11 | Jumper wires | Prototype wiring |

Pin assignment (sensor node)

| Connection | ESP32 pin(s) | Interface |
|---|---|---|
| Water sensor (analog) | GPIO 34 | ADC |
| Rain sensor (analog out) | GPIO 35 | ADC |
| LoRa RA-02: SCK / MISO / MOSI | GPIO 18 / 19 / 23 | SPI |
| LoRa RA-02: NSS / RST / DIO0 | GPIO 5 / 14 / 2 | SPI + control |
| OLED SSD1306: SDA / SCL | GPIO 21 / 22 | I2C |
| Green / Yellow / Red LED (via 220 ohm) | GPIO 25 / 26 / 27 | Digital output |
| Active buzzer | GPIO 32 | Digital output |

Wiring notes:
- The RA-02 uses 3.3 V logic. Never power it from 5 V.
- Connect the antenna before powering the module.
- Share a common ground between all modules and the ESP32.

 How it works

1. The node reads the water and rain sensors at a fixed sampling interval.
2. Each reading is averaged over several samples to suppress noise and surface ripples.
3. The rate of rise is calculated from two consecutive filtered readings:

   ```
   Rate of Rise = (Current Level - Previous Level) / Time Interval
   ```

   Example: 40 cm to 50 cm in 5 minutes gives 2 cm/min.
4. Each parameter gets a score, and the total score maps to a risk level.
5. The matching LED and buzzer pattern are activated and the OLED is updated.
6. A compact status packet is sent over LoRa to the central unit, which displays it and raises its own alarm.

 Scoring rules

Initial thresholds. They must be calibrated on site.

| Parameter | Score 0 | Score 1 | Score 2 | Score 3 |
|---|---|---|---|---|
| Water level | Low (< 30 %) | Medium (30-60 %) | High (60-85 %) | Very high (> 85 %) |
| Rate of rise | Steady or falling | Slow | Moderate | Rapid |
| Rainfall | Dry | Light | Heavy | - |

### Risk levels

| Level | Total score | Indicator | Action |
|---|---|---|---|
| SAFE | 0-1 | Green LED on, buzzer off | Normal monitoring |
| WARNING | 2-3 | Yellow LED on, short beep | Prepare and keep monitoring; status sent by LoRa |
| ALERT | 4-6 | Red LED on, intermittent buzzer | Local alert and LoRa alert to central unit |
| CRITICAL | 7-8, or level > 85 % with rapid rise | Red LED blinking, continuous buzzer | Emergency warning, LoRa alert, camera check |

Alert and Critical share the red LED and are told apart by the buzzer pattern and blinking. The risk level is lowered only after the change is confirmed over two or three consecutive samples, which prevents flickering between states.

 LoRa packet

```
NODE 01 | LEVEL: 65 cm | RISE: HIGH | RAIN: HIGH | STATUS: FLOOD ALERT
```

| Field | Example | Purpose |
|---|---|---|
| Node ID | NODE 01 | Identifies the sending node |
| Water level | 65 cm | Current filtered level |
| Rise | HIGH | LOW / MODERATE / HIGH |
| Rain | HIGH | DRY / LIGHT / HEAVY |
| Status | FLOOD ALERT | Risk level shown at the central unit |

Software

- Arduino IDE with the ESP32 board package (ESP32 and ESP32-CAM)
- LoRa library (SX127x) for the RA-02 modules
- Adafruit SSD1306 and Adafruit GFX libraries for the OLED
- ESP32 camera driver (`esp_camera`) for the ESP32-CAM
- Serial Monitor and Serial Plotter for calibration


## Future work

- Multiple sensor nodes reporting to one central unit with unique node IDs
- Waterproof level sensor (ultrasonic or pressure type) and a calibrated rain gauge
- Solar charging with battery-health monitoring
- IoT dashboard with historical graphs, without weakening the offline warning path
- TinyML-based short-term flood-risk prediction
- GPS location, rugged waterproof enclosure and field validation
