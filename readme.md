# ESP32-CAM Robot

A Wi-Fi controlled ESP32-CAM robot with live camera streaming, independent motor PWM control, synchronized motor experiments, camera settings, Wi-Fi fallback AP mode, browser OTA firmware updates, debug/event logging, log export, and a UART0 USB serial console.

## Current feature set

### Motor control

- Left motor MOSFET gate: **GPIO13**
- Right motor MOSFET gate: **GPIO12**
- Full PWM range for each motor: **0 to 255**
- Independent left/right speed sliders
- Exact **-1 / +1** PWM buttons
- **Sync motors** checkbox:
  - When enabled, changing either motor changes both motors to the same PWM value.
  - Enabling Sync uses the current left motor value as the initial reference.
- Short full-power startup kick to help overcome motor stiction.
- Forward, left, right, and stop controls.
- Reverse is intentionally disabled because the present PCB uses one low-side MOSFET per motor rather than an H-bridge.

At very low PWM values a motor may buzz without turning. This is useful for experiments because students can identify each motor's real starting threshold. Do not leave a stalled motor powered for long periods.

## Motor power hardware

The current PCB has one MOSFET per motor. Each MOSFET switches a motor in one direction only.

Because the circuit is not an H-bridge:

- Forward: supported
- Left/right steering: supported by stopping one wheel
- Stop: supported
- Reverse: not supported

For true bidirectional control, use an H-bridge such as a DRV8833 or TB6612FNG.

## Camera

The ESP32-CAM serves an MJPEG stream on port **81**.

The Settings sidebar provides:

- Resolution:
  - QQVGA 160x120
  - QVGA 320x240
  - VGA 640x480
  - SVGA 800x600
  - XGA 1024x768
  - SXGA 1280x1024
  - UXGA 1600x1200
- Display rotation:
  - 0 degrees
  - 90 degrees clockwise
  - 180 degrees
  - 270 degrees clockwise
- JPEG quality
- Brightness
- Contrast
- Saturation
- Vertical flip
- Horizontal mirror
- Reset camera defaults

High resolutions require PSRAM.

### Rotation behavior

Vertical flip and horizontal mirror are sensor settings. They are applied by the camera.

The 0/90/180/270 **Display rotation** setting rotates the stream in the browser. This avoids expensive real-time JPEG decoding, rotation, and re-encoding on the ESP32.

The selected rotation is stored in the browser with `localStorage`, so it is remembered by that browser.

## Web user interface

The robot page has two collapsible sidebars.

### Left sidebar: Settings

Includes:

- Camera controls
- USB Serial console controls
- Serial baud selector
- Select port / Connect / Disconnect
- Live UART capture terminal
- Auto-scroll
- Clear capture
- Save capture
- Command entry

### Right sidebar: Robot Debug

Shows:

- Motion state
- Left/right target PWM
- Left/right current output PWM
- Network mode
- Station status
- SSID
- Robot IP
- AP clients
- Wi-Fi RSSI
- Uptime
- Firmware build date/time
- Latest debug message
- 64-event retained event history
- Export debug log
- Clear debug log
- Browser OTA firmware update

Newest debug events are shown at the top.

## UART0 / USB serial console

ESP32-CAM UART0:

- Baud: **115200**
- Data bits: 8
- Parity: none
- Stop bits: 1

### Wiring

| ESP32-CAM | USB-TTL adapter |
|---|---|
| U0T / GPIO1 | RX |
| U0R / GPIO3 | TX |
| GND | GND |

Use a **3.3V TTL-compatible** serial adapter. Do not connect RS-232 voltage levels directly to the ESP32.

The console accepts:

- `help`
- `status`
- `log`
- `camera`
- `stop`

`stop` performs an emergency motor stop.

### Browser Web Serial limitation

Chrome/Edge Web Serial requires a secure context. A page opened directly from:

`http://192.168.x.x`

is normally not considered a secure context, so direct COM-port access may be blocked even though the robot UI itself works.

A standalone helper is included:

`ESP32_Robot_USB_Serial_Console.html`

For reliable Web Serial access, serve it from localhost:

```bash
python -m http.server 8000
```

Then open:

`http://localhost:8000/ESP32_Robot_USB_Serial_Console.html`

in Chrome or Edge.

## Wi-Fi behavior

At boot the robot:

1. Tries the configured Wi-Fi network for 10 seconds.
2. If successful, it runs in normal STA mode.
3. If it cannot connect, it creates its own fallback access point.
4. While fallback AP mode is active, it periodically retries the configured Wi-Fi.

### Fallback AP

- SSID: `ESP32-Robot-XXXX`
- Password: `88888888`
- Default AP address: `http://192.168.4.1`

`XXXX` is generated from part of the ESP32 chip ID so several classroom robots can have different SSIDs.

If normal Wi-Fi works, the robot is usually reached at the DHCP address shown in the debug sidebar or UART console.

## Debug event system

The ESP32 keeps the newest **64 events** in a circular buffer.

Examples:

- Boot started
- Camera initialized
- Wi-Fi connection state
- Fallback AP start
- Motor commands
- PWM changes
- Camera settings
- OTA start/success/failure
- Serial emergency stop

The browser requests only events newer than the last event ID it has seen.

This prevents fast button events from being lost between status polls.

## Export debug log

The Debug sidebar can save a `.txt` file containing:

- Current robot state
- PWM targets and outputs
- Network state
- SSID/IP
- Wi-Fi RSSI
- Uptime
- Build timestamp
- Latest message
- Event history, newest first

The export uses Windows-friendly CRLF line endings and UTF-8 text.

## OTA firmware update

The Debug sidebar contains a browser OTA uploader.

OTA password:

`88888888`

Before OTA works, the ESP32 must be flashed once by USB using an OTA-capable partition scheme.

A suitable Arduino IDE choice is typically:

`Minimal SPIFFS (1.9MB APP with OTA / 190KB SPIFFS)`

Exact wording may vary with the ESP32 Arduino core version.

### OTA workflow

1. Compile the current sketch in Arduino IDE.
2. Use **Sketch -> Export Compiled Binary**.
3. Select the main application file:
   - `YourSketch.ino.bin`
4. Do not select:
   - `*.bootloader.bin`
   - `*.partitions.bin`
   - `*.merged.bin`
5. Open the robot Debug sidebar.
6. Enter OTA password `88888888`.
7. Select the application `.ino.bin`.
8. Upload.
9. Motors are stopped before firmware writing.
10. After a successful update, the ESP32 reboots.

## Arduino sketch folder rule

This is important.

Arduino compiles **every `.ino` file in the same sketch folder**.

Do not put multiple complete versions of this robot firmware in the same directory.

Correct:

```text
ESP32_Robot_Complete/
  ESP32_Robot_Complete.ino
```

Incorrect:

```text
ESP32_Robot_Complete/
  ESP32_Robot_Complete.ino
  old_robot.ino
  test_robot.ino
```

The incorrect layout causes errors such as:

- redefinition of `ssid`
- redefinition of `setup()`
- redefinition of `loop()`
- redefinition of camera handlers
- redefinition of OTA variables

Keep old firmware versions in separate folders or rename them to `.txt`.

## Recommended Arduino settings

Typical settings for AI-Thinker ESP32-CAM:

- Board: AI Thinker ESP32-CAM
- Upload speed: 115200 if higher speeds are unreliable
- CPU frequency: 240 MHz
- Flash frequency: 40 MHz
- OTA-capable partition scheme
- Correct COM port

For USB flashing, GPIO0 normally needs to be held low during boot, depending on the programmer/adapter arrangement.

## Upload troubleshooting

If flashing connects at low speed and then fails after switching to a high baud rate, reduce **Upload Speed** to 115200.

Disconnect or stop motors during firmware upload and use a stable 5V supply.

## Network troubleshooting

If the robot cannot join configured Wi-Fi:

- Check SSID spelling
- Check password
- Check 2.4 GHz availability
- Check signal level
- Check router/AP security compatibility

The debug log reports status such as:

- connected
- SSID not found
- connection/authentication failed
- connection lost
- disconnected

If STA mode fails, connect directly to the fallback `ESP32-Robot-XXXX` network and open `192.168.4.1`.

## Camera troubleshooting

If higher resolutions fail:

- Confirm PSRAM is detected.
- Try VGA or QVGA.
- Increase JPEG quality number to reduce image size.
- Check power stability.
- Lower resolution if the camera stream becomes slow.

Remember: a lower JPEG quality number means higher image quality and generally larger JPEG frames.

## Security notes

The current educational configuration uses `88888888` for:

- Fallback AP password
- OTA password

This is convenient for a workshop/classroom but should be changed for an untrusted environment.

The robot control page is HTTP, not HTTPS.

Anyone who can reach the robot network may be able to control the robot unless additional authentication is added.

## Main files

- `ESP32_Robot_Complete.ino` - complete ESP32-CAM robot firmware
- `ESP32_Robot_USB_Serial_Console.html` - standalone localhost Web Serial terminal
- `README.md` - this documentation
- `README.html` - browser-friendly version of the documentation

## Suggested classroom experiments

- Find the minimum PWM where each unloaded wheel starts.
- Compare the left and right starting thresholds.
- Enable Sync and compare straight-line behavior.
- Intentionally offset one motor by 1-10 PWM counts and observe steering.
- Compare camera quality versus streaming responsiveness.
- Compare Wi-Fi RSSI with stream smoothness.
- Compare motor behavior with wheels lifted versus robot on the floor.
- Add encoders/Hall sensors later and compare requested PWM with actual RPM.

## Future extensions

Useful next steps:

- Left/right wheel encoders
- Live RPM
- Wheel linear speed in m/s
- Distance estimation
- Closed-loop speed control
- Battery voltage monitoring
- H-bridge for reverse
- Saved Wi-Fi configuration from the browser
- Authentication for robot controls
- Downloadable CSV experiment data
