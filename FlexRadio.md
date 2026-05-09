# FlexRadio Node-RED Flow

## Overview

This Node-RED flow connects to a **FlexRadio SmartSDR** transceiver over the local network, subscribing to real-time radio data (meters, slice info, TX/RX state, APD) and displaying it on a Node-RED Dashboard 2.0 panel. It also integrates with the **LA-1K Amp** flow and the **Power Flow** to automate RF power limits and dashboard visibility based on whether the radio and amplifier are active.

> **Network Requirement:** The Node-RED host must be on the **same subnet/network** as the FlexRadio. Discovery uses UDP broadcast on port **4992**. The flow will not find the radio across routers or VLANs without additional networking configuration.

---

## Prerequisites

| Requirement | Details |
|---|---|
| Node-RED palette | `node-red-contrib-flexradio` |
| Additional palettes | `node-red-node-ui-table`, `node-red-contrib-unit-converter`, `node-red-node-rbe`, `node-red-contrib-string` |
| Dashboard | Node-RED Dashboard 2.0 (`@flowfuse/node-red-dashboard`) |
| Network | Node-RED and radio on same subnet |
| Radio | FlexRadio SmartSDR (tested on 8600) |

---

## Configuring the FlexRadio Nodes for Your Radio

All four core FlexRadio nodes in the flow  -  `flexradio-discovery`, `flexradio-message`, `flexradio-meter`, and `flexradio-request`  -  share a single **radio configuration node** (internal ID `08a174e0167ffe6f`). You must update this config node to point to your radio.

### Steps

1. Open Node-RED and navigate to the **FlexRadio** tab.
2. Double-click any of the following nodes:
   - The `flexradio-discovery` node (in the **Radio Connections** group)
   - The `flexradio-message` node
   - The `flexradio-meter` node
   - The `flexradio-request` node
3. Click the **pencil icon** next to the radio config field to open the shared configuration.
4. Set the radio's **IP address** (or leave it blank to rely on auto-discovery via UDP port 4992).
5. Click **Update**, then **Deploy**.

> **Tip:** If your radio is on the same subnet and has network discovery enabled in SmartSDR settings, the `flexradio-discovery` node on port **4992** will find it automatically without a static IP.

---

## Flow Groups

The flow is organized into eight labeled groups:

### 1. Hide / Unhide Dashboard

**Purpose:** Automatically hides or shows the `Flex Radio 8600` dashboard group based on whether the radio's power relay reports the radio as on or off.

**How it works:**
- An inject node polls the **global variable `radioPower`** (a boolean) every 1 second.
- If `radioPower` is `false` → sends `{"group":{"hide":["Flex Radio 8600"]}}` to the `ui-control` node, hiding the entire dashboard panel.
- If `radioPower` is `true` → sends `{"group":{"show":["Flex Radio 8600"]}}`, making it visible again.

**Hardcoded value:** The dashboard group name `"Flex Radio 8600"` is hardcoded in the hide and show change nodes. If you rename your dashboard group, update both change nodes to match.

**Dependency:** The `global.radioPower` variable is set by the **Power Flow** tab (using a Shelly relay connected to the REM ON jack on the back of the radio). If you are not using that flow, see the section [**How to Disable Dashboard Auto-Hide**](#how-to-disable-dashboard-auto-hide) below.

---

### 2. Radio Connections

**Purpose:** Establishes and maintains the connection to the FlexRadio, subscribes to all relevant data streams, and handles client binding.

**Nodes & Functions:**

| Node | Type | Function |
|---|---|---|
| `flexradio-discovery` | flexradio-discovery | Discovers the radio on port 4992 via UDP broadcast |
| `flexradio-message` | flexradio-message | Receives all radio message data (slices, APD, transmit, client, interlock) |
| `flexradio-meter` | flexradio-meter | Receives real-time meter values |
| `flexradio-request` | flexradio-request | Sends commands back to the radio (e.g., subscribe, bind, power) |
| `Bind \| Subscribe` | function | On connect, sends 8 subscription commands: `sub client all`, `sub slice all`, `sub meter all`, `sub tx all`, `sub apd all`, `sub radio all`, `sub radio pan`, and `client bind client_id=<id>`. A `flow.subscribed` guard ensures this runs only **once per connection**  -  subsequent `client` status messages (which would otherwise echo back through the switch and trigger re-subscription repeatedly) are silently ignored. The guard is reset to `false` on disconnect so reconnect always triggers a fresh subscription. |
| `ParseClients` | function | Extracts the connected GUI client program and station name |
| `Logged in Client Name` | ui-text | Displays the connected client name on the dashboard |
| `Radio Relay Status` | switch | Passes commands to the radio only if `global.radioPower` is `true` (radio is on) |
| `NULL Payload` | switch | Filters out empty command payloads before sending to the radio |

---

### 3. Meters

**Purpose:** Processes and displays real-time meter readings from the radio on the dashboard.

**Meters displayed:**

| Meter | Source Topic Pattern | Dashboard Widget | Notes |
|---|---|---|---|
| Fan RPM | `RAD/xxx/MAINFAN` | `Fan RPM` linear gauge (0–2100 RPM) | Color: red at low/high extremes, green in normal range |
| SWR | `TX-/3/SWR` | `SWR Linear` gauge (0–10) | Color: green (1–3), orange (4–7), red (8–10) |
| Forward Power (Watts) | `TX-/1/FWDPWR` | `FWD Watts` gauge (0–100 W) | Converted from dBm to Watts using formula `10^((dBm-30)/10)` |
| PA Temperature | `TX-x/PATEMP` | `Temp` gauge (0–210 °F) | Converted from Celsius to Fahrenheit |
| PA Current | `RAD/300/PACURRENT` | Used in supply watt calculation | Stored in `global.paCurrent` |
| PA Voltage (+13.8V) | `RAD/334/+13.8A` | `Volts` linear gauge (0–15 V) | Stored in `global.paVolts` |
| PA Supply Watts | Computed | `TX Electric Power` text | Only computed during `TRANSMITTING` state: `paCurrent × paVolts` |

> **Note on PA Supply Watts:** The electrical input power calculation (current × voltage) is **only available on 8000-series FlexRadio models** (e.g., 8600). The 6000-series does not expose the required power-consumption sensor. FlexRadio has an open bug report tracking this limitation.

All meter values are passed through **report-by-exception (RBE)** nodes before display to avoid unnecessary dashboard updates.

---

### 4. Active Slice

**Purpose:** Tracks up to four VFO slices (A–D), determines which is active, and displays the active frequency and mode on the dashboard.

**How it works:**
- Incoming slice messages (topic `slice/0`, `slice/1`, etc.) are parsed by `Get Slice Status`.
- State for each slice is stored in flow variables: `sliceA`, `sliceB`, `sliceC`, `sliceD` (containing `freq`, `mode`, `active`, `in_use`, `tx`, `txant`, `rxant`).
- `Active Slice Frequency` reads all four slices and determines which has `active = 1`.
- Frequency is formatted as `MHz.KHz.Hz` (e.g., `14.225.000`) by `to flex-freq`.
- A **Vue template widget** (`Slice Display`) shows all in-use slices with frequency, mode, TX/RX badge, and TX/RX antenna assignments.

**Dashboard widgets:**

| Widget | Type | Shows |
|---|---|---|
| `Active Slice Freq` | ui-text | Active VFO frequency (formatted) |
| `Active Slice Mode` | ui-text | Active mode (e.g., USB, LSB, CW) |
| `Slice Display` | ui-template (Vue) | All in-use slices with freq, mode, TX/RX status, antenna |

---

### 5. Radio Message Outputs

**Purpose:** Handles TX/RX interlock state, RF power monitoring, client binding, and routes radio commands back to the radio.

**TX/RX State handling:**

| State | Action |
|---|---|
| `RECEIVE` | Sets `global.txstatus = "RECEIVE"`, TX Status text turns **black** with label "Receive Only" |
| `READY` | Sets `global.txstatus = "READY"`, TX Status text turns **green** |
| `TRANSMITTING` | Sets `global.txstatus = "TRANSMITTING"`, TX Status text turns **red** |

**RF Power display and limiting:**
- When a `transmit` message is received, `Number RF Power` extracts the `rfpower` value and stores it in `flow.rfpower`, then displays it on the `Watts Setting` gauge (0–100 W).
- `Limit Power if Amp Operate` checks: if `global.ampPower` is `true` and `global.amp == "operate"` and current `rfpower > 50`, it sends `transmit set rfpower=50` to cap power at **50 watts**.

**Dashboard widgets:**

| Widget | Shows |
|---|---|
| `TX Status` | Current TX/RX state with color coding |
| `Watts Setting` | Current RF power setting (gauge, 0–100 W) |

---

### 6. Amp Integration

**Purpose:** Adjusts the radio's RF power output automatically based on the state of the external amplifier.

**Input:** Receives messages from the **LA-1K Amp** flow via a `link in` node named **"From AMP"** (linked to link out ID `08f4474ef4a901ee` in the Amp flow).

**How it works:**

| Amp Message Contains | Action | Hardcoded Command Sent |
|---|---|---|
| `AM1` (Operate mode) | Amp is active  -  lower exciter power | `transmit set rfpower=30` **(30 watts)** |
| `AM2` (Standby mode) | Amp is bypassed  -  raise exciter power | `transmit set rfpower=90` **(90 watts)** |

> **These power values are hardcoded.** Edit the `Amp On Lower RF Power` and `Amp Off Raise RF Power` change nodes to match your amplifier's drive requirements.

The command is sent via the **"REQ Out 2"** link out node → **"RadioRequest In"** link in → `flexradio-request`.

> The Amp Integration comment notes: *"This integrates with the Palstar LA-1K amplifier. When the amp is in Operate mode, I set the radio's RF Power to 30 watts. When the amp is in Standby, I increase RF Power to 90 watts. Adjust these values as needed for your setup. Also check the RF Power nodes which prevent exceeding 50 watts when the amp is in Operate mode. Update the limits to match your amplifier's requirements."*

---

### 7. Automatic Pre-Distortion (APD)

**Purpose:** Monitors and controls the FlexRadio's APD (Automatic Pre-Distortion) feature from the dashboard.

**How it works:**
- An inject node fires every **1 second** and reads the `APD.enable` flow variable.
- A `switch` node checks whether APD is enabled (0 or 1) and sets the dashboard button color accordingly.
- If APD is enabled, a second check reads `apd_active` (equalizerActive) to determine calibration status.

**APD button color states:**

| Color | Meaning |
|---|---|
| Red | APD is **off** (`enable = 0`) |
| Green (+ calibrated) | APD is **on** and calibration data exists |
| Yellow | APD is **on** but **no calibration data** |

**Dashboard button:** `APD Status`  -  clicking this button toggles APD on or off by sending `apd enable=1` or `apd enable=0` to the radio.

> **Note from the flow:** *"APD seems a little buggy on my 8600 right now."*

---

### 8. Errors

**Purpose:** Catches errors from FlexRadio nodes, displays a persistent error message on the dashboard, and detects radio disconnection via a dual-path mechanism.

**Error handling:**
- A `catch` node monitors errors from: `flexradio-discovery`, `flexradio-message`, `flexradio-meter`, `flexradio-request`, and `FlexRadio Message Out`.
- Errors are formatted as `HH:MM:SS - SourceNode: error message` and stored in `flow.lastRadioError`.
- The error is logged to the persistent `Radio Status` text widget on the dashboard but does not trigger any UI clearing (to avoid false disconnect events on transient errors).

**Connection state detection  -  dual-path approach:**

The flow uses two complementary mechanisms to detect disconnection and reconnection reliably:

**Path 1  -  Connection events (fast, instant):**
- The `flexradio-message` node emits internal `connection/tcp` status messages as the TCP socket state changes.
- The `Activity Tap` function node intercepts these and acts immediately:

| `connection/tcp` payload | Action |
|---|---|
| `connected` | Sets `radioLinkOK = true`, resets `subscribed` flag, sends `Radio link OK` to Radio Status, triggers `Bind \| Subscribe` |
| `connecting` | Sends `Radio Connecting...` to Radio Status (no other action) |
| `disconnected` | Sets `radioLinkOK = false`, resets `subscribed` flag, sends `Radio Disconnected` to Radio Status, triggers `Clear Radio Displays` |

This path provides **instant** status updates when the TCP layer cleanly reports a state change (e.g., radio software restart, graceful shutdown).

**Path 2  -  Staleness watchdog (fallback, ~15 seconds):**
- Physical ethernet cable pulls do not always produce an immediate TCP socket close event  -  the OS may take 30+ seconds to declare the socket dead via keepalive timeout.
- The `Activity Tap` function also updates a `lastRadioActivity` timestamp on every normal radio data message received.
- A `Watchdog Tick` inject node fires every **5 seconds**.
- `Staleness Check` evaluates the age of `lastRadioActivity`. If no data has arrived for more than **15 seconds** and `radioLinkOK` is not already `false`, it declares the link disconnected and triggers `Clear Radio Displays`.

The two paths are independent  -  whichever fires first wins, and the other becomes a no-op (guarded by the `radioLinkOK` flag).

---

## Dashboard Panel: "Flex Radio 8600"

All nodes render into a single dashboard group named **`Flex Radio 8600`**. The widgets displayed are:

| Order | Widget | Description |
|---|---|---|
| 1 | Slice Display | Vue template showing all in-use VFO slices |
| 2 | Radio Status | Last error or link-status message |
| 3 | Client Name | Connected SmartSDR GUI client |
| 4 | TX Electric Power | Supply watt calculation (8000-series only) |
| 5 | VFO | Active slice frequency |
| 6 | (Mode) | Active slice mode |
| 7 | APD Status | Toggle button with color status |
| 8 | TX Status | TX/RX state indicator |
| 9 | Watts Setting | RF power setting gauge |
| 10 | FWD Watts | Forward RF power gauge |
| 11 | SWR Linear | SWR gauge |
| 12 | Volts | PA supply voltage gauge |
| 13 | Temp | PA temperature gauge (°F) |
| 14 | Fan RPM | Cooling fan speed gauge |

---

## Link Nodes  -  Inputs from Other Flows

| Link In Name | Source Flow | Purpose |
|---|---|---|
| `From AMP` | **LA-1K Amp** flow (link out `08f4474ef4a901ee`) | Receives amp operate/standby state to adjust radio TX power |

### Global Variables Read from Other Flows

| Variable | Set By | Used For |
|---|---|---|
| `global.radioPower` | **Power Flow** | Show/hide dashboard panel |
| `global.amp` | **LA-1K Amp** flow | Amp operate/standby state string |
| `global.ampPower` | **LA-1K Amp** flow | Amp power boolean (for 50 W cap logic) |

---

## Link Nodes  -  Outputs to Other Flows

This flow does **not** send data out to other flows via link nodes. All link outs (`FlexRadio Message Out`, `Meter-out`, `REQ Out 1/2/3`, `ADP Out`) are **internal**  -  they only connect to link ins within the same FlexRadio tab.

---

## Hardcoded Values Summary

| Location | Value | Description |
|---|---|---|
| `Amp On Lower RF Power` change node | `transmit set rfpower=30` | RF power set when amp goes to Operate |
| `Amp Off Raise RF Power` change node | `transmit set rfpower=90` | RF power set when amp goes to Standby |
| `Limit Power if Amp Operate` function | `rfpower > 50` → cap at 50 W | Safety ceiling when amp is in Operate mode |
| `Hide Radio Panel` change node | `"Flex Radio 8600"` | Dashboard group name to hide |
| `Show Radio Panel` change node | `"Flex Radio 8600"` | Dashboard group name to show |
| `Staleness Check` function | `15000 ms` (15 seconds) | Fallback timeout  -  link declared dead if no data received for 15 s and `connection/tcp` event did not already fire |
| `Watchdog Tick` inject | `5` seconds | Interval at which Staleness Check is evaluated |
| `flexradio-discovery` node | Port `4992` | FlexRadio UDP discovery port |
| `Temp` gauge | Max `210 °F` | PA temp display range |
| `FWD Watts` / `Watts Setting` gauges | Max `100 W` | RF power display range |
| `Fan RPM` gauge | Max `2100 RPM` | Fan speed display range |
| `Volts` gauge | Max `15 V` | Supply voltage display range |

---

## How to Disable Dashboard Auto-Hide

The dashboard panel is automatically hidden when `global.radioPower` is `false`. This relies on the **Power Flow** tab using a Shelly relay wired to the radio's REM ON jack. If you are **not** using the Power Flow, follow these steps so the dashboard panel is always visible:

### Option A  -  Delete the Hide/Unhide group entirely

1. In Node-RED, select all nodes inside the **Hide/Unhide Dashboard** group.
2. Delete them.
3. Deploy.
4. The dashboard group `Flex Radio 8600` will remain permanently visible.

### Option B  -  Force the dashboard to always show

1. Open the inject node at the top-left of the **Hide/Unhide Dashboard** group (it currently reads `global.radioPower`).
2. Change the `topic` field from `global.radioPower` to a **static value of `true`**.
3. Double-click the `Show Radio Panel` change node and verify it sets `payload` to `{"group":{"show":["Flex Radio 8600"]}}`.
4. Delete the wire going to the `Hide Radio Panel` change node (or delete that node).
5. Deploy.

### Option C  -  Manually show the group once via inject

1. Add a new inject node anywhere on the canvas.
2. Set its payload to JSON: `{"group":{"show":["Flex Radio 8600"]}}`
3. Wire it to the `Hide-Unhide` `ui-control` node.
4. Click the inject button once.
5. The panel will remain visible until Node-RED restarts (at which point the auto-hide logic will re-evaluate `global.radioPower`).

---

## Flow Architecture Diagram

```
[flexradio-discovery] ──► [ParseClients] ──► [Logged in Client Name]
                     └──► [D-Discovery (debug)]

[flexradio-message]  ──► [FlexRadio Message Out] ──► [APD section]
                                                 ──► [TX/RX Messages]
                                                 ──► [Slice section]
                                                 ──► [Activity Tap]
                                                         ├─ connection/tcp "connected"    ──► Radio Status "OK" + Bind|Subscribe
                                                         ├─ connection/tcp "connecting"   ──► Radio Status "Connecting..."
                                                         ├─ connection/tcp "disconnected" ──► Radio Status "Disconnected" + Clear Displays
                                                         └─ radio data                    ──► update lastRadioActivity timestamp

[Watchdog Tick]      ──► [Staleness Check] (fallback: 15 s no data ──► Radio Status + Clear Displays)

[flexradio-meter]    ──► [Meter-out] ──► [Flex Meters switch]
                                              ├──► Fan RPM gauge
                                              ├──► SWR gauge
                                              ├──► FWD Watts (dBm→W)
                                              ├──► PA Temp (°C→°F)
                                              ├──► PA Current → global
                                              └──► PA Voltage → Volts gauge

[From AMP (link in)] ──► [Set Power based on Amp]
                              ├─ AM1 (Operate) ──► rfpower=30
                              └─ AM2 (Standby) ──► rfpower=90
                                       both ──► [RadioRequest In] ──► [flexradio-request]

[global.radioPower]  ──► [Read Radio Power] ──► [Radio Power Relay Status]
                                                       ├─ false ──► Hide panel
                                                       └─ true  ──► Show panel
```

---

## Troubleshooting

| Problem | Likely Cause | Solution |
|---|---|---|
| Radio not discovered | Not on same subnet | Ensure Node-RED host and radio share the same L2 subnet |
| Dashboard panel not visible | `global.radioPower` is false | See [How to Disable Dashboard Auto-Hide](#how-to-disable-dashboard-auto-hide) |
| "Radio Disconnected" message | TCP disconnect detected instantly via `connection/tcp` event, or no meter/message data for 15+ seconds | Check network cable and that the radio is powered on; `flexradio-meter` and `flexradio-message` nodes must be configured with the correct radio config |
| RF power not changing with amp | Amp flow not running or link out ID mismatch | Verify LA-1K Amp flow is deployed and link node `08f4474ef4a901ee` exists |
| PA Supply Watts always blank | 6000-series radio | This metric requires an 8000-series radio |
| APD button not responding | APD buggy on some firmware | Noted by the flow author  -  behavior may vary by firmware version |
