# Pneumatic ATC Plugin (Beta)

> **IMPORTANT DISCLAIMER:** This plugin is part of my personal ncSender project. If you choose to use it, you do so entirely at your own risk. I am not responsible for any damage, malfunction, or personal injury that may result from the use or misuse of this plugin. Use it with caution and at your own discretion.

Automatic tool changer support for pneumatic ATC systems that use a single aux output to clamp / unclamp the collet. The plugin intercepts `M6` and expands it into a full pick / drop sequence: park at the safe Z, slide the spindle into the fork holder, actuate the pneumatic clamp, and retract.

> **Beta** — the workflow is stable but the config surface (slot layout, TLS strategy) is still evolving. Back up your `settings.json` before adopting.

## Features

### Tool Change (M6)
- Intercepts `M6 Tn` (and `Tn M6`) and drives the spindle through: approach → engage → clamp toggle → retract
- Configurable slot count (1 – 8) with per-slot clamp state tracking
- Manual-tool fallback when the requested tool number is outside the rack
- Pre / Post / Abort event hooks let you toggle coolant, open ATC covers, etc.

### Rack Layout
- **Linear array** – uniform spacing driven by Slot 1 position + orientation (X/Y) + direction (±) + slot distance
- **Custom** – per-slot X/Y coordinates in a table, useful for multi-row racks or non-uniform spacing. Switching from Linear to Custom offers to auto-populate the table from the Linear values so you can start close and fine-tune

### Tool Length Setter (TLS)
- **Probe after every tool change** – always runs TLS on `M6`
- **Use tool library offset (probe when missing)** – reuses the stored TLO from the tool library; probes only when a tool has no offset yet, then writes the value back so subsequent swaps skip the probe
- Optional automatic TLS after the first `$H` (per-session first-home)
- Configurable seek distance, seek feedrate, and TLS aux output for the probe signal

### 3D Probe Tool (T99)
- A dedicated 3D probe that lives in its **own dock** outside the rack and is picked up by the drawbar — `M6 T99` runs the same pneumatic sequence a rack tool gets: air-pressure check, release, descend onto the holder, clamp, seat. No operator prompt at any point
- **Cup** or **Fork** dock holding, with a slide axis independent of the rack's orientation (probe docks are often mounted at 90° to the magazine)
- Optional **Load / Unload Probe G-code** for waking a wireless probe or switching its receiver on and off
- Travel to and from the dock routes around the rack keepout, and starts from where the spindle actually is — including straight after a rack tool has just been put away
- Follows the normal TLS strategy, so the probe's length is measured or read from the library like any other tool
- Off by default; with it disabled T99 behaves exactly as before (manual swap)

### Aux Output Support
- Clamp / unclamp control via `M7`, `M8`, or a numeric `M64 P<n>` / `M65 P<n>` pin
- Same options for the TLS probe signal

### Positioning Commands
- `$SLOT1` … `$SLOT8` – jog the spindle over a slot's engaged position (at Z-safe)
- `$PROBEDOCK` – jog the spindle over the 3D probe dock (at Z-safe) without touching the drawbar, using the same route a real `M6 T99` takes

### Safety
- Halts and prompts the operator when a load / unload sequence needs manual intervention (out-of-rack tool)
- Grabs the current machine XY into any coordinate field so you don't hand-copy values

## Configuration

Open **Plugins → Pneumatic ATC** from the toolbar. The dialog uses a left-side navigation rail mirroring the ncSender Settings dialog.

### Rack Setup
| Section | Setting | Notes |
|---------|---------|-------|
| **Size and Holding** | Number of Slots | 1 – 8 |
| | Clamp Aux Output | `M7` / `M8` / numeric aux pin |
| | Rack Holding | `Fork` (Cup coming soon) |
| **Layout** | Layout mode | `Linear array` or `Custom` |
| | Slot 1 X/Y/Z + Grab | Engaged position of Slot 1 (Z = spindle descent depth) |
| | Orientation / Direction / Slot Distance | Linear mode only |
| | Per-slot table | Custom mode only (X/Y, Engagement Z shared) |
| **Engage Motion** | Slide Direction | ± along the axis perpendicular to Orientation |
| | Slide Distance / Speed | Horizontal travel to enter / leave the fork |
| | Z-Retract | Post-engage clearance |
| **Options** | Show G-Code Commands on Terminal | Reveals the expanded macro output |

### TLS
| Setting | Notes |
|---------|-------|
| TLS Strategy | `Probe after every tool change` (default) or `Use tool library offset (probe when missing)` |
| Tool Setter Location + Grab | Machine XY of the touch plate |
| Seek Distance / Feedrate | Probe motion parameters |
| TLS Aux Output | Optional signal to enable the probe |
| Perform TLS after first `$H` | Run TLS once per session after the first home |

### Manual
Machine XY where the spindle parks and prompts the operator when the requested tool number is outside the rack.

### Probe Tool
| Setting | Notes |
|---------|-------|
| Enable Probe Tool | Registers T99 with the main app. Off by default — with it off, T99 falls through to the manual-swap path |
| Dock X/Y/Z + Grab | Where the probe rests. Z is the **engaged** height, the same meaning as the rack's Engagement Z |
| Dock Holding | `Cup` (lift straight off) or `Fork` (slide in / out to disengage) |
| Slide Axis / Direction / Distance / Speed | Fork docks only. The axis is independent of the rack's Orientation |
| Load Probe G-code | Runs once the probe is clamped and registered as T99, before the TLS routine — e.g. wake a wireless probe |
| Unload Probe G-code | Runs before the spindle sets off for the dock, while the probe is still held — e.g. put the probe to sleep |

Keep the dock clear of the rack's Safety Margin. Travel routes around the keepout either way, but a dock inside it forces long detours.

### Events
Three Monaco G-code editors:
- **Pre Tool Change** – runs before every `M6`
- **Post Tool Change** – runs after every `M6`
- **Abort Event** – runs when the tool change is aborted

## Commands

| Command | Effect |
|---------|--------|
| `M6 Tn` | Full tool change to slot n (or manual position if `n > slots`) |
| `$TLS` | Standalone tool-length probe at the configured setter position |
| `M6 T99` | Pick up / put back the 3D probe from its dock (when the probe tool is enabled) |
| `$SLOT1` … `$SLOT8` | Move the spindle to a slot's XY at Z-safe |
| `$PROBEDOCK` | Move the spindle to the probe dock's XY at Z-safe, without actuating the drawbar |
| `$H` | Home; optionally followed by a TLS routine (see toggle above) |

## Typical Setup

1. **Wire the clamp** to an aux output that you can drive with either `M7/M8` (mist/flood) or a numeric pin (`M64 P<n>`). Confirm the output actuates the collet fixture reliably.
2. **Set the rack** by jogging the spindle to Slot 1's fully-engaged position and hitting **Grab** — this captures machine XY plus the descent Z. Configure Orientation / Direction / Slot Distance to match your rack (Linear mode) or switch to Custom for irregular racks.
3. **Set the TLS location** by touching off a known tool on the setter, then Grab. Pick your strategy — probe every time is safest until you trust the library offsets.
4. **Set up the 3D probe** (optional) on the **Probe Tool** tab: enable it, jog until the spindle nose is seated on the probe holder, and **Grab current** to capture the dock X/Y/Z. Pick Cup or Fork holding, save, then run `$PROBEDOCK` to confirm the spindle lands exactly over the holder before you let `M6 T99` drive the drawbar.
5. **Save**. The plugin registers `M6`, `$TLS`, `$SLOT1..N` and `$PROBEDOCK` handlers and updates ncSender's tool count to match your slot count.

## Installation

Install through the ncSender **Plugins** interface (Add plugin from URL or `.zip`).

## Development

Plugin source lives alongside the other ncSender plugins:
https://github.com/siganberg/ncsender-plugin-pneumaticatc

Main ncSender project: https://github.com/siganberg/ncSender
