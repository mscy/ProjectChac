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

## Build & Flash

```bash
idf.py build
idf.py -p /dev/tty.usbmodem1101 flash
idf.py monitor
```
