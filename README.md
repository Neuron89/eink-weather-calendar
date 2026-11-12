# eink-weather-calendar

ESP32 firmware for a Waveshare 7.5-inch e-ink panel that shows current weather and an hourly forecast on the left half of the screen and upcoming Microsoft Outlook calendar events on the right. The repository contains a PlatformIO project and two Arduino IDE sketches of the same design.

## Project status

This is a working prototype of the device framework; two data integrations are not finished. The table reflects what the code does today.

| Area | State |
|---|---|
| Wi-Fi provisioning through a captive portal (WiFiManager) | Implemented |
| NTP time sync, 30-minute refresh cycle with deep sleep | Implemented |
| Split-screen layout, weather panel, calendar panel (GxEPD2) | Implemented |
| Settings persistence in ESP32 NVS (`Preferences`) | Implemented |
| IP-based location lookup (ip-api.com) | City and country are resolved; coordinates are not yet passed to the weather request |
| OpenWeatherMap request and JSON parsing | Implemented against the One Call 2.5 endpoint, with placeholder coordinates |
| Microsoft Graph calendar | Refresh-token exchange is implemented; the Graph request is not, so the calendar panel renders two sample events |
| Weather icons | Drawn as labelled placeholder boxes; bitmap loading is not implemented |
| Battery indicator | Drawing routine exists with fixed example values |

Known build issues in the PlatformIO variant: `src/main.cpp` includes an `epd_driver.h` header that is not part of the project, `saveConfig()` is defined in both `src/main.cpp` and `src/config.cpp`, and `platformio.ini` lists library dependencies that the code does not use. The Arduino sketches are the more self-consistent starting point.

## Hardware

- ESP32 development board (written for the Waveshare ESP32 e-Paper Driver Board; `esp32dev` target)
- Waveshare 7.5-inch e-paper display, driven as `GxEPD2_750_T7` (black and white)
- USB cable for power and programming

| E-paper pin | ESP32 pin |
|---|---|
| BUSY | GPIO25 |
| RST | GPIO26 |
| DC | GPIO27 |
| CS | GPIO15 |
| CLK | GPIO13 |
| DIN | GPIO14 |
| GND | GND |
| VCC | 3.3V |

## Architecture

```
OpenWeatherMap    ip-api.com    Microsoft identity / Graph
       \              |              /
        +---------- ESP32 ----------+
        | WiFiManager portal        |
        | weather + calendar fetch  |
        | GxEPD2 renderer           |---- SPI ---- 7.5" e-ink panel
        | NVS settings              |
        +---------------------------+
```

On boot the firmware initialises the display, joins Wi-Fi (opening a setup access point if no credentials are stored), syncs time over NTP, fetches data, draws one full frame, and enters deep sleep for 30 minutes. More detail is in [docs/architecture.md](docs/architecture.md).

| Component | Library |
|---|---|
| Display driver and drawing | [GxEPD2](https://github.com/ZinggJM/GxEPD2) with Adafruit GFX fonts |
| Wi-Fi provisioning | [WiFiManager](https://github.com/tzapu/WiFiManager) |
| JSON parsing | [ArduinoJson](https://arduinojson.org/) 6.x |
| Time | [Time](https://github.com/PaulStoffregen/Time) |
| Storage | ESP32 `Preferences` (NVS) |

## Repository layout

```
platformio.ini         PlatformIO environment (esp32dev, Arduino framework)
src/, include/         PlatformIO variant, split into weather, calendar, display, config modules
arduino_sketch/        Arduino IDE sketch with the same modules as header files
arduino_version/       Arduino IDE sketch in a single .ino file
tools/
  ms_auth_helper.py          Local OAuth helper that obtains a Microsoft refresh token
  generate_weather_icons.py  Downloads OpenWeatherMap icons and converts them to 1-bit BMP
  setup.sh                   Interactive helper for the PlatformIO variant
docs/                  Architecture notes and setup guide
assets/                Target directory and naming convention for weather icons
```

## Getting started

### Prerequisites

- Arduino IDE with ESP32 board support, or PlatformIO
- An [OpenWeatherMap API key](https://home.openweathermap.org/api_keys)
- For calendar access: a Microsoft Entra ID (Azure AD) app registration with the delegated `Calendars.Read` and `offline_access` permissions and the redirect URI `http://localhost:8000/callback`
- Python 3 with `requests`, `flask`, and `pillow` for the scripts in `tools/`

### Secrets

No credentials are committed. Each variant reads them from a `secrets.h` that is created from a tracked template and ignored by Git.

| Variant | Template | Copy to |
|---|---|---|
| `arduino_sketch/` | `arduino_sketch/secrets.h.example` | `arduino_sketch/secrets.h` |
| `arduino_version/` | `arduino_version/secrets.h.example` | `arduino_version/secrets.h` |
| PlatformIO | `include/secrets_template.h` | `include/secrets.h` |

| Constant | Purpose | Placeholder |
|---|---|---|
| `OPENWEATHERMAP_API_KEY` | OpenWeatherMap API key | `your_api_key_here` |
| `MICROSOFT_CLIENT_ID` | App registration client ID | `your_client_id_here` |
| `MICROSOFT_CLIENT_SECRET` | App registration client secret | `your_client_secret_here` |
| `MICROSOFT_REDIRECT_URI` | OAuth redirect URI | `http://localhost` |

### Arduino IDE

1. Copy `secrets.h.example` to `secrets.h` in the sketch folder you intend to use and fill in the values.
2. Open `Eink_Weather_Calendar.ino` from `arduino_sketch/` (modular) or `arduino_version/` (single file).
3. Install the libraries with the Library Manager: GxEPD2, ArduinoJson, WiFiManager, and Time.
4. Select **Tools → Board → ESP32 Dev Module** and **Tools → Partition Scheme → Huge APP**, then choose the serial port.
5. Compile and upload.

### PlatformIO

```bash
cp include/secrets_template.h include/secrets.h   # then edit the values
pio run
pio run --target upload
pio device monitor
```

See the known build issues under [Project status](#project-status) before building this variant. A longer walkthrough is in [docs/setup_guide.md](docs/setup_guide.md).

### Microsoft refresh token

`tools/ms_auth_helper.py` runs a local Flask server on port 8000, walks through the OAuth 2.0 authorization-code flow, and writes the resulting tokens to `ms_tokens.json` and `esp32_ms_token.h` in the current directory. Both files are ignored by Git. The script contains no credentials; it prompts for the client ID and secret at run time.

```bash
pip install requests flask
cd tools
python ms_auth_helper.py
```

### Weather icons

`tools/generate_weather_icons.py` downloads the OpenWeatherMap icon set and writes 1-bit BMP files at 100x100 and 30x30 pixels into `assets/`. The naming convention is described in [assets/README.md](assets/README.md).

## First boot

1. Power the device. The display shows a Wi-Fi setup screen.
2. Join the access point named `EinkWeather_XXXXXX` from a phone or laptop.
3. Open `192.168.4.1` (the WiFiManager portal address) if the captive portal does not appear on its own.
4. Enter the Wi-Fi credentials and, optionally, a location name. The portal times out after three minutes and the device restarts.
5. After connecting, the device fetches data, draws the screen, and sleeps until the next 30-minute refresh.

## Troubleshooting

- **Sketch too large:** select the Huge APP partition scheme.
- **Upload fails:** hold the BOOT button while uploading, or try another USB cable or port.
- **Display does not update:** check the SPI wiring against the table above and power-cycle the board.
- **Cannot join Wi-Fi:** the ESP32 supports 2.4 GHz networks only.
- **Wrong orientation:** change the argument to `display.setRotation()` in the display initialisation.

## License

MIT. See [LICENSE](LICENSE).
