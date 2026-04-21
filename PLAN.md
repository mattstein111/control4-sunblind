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
| Obstruction model | readonly JSON | — | fitted parametric obstructions (overhangs, wall_edges, horizon_features) |
| Protected point sensor | binding | — | primary lux sensor at the point to defend |
| Vantage point sensor | binding | — | optional temporary lux sensor used for two-point calibration; can be removed after `Model_Converged` |
| Vantage offset | STRING | — | `(Δx=+50, Δy=0, Δz=0)` cm relative to protected point; required if vantage sensor bound |
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

### The "skyline" mental model

From the protected point, looking outward at every compass direction (azimuth), there is a lower silhouette (horizon + house wings + trees + neighbors) and an upper silhouette (overhang + eave + balcony + ceiling). Glare reaches the point only when `lower(az) < sun_elev < upper(az)` AND sun is on the window's side of the wall.

The silhouettes are **functions of azimuth**. A south-wing shadow blocks winter sun by raising the lower silhouette at southern azimuths; a roof overhang blocks summer-noon sun by pulling down the upper silhouette at those azimuths. Only where the sun can pass between the two silhouettes does glare occur.

### Parametric obstruction model (primary representation)

Rather than storing silhouettes as sampled lookup tables (which need dense observation to fill in), the driver represents the world as a **small set of parametric obstructions**, each with 2–3 geometric parameters. The silhouette values at any azimuth are *computed* from the obstruction parameters on demand. This extrapolates to azimuths the sun didn't traverse on calibration days.

Typical obstructions:

| Obstruction | Parameters | Models |
|---|---|---|
| `overhang` | depth outward from wall (cm); height above window top (cm); azimuth range where it applies | roof eave, balcony above, deep window header |
| `wall_edge` | azimuth of the edge as seen from the protected point (°); height the wall extends upward (cm, or "infinite" for the other wing of the house) | south wing of an L-shaped house, adjacent garage, neighbor's wall |
| `horizon_feature` | azimuth range; height-angle cutoff (°) | tree line, hill, fence |

A typical installation has 1 overhang + 1 wall_edge + maybe 1 horizon_feature = 5–8 fitted geometric parameters. Plus the physics params (wall_azimuth, and in Controlled mode the point coordinates and window_top_height).

If the actual geometry doesn't fit the standard obstructions (rare — e.g. a cathedral-ceilinged greenhouse room), the model falls back to piecewise silhouette segments as stored overrides.

### Parameter set

| Parameter | Units | Event Only | Controlled | Meaning |
|---|---|---|---|---|
| `wall_azimuth` | ° (0–360) | required | required | Compass direction the window's outward normal points |
| `obstructions` | list of typed obstructions (see table above) | required | required | Parametric description of the surrounding geometry |
| `silhouette_overrides` | optional piecewise segments | — | — | Fallback for geometries the parametric obstructions can't capture |
| `window_top_height` | cm above floor | — | required | Top edge of window opening |
| `point_depth` | cm from window plane | — | required | Distance to protected point |
| `point_height` | cm above floor | — | required | Height of protected point |
| `blind_travel_length` | cm | — | required | Measured during control-surface calibration, not a fitted param |

Blind bottom height at position P% closed is `window_top_height − P/100 × blind_travel_length`.

### The two solver outputs

**`sun_entering_room`** (Event Only answer): direct sun reaches the reference point right now.

```
in_wall_arc    = |angular_diff(sun_azimuth, wall_azimuth)| < 90°
above_lower    = sun_elevation > lower_silhouette(sun_azimuth)
below_upper    = sun_elevation < upper_silhouette(sun_azimuth)
sun_entering_room = in_wall_arc AND above_lower AND below_upper
```

`lower_silhouette(az)` and `upper_silhouette(az)` each look up the piecewise-constant function; fall back to the default value for azimuths not covered by any explicit segment.

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

### Fitting parameters — one pipeline, two input methods

The same least-squares solver fits the parametric obstructions from any combination of the following observation sources.

#### Primary method: two-point dual-sensor calibration

The key efficiency unlock. **One sensor at the protected point** (kept permanently); **one sensor at a deliberately-chosen vantage point** (removed after calibration). A single glare event observed at both sensors yields 4 boundary transitions (A-starts, B-starts, A-ends, B-ends) at different (az, elev) — and crucially, the spatial offset between the two points gives the solver *disparity information* that no single sensor can provide.

Workflow:

1. Bind two lux sensors: `Protected point sensor` and `Vantage point sensor`.
2. Dealer places sensors:
   - Protected: at the spot to be defended (TV, pillow, chair).
   - Vantage: at a **different** point in the room. Maximum disparity = maximum information. Good choices: 50+ cm closer to the window, or laterally offset toward the nearer-shadow edge.
3. Dealer enters the vantage point's position relative to the protected point — `(Δx cm toward window, Δy cm lateral, Δz cm vertical)`.
4. Dealer clicks `Start Two-Point Calibration`.
5. Driver watches both sensors continuously, records every lux-threshold crossing at each, tags with sun position.
6. After each glare event the fit runs; if converged, fires `Model_Converged`.

Typical convergence: **1–3 glare events (i.e. 1–3 sunny days)**. After convergence the vantage sensor can be removed; the protected sensor stays for ongoing feedback correction.

#### Variant: manual two-point observation (no sensors)

Dealer stands at each point with a phone lux-meter app during one glare event. Records 4 datetimes (A-start, B-start, A-end, B-end) + the spatial offset. One afternoon of observation feeds the same solver. No binding needed.

#### Fallback: single-point datetime observations

If only one sensor / one observation point is available, the solver still fits — it just needs more observation events (3–5 across different days) to converge. Each observation pair gives 2 boundary points at one reference position.

#### Self-learn extension

In Controlled mode with a bound lux sensor, the driver actively probes the blind by small increments during clear-sky periods, gathering many more constraint samples per day. This fills in the obstruction fit even faster when only the single protected-point sensor is bound (no vantage sensor needed).

### Observation-to-constraint encoding

Each recorded boundary point is a constraint on the obstruction model:

- **"Sun at (az, elev) reaches point P"** — the sun's ray from (az, elev) through the window plane lands inside the "visible sky cone" from P at that azimuth. Equivalent: `lower(az) < elev < upper(az)` computed from current obstruction params.
- **"Sun at (az, elev) does NOT reach point P"** — the negation.

The solver minimizes the residual: for each observation, the distance from the sun position to the nearest silhouette boundary implied by the current obstruction params. Converges when total residual is below threshold.

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

The physics model is keyed on sun position, not on date. Parametric obstructions describe physical geometry, which does not change with season — so once fit, the model predicts correctly for every day of the year, including sun positions that weren't sampled during calibration. Extrapolation across seasons is handled by the physics, not by having observed every relevant azimuth.

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
  - [ ] Piecewise-constant silhouette data structure + azimuth lookup (with default fallback)
  - [ ] Parser for `"az_start-az_end:elev; ..."` segment strings
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

## 10. Prior art — Home Assistant ecosystem

The HA community has worked this problem for years. Event Only ≈ what HA blueprints do today; Controlled ≈ what Adaptive Cover / Adaptive Cover Pro do today. SunBlinds extends both with **user-selected protected point** + **two-point dual-lux calibration**, which nobody in HA has combined.

### Baseline — HA blueprints (the v1 Event Only analog)

**[Cover Control Automation (CCA)](https://github.com/hvorragend/ha-blueprints)** — hvorragend. HA forum topic 680539 (~2,600 replies, active 2026). De-facto best-of-breed blueprint.
  - Inputs: `window_azimuth ± half_angle`, elevation min/max, outdoor lux, outdoor temp, weather state, calendar/schedule, ventilation lockout (open-window sensor), **manual-override detection with timeout**.
  - Pattern universal across community blueprints: `window_azimuth ± half_angle` arc + `elevation > threshold`, AND'd with temp/lux/weather overrides.
  - No overhang geometry, no protected-point ray-tracing.

Other notable blueprints: `daniel-sc` (forum 584240), `mougeat` (forum 405948 — the originator).

### Closest analogs — HACS integrations (the Controlled mode analog)

**[Adaptive Cover](https://github.com/basbruss/adaptive-cover)** (basbruss) — the anchor project in HACS default.
  - Computes **continuous blind position** from sun geometry to block direct sunlight.
  - Models vertical, horizontal, and tilted (venetian) covers.
  - Inputs: window azimuth, window height, distance from floor, cover height, slat depth + spacing (venetian).
  - "Protected point" is implicitly the **floor line** — not a user-selected spot.
  - **"Blindspot"** config: static azimuth range pre-shaded by neighboring buildings/trees (≈ our `silhouette_overrides`).
  - Two modes: basic (pure geometry) + climate (temp/presence/lux override).

**[Adaptive Cover Pro](https://github.com/jrhubott/adaptive-cover-pro)** (jrhubott, fork, ~67 stars) — closest existing analog to SunBlinds Controlled mode.
  - **10-handler override pipeline**: force → weather → manual → custom → motion → cloud suppression → climate → glare zones → solar → default. Clean precedence model.
  - **"Enhanced Geometric Accuracy"** — documents overhang/recess math; cross-validate our physics against this before shipping.
  - **Explicit "Glare Zones"** config.
  - Position verification + companion Lovelace card showing sun-vs-window geometry.

**[langestefan/auto-sun-blind](https://github.com/langestefan/auto-sun-blind)** — older template-sensor approach, now superseded.

### Non-HA reference math

- **[pvlib-python](https://pvlib-python.readthedocs.io/)** — sun vector computation, gold-standard reference for solar position math.
- **[ladybug-tools/honeybee-radiance](https://github.com/ladybug-tools/honeybee-radiance)** — full Radiance-based daylight/glare simulation. What architects use. Overkill for runtime blind control but the canonical reference for the geometry we're approximating.

### What to borrow / cross-check

1. **Adaptive Cover Pro's override pipeline architecture** (force > manual > climate > solar > default) — mirror this in the Controlled-mode state machine.
2. **Manual-override detection with timeout** — battle-tested across HA blueprints and Adaptive Cover Pro. Steal the exact pattern: distinguish user moves from automation moves by comparing reported position against last commanded position within a tolerance window; if mismatch → engage override timeout.
3. **Adaptive Cover Pro "Enhanced Geometric Accuracy" wiki** — cross-validate our `compute_required_blind_position` math against theirs before shipping Phase 5.5.
4. **"Blindspot" / "Glare Zones"** — HA validates the static-obstruction-mask concept. Our `silhouette_overrides` is the equivalent escape hatch for dealers who can't do two-point calibration.
5. **Lux-as-override (not training signal)** — HA pattern: close only if outdoor lux > threshold; skip on overcast days. Adaptive Cover Pro's "cloud suppression" reads cloud-cover/irradiance sensor. We're more ambitious — using lux as training signal *and* override — but keep the override path simple and dealer-understandable.

### What SunBlinds has that HA doesn't

- **User-selected protected point** (TV, bed, reading chair) — Adaptive Cover protects the floor line implicitly. Ours enables minimum-descent behavior.
- **Two-point dual-lux-sensor calibration** — nobody in HA treats lux as training signal; it's universally consumed as an override gate. Our disparity-based single-event calibration is genuinely novel for this problem space.
- **Parametric obstruction fitting** (overhang + wall_edge + horizon_feature) — Adaptive Cover's "Blindspot" is a manual mask; ours is a fitted model.

### Outreach (optional)

`basbruss` and `jrhubott` have solved adjacent problems and might be interested in the protected-point / two-point-calibration angle. Not required but could shortcut iteration. Defer until after v1 ships — easier to discuss with working code.

---

## 11. Open questions

- Final driver icon — using 🕶️☀️ (sun + sunglasses emoji) as placeholder; will design a proper one before 1.0.
- Should `Calibrate From Date/Time Window` (Event mode) *also* honor a third "glare observed midday at worst angle" datetime for peak-elevation calibration? Deferred unless needed.
- License — MIT? Apache 2.0? Decide before first public release.
- Recommended lux sensors list — Aqara TVOC, Hue motion, Zigbee2MQTT illuminance sensors, generic HA-bridged. Document after first installs.
- Tilt blinds (venetian) — deferred beyond v1. Requires a second actuator axis; Adaptive Cover models this, worth copying when we get to it.
- Multi-protected-point per blind — deferred; v1 is one protected point per window.
