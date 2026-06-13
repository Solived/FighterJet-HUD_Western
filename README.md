# Solived Fighter Jet HUD — Complete Documentation

## Overview

**Solived's Fighter Jet HUD (Western theme)** is a standalone FiveM resource that replaces the default GTA V cockpit with a physics-driven, F-16 style Head-Up Display. It is designed around real Western fighter jet HUD conventions — speed tape left, altitude tape right, heading tape top, canvas-based pitch ladder with a floating Flight Path Marker (FPM), and a full-featured electronic warfare suite including an RWR scope and lock tone system.

All sounds are synthesised in real time using the **Web Audio API** — no external `.wav` or `.mp3` files are required. Audio characteristics are based on published AN/ALR-67 radar warning receiver specifications and standard NATO cockpit alert frequencies.

The resource runs entirely **standalone** — no dependencies on ESX, QBCore, or any other framework. It communicates exclusively between a Lua client thread and an NUI HTML/JS page via `SendNUIMessage`.

---

---

## Table of Contents

1. [Overview](#overview)
2. [Features](#features)
3. [Requirements & Installation](#requirements--installation)
4. [File Structure](#file-structure)
5. [Configuration Reference](#configuration-reference)
6. [Weapon System](#weapon-system)
7. [HUD Elements](#hud-elements)
8. [Warning System](#warning-system)
9. [Sound System](#sound-system)
10. [Lock & Targeting System](#lock--targeting-system)
11. [RWR System](#rwr-system)
12. [Developer Reference — NUI API](#developer-reference--nui-api)
13. [Adding Custom Vehicles & Weapons](#adding-custom-vehicles--weapons)
14. [Troubleshooting](#troubleshooting)

---

## Credits
Original script made by SmokeyDev named "fighterjet-hud".
Continued by Solived, version v2.
https://forum.cfx.re/t/release-fighter-jet-head-up-display-hud/2282430
https://github.com/jkorek/fighterjet-hud

---

## Features

### Flight Instrumentation
- **Speed tape** (left) — vertical canvas-drawn tape, calibrated airspeed in knots, major ticks every 50 kts with labels, minor every 10 kts, current speed in highlighted centre box
- **Altitude tape** (right) — vertical canvas tape, MSL altitude in feet, major ticks every 500 ft, minor every 100 ft, current altitude box
- **Heading tape** (top) — horizontal DOM-based scrolling tape, 5 px per degree, cardinal/intercardinal labels, 300 px visible window
- **Pitch ladder** — canvas-drawn, 5° increments, solid lines above horizon (climbs), dashed lines below (descents), per F-16 Block 50 specification
- **Full-width horizon line** — the 0° pitch line spans the entire canvas width; no number is displayed at 0°, per real F-16 convention
- **Roll indicator** — arc at top of pitch canvas, tick marks at 10/20/30/45/60°, rotating triangle pointer
- **Flight Path Marker (FPM / Velocity Vector)** — circle with wing stubs and vertical tab; displaced below the gun cross (waterline) by the current Angle of Attack in degrees; not affected by roll rotation, represents true flight vector
- **Waterline / Gun Cross** — fixed screen-space reference for aircraft nose direction; the gap between waterline and FPM is always equal to AOA

### Data Readouts
- Mach number (left of speed tape)
- G-force with colour coding — normal (theme colour), caution yellow at ≥ 5.5G, critical red at ≥ 8G
- Angle of Attack in degrees (left of speed tape)
- Vertical speed in ft/min with up/down arrow, amber at > 3,000 fpm
- AGL radar altitude (right of altitude tape), derived from `GetEntityHeightAboveGround`
- Fuel level percentage with progress bar (changes colour at 20% and 10%)
- Engine health bar (maps GTA's 0–1000 range to 0–100%, changes colour at 50% and 30%)
- Gear state label — RETRACTED, DEPLOYING, DEPLOYED, RETRACTING, MALFUNCTION
- VTOL direction label when the vehicle supports it

### Weapon System
- **Seven weapon types** with distinct reticles: machinegun, cannon (CCIP), missiles (ASE circle + lock diamond), JDAM (corner-bracket designator), Paveway LGB (ring + crosshair), AGM/ARM, rockets (CCIP ring)
- **Named weapon annunciator** — bottom-centre panel displays the full military designation (`AIM-9X SIDEWINDER`), sub-designation line (`AAM · IMAGING IR SEEKER`), lock status, and target distance in feet
- Reticles auto-position to the locked or boresight screen coordinate each frame

### Lock & Targeting
- **Three-state lock system**: 0 = no guidance weapon selected, 1 = scanning (seeker active, no lock), 2 = locked
- Lock state is computed from the weapon type and GTA's `GetVehicleLockOnTarget` native
- Each state transition triggers its own audio (see Sound System)
- Target distance displayed in feet when locked

### AOA Indexer
- Three-segment staple on the left side: upper ◁ (AOA > 18°), centre ● (on-speed, 8°–18°), lower ◁ (AOA < 8° and positive)
- Colour-coded: upper = red, centre = green, lower = theme colour

### Warning System
- Six persistent cockpit warnings with individual blink speeds and audio tones
- One prominent missile threat warning
- All warnings fire only on state **transitions** from the Lua side — no NUI spam, sounds start and stop cleanly

### RWR (Radar Warning Receiver)
- Four-sector detection using `IsProjectileInArea` capsule sweeps — Front, Aft, Left, Right
- Directional indicators on a circular scope (styled after AN/ALR-67)
- Audio based on ALR-67 documented pulse-repetition frequencies

### Audio
- Zero external files — all tones synthesised with Web Audio API oscillators
- All sounds stop immediately on vehicle exit, death, or vehicle destruction — no stale audio after explosions
- Graceful AudioContext initialisation; works after first user interaction in FiveM's Chromium NUI

### Themes
- Four phosphor colours: **green** (default), **orange**, **red**, **blue**
- Configured per vehicle model in `config.lua`
- CSS custom properties cascade to all canvas drawing, DOM elements, and glow effects simultaneously

---

## Requirements & Installation

**Requirements**
- A FiveM server running any build that supports NUI resources
- No framework dependency (standalone)

**Installation**
1. Drop the `Solived-FighterJetHUD` folder into your `resources` directory
2. Add `ensure Solived-FighterJetHUD` to your `server.cfg`
3. Open `config.lua` and add any custom vehicle models you want the HUD to activate for (see [Adding Custom Vehicles](#adding-custom-vehicles--weapons))
4. Restart the resource or the server

> **Note:** The resource activates only when a player enters a vehicle whose model name or hash matches an entry in `Config.Vehicles`. Players in non-configured vehicles are completely unaffected.

---

## File Structure

```
Solived-FighterJetHUD/
├── fxmanifest.lua          — Resource manifest (client script, NUI page)
├── config.lua              — All server-owner configuration
├── client/
│   └── main.lua            — Lua client thread: physics, weapons, warnings, NUI bridge
└── html/
    ├── index.html          — HUD layout (canvases, weapon reticles, RWR, panels)
    ├── style.css           — CSS: themes, layout, animations, all HUD element styles
    └── script.js           — HUD engine: Web Audio, canvas drawing, state handling
```

### `fxmanifest.lua`
Declares `client_script 'client/main.lua'` and `ui_page 'html/index.html'`. No server-side scripts. No dependencies.

---

## Configuration Reference

All configuration lives in `config.lua` in the root of the resource. There is no in-game configuration interface; changes take effect on resource restart.

---

### `Config.Speed`
**Type:** `string` — `"knots"` | `"kilometers"` | `"miles"`  
**Default:** `"knots"`

Unit used for the speed tape and overspeed threshold comparison.

| Value | Unit | Multiplier from m/s |
|---|---|---|
| `"knots"` | Nautical miles per hour | × 1.944 |
| `"kilometers"` | km/h | × 3.6 |
| `"miles"` | mph | × 2.237 |

```lua
Config.Speed = "knots"
```

---

### `Config.Altitude`
**Type:** `string` — `"feet"` | `"meters"`  
**Default:** `"feet"`

Unit used for the altitude tape, AGL readout, and altitude warning threshold.

```lua
Config.Altitude = "feet"
```

---

### `Config.Vehicles`
**Type:** `table` of vehicle entries  
**Required fields per entry:** `model`, `color`, `retractableGear`, `vtol`

This table controls which vehicles activate the HUD and how each is configured.

```lua
Config.Vehicles = {
    { model = "lazer",   color = "green",  retractableGear = true,  vtol = false },
    { model = "hydra",   color = "red",    retractableGear = true,  vtol = true  },
    { model = "f16c",    color = "green",  retractableGear = true,  vtol = false },
    { model = "stunt",   color = "blue",   retractableGear = false, vtol = false },
}
```

| Field | Type | Description |
|---|---|---|
| `model` | `string` | Vehicle model name (as used in `GetHashKey`) |
| `color` | `string` | HUD phosphor colour: `"green"`, `"orange"`, `"red"`, `"blue"` |
| `retractableGear` | `boolean` | If `false`, the gear state panel is hidden entirely |
| `vtol` | `boolean` | If `true`, the VTOL direction is read and shown via `GetPlaneVtolDirection` |

> **Tip:** Aircraft without retractable gear (like the Stunt) should use `retractableGear = false` to prevent the gear panel from showing a permanently "DEPLOYED" state.

---

### `Config.OnlyFirstPerson`
**Type:** `boolean`  
**Default:** `true`

When `true`, the HUD is only visible in first-person cockpit view (camera mode 4). Switching to third-person automatically hides the HUD; returning to first-person re-shows it. When `false`, the HUD is always visible while in a configured vehicle.

```lua
Config.OnlyFirstPerson = true
```

---

### `Config.DisableRadar`
**Type:** `boolean`  
**Default:** `false`

Hides the minimap/radar while the HUD is active. Uses `DisplayRadar(false)` each frame when in a configured vehicle. The radar is restored on vehicle exit.

```lua
Config.DisableRadar = false
```

---

### `Config.StallWarning`
**Type:** `boolean`  
**Default:** `false`

Enables the stall warning. Triggers when the aircraft is above 80 m AGL, moving slower than 20 m/s (~39 kts), and not in VTOL mode. Plays a rapid 1050 Hz square-wave buzzer (stick-shaker simulation).

```lua
Config.StallWarning = false
```

---

### `Config.AltitudeWarning`
**Type:** `boolean`  
**Default:** `true`

Enables the PULL UP / altitude warning. Active only after the aircraft has climbed above 100 m (sets `wasInAir = true`), preventing it from firing on takeoff roll.

```lua
Config.AltitudeWarning = true
```

### `Config.AltitudeWarningHeigth`
**Type:** `number`  
**Default:** `50`

The threshold in the configured altitude unit. Warning fires when the aircraft is below this height, moving faster than 50 m/s, and with gear up.

```lua
Config.AltitudeWarningHeigth = 50   -- feet (or metres if Config.Altitude = "meters")
```

---

### `Config.GForceWarning`
**Type:** `boolean`  
**Default:** `true`

Enables the HIGH G warning.

```lua
Config.GForceWarning = true
```

### `Config.GForceWarningThreshold`
**Type:** `number`  
**Default:** `7.5`

G-force value above which the HIGH G warning triggers. The G-force computation accounts for gravity (1G at rest, increases in pulls, can go negative in push-overs). A low-pass filter with α = 0.25 removes per-frame jitter.

```lua
Config.GForceWarningThreshold = 7.5
```

---

### `Config.OverspeedWarning`
**Type:** `boolean`  
**Default:** `true`

Enables the OVERSPD warning.

```lua
Config.OverspeedWarning = true
```

### `Config.OverspeedThresholdKnots`
**Type:** `number`  
**Default:** `850`

Speed threshold **always in knots**, regardless of `Config.Speed`. The internal comparison converts this to m/s. This keeps the threshold intuitive regardless of the display unit.

```lua
Config.OverspeedThresholdKnots = 850
```

---

### `Config.LowFuelWarning`
**Type:** `boolean`  
**Default:** `true`

Enables the BINGO fuel warning.

```lua
Config.LowFuelWarning = true
```

### `Config.LowFuelThreshold`
**Type:** `number` (percentage)  
**Default:** `20`

Fuel level percentage (0–100) below which the BINGO warning fires.

```lua
Config.LowFuelThreshold = 20
```

---

### `Config.EngineDamageWarning`
**Type:** `boolean`  
**Default:** `true`

Enables the ENGINE warning.

```lua
Config.EngineDamageWarning = true
```

### `Config.EngineDamageThreshold`
**Type:** `number`  
**Default:** `400`

GTA engine health value (0–1000) below which the ENGINE warning fires. GTA's `GetVehicleEngineHealth` returns 1000 for full health, 0 for destroyed. The engine bar on the HUD converts this to a percentage.

```lua
Config.EngineDamageThreshold = 400   -- fires below 40% engine health
```

---

### `Config.ShowMach` / `Config.ShowGForce` / `Config.ShowAOA` / `Config.ShowFuel` / `Config.ShowEngineHealth`
**Type:** `boolean`  
**Default:** all `true`

These flags currently flow through the NUI data packet but are intended as future toggles for individual readout visibility. They do not suppress warnings — the warning system is independent.

---

### `Config.RWR`
**Type:** `boolean`  
**Default:** `true`

Enables the four-sector Radar Warning Receiver. When `false`, the resource falls back to a single rear-cone projectile check (`IsProjectileInArea` behind the aircraft) and the RWR scope is not updated.

```lua
Config.RWR = true
```

### `Config.RWRDetectionRange`
**Type:** `number` (metres)  
**Default:** `350.0`

The length of each sector detection beam. Each sector sweeps from 10 m out from the aircraft centre to this distance. Larger values detect projectiles at greater range but may cause false positives in very busy environments.

```lua
Config.RWRDetectionRange = 350.0
```

---

### `Config.Weapons`
**Type:** `table` — weapon hash → `{ name, type, sub }`

Maps GTA vehicle weapon hashes to display data. See the [Weapon System](#weapon-system) section for full details.

---

## Weapon System

### How It Works

Each frame, `GetCurrentPedVehicleWeapon(ped)` returns the currently selected vehicle weapon hash. The Lua script looks this hash up in `Config.Weapons` to get the human-readable name, the weapon category (`type`), and the sub-designation line.

```lua
Config.Weapons = {
    [hash] = { name = "Display Name", type = "category", sub = "Designation line" },
}
```

The `type` field controls which reticle is shown and whether the lock system activates.

### Weapon Types

| `type` value | Reticle displayed | Lock system |
|---|---|---|
| `"machinegun"` | Gun cross (small dashed ring with crosshairs) | No |
| `"cannon"` | CCIP pipper ring with dot and vertical stem | No |
| `"missiles"` | ASE dashed circle; lock diamond appears when locked | Yes |
| `"rockets"` | CCIP ring (larger than cannon), dot, stem | No |
| `"agm"` | Ring with top/bottom spokes; SEARCHING / TRACKING label | Yes |
| `"paveway"` | Dual-ring with crosshair; LASER / LASER LOCK label | Yes |
| `"jdam"` | Corner-bracket designator; NO DESIG / GPS LOCK label | Yes |

Weapons with `type` in `{ "missiles", "agm", "paveway", "jdam" }` activate the three-state lock system. All others leave `lockState = 0`.

### Default Weapon Hashes

```lua
Config.Weapons = {
    [-494786007]  = { name="M61A1 VULCAN",       type="machinegun", sub="20MM · 6000 RPM"           },
    [-647191524]  = { name="M61A1 VULCAN",       type="cannon",     sub="20MM CCIP MODE"            },
    [-821520672]  = { name="AIM-9M SIDEWINDER",  type="missiles",   sub="AAM · IR SEEKER"           },
    [-1521643342] = { name="AIM-120 AMRAAM",     type="missiles",   sub="AAM · ACTIVE RADAR"        },
    [97627437]    = { name="AIM-9X SIDEWINDER",  type="missiles",   sub="AAM · IMAGING IR SEEKER"   },
    [1614138248]  = { name="AGM-88C HARM",       type="agm",        sub="ARM · PASSIVE RADAR SEEKER"},
    [-1353401742] = { name="GBU-12 PAVEWAY II",  type="paveway",    sub="LGB · LASER GUIDED BOMB"   },
}
```

### Finding Weapon Hashes

The GTA weapon hash for a vehicle weapon can be retrieved in-game with:

```lua
local hasWeapon, hash = GetCurrentPedVehicleWeapon(GetPlayerPed(-1))
print(hash)
```

Or via any online GTA V native hash database by searching the model name. Add the hash as the table key with a **negative** sign if the raw hash is greater than `2^31 - 1` (Lua integer overflow for signed 32-bit).

### Adding a New Weapon

1. Get the weapon hash
2. Decide the category (`type`)
3. Add to `Config.Weapons`:

```lua
[1234567890] = { name="AIM-132 ASRAAM", type="missiles", sub="AAM · IMAGING IR" },
```

No code changes needed — the Lua script reads this table dynamically each frame.

---

## HUD Elements

### Speed Tape (Left)
Canvas element, 80 × 300 px logical. Scale: **2.4 px per knot**. Visible range: approximately ±62 knots from current speed. Ticks every 10 knots; labelled every 50. Current speed displayed in a bordered centre box. The tape rule line runs along the right edge.

### Altitude Tape (Right)
Canvas element, 80 × 300 px logical. Scale: **0.038 px per foot**. Visible range: approximately ±3,900 ft. Ticks every 100 ft; labelled every 500 ft. Current altitude in bordered centre box. Tape rule line along the left edge.

### Heading Tape (Top)
DOM-based scrolling div, 300 px visible window. **5 px per degree**, giving 5,400 px total across 3 looped copies for seamless wrapping. Tick marks every 5°; major marks every 10°; cardinal labels (N, 030, 060, E…) every 30°. A fixed downward caret marks the current heading. The heading box shows a zero-padded three-digit value (e.g., `270`).

### Pitch Canvas (Centre)
**500 × 420 px** logical canvas, DPI-scaled. The coordinate system centres the canvas at the aircraft's current pitch angle (`CY = 210`). Pixels-per-degree constant: `PPD = 13.5`.

Key relationships:
- **Horizon line Y position** = `CY + pitch × PPD` (below centre when pitched up, above when pitched down)
- **FPM Y position** = `CY + aoa × PPD` (always below the waterline by the current AOA; at zero AOA the FPM sits on the waterline)
- **Pitch line for degree `d`** = `CY - (d - pitch) × PPD`

The pitch ladder is drawn inside a `ctx.save/rotate(-roll)/restore` block so it rotates with aircraft bank. The FPM and waterline are drawn after `restore` and therefore remain in screen space regardless of roll — this is the correct real-world behaviour.

### Roll Indicator
Drawn on the pitch canvas above the ladder. Arc from −65° to +65° with a radius of 128 px. Tick marks at ±10/20/30/45/60°; the 30° and 60° ticks are longer for quick reference. A filled triangle pointer rotates along the arc to show current bank angle.

### AOA Indexer
Three segments left of the speed tape:
- Upper ◁ — AOA > 18° (too slow / too much alpha) — red
- Centre ● — AOA 8°–18° (on-speed bracket) — green  
- Lower ◁ — AOA > 0° but < 8° (fast / low alpha) — theme colour

### Weapon Panel
Centred at the bottom of the display. Shows:
- Weapon name (e.g., `AIM-9X SIDEWINDER`)
- Sub-designation (e.g., `AAM · IMAGING IR SEEKER`)
- Lock status: `◈ SCANNING` (amber) or `◉ LOCKED` (red)
- Target distance in feet when locked

### Systems Panel
Bottom-left. Fuel and engine health bars with percentage labels. Bar colour changes: normal → amber (fuel < 20% / engine < 50%) → red (fuel < 10% / engine < 30%).

### Status Panel
Bottom-right. Shows gear state and VTOL state when applicable.

### RWR Scope
Bottom-right corner. Circular scope with four directional threat indicators (F/A/L/R). Active sectors blink. See [RWR System](#rwr-system).

---

## Warning System

Warnings are managed as a centralised state table in Lua (`warnState`). The `CheckWarn(key, condition)` helper fires a NUI message **only on transitions** — start fires once when a condition becomes true, end fires once when it becomes false. This prevents sending hundreds of identical messages per second.

```lua
local function CheckWarn(key, condition)
    if condition and not warnState[key] then
        warnState[key] = true
        SendNUIMessage({ action = key, mode = "start" })
    elseif not condition and warnState[key] then
        warnState[key] = false
        SendNUIMessage({ action = key, mode = "end" })
    end
end
```

On vehicle exit, death, or vehicle destruction, `ResetState()` clears all flags and sends `{ action = "stopSounds" }` to the NUI, which calls `SND.stopAll()` immediately — stopping every active oscillator and clearing all pending `setTimeout` callbacks before the `hide` message fires.

### Warning Reference

| ID | Label | Blink | Colour | Condition |
|---|---|---|---|---|
| `stall` | STALL | Fast (0.38s) | Red | AGL > 80 m, speed < 20 m/s, not in VTOL |
| `altitude` | PULL UP | Medium (1.0s) | Red | AGL < threshold, speed > 50 m/s, gear up, was in air |
| `highg` | HIGH G | Fast (0.38s) | Red | `abs(smoothedG) > GForceWarningThreshold` |
| `overspeed` | OVERSPD | Medium (1.0s) | Amber | Speed > `OverspeedThresholdKnots / 1.944` m/s |
| `engine` | ENGINE | Slow (1.9s) | Amber | Engine health < `EngineDamageThreshold` |
| `fuel` | BINGO | Slow (1.9s) | Amber | Fuel level < `LowFuelThreshold` |
| `missile` | ⚠ MISSILE | Fast (0.38s) | Red | Any RWR sector active |

The missile warning is prominent — it appears at a larger font size in the centre of the display separate from the warning column.

---

## Sound System

All audio is generated programmatically using the **Web Audio API**. No `.wav`, `.mp3`, or `.ogg` files are loaded or streamed. This eliminates missing-file errors and keeps the resource self-contained. The AudioContext is created lazily on first use, satisfying Chromium's user-gesture autoplay requirement within FiveM's NUI.

### Sound Engine Architecture

The `SND` module (in `script.js`) exposes four methods:

```
SND.loop(id, beats[])     → play a repeating sequence of tones
SND.warble(id, carrier, modRate, modDepth, vol, wave) → continuous FM tone
SND.stop(id)              → stop a specific sound by ID
SND.stopAll()             → stop everything immediately
```

**`loop`** takes a beat array where each element is either a tone `{ f, t, wave, vol, decay }` or a silence `{ silence: seconds }`. The sequence loops indefinitely until `stop(id)` is called.

**`warble`** creates an FM-modulated oscillator: a modulator oscillator at `modRate` Hz drives the frequency of a carrier oscillator at `carrier` Hz with depth `modDepth`. This produces the characteristic growling/warbling tones used for missile warnings and lock acquisition.

### Warning Sounds

| Warning | Waveform | Characteristics | Real-world basis |
|---|---|---|---|
| **STALL** | Square 1050 Hz | 3 × 60 ms pulses, 180 ms pause | Stick-shaker vibration simulation |
| **PULL UP** | Sine 880 Hz | 3 × 170 ms beeps, 520 ms pause | NATO standard ground proximity alert |
| **HIGH G** | Square 110 Hz | 260 ms on / 130 ms off | G-suit inflation cue, sub-bass rumble |
| **OVERSPEED** | Sine 620 Hz | 220 ms on / 160 ms off | Vmo exceedance alert |
| **BINGO (fuel)** | Sine 440 Hz | 3 × 200 ms beeps, 1.1 s pause | Standard low-fuel advisory tone |
| **ENGINE** | Sawtooth 185/155 Hz | Irregular pattern, 2 alternating | Irregular mechanical failure cue |
| **MISSILE** | FM warble 950 Hz | Carrier 950 Hz, mod 22 Hz, depth 480 | Generic RWR threat warble |

### Lock & RWR Sounds

These are based on **published AN/ALR-67 Radar Warning Receiver specifications**:

| Sound | Specification | Behaviour |
|---|---|---|
| **Scanning (new contact)** | 550 → 520 → 490 → 470 Hz, each 83 ms, then 2.5 s silence | Fires when a guidance weapon is selected but no lock acquired. Repeats every ~2.8 s |
| **Locked (seeker acquired)** | FM sawtooth, 600 Hz carrier, 40 Hz modulator | Continuous tone while lock is maintained. Simulates AIM-9 seeker acquisition growl |
| **RWR radar lock (PRF)** | 455 ↔ 555 Hz alternating at 250 ms each | Fires if RWR scope is active but no missile is inbound (unused in current logic; available for custom integration) |
| **RWR missile launch (fast PRF)** | 455 ↔ 555 Hz alternating at **100 ms** each | Fires when any RWR sector detects an inbound projectile. The faster cadence distinguishes missile guidance from search radar |

The two PRF rates mirror the real ALR-67: slow PRF (250 ms) = search/track radar, fast PRF (100 ms) = active missile guidance uplink.

### Sound Lifecycle

1. Player enters configured vehicle → HUD shows, no sounds playing
2. Flight begins → warnings evaluate per frame, sounds start only on condition transition
3. Weapon selected (guidance type) → `lockState = 1` → scanning chirp starts
4. Lock acquired → `lockState = 2` → chirp stops, seeker growl starts
5. Weapon deselected / non-guidance type selected → `lockState = 0` → all lock sounds stop
6. **Player exits, dies, or vehicle is destroyed** → `ResetState()` → `SND.stopAll()` immediately kills all oscillators → `hide` message follows

---

## Lock & Targeting System

### State Definitions

| `lockState` | Meaning | Audio |
|---|---|---|
| `0` | No guidance weapon selected | Silent |
| `1` | Guidance weapon selected, seeker active, no target locked | ALR-67 new-contact chirp every 2.5 s |
| `2` | Target locked | AIM-9 FM seeker growl (continuous) |

### Lua Logic

```lua
local LOCK_TYPES = { missiles=true, agm=true, jdam=true, paveway=true }

-- computed each frame:
if hasWeapon and LOCK_TYPES[weaponType] then
    lockState = (hasLock == 1) and 2 or 1
else
    lockState = 0
end
```

`hasLock` comes from `GetVehicleLockOnTarget(veh)` which returns `1` when a target is locked.

### JavaScript Transition Handler

State transitions are tracked in `script.js` via `prevLock`. On any change:

```
prevLock → 1  :  SND.stop('lock')  →  LOCK_SND.scanning()
prevLock → 2  :  SND.stop('lock')  →  LOCK_SND.locked()
prevLock → 0  :  SND.stop('lock')
```

The stop-before-start pattern ensures no overlap between the chirp and the growl.

### Reticle Behaviour by Weapon Type

| Type | `hasLock = 0` | `hasLock = 1` |
|---|---|---|
| `missiles` | ASE dashed circle only | Diamond appears inside circle; target distance shown |
| `jdam` | Corner brackets; `NO DESIG` | Corner brackets; `GPS LOCK` + distance |
| `paveway` | Ring + cross; `LASER` | Ring + cross; `LASER LOCK` + distance |
| `agm` | Ring + spokes; `SEARCHING` | Ring + spokes; `TRACKING` + distance |

All reticles are positioned each frame using `World3dToScreen2d` — locked to the target entity, or to a boresight point 150 m ahead when unlocked.

---

## RWR System

The Radar Warning Receiver uses four independent `IsProjectileInArea` capsule sweeps per frame, each sweeping one directional sector. A capsule is defined by a near point (10 m from the vehicle centre) and a far point (`RWRDetectionRange` metres out), along the vehicle's local axis.

```
Front  → GetOffsetFromEntityInWorldCoords(veh,  0.0,    R, 0.0)
Aft    → GetOffsetFromEntityInWorldCoords(veh,  0.0,   -R, 0.0)
Left   → GetOffsetFromEntityInWorldCoords(veh,   -R,  0.0, 0.0)
Right  → GetOffsetFromEntityInWorldCoords(veh,    R,  0.0, 0.0)
```

`anyMissile` is `rwrFront OR rwrRear OR rwrLeft OR rwrRight`. This value drives the `missile` warning and the RWR fast-PRF audio.

When `Config.RWR = false`, a single rear-cone sweep replaces the four-sector system, providing a basic "missile from behind" warning without directional information.

The RWR scope is a circular DOM element with four triangular indicators (F/A/L/R). Active sectors animate with a step-function blink at 0.4 s. The audio (fast PRF) starts on the first active frame and stops when all sectors clear.

> **Performance note:** Four `IsProjectileInArea` calls per 15 ms is well within FiveM's native budget. In very dense combat environments you may want to increase the thread wait or reduce the detection range.

---

## Developer Reference — NUI API

The Lua client communicates with the NUI exclusively through `SendNUIMessage`. The JS side listens on `window.addEventListener('message', ...)`. Third-party resources or modified Lua files can send these messages to interact with the HUD.

### `{ action = "show", color = string }`
Shows the HUD and sets the theme. `color` must be `"green"`, `"orange"`, `"red"`, or `"blue"`.

### `{ action = "hide" }`
Hides the HUD and immediately calls `SND.stopAll()`. Always safe to call even if already hidden.

### `{ action = "stopSounds" }`
Stops all audio without hiding the HUD. Sent by `ResetState()` before `hide` as a belt-and-suspenders audio kill.

### `{ action = "update", ... }`
Full state update, sent every 15 ms. All fields:

```lua
SendNUIMessage({
    action       = "update",
    -- Navigation
    yaw          = number,      -- heading 0-359, degrees CW from north
    pitch        = number,      -- pitch degrees (positive = nose up)
    roll         = number,      -- roll degrees (positive = right bank)
    fpa          = number,      -- flight path angle (not directly drawn, included for future use)
    -- Speed / Altitude
    speed        = number,      -- display speed in configured unit (kts/km/mph)
    altitude     = number,      -- MSL altitude in configured unit (ft or m)
    rawAlt       = number,      -- AGL in metres (from GetEntityHeightAboveGround)
    vertSpeed    = number,      -- vertical speed ft/min (positive = climbing)
    -- Gear / VTOL
    gear         = string,      -- "DEPLOYED"|"RETRACTED"|"DEPLOYING"|"RETRACTING"|"MALFUNCTION"|"STATIC"
    hasVtol      = boolean,
    vtol         = string,      -- "ACTIVE"|"INACTIVE"|"SWITCHING"
    -- Weapon / Target
    hasLock      = number,      -- 0 or 1
    x_target     = number,      -- 0-1 screen X of target / boresight
    y_target     = number,      -- 0-1 screen Y
    targetDist   = number,      -- distance to target in configured altitude unit
    hasWeapon    = boolean,
    weaponType   = string,      -- "machinegun"|"cannon"|"missiles"|"rockets"|"agm"|"paveway"|"jdam"|"none"
    weaponName   = string,      -- e.g. "AIM-9X SIDEWINDER"
    weaponSub    = string,      -- e.g. "AAM · IMAGING IR SEEKER"
    lockState    = number,      -- 0|1|2
    -- Flight data
    mach         = number,      -- Mach number (2 decimal places)
    gForce       = number,      -- smoothed G-force (1 decimal place)
    aoa          = number,      -- angle of attack degrees (1 decimal place)
    -- Systems
    fuelLevel    = number,      -- 0-100 percent
    engineHealth = number,      -- 0-1000 (GTA native value)
    -- RWR
    rwrFront     = boolean,
    rwrRear      = boolean,
    rwrLeft      = boolean,
    rwrRight     = boolean,
})
```

### `{ action = warnKey, mode = "start"|"end" }`
Fires or clears a single warning. `warnKey` is one of: `"stall"`, `"altitude"`, `"missile"`, `"highg"`, `"overspeed"`, `"engine"`, `"fuel"`.

```lua
-- Example: manually trigger overspeed warning from another script
SendNUIMessage({ action = "overspeed", mode = "start" })
-- ... clear it:
SendNUIMessage({ action = "overspeed", mode = "end" })
```

---

## Adding Custom Vehicles & Weapons

### Adding a Vehicle

```lua
Config.Vehicles = {
    -- existing entries ...
    { model = "myfighterjet", color = "orange", retractableGear = true, vtol = false },
}
```

Replace `"myfighterjet"` with the exact model string used by the addon vehicle. The script uses `GetHashKey(v.model)` to convert this to a hash for comparison.

### Adding a Vehicle Weapon

1. Enter a configured vehicle in-game and equip the weapon you want to identify
2. Open F8 console and run: `TriggerEvent('__cfx_internal:gameEventTriggered', 'CEventNetworkEntityDamage', {})`  
   — or use a simple print loop:

```lua
-- paste in an in-game Lua executor:
CreateThread(function()
    local ped = GetPlayerPed(-1)
    while true do
        Wait(500)
        local has, hash = GetCurrentPedVehicleWeapon(ped)
        if has then print("Weapon hash: " .. hash) end
    end
end)
```

3. Note the printed hash, then add to `Config.Weapons`:

```lua
Config.Weapons = {
    -- existing entries ...
    [-987654321] = { name = "R-73 ARCHER", type = "missiles", sub = "AAM · ALL-ASPECT IR" },
}
```

### Adding a Custom Weapon Type with its Own Reticle

If you need a weapon category beyond the seven built-in types, you'll need to modify `script.js`:

1. Add a new `case` in the `updateWeapon` switch block
2. Create corresponding HTML elements in `index.html` with a `.ret` class
3. Style them in `style.css`
4. Optionally add a new entry to `LOCK_TYPES` in `main.lua` if the weapon should trigger the lock system

---

## Troubleshooting

### HUD doesn't appear
- Confirm the vehicle model string in `Config.Vehicles` exactly matches the model name (check with `print(GetEntityModel(GetVehiclePedIsIn(ped, false)))` and decode via a hash database)
- Confirm `Config.OnlyFirstPerson = true` and you are in first-person view (press `V` on PC)
- Check F8 console for Lua script errors

### `GetEntityUpVector` nil error (legacy installs)
This native is not exposed in FiveM Lua. The current script derives the up vector manually from `GetEntityRotation(veh, 2)` using ZXY Euler matrix decomposition:

```lua
local upVec = vector3(
    math.sin(ry) * math.cos(rz) + math.cos(ry) * math.sin(rx) * math.sin(rz),
    math.sin(ry) * math.sin(rz) - math.cos(ry) * math.sin(rx) * math.cos(rz),
    math.cos(ry) * math.cos(rx)
)
```

If you are running an older version of this script that still calls `GetEntityUpVector`, replace that line with the block above.

### Sounds keep playing after the plane explodes
This is the primary bug fixed in the current version. Two changes address it:

1. `ResetState()` now sends `{ action = "stopSounds" }` before the `hide` message
2. The vehicle exit thread explicitly checks `GetEntityHealth(veh) <= 0` in addition to `IsPedInAnyVehicle` — catching explosions within one 50 ms tick
3. The 15 ms HUD thread also checks `DoesEntityExist(veh)` and `GetEntityHealth(veh) <= 0` at the start of each iteration, triggering the exit sequence immediately if the vehicle is gone

If sounds still persist on your server, confirm you are running the latest `main.lua` and that `script.js` handles `action === 'stopSounds'`.

### No audio at all
Web Audio API requires a user gesture to initialise. In FiveM NUI this means the player must have clicked or interacted with the game window at least once. The AudioContext is created lazily on the first sound call — if that first call happens before any user gesture, it will fail silently. All subsequent calls (after any click) will succeed. This is expected browser behaviour and cannot be bypassed.

### Weapon reticle is always in the centre
This happens when `World3dToScreen2d` returns coordinates outside the valid 0–1 range (target is off-screen or behind the camera). The reticle defaults to boresight position `(0.5, 0.5)`. This is correct behaviour — the reticle will snap back to the target as soon as it re-enters the view frustum.

### G-force reads incorrect values on spawn
The G-force calculation uses `(vel - prevVelVec) / FRAME_DT + gravity`. On spawn, `prevVelVec` is `(0, 0, 0)` while the vehicle instantly has velocity, producing a spike. The low-pass filter (`smoothedG = smoothedG × 0.75 + rawG × 0.25`) damps this within 10–15 frames (~150–225 ms). If you need a cleaner transition, initialise `prevVelVec = GetEntityVelocity(veh)` at vehicle entry time.

### Heading tape shows wrong direction / is reversed
GTA's `GetEntityRotation(veh, 2)` returns the Z-rotation (heading) with **positive being counter-clockwise** when viewed from above. The conversion to a compass heading (`if direction < 0 then direction = -direction else direction = 360 - direction end`) normalises this to the standard CW-from-north convention. If the tape appears mirrored, confirm you are using rotation order `2` (ZXY).

---

*This resource is standalone. It has no framework dependencies and makes no server-side calls. All data is local to the client.*
