# .github

# Protocol Analyzer

A complete hardware + firmware + web system for real-time capture and analysis of embedded communication protocols. The system targets developers who need to monitor UART, SPI, I2C, and CAN bus traffic simultaneously — without commercial test equipment.

```
┌──────────────────────────────────────────────────────────────┐
│  Browser  (Next.js Web App)                                  │
│  Configure protocols · View live frames · Analyze data       │
└─────────────────────┬────────────────────────────────────────┘
                      │  Web Serial API  (1 Mbaud USB)
┌─────────────────────▼────────────────────────────────────────┐
│  ESP32 Bridge  (C++ / Arduino)                               │
│  Reliable frame delivery · ACK/NACK retry · SEQ numbering    │
└─────────────────────┬────────────────────────────────────────┘
                      │  UART  (3 Mbaud)
┌─────────────────────▼────────────────────────────────────────┐
│  FPGA  (VHDL / DE0-Nano Cyclone IV)                          │
│  Real-time protocol capture · 4 channels per protocol        │
└──────┬──────────┬───────────┬──────────────┬─────────────────┘
       │          │           │              │
     UART       SPI         I2C            CAN
  (×4 ch)    (×4 ch)     (×4 ch)        (×1 ch)
       │          │           │              │
       └──────────┴───────────┴──────────────┘
              Devices Under Test
```

---

## Repository Layout

```
.
├── Protocol-Analyzer-FPGA/        # VHDL firmware — capture engine
├── Protocol-Analyzer-ESP-Bridge/  # ESP32 C++ firmware — USB ↔ FPGA bridge
├── Protocol-Analyzer-WebApp/      # Next.js web app — configuration & visualization
└── OTHER/                         # Hardware references
    ├── DE0_NANO_MANUAL.pdf
    ├── pin.drawio.png
    └── schematic.drawio.png
```

---

## Components

### Protocol-Analyzer-FPGA

**Language:** VHDL | **Target:** Terasic DE0-Nano (Altera Cyclone IV EP4CE22)

The real-time capture core. Monitors physical bus lines, decodes frames according to the configured protocol, and streams results to the ESP32 over a 3 Mbaud UART link.

- 4 parallel channels for UART, SPI, and I2C; 1 channel for CAN
- Runtime reconfigurable: baud rate, SPI mode, I2C address, UART framing mode
- UART supports Raw, Modbus RTU (3.5-char silence), NMEA (`$…\n`), and Custom delimiter modes
- All inter-module communication uses a synchronous FIFO + round-robin arbiter
- Responds with ACK/NACK frames; no reprogramming needed to change config

[Full documentation →](Protocol-Analyzer-FPGA/README.md)

---

### Protocol-Analyzer-ESP-Bridge

**Language:** C++ (Arduino/PlatformIO) | **Target:** ESP32

The bridge between the host PC (USB Serial) and the FPGA (hardware UART). Provides reliable frame delivery with retry logic so the web app does not need to handle transport-level errors.

- Bidirectional relay: USB ↔ UART at 1 Mbaud / 3 Mbaud respectively
- Queue-based send buffer (256 slots) with auto SEQ injection (0–255)
- ACK/NACK handling with up to 3 retries and 1 s timeout
- Packet builders for UART, SPI, and I2C configuration frames
- Filters ACK frames from crossing the bridge (keeps the PC-side clean)

**Key files:**

| File | Purpose |
|------|---------|
| `src/main.cpp` | Entry point, sets up both serial controllers |
| `src/protocolController.*` | Frame queue, SEQ, checksum, ACK/NACK retry engine |
| `src/UART_PacketBuilder.*` | Builds UART config frames |
| `src/SPI_PacketBuilder.*` | Builds SPI config frames |
| `src/dataTypeConverter.h` | Union helpers for uint32/float ↔ byte-array conversion |

---

### Protocol-Analyzer-WebApp

**Language:** JavaScript (React / Next.js 15) | **Runtime:** Browser (Chrome/Edge required for Web Serial)

The user interface. Connects to the ESP32 via the browser's Web Serial API — no native app or driver needed. Sends configuration frames, receives captured data, and renders protocol-aware analysis in real time.

**Pages:**

| Route | Purpose |
|-------|---------|
| `/` | Main dashboard — connect, configure protocols, view live stream |
| `/analyze` | Advanced analyzer workspace with per-protocol frame decoding |
| `/modbus-tester` | Dedicated Modbus RTU tester with request/response parsing |

**Protocol library (`lib/protocol/`):**

| File | Responsibility |
|------|---------------|
| `encoder.js` | Build outgoing config frames (UART / SPI / I2C / Control) |
| `decoder.js` | State machine: byte stream → frames |
| `controller.js` | SEQ allocator + ACK/NACK retry engine (mirrors ESP32 logic) |
| `checksum.js` | `∑ mod 256` checksum |
| `modbus.js` | Modbus RTU parser (address, function, data, CRC) |
| `nmea.js` | NMEA sentence extractor and formatter |

**Key features:**
- Runtime config for all protocol parameters without page reload
- Live scrollable frame list color-coded by protocol/type
- Chart.js graphs for timing and frequency analysis
- Configuration history persisted to LocalStorage (50 records)
- Connection presets for quick reconnection
- Data export

---

## Frame Protocol

All three components share the same binary frame format:

```
[STX:0xFF] [SEQ] [TYPE] [PROTO] [CMD] [CH] [LEN] [PAYLOAD×LEN] [CHK] [ETX:0xF0]
```

| Field | Meaning |
|-------|---------|
| TYPE | `0x01` Data, `0x02` ACK, `0x03` NACK |
| PROTO | `0x00` Control, `0x01` UART, `0x02` SPI, `0x03` I2C, `0x04` CAN |
| CHK | `(SEQ + TYPE + … + last payload byte) mod 256` |

---

## Hardware Setup

1. Flash the FPGA with the bitstream compiled from `Protocol-Analyzer-FPGA/`.
2. Flash the ESP32 with the PlatformIO project in `Protocol-Analyzer-ESP-Bridge/`.
3. Connect ESP32 TX/RX (pins 18/19) to the FPGA's ESP communication UART pins.
4. Connect the target device's bus lines to the appropriate FPGA input pins (see `OTHER/pin.drawio.png`).
5. Open Chrome/Edge, navigate to the web app, click **Connect**, and select the ESP32 COM port.

---

## Tech Stack Summary

| Layer | Technology | Role |
|-------|-----------|------|
| Capture | VHDL on Cyclone IV | Real-time protocol acquisition |
| Bridge | C++ / ESP32 | Reliable USB ↔ FPGA relay |
| Interface | Next.js 15 + React 19 | Config UI and live visualization |
| Transport | Web Serial API | Browser-native serial (no driver) |
| Charts | Chart.js | Real-time timing graphs |
| Styling | Tailwind CSS 4 | UI styling |
| Build | Quartus Prime Lite + PlatformIO + Bun | Per-layer build tools |

---

## Quick Start

```bash
# 1. Web app (development server)
cd Protocol-Analyzer-WebApp
bun install
bun dev
# → open http://localhost:3000 in Chrome or Edge

# 2. ESP32 firmware
cd Protocol-Analyzer-ESP-Bridge
pio run --target upload

# 3. FPGA
# Open Protocol-Analyzer-FPGA in Quartus Prime,
# compile, and program via USB-Blaster.
```
