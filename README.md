# CHAC HW2.2 Firmware Features

## Overview

CHAC 2.2beta firmware runs on ESP32-C6, integrating oxygen analysis, dive calculations, environmental sensors, and battery management.

**Main source:** `main/chac_main.c`

### Hardware Peripherals

| Peripheral | Interface | Pins |
|------------|-----------|------|
| ST7789 LCD (240×240) | SPI 16MHz | SCLK=2, MOSI=11, DC=5, RST=4, BK=10 |
| ADS1115 (O2 sensor) | I2C 400kHz | SDA=18, SCL=19 |
| BME280 (temp/press/hum) | I2C 400kHz | SDA=18, SCL=19 |
| WS2812 LED | RMT | GPIO8 |
| Battery ADC | ADC | GPIO0 (100K/100K divider) |
| Power button | GPIO | GPIO6 (active low) |

## UI (LVGL)

System boots directly into LVGL mode with 3 tabs and a bottom status bar.

### Tab 0: O2

- Large centered O2 percentage (Montserrat 36px)
- Dive mode label:
  - Pressure > 977 hPa → "Altitude Diving"
  - Pressure ≤ 977 hPa → "Sea Level Diving"

### Tab 1: Adv

- O2 percentage (green, 36px)
- PPO2 1.2 MOD: meters/feet (white)
- PPO2 1.4 MOD: meters/feet (white)
- PPO2 1.6 MOD: meters/feet (white text on red background, warning)

### Tab 2: SysInfo

- Temp: temperature (°C)
- Press: barometric pressure (hPa)
- Hum: humidity (%)
- Batt: voltage + percentage
- Sensor Raw: ADS1115 raw mV
- Footer: "Chac 2.2beta Cysoft"

### Bottom Status Bar (22px)

- Battery icon (20px font, 4 levels):
  - \>75% → full (green)
  - \>50% → 3/4 (green)
  - \>25% → 1/4 (green)
  - ≤25% → empty (red)
- Voltage mapping: 3.3V = 0%, 4.2V = 100%

## Sensors & Calculations

### O2 Calculation

$$O2\% = \frac{V_{measured}}{V_{cal}} \times 20.9\%$$

- `V_cal`: auto-calibrated at boot (200ms settle + 16-sample average)
- Range: 0% ~ 100%, displayed to 2 decimal places

### MOD Calculation

$$MOD_{m} = \left(\frac{PPO2}{FO2} - 1\right) \times 10$$

$$MOD_{ft} = MOD_{m} \times 3.28084$$

### Battery Percentage

$$\%_{batt} = \frac{V_{batt} - 3.3}{4.2 - 3.3} \times 100$$

## Button Controls (GPIO6)

| Action | Duration | Behavior |
|--------|----------|----------|
| Short press | <5s | Cycle tabs in LVGL (O2→Adv→SysInfo→O2) |
| Long press | ≥5s | Shutdown (enter deep sleep) |
| During hold | while pressed | LED rapid random blink (30ms interval) |

## Auto Shutdown

- 3 minutes of inactivity → yellow LED for 3 seconds → enter deep sleep
- Any button press resets the timer

## Shutdown Procedure

1. Set `s_shutting_down` flag, stop button task from driving LED
2. Clear LED (WS2812 off)
3. Turn off LCD (display off + backlight off)
4. Wait for button release (prevent level-triggered immediate wake)
5. Set all non-wakeup GPIOs to high-impedance (direction disabled, pull-ups/pull-downs disabled):
   - LCD: GPIO 2, 4, 5, 10, 11
   - LED: GPIO 8
   - I2C: GPIO 18, 19
   - ADC: GPIO 0
6. Configure GPIO6 low-level wakeup
7. Enter deep sleep

## Boot Sequence

1. Clear residual deep sleep wakeup configuration
2. Initialize button GPIO (with `gpio_reset_pin` to clear residual state)
3. Initialize WS2812 LED (off)
4. Initialize ST7789 LCD (backlight stays off)
5. Clear display to black → turn on backlight (prevents old image flash)
6. Create button task (priority 10) + action task (priority 5)
7. Initialize I2C bus + ADS1115 + BME280
8. O2 sensor auto-calibration
9. Initialize battery ADC
10. Display startup logo
11. Enter LVGL mode
12. Start console REPL in background (priority 1)
13. Raise sensor loop to priority 3

## Task Architecture

| Task | Priority | Responsibility |
|------|----------|----------------|
| button_task | 10 | GPIO polling + LED blink |
| btn_action_task | 5 | Mode switching / shutdown execution |
| app_main (sensor loop) | 3 | Sensor sampling + LCD update |
| REPL | 2 | USB Serial/JTAG console |
| console_init | 1 | REPL initialization (one-shot) |

## Console Commands

| Command | Description |
|---------|-------------|
| `status` | Print sensor readings |
| `logs on\|off` | Enable/disable loop logging |
| `show` | Switch to sensor dashboard |
| `interval <ms>` | Set sampling interval (100-5000) |
| `led red\|green\|off` | Set LED color manually |
| `lcdpreview` | Run LCD graphics preview |
| `logo [n]` | Replay startup logo n times |
| `tab <0\|1\|2>` | Switch LVGL tab |
| `lvgl` | Enter LVGL mode |
| `sleep` | Enter deep sleep immediately |
| `reboot` | Reboot chip |

## Project Structure

```
main/
├── chac_main.c        # Main application (UI, sensors, buttons, power)
├── board_config.h     # Hardware pin and peripheral configuration
├── battery_adc.c/h    # Battery voltage sampling (GPIO0 ADC)
├── bme280_simple.c/h  # BME280 driver (I2C)
├── spi_debug.c        # SPI debugging utilities
├── lcd_font8x8.h      # 8×8 pixel font
├── lcd_font8x16.h     # 8×16 pixel font
├── CMakeLists.txt      # Build configuration
└── idf_component.yml   # Dependencies (led_strip, lvgl)
```

# Chac 2.2 Hardware IO & Network Specification

> **MCU:** ESP32-C6-WROOM-1 (U1)  
> **Board:** 35 × 70 mm, 2-layer PCB  
> **Power:** 3.7V Li-Ion → TPS63001 → 3.3V  

---

## 1. GPIO Pin Assignment

| GPIO | Net Label | Function | Direction | Notes |
|------|-----------|----------|-----------|-------|
| GPIO0 | OpsKey | Deep Sleep Wake Button | Input | Active low, external pull-up |
| GPIO1 | Batt_Check | Battery Voltage ADC | Input (Analog) | R31+R32 (100k+100k) voltage divider, reads VBAT/2 |
| GPIO2 | SCL-GPIO2 | I2C Clock (SCL) | Bidirectional | Shared bus: ADS1115 (0x48) + BME280 (0x76) |
| GPIO4 | Reset→GPIO4 | LCD Reset | Output | Active low, connected to ST7789 RST pin |
| GPIO5 | RS→GPIO5 | LCD Data/Command (DC) | Output | High=Data, Low=Command, connected to ST7789 DC pin |
| GPIO8 | GPIO8 | WS2812B RGB LED Data | Output | Strapping pin (pull-up required), single NeoPixel |
| GPIO9 | GPIO9 | Boot Button | Input | Strapping pin, active low, boot mode select |
| GPIO11 | SDA MOSI-GPIO11 | Shared I2C SDA / SPI MOSI | Bidirectional | Dual-purpose: I2C SDA for sensors + SPI MOSI for LCD |
| GPIO12 | USB_D- | USB Data Minus | Bidirectional | USB 2.0, via TPD2EUSB30A (U6) ESD protection |
| GPIO13 | USB_D+ | USB Data Plus | Bidirectional | USB 2.0, via TPD2EUSB30A (U6) ESD protection |
| GPIO16 | U0TXD | UART0 TX | Output | Debug console / programming |
| GPIO17 | U0RXD | UART0 RX | Input | Debug console / programming |
| GPIO20 | LCD_BL_GPIO | LCD Backlight Control | Output | PWM capable, drives AO3400A (Q1) N-MOS gate via R5 (100Ω) |
| EN | EN_Chip_PU | Chip Enable / Power Up | Input | R21 (10k) pull-up + C15 (100nF) debounce to GND |

### Unused / Available GPIOs
| GPIO | Status | Notes |
|------|--------|-------|
| GPIO3 | Free | Available for expansion |
| GPIO6 | Free | Available (FSPI CLK capable) |
| GPIO7 | Free | Available (FSPI MOSI capable) |
| GPIO10 | Free | Available |
| GPIO14 | Free | Available |
| GPIO15 | Free | Available |
| GPIO18 | Free | Available |
| GPIO19 | Free | Available |
| GPIO21 | Free | Available |
| GPIO22 | Free | Available |
| GPIO23 | Free | Available |

---

## 2. I2C Bus

| Parameter | Value |
|-----------|-------|
| SDA | GPIO11 (shared with SPI MOSI) |
| SCL | GPIO2 |
| Pull-ups | R23 + R24 (4.7kΩ each) + R81 + R82 (4.7kΩ each) |
| Speed | Standard/Fast (100/400 kHz) |

### I2C Devices
| Device | Address | Function | Reference |
|--------|---------|----------|-----------|
| ADS1115IDGS | 0x48 | 16-bit ADC, 4-channel | U5, ADDR→GND |
| BME280 | 0x76 | Temperature, Humidity, Pressure | U8, SDO→GND |

---

## 3. SPI Bus (LCD)

| Parameter | Value |
|-----------|-------|
| MOSI | GPIO11 (shared with I2C SDA) |
| SCLK | GPIO2 (shared with I2C SCL) |
| CS | Directly tied to GND (always selected) |
| DC (RS) | GPIO5 |
| Reset | GPIO4 |
| Backlight | GPIO20 → R5 (100Ω) → AO3400A (Q1) gate |

### LCD Module
| Parameter | Value |
|-----------|-------|
| Controller | ST7789V |
| Size | 1.54 inch |
| Resolution | 240 × 240 pixels |
| Color | RGB565 (65K colors) |
| Interface | 4-wire SPI |
| Connector | J4, HRS FH34SRJ-8S-0.5SH(50), 8-pin FPC |

### J4 FPC Pin Mapping
| Pin | Signal | Connection |
|-----|--------|------------|
| 1 | GND | Ground |
| 2 | LCD_GND | Backlight Cathode |
| 3 | LCD_BL | Backlight Anode (via Q1) |
| 4 | SDA/MOSI | GPIO11 |
| 5 | SCL/SCLK | GPIO2 |
| 6 | RS/DC | GPIO5 |
| 7 | Reset | GPIO4 |
| 8 | VCC | +3.3V |

> **Note:** I2C and SPI share GPIO2 (SCL/SCLK) and GPIO11 (SDA/MOSI). The LCD CS is tied to GND, so the bus is always selected. Firmware must manage bus contention between I2C sensor reads and SPI LCD writes.

---

## 4. ADC Channels (ADS1115)

| Channel | Net Label | Function | Input Range |
|---------|-----------|----------|-------------|
| AIN0 | SensorP | O₂ Sensor Positive | 0–3.3V (differential with AIN1) |
| AIN1 | SensorN | O₂ Sensor Negative | Reference/GND side |
| AIN2 | — | Unused | Available |
| AIN3 | — | Unused | Available |

### O₂ Sensor Interface
| Parameter | Value |
|-----------|-------|
| Connector | J1, SMB Jack Vertical |
| Input Protection | R1 (100Ω) series resistor |
| Measurement | Differential (AIN0 - AIN1) |
| Sensor Type | Electrochemical oxygen cell |

---

## 5. Internal ADC (ESP32-C6)

| GPIO | Channel | Net Label | Function |
|------|---------|-----------|----------|
| GPIO1 | ADC1_CH1 | Batt_Check | Battery voltage monitoring |

### Battery Voltage Divider
```
Bat+ ─── R31 (100kΩ) ──┬── R32 (100kΩ) ─── GND
                        │
                     GPIO1 (ADC)
                     Reads VBAT / 2
```
- Full battery (4.2V) → ADC reads ~2.1V
- Empty battery (3.0V) → ADC reads ~1.5V
- ESP32-C6 ADC range: 0–3.3V, 12-bit resolution

---

## 6. Power System

### Power Architecture
```
USB-C (5V) ──→ BQ24072RGT (U3) ──→ PM_VOut ──→ TPS63001 (U2) ──→ 3.3V
                    │       │
                 Bat+ ←──→ J3 (JST-PH, 3.7V Li-Ion)
                    │
               Charge Management
```

### USB-C (J20)
| Parameter | Value |
|-----------|-------|
| Connector | GCT USB4105, 16-pin, top mount |
| CC Resistors | R8, R9 (5.1kΩ) — UFP (device) mode |
| ESD Protection | TPD2EUSB30A (U6) on D+/D- |
| Shield Filter | R7 (1MΩ) + C100 (10nF) to GND |
| Data Lines | GPIO12 (D-), GPIO13 (D+) |

### Battery Charger — BQ24072RGT (U3)
| Parameter | Value |
|-----------|-------|
| Input | +5V (USB) |
| Output | PM_VOut (tracks battery voltage) |
| Charge Current | ~500mA (ISET: R4 = 2kΩ) |
| Input Current Limit | ~500mA (ILIM: R40 = 2kΩ) |
| Mode | EN1=HIGH, EN2=LOW → USB 500mA mode |
| Termination | ~50mA (10% of charge current) |
| Battery Connector | J3, JST-PH 2-pin |
| Status LEDs | D1 (Red, ~CHG), D2 (Green, ~PGOOD) via R14, R18 (1kΩ) |
| TS Pin | R10 (10kΩ) to GND — NTC disabled |
| TMR Pin | R80 (10kΩ) to GND — safety timer |

### Voltage Regulator — TPS63001 (U2)
| Parameter | Value |
|-----------|-------|
| Type | Buck-Boost switching regulator |
| Input | PM_VOut (2.5V–5.5V) |
| Output | 3.3V fixed |
| Inductor | L1 (2.2µH) |
| Feedback | Internal (fixed 3.3V output) |
| Input Caps | C7 (10µF), C8 (1µF) |
| Output Caps | C9 (100nF), C10 (10µF) |

### Power Rails Summary
| Rail | Voltage | Source | Consumers |
|------|---------|--------|-----------|
| +5V | 5.0V | USB-C | BQ24072 input |
| PM_VOut | 3.0–4.2V | BQ24072 output (tracks battery) | TPS63001 input, LED indicators |
| +3.3V | 3.3V | TPS63001 output | ESP32-C6, ADS1115, BME280, LCD, WS2812B |
| Bat+ | 3.0–4.2V | Li-Ion battery | BQ24072 BAT pin |

---

## 7. User Interface

### Buttons
| Ref | Function | GPIO | Type | Notes |
|-----|----------|------|------|-------|
| SW1 | Reset | EN_Chip_PU | Momentary | Resets ESP32 via EN pin |
| SW2 | Boot | GPIO9 | Momentary | Enter download mode (hold during reset) |
| SW4 | OpsKey | GPIO0 | Momentary | User button, deep sleep wake source |

### Status LEDs
| Ref | Color | Signal | Meaning |
|-----|-------|--------|---------|
| D1 | Red | ~CHG (BQ24072 Pin 9) | ON = Charging, OFF = Complete |
| D2 | Green | ~PGOOD (BQ24072 Pin 7) | ON = USB power good |
| D3 | RGB | GPIO8 (WS2812B) | Programmable status indicator |

### LCD Display
| Parameter | Value |
|-----------|-------|
| Type | ST7789V TFT |
| Size | 1.54" |
| Resolution | 240 × 240 |
| Interface | 4-wire SPI via FPC |
| Backlight | PWM controlled via GPIO20 |
| Color Format | RGB565 byte-swapped |

---

## 8. Sensor Interface

### BME280 Environmental Sensor (U8)
| Parameter | Value |
|-----------|-------|
| Interface | I2C (address 0x76) |
| Measurements | Temperature, Humidity, Barometric Pressure |
| Use Case | Altitude compensation for O₂ readings |
| Decoupling | C13 (100nF) on VDD, C14 (100nF) on VDDIO |

### ADS1115 16-bit ADC (U5)
| Parameter | Value |
|-----------|-------|
| Interface | I2C (address 0x48, ADDR→GND) |
| Channels | 4 single-ended or 2 differential |
| Resolution | 16-bit |
| Sample Rate | 8–860 SPS |
| Input Protection | R1 (100Ω) on sensor input |
| Use Case | O₂ electrochemical sensor measurement |
| Decoupling | C23, C24 (100nF) |

### O₂ Sensor
| Parameter | Value |
|-----------|-------|
| Connector | J1, SMB vertical jack |
| Type | Electrochemical oxygen cell |
| Output | mV-level signal (proportional to O₂ partial pressure) |
| ADC Channel | ADS1115 AIN0/AIN1 (differential) |

---

## 9. Wireless

### ESP32-C6-WROOM-1 (U1)
| Parameter | Value |
|-----------|-------|
| Module | ESP32-C6-WROOM-1 |
| Chip | ESP32-C6 (RISC-V single core, 160MHz) |
| Flash | 4MB (embedded) |
| WiFi | 802.11b/g/n (2.4GHz), WiFi 6 (802.11ax) |
| Bluetooth | BLE 5.0 + Bluetooth Mesh |
| Zigbee | IEEE 802.15.4 (Thread/Zigbee) |
| Antenna | On-module PCB antenna |
| Antenna Jumper | R20 (0Ω) — connects to on-module antenna |

### Wireless Capabilities
| Protocol | Standard | Use Case |
|----------|----------|----------|
| WiFi 6 | 802.11ax | OTA updates, data logging to cloud |
| BLE 5.0 | Bluetooth Low Energy | Mobile app connectivity |
| Thread | IEEE 802.15.4 | Mesh networking (future) |
| Zigbee | IEEE 802.15.4 | Smart home integration (future) |

### Antenna
| Parameter | Value |
|-----------|-------|
| Type | PCB antenna (on WROOM module) |
| Feed | R20 (0Ω jumper) |
| Frequency | 2.4 GHz |
| Note | Keep ground plane clear under antenna area |

---

## 10. Protection & ESD

| Component | Function | Location |
|-----------|----------|----------|
| U6 (TPD2EUSB30A) | USB D+/D- ESD protection | Between USB-C and ESP32 |
| D10 (PESD5V0S1BA) | TVS diode, 5V clamp | USB power line |
| D4 (1N4148W) | Reverse polarity protection | Power path |
| R7 (1MΩ) + C100 (10nF) | USB shield RC filter | USB-C shell to GND |
| R8, R9 (5.1kΩ) | USB-C CC pull-down | UFP identification |

---

## 11. Mechanical

| Parameter | Value |
|-----------|-------|
| PCB Size | 35 × 70 mm |
| Layers | 2 |
| Mounting Holes | H1-H4, M2, 2.2mm diameter |
| Hole Positions | (2.79, 2.79), (32.21, 2.79), (32.21, 67.21), (2.79, 67.21) mm from board origin |
| Board Thickness | 1.6 mm |
| Corner Radius | 3 mm (rounded) |
| Components | Top side only (single-side assembly) |
| Total Components | 66 |

---

## 12. Connector Summary

| Ref | Type | Function | Pin Count |
|-----|------|----------|-----------|
| J1 | SMB Vertical Jack | O₂ sensor input | 2 (signal + shield) |
| J2 | JST-ZH 2-pin | Auxiliary connector | 2 |
| J3 | JST-PH 2-pin | Battery (3.7V Li-Ion) | 2 |
| J4 | HRS FH34SRJ-8S FPC | LCD display (ST7789) | 8 |
| J20 | USB-C 16-pin | Power + data | 16 |

---

*Generated from Chac 2.2 KiCad schematic — April 2026*
*by Cysoft & Chac 🐕*

