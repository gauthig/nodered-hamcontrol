# Node-RED PST Rotator Controller with Great Circle Map

A Node-RED Dashboard 2.0 flow for controlling a **PST rotator** over UDP, featuring a live **azimuthal equidistant (great circle) world map** that displays your current beam heading in real time.

> Based on the original work by [LA9VKA](https://flows.nodered.org/user/LA9VKA). Migrated to Dashboard 2.0 with a new D3.js great circle map and preset bearings dropdown.

---

## Screenshot

> *(Add your own screenshot here)*

---

## Features

- 🌍 **Live great circle map** centered on your QTH — the classic ham radio azimuthal equidistant projection
- 🔴 **Animated beam line** that updates as the rotator moves
- 📡 **UDP control** — sends PST-format XML commands to the rotator controller
- 🎯 **Preset bearings** dropdown for common DX destinations (fully customizable)
- 🔢 **Manual azimuth input** — type any bearing 0–360°
- ⛔ **Stop button** — immediately halts rotation
- 🔄 **Auto-polls** azimuth every 10 seconds to stay in sync while idle

---

## Requirements

| Requirement | Notes |
|---|---|
| [Node-RED](https://nodered.org/) | v3.0 or later recommended |
| [@flowfuse/node-red-dashboard](https://dashboard.flowfuse.com/) | Dashboard 2.0 (not the legacy `node-red-dashboard`) |
| PST rotator controller | Listening on UDP port 12001, accepting commands on UDP port 12000 |
| Internet access (first load) | D3.js and world map data load from CDN on first render |

---

## How It Works

```
UDP In (port 12001)
    └── Switch (filter "AZ:" messages)
        └── Parse AZ (extract 3-digit bearing)
            ├── ui-number-input  (shows / accepts azimuth)
            └── ui-template      (great circle map)

ui-number-input  ──► AZ Input function ──► UDP Out (port 12000)
ui-button (Stop) ──► AZ Stop function  ──► UDP Out (port 12000)
ui-dropdown      ──► AZ Input function ──► UDP Out (port 12000)

inject (every 10s) ──► UDP Out   (polls "AZ?" to keep display current)
```

**UDP message formats (PST XML protocol):**

| Direction | Example |
|---|---|
| Poll current azimuth | `<PST>AZ?</PST>` |
| Received azimuth | `AZ:270` |
| Command: rotate to bearing | `<PST><AZIMUTH>270</AZIMUTH></PST>` |
| Command: stop | `<PST><STOP>1</STOP></PST>` |

---

## Installation

### 1. Install Dashboard 2.0

In Node-RED, go to **Menu → Manage Palette → Install** and search for:

```
@flowfuse/node-red-dashboard
```

### 2. Import the flow

1. Copy the contents of [`rotator-flow.json`](rotator-flow.json) *(export from your Node-RED instance via Menu → Export)*
2. In Node-RED: **Menu → Import → Clipboard**
3. Paste and click **Import**

### 3. Personalise your QTH location

The great circle map is centered on a hardcoded lat/lon. You **must** update this to your own location or the map will be centered on someone else's QTH.

**Step 1 — Find your lat/lon**

Go to [nominatim.openstreetmap.org](https://nominatim.openstreetmap.org/) and search your address, or use Google Maps (right-click on your location → the coordinates appear at the top of the context menu).

**Step 2 — Update the map node**

Open the **`Azimuth Map`** `ui-template` node and find these two lines near the top of the `<script>` section:

```javascript
qthLat: 34.5134,   // ← replace with your latitude
qthLon: -117.2414, // ← replace with your longitude (negative = West)
```

Replace the values with your own coordinates. For example, if you are in London:

```javascript
qthLat: 51.5074,
qthLon: -0.1278,
```

> **Note:** Western longitudes are **negative** (Americas). Eastern longitudes are **positive** (Europe, Asia, Australia).

**Step 3 — Deploy**

Click the red **Deploy** button. The map will re-render centered on your QTH.

---

## Customising the Preset Bearings

The **Preset** dropdown comes loaded with example destinations. Edit the `ui-dropdown` node to replace them with your own:

| Field | Description |
|---|---|
| `label` | The name shown in the dropdown (e.g. `"Japan"`) |
| `value` | The true bearing in degrees from your QTH (e.g. `"305"`) |

To find the true bearing from your QTH to a destination, use a great circle calculator such as [HamCalc](http://www.csgnetwork.com/bearingcalc.html) or [ARRL's online tools](http://www.arrl.org/great-circle-maps-and-antenna-aiming).

---

## Dashboard Layout

The flow uses a single Dashboard 2.0 group. Recommended group width: **6 columns**.

| Widget | Type | Width | Purpose |
|---|---|---|---|
| Preset | Dropdown | 1 | Select a preset bearing |
| Azimuth | Number input | 1 | Manual bearing entry / live display |
| Azimuth Map | Template (great circle map) | 6 | Visual beam heading display |
| Stop | Button (red) | 6 | Halt rotation immediately |

---

## Map Technical Details

The great circle map is rendered using **[D3.js v7](https://d3js.org/)** with the `geoAzimuthalEquidistant` projection — the standard projection used in ham radio for antenna work because **straight lines equal great circle paths**.

| Detail | Value |
|---|---|
| Projection | Azimuthal Equidistant |
| World data | [Natural Earth 110m](https://cdn.jsdelivr.net/npm/world-atlas@2/) via CDN |
| Libraries | D3.js v7, TopoJSON Client v3 (loaded from CDN on first render) |
| Map size | 270 × 270 px (adjustable via `mapSize` in the template) |

The red beam line is drawn using SVG directly — it does **not** require any special Node-RED SVG nodes, which is why this works in Dashboard 2.0.

**Adjusting map size:** If the map is too large or small for your dashboard layout, open the `Azimuth Map` template node and change:

```javascript
mapSize: 270,  // ← increase or decrease this value (pixels)
```

Then re-deploy.

---

## UDP Port Configuration

| Port | Direction | Purpose |
|---|---|---|
| `12001` | Incoming | Receive azimuth status from rotator controller |
| `12000` | Outgoing | Send commands to rotator controller |

Both ports are configured for `127.0.0.1` (localhost). If your rotator controller runs on a different machine, update the **UDP Out** node's address accordingly.

---

## Credits

- Original flow concept and design: **[LA9VKA](https://flows.nodered.org/user/LA9VKA)**
- Dashboard 2.0 migration, D3.js great circle map, and preset dropdown: adapted from LA9VKA's work

---

## License

MIT — free to use, modify, and share. If you improve it, consider sharing your changes back with the ham radio Node-RED community.

---

*73 de your-callsign*
