# AeroBeam Tech: Hybrid LiFi / RF Audio Link

Audio is sent from one ESP32 to another over a laser beam (LiFi). A camera-guided
pan/tilt gimbal keeps the laser on the receiver's solar panel. When the beam drifts off
target, the receiver switches to a backup radio link (ESP-NOW), so the audio keeps playing.

##  Demo

📹 **[Watch the AeroBeam Tech Simulation Video](https://drive.google.com/file/d/1SuhYatWLwRtfAIDrSVdV4MZzIzGSnFBn/view?usp=sharing)**

   **[▶ Try the live simulation](https://mohammedfiromsa06-design.github.io/AeroBeam-Tech-Hybrid-LiFi-RF/simulation/lifi_tracking_link.html)**
   in your browser, with no hardware needed.

📹 **[Watch this real life demo here](https://drive.google.com/file/d/1WU9ODq8sNfBRY5bi-inhxa71GcVHW303/view?usp=sharing)**
   

```
 Laptop (tracker_bridge.py)
   │  camera stream ◄── phone (IP Webcam) watching the ArUco marker on the panel
   │
   ├── USB serial: servo angles ──► SENDER ESP32 ── laser ─────────► solar panel ─► speaker
   │                                    │  (pan/tilt + mic)                (LiFi path)
   │                                    └── ESP-NOW audio ──┐
   │                                                        ▼
   └── Wi-Fi UDP: "use LiFi / use RF" ─────────────► RECEIVER ESP32 ─► DAC GPIO 25 ─► speaker
                                                     (OLED status)        (RF path)
```

## How the failover works

The receiver has no beam sensor. The laptop decides from the camera image:

1. `tracker_bridge.py` finds ArUco marker **ID 0** (dictionary `DICT_4X4_50`) on the panel and
   steers the gimbal to keep it on the crosshair.
2. It measures the **offset**, the distance in pixels from the crosshair to the marker.
   - If the offset is more than `BEAM_LOST_PX`, or the marker isn't visible, it switches to **RF** immediately.
   - It switches back to **LiFi** once the offset has stayed within `BEAM_RELOCK_PX` for `RELOCK_HOLD_S`.
3. It sends the receiver the current mode over UDP every 100 ms.
4. The receiver plays ESP-NOW audio on its DAC only in RF mode. If it hears nothing from
   the laptop for 1 s, it switches to RF on its own.

## Repository layout

| Path | What it is |
|---|---|
| `tracker_bridge.py` | Laptop script: camera tracking, servo control, LiFi/RF decision |
| `requirements.txt` | Python packages for the script |
| `firmware/platformio.ini` | One PlatformIO project with two environments: `sender` and `receiver` |
| `firmware/src/sender/main.cpp` | Laser-end board: servos (GPIO 18/19), microphone (GPIO 34) → ESP-NOW |
| `firmware/src/receiver/main.cpp` | Panel-end board: UDP link commands, ESP-NOW audio → DAC (GPIO 25), SH1106 OLED (GPIO 21/22) |
| `firmware/include/audio_packet.h` | Audio packet format shared by both boards |
| `firmware/include/wifi_credentials.example.h` | Template for your Wi-Fi name and password |
| `simulation/lifi_tracking_link.html` | Interactive browser simulation of the tracking and LiFi/RF failover (no hardware needed) |

## What you need

- 2 × ESP32 (uPesy ESP32 WROOM)
- Pan/tilt servo gimbal and laser module (sender side), MAX4466 microphone
- Solar panel (laser audio output), 1.3" SH1106 I²C OLED, speaker/amplifier (receiver side)
- A phone running an IP camera app (e.g. IP Webcam), mounted on the gimbal
- A **2.4 GHz** Wi-Fi network shared by the laptop, the phone and both ESP32s
- VS Code with the PlatformIO extension, and Python 3

## Setup

### 1. Wi-Fi credentials

Copy `firmware/include/wifi_credentials.example.h` to `firmware/include/wifi_credentials.h`,
then fill in your network name and password. Git ignores this file, so the password stays
off GitHub.

Both boards join this network. The receiver needs it to get commands from the laptop.
The sender needs it so that its ESP-NOW audio goes out on the same Wi-Fi channel as the receiver.

### 2. Flash the boards

Plug in **one board at a time**. In VS Code, pick the environment and the COM port in the
bottom status bar, then click Upload. Or use the terminal, from the `firmware` folder:

```
pio run -e sender   -t upload --upload-port COMx   # laser-end board
pio run -e receiver -t upload --upload-port COMy   # panel-end board
```

Don't upload without choosing an environment. That flashes both firmwares onto the same
board, one after the other.

To find the COM port, look in **Device Manager → Ports (COM & LPT)** for "USB-SERIAL CH340"
or "CP210x". The "Standard Serial over Bluetooth link" entries are not the ESP32. If no USB
port appears, install the CH340 or CP210x driver. If the upload stalls at `Connecting...`,
hold the **BOOT** button.

Each board prints its firmware name at boot (`AeroBeam SENDER firmware` /
`AeroBeam RECEIVER firmware`) at 115200 baud.

**Receiver MAC:** the sender sends audio to the MAC address in `receiverMacAddress`
(`firmware/src/sender/main.cpp`). The receiver's upload log prints its MAC near the
start. If the two don't match, update the address in the sender code.

### 3. Python script

```
python -m venv .venv
.venv\Scripts\activate
pip install -r requirements.txt
```

Use `pyserial`, not the unrelated `serial` package. If both are installed, `import serial` breaks.

Edit the settings at the top of `tracker_bridge.py`:

| Setting | Set it to |
|---|---|
| `SERIAL_PORT` | The sender board's USB COM port |
| `RECEIVER_IP` | The IP shown on the receiver's OLED |
| `STREAM_URL` | The phone's camera stream URL |

Then run:

```
python tracker_bridge.py
```

Press **q** in the video window to quit. On quit, the script tells the receiver to switch to RF.

## Calibrating the failover boundary

The boundary values depend on the camera, the distance and the panel size:

1. Run the script with the beam on the panel.
2. Move the gimbal or the target slowly until the panel stops receiving the beam.
   Note the **Offset** shown in the video window.
3. Set `BEAM_LOST_PX` a little below that value, and `BEAM_RELOCK_PX` about 15 px lower.

Keep `BEAM_LOST_PX` above `DEADBAND` (12). Otherwise the tracker's normal resting error
triggers RF failover.

## Simulation

[Open it online](https://mohammedfiromsa06-design.github.io/AeroBeam-Tech-Hybrid-LiFi-RF/simulation/lifi_tracking_link.html),
or open `simulation/lifi_tracking_link.html` locally in a browser. You can watch the gimbal track the marker, see the
audio signal at each stage, and see the link switch to RF when the beam is lost. It's useful
for demos and for understanding the system before setting up the hardware.

It is a simplified model, so some details differ from the real build.
For example, it uses ArUco marker ID 7 (the real script tracks ID 0).

## Status displays

**Receiver OLED**

- **Link:** `LiFi (Laser)` or `RF (ESP-NOW)`
- **Host:** `waiting` (no command yet), `off N px` (normal), `target lost`, or `TIMEOUT` (no command for 1 s)
- **IP:** the address to put in `RECEIVER_IP`. `WiFi: connecting...` means the board hasn't joined the network.

**Laptop window:** the ring around the crosshair marks the failover boundary. It is green
on LiFi and red on RF. The top lines show the active link, the offset and the servo angles.

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| Servos don't move | Wrong `SERIAL_PORT`, or the wrong `serial` package installed (see step 3) |
| OLED stuck on `WiFi: connecting...` | Wrong credentials, or the network is 5 GHz only |
| OLED shows `Host: waiting` / `TIMEOUT` | Wrong `RECEIVER_IP`, laptop on a different network, or a firewall blocking outgoing UDP port 4210 |
| No RF audio | Receiver MAC doesn't match `receiverMacAddress`, or the sender isn't on Wi-Fi (its serial monitor should print `Wi-Fi joined, ESP-NOW on channel N`) |
| Link flickers between LiFi and RF | Make the gap between `BEAM_LOST_PX` and `BEAM_RELOCK_PX` bigger, or increase `RELOCK_HOLD_S` |
| VS Code: "Cannot read properties of undefined (reading 'id')" | PlatformIO extension glitch. Run **Developer: Reload Window**. |

## License

Released under the [MIT License](LICENSE).
