# Arduino IDE Version

This directory contains a simplified version of the project that can be directly opened in the Arduino IDE.

## Files

- `Eink_Weather_Calendar.ino` - Main Arduino sketch file
- `secrets.h.example` - Template for API keys; copy it to `secrets.h` (ignored by Git) and add your credentials

## Setup Instructions

1. Open `Eink_Weather_Calendar.ino` in Arduino IDE
2. Install the required libraries through the Arduino Library Manager:

   - GxEPD2 by Jean-Marc Zingg
   - ArduinoJson by Benoit Blanchon
   - WiFiManager by tzapu
   - Time by Paul Stoffregen

3. Copy `secrets.h.example` to `secrets.h` and edit it with your API keys
4. Select the ESP32 board from the Tools menu
5. Compile and upload to your ESP32

## Hardware Connections

Connect the Waveshare 7.5-inch E-Paper HAT (H) to the ESP32 as follows:

| E-Paper Display | ESP32  |
| --------------- | ------ |
| BUSY            | GPIO25 |
| RST             | GPIO26 |
| DC              | GPIO27 |
| CS              | GPIO15 |
| CLK             | GPIO13 |
| DIN             | GPIO14 |
| GND             | GND    |
| VCC             | 3.3V   |

## Troubleshooting

If you encounter compilation errors:

- Make sure all required libraries are installed
- Try changing the Partition Scheme to "Huge APP" in the Tools menu
- Check that your ESP32 board support is properly installed
