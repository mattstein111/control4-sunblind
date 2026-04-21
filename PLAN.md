# SunBlinds — Design Plan

Last updated: 2026-04-21

---

## 1. Scope

SunBlinds is a Control4 DriverWorks driver suite that models each window as a **physical geometry** — wall orientation, horizon obstructions, overhang — and uses that geometry to fire per-window glare events. Optionally, each window can be upgraded to **Controlled mode** by adding protected-point geometry and binding a motorized shade — in which case the same physics model also computes the minimum blind descent needed to block direct sun from the protected point.

One physics model, shared across both modes. Event Only is the same solver asking "does any direct sun enter the room?"; Controlled is the same solver asking "does direct sun reach *this point*, and if so how much blind descent blocks it?"

Each window proxy runs in one of two modes:

- **Event Only** — fires `Glare_Start`/`Glare_End` events when direct sun is entering the room. Dealer programs the response. No direct shade control. Requires 3 geometric parameters (see §8).
- **Controlled** — commands a bound motorized shade directly, descending only as far as required to block direct sun from the configured protected point. Requires the 3 Event-Only params plus 4 protected-point params (7 total).

Both modes share the same calibration machinery:

- **Manual calibration** — dealer records datetime stamps of observed glare start and end at a reference point (any sunny spot for Event Only; the specific protected point for Controlled). Each pair gives 2 constraints. 2+ pairs fit the model.
- **Self-learn with lux sensor** — lux sensor at the reference point drives automated micro-probing and least-squares fitting. Raw samples discarded after convergence; only the fitted parameters persist.

Two driver types are packaged together:

| Driver | Role | Quantity per project |
|---|---|---|
| `SunBlinds Manager` (parent) | Holds location, polls weather, computes sun position, aggregates window state | 1 |
| `SunBlinds Window` (child) | One per window. Runs the physics solver. Fires events. In Controlled mode, also binds a shade and commands position. | N |

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

### Common properties (both modes)

| Property | Type | Default | Notes |
|---|---|---|---|
| Mode | LIST | `Event Only` | `Event Only` / `Controlled` |
| Wall azimuth (°) | NUMBER | 180 | 0=N, 90=E, 180=S, 270=W — outward normal of the window's wall |
| Horizon obstruction elevation (°) | NUMBER | 5 | sun below this angle cannot illuminate the window (trees, hill, neighbor) |
| Overhang cutoff elevation (°) | NUMBER | 70 | sun above this angle is blocked by roof eave / overhang |
| Cloud cover threshold (%) | NUMBER | 40 | direct-sun gating threshold — skip glare when clouds above this |
| On-delay (min) | NUMBER | 5 | hysteresis — condition must hold this long before `Glare_Start` |
| Off-delay (min) | NUMBER | 10 | hysteresis — condition must clear this long before `Glare_End` |
| Calibration method | LIST | `Manual datetime` | `Manual datetime` / `Self-learn (lux sensor)` |
| Lux sensor binding | binding | — | optional in Event Only, enables self-learn calibration |
| Lux threshold | NUMBER | 1000 | glare threshold at reference point (lux) — only used if lux sensor bound |
| Observation log | STRING (readonly) | — | list of recorded `(start_dt, end_dt)` calibration pairs |
| Model converged | readonly bool | — | fit has enough constraints and residual is acceptable |
| Model parameters | readonly string | — | e.g. `"wall_az=182°, horizon=4°, overhang=64°"` |
| Glare Active | readonly bool | — | live |
| Last reason | readonly string | — | e.g. `"sun ray enters room (az 205°, elev 34°), clouds 22%"` |

### Controlled-mode properties (additional)

| Property | Type | Default | Notes |
|---|---|---|---|
| Shade binding | binding | — | C4 connection to a motorized shade/blind driver |
| Control surface | LIST | `Auto-detect` | `Percent` / `Timed Up-Down-Stop` / `Preset Slots` / `Binary` / `Auto-detect` |
| Shade travel time (s) | NUMBER | — | for Timed control surface — seconds from fully up to fully down |
| Shade travel length (cm) | NUMBER | — | physical distance blind covers from fully up to fully down |
| Preset slot map | STRING | — | for Preset control surface — e.g. `"1=25, 2=50, 3=75, 4=100"` |
| Blind opacity | LIST | `Blackout` | `Blackout` / `Room-darkening` / `Sheer` |
| Window top height (cm) | NUMBER | — | top edge of window above the floor (physics param, can be calibrated) |
| Protected point depth (cm) | NUMBER | — | distance from window plane into room (physics param, can be calibrated) |
| Protected point height (cm) | NUMBER | — | height of protected point above floor (physics param, can be calibrated) |
| Probe interval (min) | NUMBER | 10 | minimum time between probe moves during self-learn |
| Probe step (%) | NUMBER | 5 | smallest blind position increment |
| Manual override timeout (min) | NUMBER | 30 | pause auto after a detected manual move |
| Target position (%) | readonly | — | last computed target |
| Current position (%) | readonly | — | last commanded / reported position |
| Learning samples count | readonly | — | during self-learn |

### Actions (both modes)

- `Record Calibration Observation` — reads the Calibrate Start/End datetime fields (entered by dealer) and appends this pair to the observation log. Calibration fits as many model parameters as the current observations can constrain.
  - Event Only mode: dealer enters the datetimes when glare was observed starting and ending anywhere in the sunlit zone. Each pair constrains 2 of the 3 Event-Only params; 2+ pairs fit the model.
  - Controlled mode: dealer enters the datetimes while **blind was fully closed** and glare was observed at the protected point. Each pair constrains 2 of the 7 params; 4+ pairs fit the full model (fewer if some params are provided by tape measure).
- `Start Self-Learn` — requires lux sensor binding. Records `(timestamp, az, elev, cloud_cover, blind_position, lux)` at 15-min resolution. In Controlled mode, actively nudges the blind by probe step every probe interval during clear-sky periods.
- `Stop Self-Learn` — freezes learning, keeps current model in effect.
- `Refit Model Now` — re-runs least-squares fit on current observations.
- `Discard Calibration Data` — wipes observations and model parameters.
- `Test Glare Now` — logs current reason, and in Controlled mode, what position would be commanded.
- `Manual Pause (N minutes)` — Controlled mode — temporarily halts auto control.

### Events

- `Glare_Start` / `Glare_End` — both modes
- `Blind_Adjusted` — Controlled mode (from%, to%, reason)
- `Manual_Override_Detected` — Controlled mode when position feedback disagrees with last command
- `Model_Converged` — both modes, when fit is confident

### Variables

- `GLARE_ACTIVE` (bool) — **primary state variable** for dealer programming. True while the window is currently in glare. Use this in IF-conditions when triggering off some other event (TV on, light switched on, door opened) to check whether glare is currently a concern.
- `SUN_ENTERING_ROOM` (bool) — true whenever direct sun is geometrically reaching the room interior, regardless of cloud cover. Useful if the dealer wants to ignore the cloud threshold.
- `WALL_AZIMUTH` (number)
- `LAST_GLARE_START` / `LAST_GLARE_END` (timestamp) — useful for "has glare been active in the last N minutes" conditional logic
- `GLARE_ACTIVE_DURATION_SEC` (number) — seconds since `LAST_GLARE_START`; 0 when not active. Lets dealer write `IF glare has been active for > 10 minutes THEN ...`
- `MODEL_CONVERGED` (bool)
- Controlled-only: `TARGET_POSITION`, `CURRENT_POSITION`, `LUX_CURRENT`, `LEARNING_SAMPLES_COUNT`

---

## 4. Glare decision logic

For each window, every tick, the physics solver is invoked with the current sun position and the window's fitted parameters. The solver returns:

- `sun_entering_room` (bool) — does a direct sun ray currently pass through the window plane into the room interior, given wall_azimuth + horizon_obstruction + overhang_cutoff?
- `sun_reaches_protected_point` (bool, Controlled only)
- `required_blind_position` (%, Controlled only)

### Event Only

```
raw_glare = sun_entering_room AND cloud_cover <= cloud_threshold
```

### Controlled

```
raw_glare = sun_reaches_protected_point AND cloud_cover <= cloud_threshold
# raw_glare implies required_blind_position > 0
```

### Common hysteresis + dispatch

```
if raw_glare and not glare_active and raw_glare held for on_delay:
    glare_active = true, fire Glare_Start
if not raw_glare and glare_active and not_raw_glare held for off_delay:
    glare_active = false, fire Glare_End

# Controlled only
if Mode == Controlled and model_converged:
    command shade to required_blind_position, rounded to probe_step
    if position changed: fire Blind_Adjusted
```

### `sun_entering_room` (Event Only subset of the physics)

```
in_azimuth_range = |angular_diff(sun_azimuth, wall_azimuth)| < 90°   -- sun must be on window's side of the wall
elev_ok          = horizon_obstruction < sun_elevation < overhang_cutoff
sun_entering_room = in_azimuth_range AND elev_ok
```

This is the degenerate case of the full physics solver when no protected point is configured. The 90° half-arc is physical fact (a window can only see sun that is on the exterior side of its wall plane), not a tuning knob.

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

## 8. Unified physics model

Every window is a physics model. Event Only and Controlled differ only in which parameters are in play and which questions the solver answers.

### Parameter set

| Parameter | Units | Event Only | Controlled | Meaning |
|---|---|---|---|---|
| `wall_azimuth` | ° (0–360) | required | required | Compass direction the window's outward normal points |
| `horizon_obstruction_elevation` | ° | required | required | Sun below this angle cannot illuminate the window (trees, hill, building) |
| `overhang_cutoff_elevation` | ° | required | required | Sun above this angle is blocked by roof eave |
| `window_top_height` | cm above floor | — | required | Top edge of window opening |
| `point_depth` | cm from window plane | — | required | Distance to protected point |
| `point_height` | cm above floor | — | required | Height of protected point |
| `blind_travel_length` | cm | — | required | Measured during control-surface calibration, not a fitted param |

Blind bottom height at position P% closed is `window_top_height − P/100 × blind_travel_length`.

### The two solver outputs

**`sun_entering_room`** (Event Only answer): direct sun reaches the room interior right now.

```
in_wall_arc    = |angular_diff(sun_azimuth, wall_azimuth)| < 90°
elev_ok        = horizon_obstruction_elevation < sun_elevation < overhang_cutoff_elevation
sun_entering_room = in_wall_arc AND elev_ok
```

**`required_blind_position`** (Controlled answer): smallest P% that blocks the sun ray from reaching the protected point.

```
if not sun_entering_room:
    required_blind_position = 0   -- sun can't reach the protected point anyway
else:
    project sun vector into room-local frame (rotate by wall_azimuth)
    compute the ray from protected point back toward the sun
    find h_window = height where that ray crosses the window plane
    if h_window > window_top_height:
        required_blind_position = 0   -- sun comes in above window top (geometrically impossible given overhang, but clamp)
    else:
        required_blind_bottom = h_window − safety_margin
        required_blind_position = clamp(0..100, (window_top_height − required_blind_bottom) / blind_travel_length × 100)
        round to nearest probe_step
```

### Fitting parameters — one pipeline, two flavors

Both modes use the same calibration actions and the same least-squares solver. The only difference is how many parameters are being fit and what the observations constrain.

**Manual datetime observations**

Each observation pair `(start_dt, end_dt)` encodes: *"between these two moments, the [sun was reaching the reference point / blind was no longer blocking sun at the protected point]."*

For Event Only — reference point is any sunny spot in the room's interior. Each pair constrains `wall_azimuth`, `horizon_obstruction_elevation`, `overhang_cutoff_elevation`:
- `start_dt` sun position is on the boundary where `sun_entering_room` transitions false → true
- `end_dt` sun position is on the boundary where `sun_entering_room` transitions true → false

For Controlled (`blind fully closed`) — reference point is the protected point. Each pair constrains all 7 params:
- sun position at `start_dt` is exactly aligned with the ray from the fully-closed blind's bottom edge to the protected point (glare just started leaking past)
- sun position at `end_dt` similarly (glare just stopped reaching the protected point as the sun moved)

**Observation count needed for convergence**:
- Event Only (3 params): **2 pairs minimum**, 3+ recommended for robustness
- Controlled (7 params): **4 pairs minimum** (preferably spread across seasons), 5–6 for robustness. Tape-measure shortcut — dealer enters `window_top_height`, `point_depth`, `point_height` directly, reducing to 4 fit params (same as Event Only + `wall_azimuth` already known) and 2 pairs suffice.

**Self-learn with lux sensor** — same flow as previously designed (15-min sampling, probing in Controlled mode only, least-squares fit). Sensor placement determines which reference point is being calibrated; same solver fits either 3 or 7 params accordingly.

### After convergence

Raw samples and observation datetimes are discarded; only the fitted parameters + small metadata (convergence date, residual error, sample count) persist. Feedback correction loop (§8b below) runs continuously in Controlled mode.

### 8b. Feedback correction loop (Controlled only)

After each blind move in production:

1. Wait settle time (30 s).
2. Read lux.
3. If lux > threshold but the model said it should be blocked → slight geometry error or obstruction; bump target position up 1 probe step next time this (az, elev) cell is approached.
4. If lux << threshold and the model said it should be just-barely blocked → target is descending farther than needed; back off 1 probe step next time.

These corrections are applied as a small, bounded delta; if the delta grows large, the driver surfaces a `MODEL_DRIFT_DETECTED` state and suggests the operator re-run self-learn.

### Seasonal handling

The physics model is keyed on sun position, not on date. The same `(az, elev)` cell produces the same required blind position regardless of month — the model transfers across the year by construction. The sun traces different (az, elev) envelopes in different seasons, but the 4 fitted parameters describe **geometry**, and geometry does not change with season.

Out of scope for v1: tree leaf-on/leaf-off overlays, snow reflections, furniture moves. If support is added later, the design calls for a sparse `(week_of_year, az_bin, elev_bin) → correction_delta` overlay table — weekly granularity.

### Control surfaces (Bond and friends)

The driver adapts to whatever command interface the bound shade exposes:

| Control surface | How the driver commands position | Position feedback |
|---|---|---|
| `Percent` | Issue `SetLevel(P)` directly | Yes, if driver supports it |
| `Timed Up-Down-Stop` | Go up or down for (P_target − P_current) × travel_time / 100 seconds, then Stop | No — driver tracks assumed position; nightly "re-zero" by going fully up with safety margin |
| `Preset Slots` | Pick the preset whose mapped % is closest to target; quantized resolution | Yes (preset ID) |
| `Binary` | Commanded fully closed when target ≥ 50%, else fully open | Falls back to Event-Only behavior effectively |

Manual override detection on no-feedback motors is best-effort; relies on lux sensor discontinuities where possible.

### Storage during learning

During self-learn, raw samples are stored at **15-minute resolution** in a ring buffer (capped, e.g. 2000 samples ≈ 20 days of daylight samples). Buffer lives in driver persistent state. After model convergence the raw samples are discarded and only the 4 fitted parameters plus metadata remain.

---

## 9. Roadmap

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

- **Phase 4 — Physics model (Event Only subset)**
  - [ ] Shared physics solver in Lua — `sun_entering_room` path
  - [ ] Per-window decision loop with hysteresis state machine
  - [ ] Parent aggregation (`ANY_GLARE_ACTIVE`, count)

- **Phase 5 — Calibration pipeline**
  - [ ] `Record Calibration Observation` action — appends datetime pairs to observation log
  - [ ] Least-squares solver in Lua (Levenberg-Marquardt or simple Gauss-Newton, handles 3-param Event Only fit and 7-param Controlled fit)
  - [ ] `Start Self-Learn` / `Stop Self-Learn` / `Refit Model Now` / `Discard Calibration Data` actions
  - [ ] Lux sensor binding; 15-min sampling with ring buffer
  - [ ] Convergence test + `Model_Converged` event
  - [ ] Discard raw samples after convergence; retain only fitted params + metadata

- **Phase 5.5 — Controlled mode (blind control on top of the physics model)**
  - [ ] Mode property; gate Controlled-only properties/actions/events
  - [ ] Shade binding and control-surface abstraction (`Percent`, `Timed Up-Down-Stop`, `Preset Slots`, `Binary`)
  - [ ] Extended physics solver — `required_blind_position` using the 4 additional params
  - [ ] Active probing during self-learn (Controlled mode only)
  - [ ] Feedback correction loop after each commanded move
  - [ ] Manual override detection + pause
  - [ ] `Blind_Adjusted`, `Manual_Override_Detected` events

- **Phase 6 — Polish**
  - [ ] GitHub Releases autoupdater (self-install via `UpdateProjectC4i` SOAP + esphome handshake — see `~/.claude/c4-conventions.md` §3 and §3a)
  - [ ] Real icon (replace sunglasses emoji)
  - [ ] Dealer documentation HTML (`www/documentation.html`)
  - [ ] Release 1.0.0

---

## 10. Open questions

- Final driver icon — using 🕶️☀️ (sun + sunglasses emoji) as placeholder; will design a proper one before 1.0.
- Should `Calibrate From Date/Time Window` (Event mode) *also* honor a third "glare observed midday at worst angle" datetime for peak-elevation calibration? Deferred unless needed.
- License — MIT? Apache 2.0? Decide before first public release.
- Recommended lux sensors list — Aqara TVOC, Hue motion, Zigbee2MQTT illuminance sensors, generic HA-bridged. Document after first installs.
- Tilt blinds (venetian) — deferred beyond v1. Requires a second actuator axis and is uncommon in the install base we're targeting.
- Multi-protected-point per blind — deferred; v1 is one protected point per window.
