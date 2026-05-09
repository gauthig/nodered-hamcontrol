# LA-1K Amp — Node-RED Flow

Node-RED flow for monitoring and controlling a **Palstar LA-1K** HF linear amplifier over a serial connection. It polls the amp every 5 seconds, parses the response, and updates a Dashboard 2.0 group with live telemetry. It also sends antenna-selection and frequency commands, and reports operate/standby status back to the radio.

---

## Table of Contents

- [Overview](#overview)
- [Requirements](#requirements)
- [Serial Port Configuration](#serial-port-configuration)
- [Flow Groups](#flow-groups)
  - [Main Polling & Telemetry](#main-polling--telemetry)
  - [Mode Select (Operate / Standby)](#mode-select-operate--standby)
  - [Antenna Select](#antenna-select)
  - [Flex API Frequency Sync](#flex-api-frequency-sync)
- [Dashboard Widgets](#dashboard-widgets)
- [Global Variables](#global-variables)
- [Inter-Flow Interfaces (Link Nodes)](#inter-flow-interfaces-link-nodes)
- [Show / Hide Logic](#show--hide-logic)
- [AR1 Response Parsing](#ar1-response-parsing)
- [Configuration Checklist](#configuration-checklist)

---

## Overview

```
Inject (5s) ──► Check Power ──► LA-1K Command (AR1;) ──► Serial Port
                                                              │
                                                         Serial Response
                                                              │
                                                       String (between "AD")
                                                              │
                                                          rbe filter
                                                              │
                                                       Split on ";"
                                                              │
                                                          Switch (index)
                                                    ┌─────┬──┴──┬─────┐
                                                  Freq  Power  Ant  Band  Temp  Key ...
                                                    │     │     │     │     │     │
                                                 ui-text gauge change text gauge text
```

The amp is polled via a **serial request** node. The raw response string is trimmed to the data between `AD` markers, deduplicated by an `rbe` node (only propagates when the value changes), split on `;`, and routed by field index to the appropriate function node and dashboard widget.

---

## Requirements

| Package | Version |
|---|---|
| `@flowfuse/node-red-dashboard` | ≥ 1.30.x (Dashboard 2.0) |
| `node-red-node-serialport` | Any compatible version |
| `node-red/rbe` | Built-in (Node-RED ≥ 3.x) |
| `node-red-contrib-string` | For the `string` between-extraction node |
| `node-red-contrib-unit-converter` | For °C → °F temperature conversion |

Install missing nodes via **Manage Palette** in the Node-RED editor.

---

## Serial Port Configuration

The flow uses a shared **Serial Port** config node (`a6924d43.0c47e8`). Configure it to match your system:

| Setting | Value |
|---|---|
| Serial Port | e.g. `COM3` (Windows) or `/dev/ttyUSB0` (Linux) |
| Baud Rate | Per LA-1K manual (typically `9600`) |
| Data Bits | `8` |
| Stop Bits | `1` |
| Parity | `None` |

Open the `com-LA1K` serial request node and select or create this config node.

---

## Flow Groups

### Main Polling & Telemetry

The heart of the flow. An **Inject** node fires every 5 seconds (and once at startup with a 0.2 s delay). Before sending the poll command, a **Check Power** switch node tests `msg.topic` (bound to `global.ampPower`) to confirm the amp is powered on:

- **Power off** → triggers the **Power Off** change node, which hides the dashboard group.
- **Power on** → sends `AR1;\r\n` to the amp via the **LA-1K Command** function node, and shows the dashboard group via **ui-control**.

The serial response is processed by:

1. **String node** — extracts the substring between the two `AD` delimiters in the AR1 response.
2. **rbe node** (`cd5bc1e866595ddb`) — blocks duplicate responses; only passes when data changes.
3. **Split node** — splits the string on `;` into an array of fields.
4. **Switch node** — routes each field by array index to the correct handler.

### Mode Select (Operate / Standby)

A **ui-switch** widget (OP/STBY) in the dashboard sends either `"operate"` or `"standby"` to a switch node that routes to:

- **Operate function** — sends `AM1;\r\n` to the amp, then sets `global.amp = "operate"` and notifies the radio via the **Tell Radio Amp Status** link out.
- **Standby function** — sends `AM2;\r\n` to the amp, then sets `global.amp = "standby"` and notifies the radio.

An **Initialize** inject node fires once at startup (1 s delay) with `"standby"` to set the switch to its default position.

### Antenna Select

Three **ui-button** widgets (ANT->Tuner, ANT->OPEN × 2) each send a payload of `"1"`, `"2"`, or `"3"`. The **Ant Output** function builds the command `AA<n>;\r\n` and sends it to the amp via the **LA1K Ant Select Command** link out → **LA1K Serial In** link in → serial port.

Button colors reflect the currently selected antenna:
- Each AR1 poll response routes through the **Antenna change** node, which saves the response value to `flow.antenna.response` and then triggers all three **ANT Button Config** function nodes.
- Each config function reads `flow.antenna.response` and sets `msg.ui_update.buttonColor` to `"green"` (selected) or `"red"` (not selected).

### Flex API Frequency Sync

An **Inject** node fires every 1 second and reads `global.TXFreq` (the radio's current TX frequency in kHz, as a float). An **rbe** node (`b21dbde0.58adb8`) suppresses updates when the frequency hasn't changed. When it changes, the **Flex API Frequency Command** function converts it to the LA-1K IF command format:

```js
var freq = Math.round(msg.payload * 1000);   // kHz → Hz, integer
msg.payload = "IF" + String(freq).padStart(8, '0') + ";\r\n";
```

The command is sent via the **N1MM Freq Command** link out → **LA1K Serial In** → serial port.

---

## Dashboard Widgets

All widgets belong to the dashboard group **"Palstar LA-1K Amp"** (group ID `66e9faf9106a437d`) on Dashboard 2.0 UI `549e4fdcf9161fde`.

| Order | Widget | Type | Source Field (AR1 index) | Notes |
|---|---|---|---|---|
| 1 | Freq. | `ui-text` | Index 0 | Divided by 1000 (Hz → kHz) |
| 2 | Band | `ui-text` | Index 5 | Decoded: 0=blank, 1=160m, 2=80m, 3=40m, 4=20-15m, 5=12-10m |
| 3 | OP/STBY | `ui-switch` | — | User input; sends AM1/AM2 to amp |
| 4 | Key | `ui-text` | Index 10 | See [Key Status codes](#ar1-response-parsing) |
| 5 | FWD Power | `ui-gauge-linear` | Index 2 | Multiplied ×10 when in Operate mode |
| 6 | ANT->Tuner | `ui-button` | — | Sends AA1; |
| 7 | ANT->OPEN | `ui-button` | — | Sends AA3; |
| 8 | Temp | `ui-gauge` (half) | Index 8 | Converted °C → °F; segments: green/yellow/red |
| 9 | ANT->OPEN | `ui-button` | — | Sends AA2; |

---

## Global Variables

| Variable | Written by | Read by | Values |
|---|---|---|---|
| `global.amp` | Amp Operate / Amp Standby change nodes | FWD Power function | `"operate"` / `"standby"` |
| `global.ampPower` | External flow (radio interface) | Check Power switch | `true` / `false` |
| `global.ampStatus` | Power Off change node | External flows | `"standby"` |
| `global.TXFreq` | External flow (radio interface) | Flex Freq group inject | Frequency in kHz (float) |

> **Note:** `global.amp` controls how FWD Power is scaled. When `"operate"`, the raw value from the amp is multiplied by 10 to give watts. When `"standby"` (or unset), the raw value is passed through unchanged.

---

## Inter-Flow Interfaces (Link Nodes)

The flow communicates with the rest of the Node-RED project through named link nodes.

### Link In

| Node Name | ID | Accepts messages from |
|---|---|---|
| `LA1K Serial In` | `1c20288d.2e3437` | `LA1K Mode Command`, `LA1K Ant Select Command`, `N1MM Freq Command` |

All commands (mode, antenna, frequency) funnel through this single link-in node to the serial request node. This ensures only one message reaches the serial port at a time.

### Link Out

| Node Name | ID | Purpose | Connects to |
|---|---|---|---|
| `LA1K Mode Command` | `6a997dd0.80a73c` | Sends AM1/AM2 serial commands | `LA1K Serial In` |
| `LA1K Ant Select Command` | `6a333069.8531f8` | Sends AA1/AA2/AA3 serial commands | `LA1K Serial In` |
| `N1MM Freq Command` | `2705c395.dbbfdc` | Sends IF frequency commands | `LA1K Serial In` |
| `Tell Radio Amp Status` | `08f4474ef4a901ee` | Reports operate/standby to radio flow | `941465de770ff4a6` (radio interface flow) |

### What other flows must provide

| Global variable | Required by | Description |
|---|---|---|
| `global.ampPower` | Check Power switch | Set `true` when the amp is powered on, `false` when off. The polling inject uses this as its `msg.topic`. |
| `global.TXFreq` | Flex Freq group | Current radio TX frequency in kHz (e.g. `14225.0`). Updated by the radio interface flow. |

The link-out node **Tell Radio Amp Status** (ID `08f4474ef4a901ee`) delivers the current amp mode (`"operate"` or `"standby"`) to link-in `941465de770ff4a6` in the radio interface flow, which can then pass that state to the radio.

---

## Show / Hide Logic

The dashboard group is shown or hidden based on amp power state using a **ui-control** node (`Hide-Unhide`, `c7194fc483948114`) configured for `events: "all"`:

- **Show:** `{"group": {"show": ["Palstar LA-1K Amp"]}}`
- **Hide:** `{"group": {"hide": ["Palstar LA-1K Amp"]}}`

These payloads are set by the **Power On** and **Power Off** change nodes respectively, triggered by the 5-second poll inject when it checks `global.ampPower`.

---

## AR1 Response Parsing

The amp responds to `AR1;` with a semicolon-delimited string wrapped in `AD...AD`. After extracting the inner data and splitting on `;`, each index maps to:

| Index | Field | Processing |
|---|---|---|
| 0 | Frequency (Hz) | ÷ 1000 → kHz, displayed as-is |
| 2 | FWD Power (raw) | ×10 if `global.amp == "operate"` |
| 4 | Antenna (1/2/3) | Stored in `flow.antenna.response`; drives button colors |
| 5 | Band code | 0=blank, 1=160m, 2=80m, 3=40m, 4=20-15m, 5=12-10m |
| 8 | Temperature (°C) | Converted to °F via unit-converter node |
| 10 | Key status code | See table below |

**Key Status codes:**

| Code | Display |
|---|---|
| 0 | Unkeyed |
| 1 | TX Wait |
| 2 | TX Amp |
| 3 | --- |
| 4 | TX Bypass |
| 5 | --- |
| 6 | Fault FQ |
| 7 | Fault SWR |
| 8 | Fault Temp |

---

## Configuration Checklist

1. **Serial port** — update the `com-LA1K` serial request node to point to the correct COM port and baud rate for your LA-1K.
2. **Dashboard UI** — confirm the dashboard group `66e9faf9106a437d` and UI `549e4fdcf9161fde` exist in your Dashboard 2.0 setup. Re-assign all widgets if you created a new dashboard.
3. **`global.ampPower`** — ensure your radio interface flow sets this global variable to `true`/`false` based on whether the amp is powered on.
4. **`global.TXFreq`** — ensure your radio interface flow continuously updates this to the current TX frequency in kHz.
5. **Link node `941465de770ff4a6`** — verify a link-in node with this ID exists in your radio interface flow to receive the amp operate/standby state.
6. **Installed nodes** — verify `node-red-contrib-string` and `node-red-contrib-unit-converter` are installed (Manage Palette).
