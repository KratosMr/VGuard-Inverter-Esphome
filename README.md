# ⚡ V-Guard UPS / Inverter UART Reverse Engineering

Reverse engineering notes for the UART protocol used between the ESP32 communication board and the main inverter MCU found in V-Guard/OEM UPS systems.

Includes:

- 🔍 UART packet structure
- 🧭 Register map
- 📊 Telemetry decoding
- 🎛️ Write commands
- ☁️ MQTT findings
- 🏠 ESPHome integration
- 🔄 Live polling
- ⚙️ Settings synchronization

---

# 🏗️ Architecture

```text
Mobile App
    ↓
MQTT Cloud
    ↓
ESP32 Communication Board
    ↓ UART
Main Inverter MCU
```

The MCU does not have WiFi or MQTT support.

The ESP32 acts as:
- 🔄 UART polling bridge
- ☁️ MQTT client
- 📱 cloud/app interface

---

# 🔌 UART Configuration

| Parameter | Value |
|---|---|
| Baud Rate | 9600 |
| Data Bits | 8 |
| Stop Bits | 1 |
| Parity | NONE |

---

# 📡 UART Protocol

## 📥 Read Request

ESP → MCU

```text
FF FF FF [REGISTER] 0C 01 FF FF
```

Example:

```text
FF FF FF 06 0C 01 FF FF
```

---

## 📤 Read Response

MCU → ESP

```text
FF FF FF [REGISTER] 0C 01 [LOW] [HIGH]
```

Little-endian payload:

```python
raw = (high << 8) | low
```

---

# 🧩 Packet Types

| Type | Meaning |
|---|---|
| `0x0C` | 📊 Register read / telemetry |
| `0x0B` | ❤️ Heartbeat / keepalive |

Example:

```text
FF FF FF C8 0B 01 FF FF
```

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

Firmware strings revealed:

```ini
BrokerAddress = vguardbox.com
BrokerPort    = 8883
ServerUname   = vguard
Serverpass    = vguard1234
```

Meaning:

- 🔄 ESP32 polls MCU over UART
- ☁️ publishes telemetry over MQTT
- 📱 receives app commands over MQTT
- ⚙️ converts commands into UART write packets

---

# 🏠 ESPHome Integration

Working ESPHome implementation includes:

## 📊 Sensors

- ⚡ Grid Voltage
- 🔋 Battery Voltage
- 🔌 Output Voltage
- 🔋 Charge Current
- 🔄 Discharge Current
- ☀️ Solar Voltage
- ☀️ Solar Current
- ☀️ Solar Power
- 🔋 Battery %
- 📊 Load %

## 🎛️ Settings Controls

- 🔄 Inverter Mode
- 🔔 Mains Buzzer
- 📊 Output Limit
- 🔌 Appliance Mode
- ✂️ Grid Force Cut
- ⚡ Turbo Charging
- 🚨 Load Alarm Threshold
- 🔋 Advanced Low Battery Alarm
- ☀️ Daytime Load Usage

### ✅ Features

- 🔁 bidirectional synchronization
- 📡 live polling
- 🏠 Home Assistant entities
- 🔒 local-only control
- ❌ no OEM cloud required

---

# 🔍 Key Findings

- 📡 MCU protocol is plaintext UART
- 📦 fixed 8-byte packets
- ❌ no checksum/CRC observed
- 🔄 ESP continuously polls MCU
- ✍️ settings are fully writable
- 📱 app opening increases polling rate
- ☁️ ESP acts as UART ↔ MQTT bridge

---

# ⚠️ Safety Warning

Unknown register writes may:
- ❌ disable protections
- 🔥 damage batteries
- ⚡ alter charging behavior
- 💥 crash inverter MCU

Only documented registers should be modified.

---

# 📜 Disclaimer

Unofficial reverse engineering project.

Not affiliated with:
- V-Guard
- OEM vendors
- original firmware authors.
