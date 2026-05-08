# HF Auto — Node-RED Flow Documentation

## Overview

The **HF Auto** flow integrates with an HFAuto ATU (Automatic Antenna Tuner) controller over UDP. It receives telemetry from the tuner, displays live power and SWR readings on a dashboard, and allows the operator to select antennas and control tuner operating mode — with automatic mode enforcement based on antenna type.

---

## Architecture

```
UDP in :13080
    │
    ├─► Debug (D-HFAuto) [inactive]
    │
    └─► RBE filter (deduplicate)
            │
            └─► XML parser
                    │
                    ├─► Change → Peak Power gauge  (ATU_PWR_MAX)
                    ├─► Change → TX Watts gauge     (ATU_PWR)
                    ├─► Change → SWR gauge          (ATU_SWR)
                    ├─► Mode Config fn → Tuner Mode radio (UI sync)
                    └─► Get Ant fn → Ant Selection radio
                                          │
                                    Ant Control switch
                                          │
                          ┌───────────────┼───────────────┐
                        ANT1            ANT2            ANT3
                          │               │               │
                      delay 2s        delay 2s        delay 2s
                          │               └───────────────┘
                     Force BYPASS      Force AUTO (shared)
                          │               │
                          └───────┬───────┘
                              UDP out :12020
```

---

## Interfaces

| Direction | Protocol | Port  | Description                        |
|-----------|----------|-------|------------------------------------|
| In        | UDP      | 13080 | Telemetry XML from HFAuto controller |
| Out       | UDP      | 12020 | Command XML to HFAuto controller (loopback 127.0.0.1) |

---

## Telemetry Fields (Incoming XML)

The HFAuto controller broadcasts XML packets to port 13080. The flow extracts:

| XML Field              | Dashboard Widget  | Notes                        |
|------------------------|-------------------|------------------------------|
| `HFAUTO.ATU_PWR_MAX`   | Peak Watts gauge  | 0–1100 W, green/amber/red    |
| `HFAUTO.ATU_PWR`       | TX Watts gauge    | 0–1100 W, green/amber/red    |
| `HFAUTO.ATU_SWR`       | SWR gauge         | 0–5.0, 2 decimal places      |
| `HFAUTO.ATU_OPER_MODE` | Tuner Mode radio  | BYPASS / AUTO / MANUAL       |
| `HFAUTO.ATU_ANT_NR`    | Ant Selection radio | 1 / 2 / 3                  |

The **RBE (Report By Exception)** node filters duplicate packets — the XML parser only runs when the payload actually changes, reducing unnecessary processing.

---

## Dashboard Controls

### Tuner Mode

A radio button group with three options sent directly to the tuner via UDP:

| Label   | Command sent to tuner |
|---------|-----------------------|
| Bypass  | `<CMD>BYPASS</CMD>`   |
| Auto    | `<CMD>AUTO</CMD>`     |
| Manual  | `<CMD>MANUAL</CMD>`   |

> **Note:** When an antenna is selected via the Ant Selection control, the flow automatically overrides this mode after 2 seconds (see Antenna Selection below). Manual changes during that 2-second window will be overridden.

### Ant Selection

A radio button group with three antennas. Selecting an antenna does two things:

1. **Immediately** sends the antenna select command to the tuner.
2. **After a 2-second delay**, sends an automatic mode command appropriate for the antenna type.

| Label    | ANT# | Auto mode forced | Rationale                          |
|----------|------|------------------|------------------------------------|
| HexBeam  | ANT1 | BYPASS           | Resonant antenna — no tuner needed |
| EFHW     | ANT2 | AUTO             | Non-resonant — tuner required      |
| ZeroFive | ANT3 | AUTO             | Non-resonant — tuner required      |

The 2-second delay gives the tuner time to process the antenna switch before receiving the mode command.

---

## Gauge Color Ranges

### Power Gauges (Peak Watts, TX Watts) — 0 to 1100 W

| Range      | Color  | Meaning         |
|------------|--------|-----------------|
| 0–1400 W   | Green  | Normal operation |
| 1400–1600 W | Amber | Approaching limit |
| 1600–2200 W | Red   | Over limit       |

*(Each color segment represents 100 W; 20 segments total across 0–2000 W scale.)*

### SWR Gauge — 0 to 5.0

| Range    | Color  | Meaning             |
|----------|--------|---------------------|
| 0–1.6    | Green  | Good match          |
| 1.6–2.2  | Amber  | Marginal            |
| 2.2–5.0  | Red    | Poor match / fault  |

---

## Command XML Format

All commands sent to the tuner use this structure:

```xml
<?xml version="1.0" encoding="iso-8859-1"?>
<EXT_CMD><CMD>COMMAND</CMD></EXT_CMD>
```

Where `COMMAND` is one of: `ANT1`, `ANT2`, `ANT3`, `BYPASS`, `AUTO`, `MANUAL`.

---

## Global Variables Written

| Key                          | Type    | Set by       | Value                    |
|------------------------------|---------|--------------|--------------------------|
| `tuner.HFAUTO.ATU_ANT_NR[0]` | integer | `Get Ant` fn | Current antenna number (1/2/3) |

---

## Node Reference

| Node ID              | Type              | Name                        | Role |
|----------------------|-------------------|-----------------------------|------|
| `f61aadba.3187c`     | udp in            | —                           | Receives telemetry on :13080 |
| `36b3091a4078fb6b`   | rbe               | —                           | Drops duplicate packets |
| `76de83fb.8325bc`    | xml               | —                           | Parses incoming XML |
| `ed69e658.3af228`    | change            | —                           | Extracts ATU_PWR_MAX |
| `dcd1b2c5.c5faf8`    | change            | —                           | Extracts ATU_PWR |
| `c07dbd25.ad85f`     | change            | —                           | Extracts ATU_SWR |
| `de104be27d6fcf35`   | function          | Mode Config                 | Maps mode string → command XML; syncs Mode radio |
| `c1f5c51601664052`   | function          | Get Ant                     | Maps antenna number → command XML; syncs Ant Selection radio |
| `0dce0add8c589d92`   | ui-gauge-linear   | Peak Power                  | Displays ATU_PWR_MAX |
| `3a6e46e8e4fe86af`   | ui-gauge-linear   | TX Watts                    | Displays ATU_PWR |
| `1fd3659ee5284b39`   | ui-gauge-linear   | SWR                         | Displays ATU_SWR |
| `1fce4196c0c55488`   | ui-radio-group    | Mode                        | Tuner mode selector; also receives UI sync from Mode Config |
| `9f7d73a86f8576b2`   | ui-radio-group    | Ant Selection               | Antenna selector; also receives UI sync from Get Ant |
| `0a868cebee90b619`   | switch            | Ant Control                 | Routes antenna command to correct delay path |
| `a019a2910581f218`   | delay             | —                           | 2s delay before HexBeam BYPASS enforcement |
| `50e3cb60a5cc4428`   | delay             | —                           | 2s delay before EFHW/ZeroFive AUTO enforcement |
| `3ca910667e902369`   | change            | HexBeam - Force Bypass      | Sets BYPASS command after ANT1 selection |
| `43f27492c8999b1d`   | change            | Non-resonant - Force Auto Tune | Sets AUTO command after ANT2/ANT3 selection |
| `a908fd67.bcfa98`    | udp out           | —                           | Sends commands to tuner on 127.0.0.1:12020 |
| `1922b234.3d88d6`    | debug             | D-HFAuto                   | Debug node (inactive); enable in editor to inspect raw UDP |

---

## Troubleshooting

**Gauges not updating**
- Confirm the HFAuto controller is broadcasting to UDP port 13080 on this machine.
- Enable the `D-HFAuto` debug node in the Node-RED editor to inspect raw incoming packets.
- Check that the RBE node isn't suppressing updates — it passes only on change, so the first packet after a restart or re-deploy will always pass through.

**Antenna commands not being accepted by tuner**
- Verify the tuner is listening on UDP port 12020 on 127.0.0.1.
- The `Ant Control` switch will silently drop messages that don't match any antenna command string exactly. If the `Get Ant` function returns for an unexpected antenna number it logs nothing — check the debug node.

**Mode not switching after antenna change**
- The mode override fires 2 seconds after the antenna command. If the tuner is slow to acknowledge the antenna change, increase the delay node timeouts.
- Manually clicking the Mode radio group during the 2-second window will be overridden by the auto-mode command.

**SWR reading shows 0.00 after transmit**
- Expected — `zeros: true` is set on the SWR gauge so it displays 0.00 when no RF is present. The power gauges suppress zero values (`zeros: false`).
