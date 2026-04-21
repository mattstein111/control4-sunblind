# SunBlinds — Design Plan

Last updated: 2026-04-21

---

## 1. Scope

SunBlinds is an **event-emitter** Control4 DriverWorks driver. It does not directly command shades, blinds, lights, or any other devices. Its sole output is Control4 events and driver variables that dealers/programmers wire into their own automations.

Two driver types are packaged together:

| Driver | Role | Quantity per project |
|---|---|---|
| `SunBlinds Manager` (parent) | Holds location, polls weather, computes sun position, aggregates window state | 1 |
| `SunBlinds Window` (child) | One per window, holds geometry, fires per-window glare events | N |

---

## 2. Parent driver — `SunBlinds Manager`

### Properties

| Property | Type | Default | Notes |
|---|---|---|---|
| Latitude | NUMBER | prefilled from C4 project location | Overridable |
| Longitude | NUMBER | prefilled from C4 project location | Overridable |
| Weather provider | LIST | `Open-Meteo` | v1 fixed |
| Weather poll interval (min) | NUMBER | 10 | |
| Horizon band (±°) | NUMBER | 2 | "at horizon" elevation tolerance |
| Near-horizon band (±°) | NUMBER | 6 | triggers `Sun_Approaching_Horizon` |
| Golden hour elevation range | STRING | `0,6` | min,max in degrees |
| Sun azimuth | readonly | — | live readout |
| Sun elevation | readonly | — | live readout |
| Cloud cover % | readonly | — | live readout |
| Weather condition | readonly | — | e.g. "Clear", "Partly cloudy" |
| Last weather update | readonly | — | ISO timestamp |

### Actions

- `Add Window` — creates a new `SunBlinds Window` child proxy at the parent; dealer moves it to the target room in Composer.
- `Refresh Weather Now`
- `Recompute Sun Now`

Windows are removed by deleting the child proxy in Composer; the parent reconciles on bind scan.

### Events

- `Sunrise`
- `Sunset`
- `Sun_Below_Horizon`
- `Sun_Approaching_Horizon`
- `Sun_At_Horizon`
- `Sun_Above_Horizon`
- `Golden_Hour_Start`
- `Golden_Hour_End`
- `Any_Glare_Active`
- `All_Glare_Cleared`

### Variables

- `SUN_AZIMUTH` (number, degrees 0–360)
- `SUN_ELEVATION` (number, degrees -90..90)
- `CLOUD_COVER` (number, 0–100 %)
- `WEATHER_CONDITION` (string)
- `ANY_GLARE_ACTIVE` (bool)
- `ACTIVE_GLARE_COUNT` (number)

---

## 3. Child driver — `SunBlinds Window`

### Properties

| Property | Type | Default | Notes |
|---|---|---|---|
| Window azimuth (°) | NUMBER | 180 | 0=N, 90=E, 180=S, 270=W |
| View half-angle (°) | NUMBER | 45 | ± around azimuth |
| Min sun elevation (°) | NUMBER | 5 | sun below = no glare |
| Max sun elevation (°) | NUMBER | 70 | sun above = no glare (overhang) |
| Cloud cover threshold (%) | NUMBER | 40 | fires glare only when cloud cover ≤ this |
| On-delay (min) | NUMBER | 5 | hysteresis — sun must stay in zone this long |
| Off-delay (min) | NUMBER | 10 | hysteresis — sun must stay out of zone this long |
| Calibrate: Glare Start datetime | STRING | — | ISO 8601 local, e.g. `2026-04-21T14:30` |
| Calibrate: Glare End datetime | STRING | — | ISO 8601 local |
| Glare Active | readonly bool | — | live |
| Last reason | readonly string | — | e.g. `"sun in arc (az 205°), cloud 22%, elev 34°"` |

### Actions

- `Calibrate From Date/Time Window` — reads both datetime fields; computes:
  - **Window azimuth** = midpoint of sun azimuth at start and end times
  - **View half-angle** = half the azimuthal spread (with a small safety margin)
  - **Min sun elevation** = min of the two elevations (minus a margin)
  - **Max sun elevation** = max of the two elevations (plus a margin) — or leave at default if overhang behavior unknown
  - Stamps the results into the corresponding properties.
- `Test Glare Now` — forces a recompute and logs current reason string.

### Events

- `Glare_Start`
- `Glare_End`

### Variables

- `GLARE_ACTIVE` (bool)
- `WINDOW_AZIMUTH` (number)
- `LAST_GLARE_START` (timestamp)
- `LAST_GLARE_END` (timestamp)

---

## 4. Glare decision logic

For each window, every tick (after any sun or weather update):

```
in_arc         = angular_diff(sun_azimuth, window_azimuth) <= view_half_angle
elev_ok        = min_elevation <= sun_elevation <= max_elevation
sunny_enough   = cloud_cover <= cloud_threshold
raw_glare      = in_arc AND elev_ok AND sunny_enough

# hysteresis
if raw_glare and not glare_active:
    if raw_glare has been true continuously for on_delay → glare_active = true, fire Glare_Start
if not raw_glare and glare_active:
    if raw_glare has been false continuously for off_delay → glare_active = false, fire Glare_End
```

`angular_diff` wraps modulo 360 and always returns the shortest arc.

---

## 5. Sun position algorithm

Standard NOAA solar position algorithm, implemented in pure Lua:

- Inputs: latitude, longitude, UTC datetime.
- Outputs: azimuth (0–360°, clockwise from north), elevation (-90 to +90°).
- Precision target: ±0.1° — more than enough for glare detection.
- Reference: https://gml.noaa.gov/grad/solcalc/solareqns.PDF

Computed on a timer (once per minute by default) and on demand from the `Recompute Sun Now` action and from the calibration action.

---

## 6. Weather polling

Open-Meteo endpoint:

```
https://api.open-meteo.com/v1/forecast?latitude={lat}&longitude={lon}&current=cloud_cover,weather_code&timezone=auto
```

- Poll interval configurable (default 10 min).
- `cloud_cover` drives glare decisions.
- `weather_code` maps to a human-readable `WEATHER_CONDITION` string (Clear, Mainly clear, Partly cloudy, Overcast, Fog, Drizzle, Rain, Snow, Thunderstorm — per Open-Meteo's WMO code table).
- Cached with last-updated timestamp; if a poll fails, the previous value is reused until the next success.

---

## 7. Horizon and golden-hour semantics

| State | Condition |
|---|---|
| `Sun_Below_Horizon` | elevation < −(near-horizon band) |
| `Sun_Approaching_Horizon` | elevation within ± near-horizon band, not yet within horizon band |
| `Sun_At_Horizon` | elevation within ± horizon band |
| `Sun_Above_Horizon` | elevation > +(near-horizon band) |
| `Sunrise` | transition from below → at/above horizon |
| `Sunset` | transition from at/above → below horizon |
| `Golden_Hour_Start/End` | elevation enters/exits the configured range |

`Sun_Approaching_Horizon` fires twice per day (ascending at dawn, descending at dusk); direction is inferred by the dealer from adjacent `Sunrise`/`Sunset` events.

---

## 8. Roadmap

- **Phase 1 — Spec & scaffold** (this plan)
  - [x] Create GitHub repo
  - [x] Write README and PLAN
  - [ ] Create Notion project page under Active Projects
  - [ ] Open GitHub issues for each phase-2 task

- **Phase 2 — Core driver skeletons**
  - [ ] `driver.xml` for parent and child with properties, events, variables, actions
  - [ ] Bindings between parent and child proxies
  - [ ] Build script (`make c4z` or similar) producing two `.c4z` files
  - [ ] Icons: 16×16 `device_sm.png`, 32×32 `device_lg.png`, full experience ladder (20…1024) — sun-with-sunglasses emoji as source for now

- **Phase 3 — Sun + weather engines**
  - [ ] Pure-Lua NOAA solar position module, unit-tested against known fixtures
  - [ ] Open-Meteo client with retry/backoff
  - [ ] Parent periodic tick (sun every 1 min, weather every 10 min)

- **Phase 4 — Glare logic**
  - [ ] Per-window decision loop with hysteresis state machine
  - [ ] Parent aggregation (`ANY_GLARE_ACTIVE`, count)

- **Phase 5 — Calibration**
  - [ ] `Calibrate From Date/Time Window` action — back-calculates azimuth, half-angle, elevation bounds from two observed timestamps

- **Phase 6 — Polish**
  - [ ] GitHub Releases autoupdater (self-install via `UpdateProjectC4i` SOAP + esphome handshake — see `~/.claude/c4-conventions.md` §3 and §3a)
  - [ ] Real icon (replace sunglasses emoji)
  - [ ] Dealer documentation HTML (`www/documentation.html`)
  - [ ] Release 1.0.0

---

## 9. Open questions

- Final driver icon — using 🕶️☀️ (sun + sunglasses emoji) as placeholder; will design a proper one before 1.0.
- Should `Calibrate From Date/Time Window` *also* honor a third "glare observed midday at worst angle" datetime for peak-elevation calibration? Deferred unless needed.
- License — MIT? Apache 2.0? Decide before first public release.
