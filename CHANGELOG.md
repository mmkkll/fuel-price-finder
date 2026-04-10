# Changelog

All notable changes to this project are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.1.0] — 2026-04-10

### Added
- **EV charging stations** — new mode for finding the nearest electric vehicle charging points, complementing the existing fuel price lookup.
- **Open Charge Map integration** as the primary EV data source (rich, structured data: operator, connectors, power kW, number of charge points, operational status).
- **OpenStreetMap Overpass fallback** (no API key required) via the Kumi mirror, queried in parallel with OCM and merged by geographic proximity (< 80 m dedup, OCM wins on overlaps).
- **Step-by-step guide** in the README and the skill template for registering a free Open Charge Map API key (6 steps, ~60 seconds).
- **Trigger words** to switch the skill into EV mode: `EV charging`, `charging station`, `charge point`, `colonnina`, `ricarica elettrica`, `punto di ricarica`.
- `.gitignore`: `.secrets/` and `*.key` excluded by default to prevent accidental key leaks.

### Changed
- The fuel-prices skill is now titled **Fuel Prices & EV Charging** to reflect the broader scope.
- The Python snippet in the README has been replaced with a full **OCM + OSM merge** implementation that runs both queries in parallel and dedupes results by Haversine distance.
- README "What You'll Build" section now lists EV charging stations alongside fuel pumps.

### Fixed
- **Open Charge Map now requires an API key** — anonymous access started returning `403 — You must specify an API key` in April 2026. The previous "no key required" guidance was wrong and has been corrected throughout.
- The example query string no longer uses `compact=true&verbose=false` — those flags strip `OperatorInfo.Title`, `ConnectionType.Title`, and `StatusType` and replace them with bare numeric IDs, breaking the Telegram output.

### Security
- Documented that the OCM API key must be stored **outside the repo** (e.g. `~/mission-control/.secrets/openchargemap.key` with `chmod 600`) and read at runtime. Never commit the key.

## [1.0.0] — 2026-04-09

### Added
- Initial public release of **Fuel Price Finder**.
- Step-by-step guide to building a Telegram-driven fuel price lookup with Claude Code, government open data (Italy MIMIT), and a Python parser.
- Haversine distance calculation, top-3 cheapest stations within a configurable radius, multi-fuel support (Diesel, Gasoline, LPG, CNG).
- Telegram location pin handler (plugin patch) so a location share triggers the search automatically.
- Data source table for adapting the parser to France, Germany, Spain, UK, and the USA.
- Generic skill template (`skill-template.md`) ready to be filled in for any country.
