# Data sources

Notes on external data sources for the planned route optimiser: a traveling
salesman solver with traffic, weather and crime weightings.

## Weather: Open-Meteo (chosen)

- Site: https://open-meteo.com/
- Geo based: query by `latitude` / `longitude`, global coverage.
- No API key required.
- Hourly forecast up to 16 days ahead (requirement was at least 24 hrs).
- Free for non-commercial use only. Commercial use needs a paid plan, so check
  the terms before deploying anywhere commercial.

Example (24 hrs of hourly data; `forecast_days=2` guarantees a full 24 hrs
whatever the time of day):

```
https://api.open-meteo.com/v1/forecast?latitude=59.33&longitude=18.07&hourly=temperature_2m,precipitation,weather_code&forecast_days=2
```

Status: not yet verified against a live response. The sandbox network policy
blocked `api.open-meteo.com` when this was written. Confirm the response shape,
rate limits and terms on the official docs before building on it.

### Alternatives considered

- Pirate Weather: Dark Sky style API, 48 hrs hourly by default, 168 with
  `extend=hourly`. Key requirement not confirmed.
- US National Weather Service (api.weather.gov): free, coordinate based, US
  only.
- meteoblue: requires an API key on every call.

## Still to choose

- Traffic: TBD
- Crime: TBD
