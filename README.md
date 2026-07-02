# MeshCore Ethernet Firmware (Elecrow ThinkNode M7)

This is a fork of [MeshCore](https://github.com/ripplebiz/MeshCore) that adds **wired Ethernet** support to the **Elecrow ThinkNode M7** (ESP32-S3 + LR1110) companion firmware, alongside the existing Wi-Fi transport.

The board's onboard **WCH CH390** SPI Ethernet controller is brought up as a second network interface. Both Wi-Fi and Ethernet share the same TCP companion server, so a MeshCore client can connect over either link at **`<device-ip>:5000`**.

## What this fork adds

- **Dual-stack networking**: Wi-Fi and Ethernet are both active by default. The device gets an IP on each interface (typically via DHCP) and answers the companion TCP server on **port 5000** on either.
- **CH390 SPI Ethernet driver**: uses [`meshtastic/ESP32-CH390`](https://github.com/meshtastic/ESP32-CH390) and registers the CH390 as an `esp_netif` on the shared lwIP stack. No client-side changes are needed — it looks like a normal network interface.
- Target build environment: **`ThinkNode_M7_companion_radio_wifi`**.

## Hardware pin reference (ThinkNode M7)

The ThinkNode M7 is an ESP32-S3 board with an onboard **LR1110** LoRa transceiver and an onboard **CH390** SPI Ethernet controller. Everything is on the PCB — no manual wiring is required — but the full pin map is documented here for reference and for anyone porting the firmware. All values come from [`variants/thinknode_m7/platformio.ini`](./variants/thinknode_m7/platformio.ini) and [`variants/thinknode_m7/pins_arduino.h`](./variants/thinknode_m7/pins_arduino.h).

The two SPI peripherals sit on **separate SPI hosts** so they never share a bus:

- **LR1110 LoRa** → default `SPIClass` host = **`HSPI` / `SPI3_HOST`**
- **CH390 Ethernet** → **`SPI2_HOST` / FSPI**

### LR1110 LoRa radio (SPI3 / HSPI)

| Signal        | GPIO | Build flag      |
| ------------- | ---- | --------------- |
| SCLK          | 11   | `P_LORA_SCLK`   |
| MISO          | 9    | `P_LORA_MISO`   |
| MOSI          | 10   | `P_LORA_MOSI`   |
| NSS / CS      | 12   | `P_LORA_NSS`    |
| BUSY          | 13   | `P_LORA_BUSY`   |
| DIO1 (IRQ)    | 38   | `P_LORA_DIO_1`  |
| RESET         | 39   | `P_LORA_RESET`  |
| TX LED (blue) | 46   | `P_LORA_TX_LED` |

> The RF switch is driven by the LR1110's internal DIO5/DIO6, and the TCXO is on DIO3 @ 1.8 V (`LR11X0_DIO3_TCXO_VOLTAGE`).

### CH390 Ethernet (SPI2 / FSPI)

| Signal | GPIO | Build flag     |
| ------ | ---- | -------------- |
| SCLK   | 47   | `ETH_SCLK_PIN` |
| MISO   | 14   | `ETH_MISO_PIN` |
| MOSI   | 48   | `ETH_MOSI_PIN` |
| CS     | 21   | `ETH_CS_PIN`   |
| INT    | 45   | `ETH_INT_PIN`  |
| RESET  | —    | not used (-1)  |

### I2C, buttons & LEDs

| Signal             | GPIO | Notes                                             |
| ------------------ | ---- | ------------------------------------------------- |
| I2C SDA            | 17   | `Wire` — PMU / RTC bus                             |
| I2C SCL            | 18   | `Wire` — PMU / RTC bus                             |
| User button        | 4    | `PIN_USER_BTN_ANA` (analog)                       |
| Status LED (green) | 3    | `PIN_STATUS_LED`, active-low (`LED_STATE_ON=LOW`) |
| TX LED (blue)      | 46   | shared with `P_LORA_TX_LED`                       |

## Build & flash

Uses [PlatformIO](https://docs.platformio.org). Build and upload the Wi-Fi/Ethernet companion firmware:

```bash
pio run -e ThinkNode_M7_companion_radio_wifi -t upload
```

Then connect a [MeshCore client](#-meshcore-clients) to the device over Wi-Fi (TCP), or plug in an Ethernet cable and connect to the wired IP — both listen on port `5000`.

## Configuring your own Wi-Fi credentials

Wi-Fi credentials are compiled into the firmware as build flags. Edit them in [`variants/thinknode_m7/platformio.ini`](./variants/thinknode_m7/platformio.ini) under the `[env:ThinkNode_M7_companion_radio_wifi]` section:

```ini
  -D WIFI_SSID='"myssid"'
  -D WIFI_PWD='"mypwd"'
```

Replace `myssid` / `mypwd` with your network's SSID and password (keep the `'"..."'` quoting exactly as shown), then rebuild and flash.

> ⚠️ **Do not commit your real Wi-Fi credentials.** These values are placeholders on purpose. Change them locally to build, but leave the committed copy as `myssid` / `mypwd` (e.g. use `git update-index --assume-unchanged` or simply avoid staging that change).

### Network modes

- **Dual-stack (default)** — leave the config as-is: both Wi-Fi and Ethernet come up.
- **Ethernet only** — remove/comment the `WIFI_SSID` and `WIFI_PWD` lines (keep the `USE_CH390D` block).
- **Wi-Fi only** — remove the CH390 block (`USE_CH390D` + `ETH_*_PIN` flags) and the `ESP32-CH390` entry from `lib_deps`.

---

## About MeshCore

MeshCore is a lightweight, portable C++ library that enables multi-hop packet routing for embedded projects using LoRa and other packet radios. It is designed for developers who want to create resilient, decentralized communication networks that work without the internet.

## 🔍 What is MeshCore?

MeshCore now supports a range of LoRa devices, allowing for easy flashing without the need to compile firmware manually. Users can flash a pre-built binary using tools like Adafruit ESPTool and interact with the network through a serial console.
MeshCore provides the ability to create wireless mesh networks, similar to Meshtastic and Reticulum but with a focus on lightweight multi-hop packet routing for embedded projects. Unlike Meshtastic, which is tailored for casual LoRa communication, or Reticulum, which offers advanced networking, MeshCore balances simplicity with scalability, making it ideal for custom embedded solutions, where devices (nodes) can communicate over long distances by relaying messages through intermediate nodes. This is especially useful in off-grid, emergency, or tactical situations where traditional communication infrastructure is unavailable.

## ⚡ Key Features

* Multi-Hop Packet Routing
  * Devices can forward messages across multiple nodes, extending range beyond a single radio's reach.
  * Supports up to a configurable number of hops to balance network efficiency and prevent excessive traffic.
  * Nodes use fixed roles where "Companion" nodes are not repeating messages at all to prevent adverse routing paths from being used.
* Supports LoRa Radios – Works with Heltec, RAK Wireless, and other LoRa-based hardware.
* Decentralized & Resilient – No central server or internet required; the network is self-healing.
* Low Power Consumption – Ideal for battery-powered or solar-powered devices.
* Simple to Deploy – Pre-built example applications make it easy to get started.

## 🎯 What Can You Use MeshCore For?

* Off-Grid Communication: Stay connected even in remote areas.
* Emergency Response & Disaster Recovery: Set up instant networks where infrastructure is down.
* Outdoor Activities: Hiking, camping, and adventure racing communication.
* Tactical & Security Applications: Military, law enforcement, and private security use cases.
* IoT & Sensor Networks: Collect data from remote sensors and relay it back to a central location.

## 🚀 How to Get Started

- Watch the [MeshCore QuickStart Playlist](https://www.youtube.com/watch?v=iaFltojJrAc&list=PLshzThxhw4O4WU_iZo3NmNZOv6KMrUuF9) by The Comms Channel
- Watch the [MeshCore Technical Presentation](https://www.youtube.com/watch?v=OwmkVkZQTf4) by Liam Cottle.
- Read through our [Frequently Asked Questions](./docs/faq.md) and [Documentation](https://docs.meshcore.io).
- Flash the MeshCore firmware on a supported device.
- Connect with a supported client.

For developers:

- Install [PlatformIO](https://docs.platformio.org) in [Visual Studio Code](https://code.visualstudio.com).
- Clone and open the MeshCore repository in Visual Studio Code.
- See the example applications you can modify and run:
  - [Companion Radio](./examples/companion_radio) - For use with an external chat app, over BLE, USB or Wi-Fi.
  - [KISS Modem](./examples/kiss_modem) - Serial KISS protocol bridge for host applications. ([protocol docs](./docs/kiss_modem_protocol.md))
  - [Simple Repeater](./examples/simple_repeater) - Extends network coverage by relaying messages.
  - [Simple Room Server](./examples/simple_room_server) - A simple BBS server for shared Posts.
  - [Simple Secure Chat](./examples/simple_secure_chat) - Secure terminal based text communication between devices.
  - [Simple Sensor](./examples/simple_sensor) - Remote sensor node with telemetry and alerting.

The Simple Secure Chat example can be interacted with through the Serial Monitor in Visual Studio Code, or with a Serial USB Terminal on Android.

## ⚡️ MeshCore Flasher

We have prebuilt firmware ready to flash on supported devices.

- Launch https://meshcore.io/flasher
- Select a supported device
- Flash one of the firmware types:
  - Companion, Repeater or Room Server
- Once flashing is complete, you can connect with one of the MeshCore clients below.

## 📱 MeshCore Clients

**Companion Firmware**

The companion firmware can be connected to via BLE, USB or Wi-Fi depending on the firmware type you flashed.

- Web: https://app.meshcore.nz
- Android: https://play.google.com/store/apps/details?id=com.liamcottle.meshcore.android
- iOS: https://apps.apple.com/us/app/meshcore/id6742354151?platform=iphone
- NodeJS: https://github.com/liamcottle/meshcore.js
- Python: https://github.com/fdlamotte/meshcore-cli

**Repeater and Room Server Firmware**

The repeater and room server firmware can be set up via USB in the web config tool.

- https://config.meshcore.io

They can also be managed via LoRa in the mobile app by using the Remote Management feature.

## 🛠 Hardware Compatibility

MeshCore is designed for devices listed in the [MeshCore Flasher](https://meshcore.io/flasher)

## 📜 License

MeshCore is open-source software released under the MIT License. You are free to use, modify, and distribute it for personal and commercial projects.

## Contributing

Please submit PR's using 'dev' as the base branch!
For minor changes just submit your PR and we'll try to review it, but for anything more 'impactful' please open an Issue first and start a discussion. It is better to sound out what it is you want to achieve first, and try to come to a consensus on what the best approach is, especially when it impacts the structure or architecture of this codebase.

Here are some general principles you should try to adhere to:
* Keep it simple. Please, don't think like a high-level lang programmer. Think embedded, and keep code concise, without any unnecessary layers.
* No dynamic memory allocation, except during setup/begin functions.
* Use the same brace and indenting style that's in the core source modules. (A .clang-format is probably going to be added soon, but please do NOT retroactively re-format existing code. This just creates unnecessary diffs that make finding problems harder)

Help us prioritize! Please react with thumbs-up to issues/PRs you care about most. We look at reaction counts when planning work.

### Running unit tests

To run unit tests, run the following command:

```bash
pio test --environment native --verbose
```

## Road-Map / To-Do

There are a number of fairly major features in the pipeline, with no particular time-frames attached yet. In very rough chronological order:
- [X] Companion radio: UI redesign
- [X] Repeater + Room Server: add ACL's (like Sensor Node has)
- [X] Standardise Bridge mode for repeaters
- [ ] Repeater/Bridge: Standardise the Transport Codes for zoning/filtering
- [X] Core + Repeater: enhanced zero-hop neighbour discovery
- [ ] Core: round-trip manual path support
- [ ] Companion + Apps: support for multiple sub-meshes (and 'off-grid' client repeat mode)
- [ ] Core + Apps: support for LZW message compression
- [ ] Core: dynamic CR (Coding Rate) for weak vs strong hops
- [ ] Core: new framework for hosting multiple virtual nodes on one physical device
- [ ] V2 protocol spec: discussion and consensus around V2 packet protocol, including path hashes, new encryption specs, etc

## 📞 Get Support

- Report bugs and request features on the [GitHub Issues](https://github.com/ripplebiz/MeshCore/issues) page.
- Find additional guides and components on [my site](https://buymeacoffee.com/ripplebiz).
- Join [MeshCore Discord](https://meshcore.gg) to chat with the developers and get help from the community.
