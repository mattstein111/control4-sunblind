# SunBlinds 🕶️☀️

A Control4 DriverWorks driver suite that fires per-window glare events and, optionally, actively controls motorized blinds — descending them only as far as needed to block direct sun from a specific "protected point" in the room (a TV, a reading chair, a pillow).

Each window runs in one of two modes:

- **Event Only** — fires `Glare_Start` / `Glare_End`; dealer programs the response.
- **Controlled** — binds to a motorized shade and commands it directly using a 4-parameter physics model of the window + protected point. The model is calibrated either by entering a few datetime pairs of observed glare events (blind fully closed), or by self-learning from a lux sensor placed at the protected point.

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
