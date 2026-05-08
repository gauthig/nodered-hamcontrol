# nodered-hamcontrol

Node-RED automation flows for a ham radio station — integrating a FlexRadio transceiver, Palstar LA-1K amplifier, HFAuto ATU, and PST antenna rotator into a unified Dashboard 2.0 control surface.

> **Station:** Apple Valley, CA (34.5134° N / 117.2414° W)
> **Platform:** Node-RED with Dashboard 2.0 (`@flowfuse/node-red-dashboard`)

---

## Flows

| Flow | Equipment | Transport | Documentation |
|------|-----------|-----------|---------------|
| [FlexRadio](#flexradio) | FlexRadio SmartSDR transceiver | TCP (FlexRadio API) | [FlexRadio.md](FlexRadio.md) |
| [HF Auto](#hf-auto-palstar-atu) | HFAuto / Palstar ATU | UDP | [HF-Auto-Palstar.md](HF-Auto-Palstar.md) |
| [LA-1K Amp](#la-1k-amplifier) | Palstar LA-1K linear amplifier | Serial (RS-232) | [LA1K-Amp.md](LA1K-Amp.md) |
| [Rotator](#rotator) | PST antenna rotator controller | UDP | [ROTATOR.md](ROTATOR.md) |

---

## Flow Descriptions

### FlexRadio

Integrates with a **FlexRadio SmartSDR** transceiver via the `node-red-contrib-flexradio` package. Discovers the radio on the local subnet via UDP broadcast on port 4992, then maintains a persistent connection.

**Key capabilities:**
- Monitors up to four VFO slices with live frequency and mode display
- Displays real-time meters — PA temperature, supply voltage, and RF power
- Automatically shows/hides dashboard panels based on radio power state
- Limits transmit power when the LA-1K amplifier is in standby (30 W) vs. operate (90 W)
- APD (automatic pre-distortion) state monitoring
- 15-second link-staleness detection with error reporting

**Provides to other flows:**
- `global.TXFreq` — current TX frequency in kHz, consumed by the LA-1K amp flow for band/frequency sync
- `global.ampPower` — amp power state flag, controls LA-1K polling
- Link-in `941465de770ff4a6` — receives amp operate/standby state from the LA-1K flow and adjusts radio output accordingly

→ Full documentation: [FlexRadio.md](FlexRadio.md)

---

### HF Auto (Palstar ATU)

Controls an **HFAuto** automatic antenna tuner over UDP, with live power and SWR monitoring. Receives telemetry XML on port 13080 and sends command XML to port 12020.

**Key capabilities:**
- Live gauges for Peak Watts, TX Watts, and SWR
- Antenna selection for three antennas (HexBeam, EFHW, ZeroFive)
- Automatic tuner mode enforcement per antenna: HexBeam → BYPASS (resonant), EFHW/ZeroFive → AUTO (non-resonant)
- 2-second sequencing delay between antenna change and mode enforcement
- RBE (report-by-exception) filtering to suppress duplicate UDP packets
- Dashboard radio groups stay in sync with incoming tuner state

**Network ports:**

| Port | Direction | Purpose |
|------|-----------|---------|
| 13080 | In | Telemetry XML from ATU |
| 12020 | Out | Command XML to ATU (loopback) |

→ Full documentation: [HF-Auto-Palstar.md](HF-Auto-Palstar.md)

---

### LA-1K Amplifier

Monitors and controls a **Palstar LA-1K** HF linear amplifier over a serial (RS-232) connection. Polls the amp every 5 seconds via the `AR1;` command and parses the semicolon-delimited response.

**Key capabilities:**
- Live FWD power gauge, frequency, band, temperature (°F), and key status
- Operate / Standby control with immediate serial command
- Three-antenna selection (Tuner, OPEN × 2) with colored button feedback showing active antenna
- Automatic frequency sync from the radio (`global.TXFreq`) — sends `IF` command to amp every second on frequency change
- Show/hide dashboard panel based on amp power state (`global.ampPower`)
- Reports operate/standby state back to the FlexRadio flow via link node

**Dependencies on other flows:**

| Global / Link | Provided by | Purpose |
|---------------|------------|---------|
| `global.TXFreq` | FlexRadio flow | TX frequency for amp frequency sync |
| `global.ampPower` | FlexRadio flow | Controls whether the amp is polled |
| Link-out `Tell Radio Amp Status` | This flow → FlexRadio | Reports operate/standby to radio |

**Required Node-RED packages:**
- `node-red-node-serialport`
- `node-red-contrib-string`
- `node-red-contrib-unit-converter`

→ Full documentation: [LA1K-Amp.md](LA1K-Amp.md)

---

### Rotator

Controls a **PST antenna rotator** over UDP, featuring a live azimuthal equidistant (great circle) world map rendered with D3.js — the standard ham radio projection where straight lines equal great circle paths.

**Key capabilities:**
- Great circle map centered on your QTH, with an animated red beam line that updates as the rotator moves
- Manual azimuth entry (0–360°) and preset bearings dropdown for common DX destinations
- Stop button to immediately halt rotation
- Auto-polls azimuth every 10 seconds to stay in sync while idle

**Network ports:**

| Port | Direction | Purpose |
|------|-----------|---------|
| 12001 | In | Azimuth status from rotator controller |
| 12000 | Out | Commands to rotator controller |

**UDP protocol:**

| Message | Format |
|---------|--------|
| Poll azimuth | `<PST>AZ?</PST>` |
| Rotate to bearing | `<PST><AZIMUTH>270</AZIMUTH></PST>` |
| Stop | `<PST><STOP>1</STOP></PST>` |
| Azimuth reply | `AZ:270` |

> Based on original work by [LA9VKA](https://flows.nodered.org/user/LA9VKA), migrated to Dashboard 2.0 with D3.js great circle map.

→ Full documentation: [ROTATOR.md](ROTATOR.md)

---

## Inter-Flow Data Map

The FlexRadio and LA-1K flows are tightly coupled. The HF Auto and Rotator flows are independent.

```
┌──────────────────────────────────────────────────────────┐
│                      FlexRadio Flow                      │
│  Writes: global.TXFreq, global.ampPower                  │
│  Reads:  global.amp (from LA-1K via link node)           │
└─────────────────────┬────────────────────────────────────┘
                      │ global.TXFreq
                      │ global.ampPower
                      │ link-in 941465de770ff4a6 (amp status)
                      ▼
┌──────────────────────────────────────────────────────────┐
│                     LA-1K Amp Flow                       │
│  Reads:  global.TXFreq, global.ampPower                  │
│  Writes: global.amp ("operate"/"standby")                │
│  Sends:  amp status → FlexRadio (link out)               │
└──────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────┐
│                   HF Auto Flow (ATU)                     │
│  UDP in  :13080  ← HFAuto telemetry                      │
│  UDP out :12020  → HFAuto commands                       │
│  Independent — no inter-flow dependencies                │
└──────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────┐
│                     Rotator Flow                         │
│  UDP in  :12001  ← PST azimuth status                    │
│  UDP out :12000  → PST rotation commands                 │
│  Independent — no inter-flow dependencies                │
└──────────────────────────────────────────────────────────┘
```

---

## Requirements

### Node-RED Packages

Install via **Manage Palette** in the Node-RED editor:

| Package | Used by |
|---------|---------|
| `@flowfuse/node-red-dashboard` | All flows (Dashboard 2.0) |
| `node-red-contrib-flexradio` | FlexRadio flow |
| `node-red-node-serialport` | LA-1K Amp flow |
| `node-red-contrib-string` | LA-1K Amp flow |
| `node-red-contrib-unit-converter` | LA-1K Amp flow |

### Network / Hardware

| Equipment | Connection | Address |
|-----------|------------|---------|
| FlexRadio SmartSDR | TCP — FlexRadio API | Same subnet (auto-discovered) |
| HFAuto ATU | UDP loopback | 127.0.0.1:13080 in / :12020 out |
| Palstar LA-1K | Serial RS-232 | COM port (configure in flow) |
| PST Rotator controller | UDP loopback | 127.0.0.1:12001 in / :12000 out |

---

## Installation

1. Clone this repo into your Node-RED user directory (or a subdirectory):
   ```
   git clone https://github.com/gauthig/nodered-hamcontrol.git
   ```
2. Copy `flows.json` to your Node-RED user directory and restart Node-RED, **or** import the flows individually via **Menu → Import**.
3. Install required Node-RED packages listed above.
4. Configure the **serial port** in the LA-1K flow to match your COM port.
5. Configure the **FlexRadio** node with your radio's IP address.
6. Update the QTH coordinates in the **Rotator** flow's `Azimuth Map` template node if not already set to your location.
7. Deploy all flows.

---

## License

MIT
