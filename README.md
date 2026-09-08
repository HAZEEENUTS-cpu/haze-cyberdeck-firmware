# HAZE Cyberdeck OS

Custom firmware for the LilyGo T-Embed CC1101 ESP32-S3 device.

## Features

- **220x170 IPS Display** with LovyanGFX driver
- **Rotary Encoder** input with tactile feedback
- **WiFi Tools** (scan, connect, status)
- **Bluetooth** server mode with media control
- **Clock** with NTP sync and timezone support
- **SD Card** support with file browser
- **Settings** persistence via NVS
- **Safe Boot Mode** for display recovery
- **Animated Home Screen** with procedural face
- **4 Themes** (Cyberpunk, Retro Teal, Terminal, Light)
- **Mini Apps** framework (calculator, stopwatch, timer, dice, monitor)

## Hardware

- **MCU**: ESP32-S3
- **Display**: ST7789V 170×320 (rotated to 320×170 landscape)
- **Input**: Rotary encoder + tactile button
- **Storage**: microSD card (up to 32GB FAT32)
- **Wireless**: WiFi 802.11 b/g/n, Bluetooth 5.0 LE
- **Power**: Li-ion battery with charging circuit

## Pin Configuration

```
Display SPI:
  MOSI (GPIO 9)   → Display data
  SCLK (GPIO 11)  → Display clock
  CS (GPIO 41)    → Display chip select
  DC (GPIO 16)    → Display data/command
  RST (GPIO 40)   → Display reset
  BL (GPIO 21)    → Backlight

Encoder:
  A (GPIO 4)      → Rotary A
  B (GPIO 5)      → Rotary B
  SW (GPIO 0)     → Button (also boot button)

SD Card SPI:
  MOSI (GPIO 9)   → SD data
  SCLK (GPIO 11)  → SD clock
  MISO (GPIO 10)  → SD input
  CS (GPIO 13)    → SD chip select

Status:
  WS2812 (GPIO 14) → RGB LED (8 pixels, not yet implemented)
```

## Installation

### Requirements

- Arduino IDE 1.8.19+ or VSCode with PlatformIO
- ESP32 Board Package 2.0.17+
- LovyanGFX library (latest)

### Steps

1. **Install ESP32 board support**:
   - Arduino: Tools → Board Manager → search "esp32" → install
   - PlatformIO: Auto-installed with platform definition

2. **Install LovyanGFX**:
   - Arduino: Sketch → Include Library → Manage Libraries → search "LovyanGFX" → install
   - PlatformIO: Add to `platformio.ini`:
     ```ini
     lib_deps = lovyan03/LovyanGFX
     ```

3. **Configure WiFi** (optional):
   - Open `TEmbedOS.ino`
   - Find `const char* WIFI_SSID = "YOUR_WIFI_NAME";`
   - Replace with your network SSID and password
   - If not configured, WiFi menu will show "SSID NOT CONFIGURED"

4. **Flash firmware**:
   - Connect device via USB-C
   - Hold **BOOT** button, press **RESET**, release **BOOT** to enter bootloader
   - Arduino: Tools → Port → select device → Upload
   - PlatformIO: Platform → Upload

5. **First boot**:
   - Device will test display (red → green → blue)
   - Home screen shows animated face
   - Press encoder to open menu
   - Hold encoder for 850ms to access settings

## Configuration

### Display Offset

If the image is shifted or clipped:

1. Open `TEmbedOS.ino`
2. Find `static const int DISPLAY_OFFSET_X = 35;`
3. Try values: 0, 34, 35, 36
4. Recompile and test

The offset depends on your ST7789 panel controller revision. 35 is common for LilyGo T-Embed CC1101 v1.0.

### WiFi Credentials

```cpp
const char* WIFI_SSID = "My Network";
const char* WIFI_PASSWORD = "password123";
```

Leave as `"YOUR_WIFI_NAME"` to disable WiFi.

### Themes

Press encoder to navigate → Settings → Display Settings → Theme:
- **0**: Cyberpunk (neon green/purple on dark blue)
- **1**: Retro Teal (cyan on black)
- **2**: Terminal (green on black, monospace-like)
- **3**: Light (dark blue on light gray)

### Idle Timeout

Settings → Display Settings → Timeout:
- Range: 15–180 seconds
- Default: 45 seconds
- When idle, screen enters low-power mode with animated status display

## Usage

### Navigation

- **Rotate encoder**: Scroll up/down
- **Press encoder**: Select / open menu
- **Hold encoder (850ms+)**: Open settings or return to menu
- **Top button (GPIO 0)**: Back / sleep device

### Home Screen

- Displays time and status (WiFi, Bluetooth, SD card)
- Animated face with blinking and eye motion
- Press encoder to open menu

### Main Menu

1. **Remote Control** — Placeholder for local HTTP commands (not yet implemented)
2. **WiFi Tools** — Connect, disconnect, scan networks, view status
3. **Bluetooth Tools** — Toggle BLE advertising, view connection status
4. **Clock** — Display time and date, toggle 12/24-hour format
5. **SD Card** — Browse files on inserted microSD card
6. **Mini Apps** — Calculator, stopwatch, timer, dice roller, system monitor
7. **Settings** — Display, system, connectivity options
8. **Hardware Test** — Display specifications and component status
9. **About HAZE** — Version and disclaimer

### WiFi Menu

- **Connect**: Attempt connection to configured SSID
- **Disconnect**: Close WiFi connection
- **Scan**: List nearby networks (blocking operation)
- **Status**: Show current connection IP or error

### Bluetooth Menu

- Toggle BLE advertising on/off
- Device name: "HAZE CYBERDECK"
- Advertises custom GATT service (7e4a0001-9d5d-4a26-9e6b-9e4b8f2c3001)
- Accepts: STATUS, PLAY, PAUSE, NEXT, PREVIOUS, VOLUME_UP, VOLUME_DOWN

### Safe Boot Mode

1. Hold **BOOT** button (GPIO 0, center of encoder)
2. Press and release **RESET** button (back of device)
3. Release **BOOT** button
4. Firmware enters recovery mode:
   - Color test (red → green → blue)
   - No peripherals initialized
   - Safe to test display or reset settings

## Troubleshooting

### Display shows wrong orientation

- Check DISPLAY_OFFSET_X (should be 35 for T-Embed v1.0)
- Serial output shows: `[DISPLAY] runtime=320x170`
- If dimensions mismatch, display geometry error is shown

### WiFi won't connect

- Verify SSID and password are correct in code
- Check router is not hidden or using unusual security
- Confirm antenna is seated properly
- View error in status bar: "W:FAIL" with detailed message

### Settings don't save

- Check Serial output for "[SETTINGS] Preferences initialization failed"
- If NVS corrupted, device falls back to defaults
- Try safe boot mode to clear settings

### Bluetooth not advertising

- Some Arduino ESP32 cores lack BLE support
- Check Serial: "[BLE] Headers unavailable" or "[BLE] Advertising HAZE CYBERDECK"
- Bluetooth Tools menu shows "BLE N/A" if unavailable

### Face animation has ghosting

- This is a known issue in v0.6.1
- Upgrade to v0.6.2 which includes display transaction fixes
- Or check that LovyanGFX library is up to date

## Firmware Patches

See `PATCHES.md` for detailed information on safety fixes applied in v0.6.2:

1. Preferences initialization validation
2. BLE null-safety checks
3. Display write transaction safety
4. WiFi scan cleanup
5. Setup initialization order
6. Remote control feedback clarification

## Architecture

### State Machine

- **currentScreen**: Active screen ID
- **screenDirty**, **contentDirty**, **statusBarDirty**, **faceDirty**: Partial redraw flags
- **idleMode**: Deep sleep with animated display
- **inputAction**: Result of encoder/button polling

### Redraw Scheduler

- Full redraw if `screenDirty` or `contentDirty`
- Partial redraws for status bar and face animation
- Idle mode draws at ~2 Hz for power savings
- All SPI operations wrapped in `startWrite()`/`endWrite()` transactions

### Manager Functions

- `updateWifiManager()`: Handle WiFi state machine
- `updateClockManager()`: Sync NTP, update status bar
- `updateSdManager()`: Detect card insertion/removal
- `updateIdleMode()`: Activate after timeout
- `updateFace()`: Animate blinking and eye motion

## Performance

- **Loop time**: ~5–10ms (idle), ~20–50ms (active rendering)
- **Heap usage**: ~150 KB free (typical)
- **Display refresh**: ~60 FPS internal, ~30 FPS practical (WiFi/BLE interference)

## Known Limitations

- ✗ CC1101 RF module not implemented (reserved for future use)
- ✗ NFC not implemented (missing PN532 support)
- ✗ Battery percentage not displayed
- ✗ Audio/IR not implemented
- ✗ Remote control not yet functional
- ✗ SCREEN_KEYBOARD and SCREEN_NOTES are placeholder stubs
- ✗ Mini apps are framework only (not fully implemented)

## Future Work

- [ ] Implement RF scanning/transmission (CC1101)
- [ ] Add NFC tag reading
- [ ] Populate mini apps (calculator, stopwatch, etc.)
- [ ] Implement remote HTTP control
- [ ] Add notes app with SD card persistence
- [ ] Battery voltage display
- [ ] WS2812 LED status indicator
- [ ] Bluetooth HID media control (if feasible)

## License

This firmware is provided as-is for educational and personal use. No warranties. Use at your own risk.

## Credits

- **LovyanGFX**: ST7789 display driver library
- **ESP32 Arduino Core**: Espressif Systems
- **LilyGo**: Hardware design and reference firmware

## Support & Issues

For questions or bug reports:

1. Check Serial output (baud 115200) for diagnostic messages
2. Verify pin configuration matches your board variant
3. Test with safe boot mode (see Troubleshooting)
4. Consult PATCHES.md for recent changes

---

**Version**: 0.6.2  
**Last Updated**: 2025-09-08  
**Hardware Target**: LilyGo T-Embed CC1101 v1.0
