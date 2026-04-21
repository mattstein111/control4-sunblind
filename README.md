# SunBlinds 🕶️☀️

A Control4 DriverWorks driver suite that models each window as a **physical geometry** (wall orientation, horizon obstructions, overhang) and — using the sun's position and live weather — decides whether direct sun is currently entering the room. Optionally, each window can be upgraded to **Controlled mode** by providing protected-point geometry (TV, reading chair, pillow) and a motorized shade binding; the same physics model then computes the minimum blind descent needed to block direct sun from that point.

Each window runs in one of two modes:

- **Event Only** — fires `Glare_Start` / `Glare_End`. Fits 3 geometric parameters. Dealer programs the response.
- **Controlled** — binds to a motorized shade and commands it directly. Fits the same 3 parameters plus 4 protected-point parameters.

Both modes share the same calibration pipeline — dealer enters a few observation-pair datetimes, or binds a lux sensor and lets the driver self-learn. Raw samples are discarded after convergence; only the fitted parameters persist. The model is keyed on sun position, so it transfers across seasons by construction.

---

## Status

**Planning.** See [PLAN.md](PLAN.md) for the architecture and spec. No code yet.

---

## Concept

Every window faces a different direction, and glare depends on where the sun is, how high it is, and whether cloud cover is blocking it. SunBlinds computes sun position locally from the home's latitude/longitude, polls a public weather API for cloud cover, and fires events when each configured window enters or leaves its "glare zone."

Dealers calibrate each window by entering the date/time glare was *observed starting* and the date/time it *stopped*. The driver back-calculates azimuth, view half-angle, and elevation bounds automatically — no compass or protractor required.

---

## Architecture at a glance

- **Parent driver** (`SunBlinds Manager`) — one per home. Holds location, polls weather, computes sun position, fires sun/golden-hour/horizon events, aggregates per-window state into `Any_Glare_Active` / `All_Glare_Cleared`.
- **Child driver** (`SunBlinds Window`) — one per window. Holds window azimuth, view half-angle, elevation bounds, cloud cover threshold, hysteresis. Fires `Glare_Start` / `Glare_End`. Dealer creates proxies from the parent via an `Add Window` action, then moves each proxy into the appropriate room in Composer.

---

## Events

### Parent
- `Sunrise`, `Sunset`
- `Sun_Below_Horizon`, `Sun_Approaching_Horizon`, `Sun_At_Horizon`, `Sun_Above_Horizon`
- `Golden_Hour_Start`, `Golden_Hour_End`
- `Any_Glare_Active`, `All_Glare_Cleared`

### Child (per window)
- `Glare_Start`, `Glare_End`

---

## Weather source

[Open-Meteo](https://open-meteo.com) — no API key, free tier generous, returns cloud cover %. Polled every 10 min by default.

---

## Sun position

Computed locally in Lua from lat/long + current time using the standard NOAA solar position algorithm. No network required.

---

## License

TBD.
