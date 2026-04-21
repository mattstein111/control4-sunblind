# SunBlinds — Design Plan

Last updated: 2026-04-21

---

## 1. Scope

SunBlinds is a Control4 DriverWorks driver suite that fires per-window glare events and, optionally, **actively controls motorized blinds** to block the sun from reaching a specific "protected point" in the room. Each window proxy runs in one of two modes:

- **Event Only** — fires `Glare_Start`/`Glare_End` events. Dealer programs the response. No direct control. This is the default and the v1 baseline.
- **Controlled** — commands a bound motorized shade directly, descending only as far as required to keep direct sun off the protected point. Uses a 4-parameter physics model (see §10) calibrated in one of two ways:
  - **Manual calibration** — dealer enters datetime stamps of observed glare start and end while the blind is fully closed (gives two constraints).
  - **Self-learn with lux sensor** — a lux sensor at the protected point drives automated micro-probing; the driver moves the blind in small increments and learns the geometry from observed lux responses.

Two driver types are packaged together:

| Driver | Role | Quantity per project |
|---|---|---|
| `SunBlinds Manager` (parent) | Holds location, polls weather, computes sun position, aggregates window state | 1 |
| `SunBlinds Window` (child) | One per window. In Event Only mode: holds geometry, fires glare events. In Controlled mode: also binds to a motorized shade and (optionally) a lux sensor, runs the physics solver, commands the shade. | N |

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
| Window azimuth (°) | NUMBER | 180 | 0=N, 90=E, 180=S, 270=W |
| View half-angle (°) | NUMBER | 45 | ± around azimuth |
| Min sun elevation (°) | NUMBER | 5 | sun below = no glare |
| Max sun elevation (°) | NUMBER | 70 | sun above = no glare (overhang) |
| Cloud cover threshold (%) | NUMBER | 40 | direct-sun gating threshold |
| On-delay (min) | NUMBER | 5 | hysteresis — sun must stay in zone this long |
| Off-delay (min) | NUMBER | 10 | hysteresis — sun must stay out of zone this long |
| Calibrate: Glare Start datetime | STRING | — | ISO 8601 local, e.g. `2026-04-21T14:30` |
| Calibrate: Glare End datetime | STRING | — | ISO 8601 local |
| Glare Active | readonly bool | — | live |
| Last reason | readonly string | — | e.g. `"sun in arc (az 205°), cloud 22%, elev 34°"` |

### Controlled-mode properties (additional)

| Property | Type | Default | Notes |
|---|---|---|---|
| Shade binding | binding | — | C4 connection to a motorized shade/blind driver |
| Lux sensor binding | binding | — | optional; C4 connection to an illuminance source (any lux-reporting device) |
| Control surface | LIST | `Auto-detect` | `Percent` / `Timed Up-Down-Stop` / `Preset Slots` / `Binary` / `Auto-detect` |
| Shade travel time (s) | NUMBER | — | for Timed control surface — seconds from fully up to fully down |
| Preset slot map | STRING | — | for Preset control surface — e.g. `"1=25, 2=50, 3=75, 4=100"` |
| Blind opacity | LIST | `Blackout` | `Blackout` / `Room-darkening` / `Sheer` — affects settle tolerance |
| Calibration method | LIST | `Self-learn (lux sensor)` | `Self-learn (lux sensor)` / `Manual datetime (blind fully closed)` / `Tape measure (geometry)` |
| Lux threshold | NUMBER | 1000 | glare threshold at the protected point, in lux |
| Probe interval (min) | NUMBER | 10 | minimum time between probe moves during self-learn |
| Probe step (%) | NUMBER | 5 | smallest blind position increment |
| Manual override timeout (min) | NUMBER | 30 | pause auto after a detected manual move |
| Learning status | readonly string | — | e.g. `"Self-learning, 47 samples, model not yet converged"` |
| Model parameters | readonly string | — | e.g. `"window_top=210cm, point_depth=180cm, point_height=95cm, wall_az=182°"` |
| Target position (%) | readonly | — | last computed target |
| Current position (%) | readonly | — | last commanded / reported position |

### Actions

- `Calibrate From Date/Time Window` (Event Only mode) — reads both datetime fields; computes azimuth, half-angle, elevation bounds; stamps into properties.
- `Calibrate From Date/Time Window` (Controlled mode, `Manual datetime` method) — interprets the two datetimes as "while the blind was fully closed, glare was observed starting at T1 and ending at T2" at the protected point. Combines the two sun positions into constraints on the 4-parameter physics model. Repeat across several observed glare events over different days to fully constrain the model.
- `Start Self-Learn` (Controlled mode) — enters passive+active learning mode. Records `(timestamp, az, elev, cloud_cover, blind_position, lux)` samples at 15-min resolution. During clear-sky daytime periods, automatically nudges the blind by the probe step every probe interval to gather constraint samples. Runs until operator calls `Stop Self-Learn` or the model converges.
- `Stop Self-Learn` — freezes learning, keeps the current physics model in effect.
- `Refit Model Now` — re-runs the least-squares fit on accumulated samples.
- `Discard Learning Data` — wipes the sample buffer and model parameters; starts over.
- `Test Glare Now` — logs current reason and, if Controlled, what blind position *would* be commanded.
- `Manual Pause (N minutes)` — temporarily halts auto control.

### Events

- `Glare_Start` / `Glare_End` (both modes)
- `Blind_Adjusted` (Controlled mode; includes fromPercent, toPercent, reason)
- `Manual_Override_Detected` (Controlled mode with position feedback)
- `Model_Converged` (Controlled mode, once fit is confident)

### Variables

- `GLARE_ACTIVE` (bool)
- `WINDOW_AZIMUTH` (number)
- `LAST_GLARE_START` / `LAST_GLARE_END` (timestamp)
- Controlled-only: `TARGET_POSITION`, `CURRENT_POSITION`, `MODEL_CONVERGED` (bool), `LUX_CURRENT`, `LEARNING_SAMPLES_COUNT`

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

## 8. Controlled mode — physics model

### The 4 parameters

Each Controlled-mode window proxy fits this parameter set once, then uses it forever:

| Parameter | Units | Meaning |
|---|---|---|
| `wall_azimuth` | ° (0–360) | Compass direction the window's outward normal points |
| `window_top_height` | cm, above floor | Height of the window's top edge (blind in fully-up state reveals sun ≤ this height) |
| `point_depth` | cm, from window plane into the room | Horizontal distance from the window plane to the protected point |
| `point_height` | cm, above floor | Height of the protected point above the floor |

Blind bottom height (the blind's lower edge when at position P% closed) is:
`blind_bottom_height = window_top_height − P/100 × blind_travel_length`

where `blind_travel_length` is measured during control-surface calibration.

### Required-position solver

Given current `(sun_az, sun_elev)` and the 4 parameters, the driver computes:

1. Project sun vector onto the room coordinate frame (wall_azimuth rotates the horizontal component).
2. Compute the ray from the protected point toward the sun.
3. Find where that ray crosses the window plane — its height `h_window`.
4. If `h_window > window_top_height` → sun comes in from above the window top; the blind can't help anyway (overhang / roof blocks sun). Command: 0% (fully up).
5. If `h_window < floor` → ray goes down; no glare. Command: 0%.
6. Else required blind_bottom_height = `h_window − safety_margin`; translate to P% via blind_travel_length. Command: clamp to [0, 100] and round to nearest probe step.

If `cloud_cover > cloud_threshold` (not sunny enough to cause glare), command 0% regardless.

### Fitting the 4 parameters

Two calibration paths, chosen via `Calibration method` property:

**(a) Manual datetime calibration (blind fully closed)**

Dealer observes glare at the protected point while the blind is at 100% closed. They note the datetime glare started and the datetime it ended. These two moments are the sun positions where the bottom edge of the fully-closed blind is *exactly* aligned with the ray from sun to protected point — two equality constraints on the 4 parameters.

One observation pair gives 2 constraints; two pairs give 4 constraints (fully determined). Dealer records 3–4 observation pairs across different sun positions (different times of year, different windows of the day) for a robust overdetermined fit.

Dealer enters repeated observations via a list-style property or by calling `Calibrate From Date/Time Window` multiple times; each call appends to the constraint set.

**(b) Self-learn with lux sensor**

Lux sensor binding provides live illuminance at (or near) the protected point. Driver does:

1. **Sample every 15 minutes** during sun-above-horizon hours, recording `(az, elev, cloud_cover, blind_position, lux)`.
2. **Automated probing** during low-cloud daytime periods: every `probe_interval` minutes, move the blind by `probe_step` (up or down within [0, 100]); after settle time (~30 s), read lux. The *transition* in lux for a given move at a given sun position is the constraint.
3. **Classify each sample** as "sun reaches protected point" (lux > threshold with blind at position P) or "sun blocked" (lux ≤ threshold with blind at P). Each classified sample is a constraint: the ray from sun to protected point either does or does not pass above the blind's current bottom edge.
4. **Fit** the 4 parameters via nonlinear least-squares (Levenberg-Marquardt or similar simple iterative solver in Lua) to satisfy as many constraints as possible.
5. **Convergence test** — parameters stable within tolerance across last 3 refits, ≥N successful constraint samples (N ~ 50 with diverse sun positions). Fire `Model_Converged` and stop active probing.

After convergence: **raw sample data is discarded**. Only the 4 fitted parameters, `blind_travel_length`, and a small metadata record (convergence date, sample count used, residual error) are persisted. Ongoing observations are used for feedback correction (below) but not re-stored.

### Feedback correction loop

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

- **Phase 4 — Glare logic (Event Only)**
  - [ ] Per-window decision loop with hysteresis state machine
  - [ ] Parent aggregation (`ANY_GLARE_ACTIVE`, count)

- **Phase 5 — Event-mode calibration**
  - [ ] `Calibrate From Date/Time Window` action — back-calculates azimuth, half-angle, elevation bounds from two observed timestamps

- **Phase 5.5 — Controlled mode (physics-model blind control)**
  - [ ] Mode property; skeleton for Controlled path
  - [ ] Shade binding and control-surface abstraction (`Percent`, `Timed Up-Down-Stop`, `Preset Slots`, `Binary`)
  - [ ] Optional lux sensor binding
  - [ ] Required-position solver (4-parameter physics, ray-to-window-plane intersection)
  - [ ] Manual datetime calibration (multi-observation fit)
  - [ ] Self-learn with lux sensor (15-min sampling, active probing, least-squares fit, convergence detection)
  - [ ] Feedback correction loop after each commanded move
  - [ ] Manual override detection + pause
  - [ ] `Blind_Adjusted`, `Model_Converged` events

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
