# ⚡ V-Guard UPS / Inverter UART Reverse Engineering Guide

> A practical guide to understanding and replacing the OEM ESP32 communication module used in V-Guard/OEM UPS and inverter systems.

---

# 📖 Overview

This guide documents the reverse engineering of the UART communication between:

- 🧠 Main inverter MCU
- 📡 ESP32 WiFi communication board
- ☁️ OEM MQTT cloud infrastructure

The goal was to:

- understand the UART protocol
- decode telemetry
- control inverter settings
- replace the OEM cloud
- integrate with ESPHome/Home Assistant

---

# 🏗️ System Architecture

The OEM design works like this:

```text
Mobile App
    ↓
MQTT Cloud
    ↓
ESP32 Communication Board
    ↓ UART
Main Inverter MCU
```

The inverter MCU itself:
- has no WiFi
- has no MQTT
- has no cloud logic

The ESP32 board acts as a bridge between:
- UART
- MQTT cloud
- mobile app

---

# 🔍 Reverse Engineering Process

The protocol was discovered by:

- sniffing UART traffic
- observing polling patterns
- changing inverter settings
- correlating packets with UI changes
- dumping OEM ESP32 firmware
- extracting MQTT configuration

---

# 🔌 UART Configuration

| Parameter | Value |
|---|---|
| Baud Rate | 9600 |
| Data Bits | 8 |
| Stop Bits | 1 |
| Parity | NONE |

---

# 📡 UART Packet Structure

## 📥 Read Request (ESP → MCU)

The ESP continuously polls registers from the MCU.

Packet format:

```text
FF FF FF [REGISTER] 0C 01 FF FF
```

Example:

```text
FF FF FF 06 0C 01 FF FF
```

Meaning:

```text
Read register 0x06
```

---

## 📤 Read Response (MCU → ESP)

The MCU replies with:

```text
FF FF FF [REGISTER] 0C 01 [LOW] [HIGH]
```

Payload is little-endian.

Decode logic:

```python
raw = (high << 8) | low
```

Example:

```text
FF FF FF 06 0C 01 29 05
```

Decoded:

```text
0x0529 = 1321
1321 / 100 = 13.21V
```

Meaning:

```text
Battery Voltage = 13.21V
```

---

# 🧩 Packet Types

| Type | Meaning |
|---|---|
| `0x0C` | 📊 Register read / telemetry |
| `0x0B` | ❤️ Heartbeat / keepalive |

Heartbeat example:

```text
FF FF FF C8 0B 01 FF FF
```

---

# 🔄 Polling Mechanism

The ESP32 continuously polls the MCU.

## 💤 Idle Polling

During idle state:

```text
60
1E
90
8A
40
50
42
52
...
```

---

## 📱 App Open Polling

When the mobile app opens:

- polling frequency increases
- more registers queried
- nearly full register map scanned

This revealed most telemetry and settings registers.

---

# 📈 Telemetry Registers

| Register | Meaning | Scale |
|---|---|---|
| `0x06` | 🔋 Battery Voltage | `/100` |
| `0x08` | ⚡ Grid Voltage | `/10` |
| `0x0A` | 🔌 Output Voltage | `/10` |
| `0x0C` | 🔄 Discharge Current | `/10` |
| `0x10` | 🔋 Charge Current | `/10` |
| `0x38` | ☀️ Solar Power | `/10` |
| `0x3C` | 🔋 Battery % | `/278` |
| `0x2C` | 📊 Load % | raw |
| `0x76` | ☀️ Solar Current | `/100` |
| `0x78` | ☀️ Solar Voltage | `/100` |

---

# ⚙️ Settings Registers

| Register | Setting |
|---|---|
| `0x18` | 🔄 Inverter Mode |
| `0x20` | 🔔 Mains Buzzer |
| `0x24` | 📊 Output Limit |
| `0x26` | 🔌 Appliance Mode |
| `0x28` | ✂️ Grid Force Cut |
| `0x30` | ⚡ Turbo Charging |
| `0x3A` | 🚨 Load Alarm Threshold |
| `0x3E` | 🔋 Advanced Low Battery Alarm |
| `0x84` | ☀️ Daytime Load Usage |

---

# 🎛️ Write Commands

## 🔄 Inverter Mode

| Mode | Packet |
|---|---|
| Normal | `FF 00 00 18 0C 00 FF FF` |
| UPS | `FF 01 00 18 0C 00 FF FF` |
| Equipment | `FF 02 00 18 0C 00 FF FF` |

---

## 🔔 Mains Buzzer

| State | Packet |
|---|---|
| ON | `FF 00 00 20 0C 00 FF FF` |
| OFF | `FF 02 00 20 0C 00 FF FF` |

---

## 📊 Output Limit

| Value | Packet |
|---|---|
| 0% | `FF 01 00 24 0C 00 FF FF` |
| 20% | `FF 02 00 24 0C 00 FF FF` |
| 40% | `FF 03 00 24 0C 00 FF FF` |
| 50% | `FF 04 00 24 0C 00 FF FF` |
| 60% | `FF 05 00 24 0C 00 FF FF` |
| 90% | `FF 06 00 24 0C 00 FF FF` |
| 100% | `FF 07 00 24 0C 00 FF FF` |

---

# ☁️ MQTT Findings

Firmware extraction revealed:

```ini
BrokerAddress = vguardbox.com
BrokerPort    = 8883
ServerUname   = vguard
Serverpass    = vguard1234
```

This confirms:

- ESP32 polls MCU via UART
- ESP32 publishes telemetry via MQTT
- mobile app communicates through MQTT
- app commands become UART write packets

---

# 🧠 OEM Firmware Findings

The OEM ESP firmware:

✅ continuously polls the MCU  
✅ decodes UART packets locally  
✅ publishes MQTT telemetry  
✅ receives remote commands  
✅ synchronizes settings  

The MCU itself only understands:
- UART register reads
- UART register writes

---

# 🏠 ESPHome Replacement

A complete ESPHome replacement was successfully implemented using:

- ESP32-C3-DevKitM-1
- UART polling
- Home Assistant entities
- live telemetry
- writable settings
- bidirectional synchronization

Features:

✅ local-only control  
✅ no OEM cloud required  
✅ no mobile app required  
✅ real-time telemetry  
✅ Home Assistant integration  

---

# 🔍 Key Findings

- 📡 protocol is plaintext UART
- 📦 fixed 8-byte packets
- ❌ no checksum/CRC observed
- 🔄 ESP continuously polls MCU
- ✍️ settings are fully writable
- ☁️ ESP acts as UART ↔ MQTT bridge
- 📱 app opening increases polling frequency

---

# ⚠️ Safety Warning

Writing unknown registers may:

- disable protections
- alter charging behavior
- damage batteries
- crash inverter MCU

Only documented registers should be modified.

---

# 🚀 Future Work

Potential future improvements:

- full register discovery
- fault code mapping
- MQTT topic extraction
- OTA analysis
- web dashboard
- native ESPHome component
- InfluxDB/Grafana integration

---

# 📜 Disclaimer

Unofficial reverse engineering project.

Not affiliated with:
- V-Guard
- OEM vendors
- original firmware authors
