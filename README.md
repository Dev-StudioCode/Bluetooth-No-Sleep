# Bluetooth No Sleep (Bluetooth Audio Keep-Alive)

[![Platform](https://img.shields.io/badge/Platform-Windows%2010%20%7C%2011-blue.svg)](https://microsoft.com/windows)
[![Windhawk Mod](https://img.shields.io/badge/Windhawk-Mod%20Available-orange.svg)](https://windhawk.net/)
[![Audio API](https://img.shields.io/badge/API-WASAPI%20Shared%20Mode-green.svg)](https://learn.microsoft.com/en-us/windows/win32/coreaudio/wasapi)
[![License](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

A lightweight Windows background utility and Windhawk mod designed to prevent Bluetooth audio endpoints (headphones, soundbars, DACs, and wireless speakers) from entering aggressive low-power sleep states, disconnecting on idle timeout, or truncating the initial seconds of audio playback.

---

## Overview

Bluetooth audio devices frequently employ aggressive internal power-management timers to conserve battery life. While beneficial on paper, these power-saving behaviors introduce practical operational issues in daily desktop environments:

1. **Audio Truncation (Startup Latency):** When no audio is rendered for a short interval, the device enters a sleep state and disables its internal Digital-to-Analog Converter (DAC). When playback resumes, the device requires 1 to 2 seconds to renegotiate the wireless link and wake the amplifier, causing the first few seconds of notifications, speech, or media playback to be dropped.
2. **Hardware Popping and Transient Static:** Abrupt DAC initialization upon incoming audio streams often produces audible transients, clicks, or static bursts.
3. **Inactivity Disconnections:** Many wireless peripherals enforce automatic shutdown after 5 to 10 minutes of silence, requiring manual hardware power cycling and reconnection.

---

## Technical Approach

### Analysis of Existing Approaches
Virtual audio routing solutions such as VoiceMeeter maintain an active audio stream handle indefinitely, continuously streaming zeroed audio packets. While this keeps Bluetooth hardware powered on and eliminates sleep latency, running a virtual mixing console adds overhead, modifies device routing, and introduces unnecessary architectural complexity.

### The Bluetooth No Sleep Solution
Bluetooth No Sleep addresses this issue using a minimal Core Audio session architecture:

* **Inaudible Digital Silence Stream:** Emits continuous zero-amplitude PCM packets (`0x00`) to maintain an active streaming session without introducing audible artifacts or modifying source volume.
* **WASAPI Shared Mode:** Operates strictly within Windows Audio Session API (WASAPI) Shared Mode (`AUDCLNT_SHAREMODE_SHARED`). It does not request exclusive hardware access, allowing system sounds, media players, and voice communication applications to operate normally without contention.
* **Dynamic Endpoint Tracking:** Implements the `IMMNotificationClient` interface to monitor hardware state transitions and default endpoint reassignments (`OnDefaultDeviceChanged`, `OnDeviceStateChanged`). When a Bluetooth device connects, disconnects, or switches roles, the worker gracefully releases existing COM pointers and binds to the active device.
* **Efficient Scheduling:** Uses buffer-proportional thread scheduling (`Sleep(buffer_duration / 2)`) to yield CPU cycles, maintaining processor utilization below 0.1%.

---

## Architecture

```
                                +-----------------------------+
                                |    Windows Audio Endpoint   |
                                |  (Bluetooth Headset / DAC)  |
                                +--------------^--------------+
                                               |
                                    [WASAPI Shared Stream]
                                               |
+--------------------------+       +-----------+-------------+
| IMMNotificationClient    | ----> | IAudioRenderClient      |
| - OnDefaultDeviceChanged |       | - Dispatches 0x00 PCM   |
| - OnDeviceStateChanged   |       +-----------^-------------+
+--------------------------+                   |
                                   +-----------+-------------+
                                   | Audio Keep-Alive Worker |
                                   | - Detached thread       |
                                   | - Minimal CPU footprint |
                                   +-------------------------+
```

### Core Execution Flow
1. **Endpoint Acquisition:** Queries `IMMDeviceEnumerator` for the active multimedia audio rendering endpoint (`eRender`, `eConsole` / `eMultimedia`).
2. **Session Initialization:** Activates `IAudioClient` with standard negotiated PCM parameters (e.g., 44.1 kHz / 48 kHz stereo, 16-bit PCM).
3. **Buffer Dispatch:** Uses `IAudioRenderClient` to obtain buffer space, fills memory with digital silence via `memset`, and releases the buffer back to the audio engine.
4. **Lifecycle Management:** Traps endpoint invalidation errors (`AUDCLNT_E_DEVICE_INVALIDATED`) and notification callbacks to handle disconnects and device switching cleanly.

---

## Features

- **Continuous Audio Stream Maintenance:** Prevents DAC power-down and radio idle timeouts across connected wireless endpoints.
- **Zero Playback Truncation:** Ensures immediate, latency-free output for notifications and brief system alerts.
- **Dynamic Endpoint Adaptation:** Responds to system-wide default device changes and Bluetooth state changes in real time.
- **Low Overhead:** Built entirely in native C++ using Windows Core Audio APIs without external frameworks or runtimes.
- **Windhawk Integration:** Implements the Windhawk "Mod as a Tool" model, executing inside `windhawk.exe` without injecting hooks into system processes such as `explorer.exe` or `audiosrv.dll`.
- **Standalone Compatibility:** Can be compiled and deployed as an independent background process or Windows system tray utility.

---

## Comparison of Solutions

| Feature | Bluetooth No Sleep | VoiceMeeter | SoundKeeper | Silent BT Awake |
| :--- | :---: | :---: | :---: | :---: |
| Keep-Alive Digital Silence | Yes | Yes | Yes | Yes |
| Native Windhawk Mod | Yes | No | No | No |
| Resource Utilization | Minimal (<0.1% CPU) | Moderate to High | Minimal | Low |
| System Audio Routing Alterations | None | Requires Bus Routing | None | None |
| Dynamic Device Recovery | Automatic (WASAPI) | Manual / Restart | Automatic | Periodic Loop |
| Audio Session Mode | Shared Mode | Direct / Exclusive / Shared | Shared Mode | Shared Mode |

---

## Installation and Usage

### Option 1: Windhawk Mod (Recommended)
1. Install [Windhawk](https://windhawk.net/) on Windows 10 or 11.
2. In the Windhawk Mod Manager, navigate to **Mods** and search for **Bluetooth Audio Keep-Alive** (or import the local source file).
3. Click **Details** and select **Install**.
4. The mod runs immediately in the background under the main Windhawk process.

### Option 2: Standalone Background Process
1. Download the compiled executable from the [Releases](https://github.com/Dev-StudioCode/Bluetooth-No-Sleep/releases) section.
2. Place `BluetoothNoSleep.exe` in a preferred utility path.
3. Configure the binary to start automatically via the Windows Startup folder (`shell:startup`) or Windows Task Scheduler.

---

## Building from Source

### Requirements
* Windows 10 or 11 SDK (version 10.0.19041.0 or higher)
* C++20 compatible compiler (Microsoft Visual C++ 2022 or Clang-cl)
* CMake 3.20 or newer (for standalone builds)

### Build Instructions (Standalone)
```powershell
# Clone the repository
git clone https://github.com/Dev-StudioCode/Bluetooth-No-Sleep.git
cd Bluetooth-No-Sleep

# Generate build configuration
cmake -B build -DCMAKE_BUILD_TYPE=Release

# Compile release binaries
cmake --build build --config Release
```

---

## Configuration Options

- **Target Endpoints (`targetMode`):**
  - `bluetoothOnly` (Default): Restricts keep-alive streaming strictly to the default playback device when connected over Bluetooth, preserving power on internal laptop speakers or monitors.
  - `defaultDevice`: Keeps the current default Windows playback endpoint active regardless of hardware bus type.
  - `allDevices`: Maintains simultaneous keep-alive streams across all connected playback endpoints.
- **Silence Generation Mode (`silenceMode`):**
  - `zero` (Default): Continuous zero-amplitude PCM buffer (`0x00`).
  - `dither`: Sub-audible 1-LSB dither (~ -90 dBFS) for hardware DACs that require subtle bit oscillation to stay active.
- **Keep-Alive Strategy (`keepAliveStrategy`):**
  - `continuous` (Default): Keeps audio streams active whenever the targeted endpoint is connected.
  - `timeoutGate`: Pauses keep-alive streaming after a period of user inactivity (`idleTimeoutMinutes`), permitting natural Bluetooth battery conservation and standard Windows system idle sleep (`powercfg /requests`). Resumes instantly upon user input.
- **Inactivity Timeout Duration (`idleTimeoutMinutes`):**
  - Number of minutes of user inactivity before pausing the stream (Range: 1–240 minutes, default: 30 minutes).

---

## Contributing

Contributions, bug reports, and pull requests are welcome. Please ensure changes follow modern C++ standards, adhere to Windows Core Audio API best practices, and maintain minimal resource footprint.

---

## License

This project is licensed under the [MIT License](LICENSE).
