# Price Server User Guide

Public documentation for the [Grid Coordination](https://grid-coordination.energy) price server — an [OpenADR 3.1.0](https://www.openadr.org/) VTN serving hourly California electricity prices and GHG emissions data.

## What is the price server?

The price server is a universal electricity price signal layer. It publishes hourly prices from two kinds of tariffs, plus GHG emissions data — all as [OpenADR 3.1.0](https://www.openadr.org/) programs and events.

- **Feed-based pricing** — dynamic CAISO Day-Ahead Market prices via [GridX](https://www.gridx.com/) CalFUSE, updated hourly
- **Computed pricing** — published rate schedules from [OpenEI URDB](https://openei.org/wiki/Utility_Rate_Database), prices generated from the tariff definition
- **GHG emissions** — Marginal Operating Emissions Rate (MOER) in g CO2/kWh from [SGIP Signal](https://sgipsignal.com/) (operated by WattTime for the CPUC)

An EMS or appliance (VEN) just sees hourly price intervals and optimizes its energy use — it doesn't need to know whether the price comes from a dynamic market or a published TOU schedule. When prices vary, the appliance optimizes. When they don't (flat rate), there's nothing to optimize — but the protocol is the same.

## Feed-based tariffs (GridX) — 31 rate schedules

**PG&E — 16 tariffs** (each served across 59 distribution feeders)

| Rate | Type |
|------|------|
| `EELEC` | Standard residential electric |
| `BEV1` | Commercial EV charging |
| `BEV2P`, `BEV2S` | Business EV 2 (Primary / Secondary) |
| `B6` | Small commercial |
| `B10P`, `B10S` | Medium commercial 10 kW (Primary / Secondary) |
| `B19P`, `B19S` | Medium commercial (Primary / Secondary) |
| `B20P`, `B20S` | Large commercial (Primary / Secondary) |
| `AG-A1`, `AG-A2` | Agricultural |
| `AGBP`, `AGBS` | Agricultural Business (Primary / Secondary) |
| `AGCP`, `AGCS` | Agricultural Commercial (Primary / Secondary) |

**SCE — 15 tariffs** (each served across 46 substations)

| Rate | Type |
|------|------|
| `TOU-PRIME` | Residential time-of-use premium |
| `TOU-D-49` | Residential time-of-use 4–9 PM peak |
| `TOU-D-58` | Residential time-of-use 5–8 PM peak |
| `TOU-EV-8` | EV time-of-use 8 PM peak |
| `TOU-EV-9P`, `TOU-EV-9S`, `TOU-EV-9ST` | EV time-of-use 9 PM (Primary / Secondary / Super TOU) |
| `TOU-GS-1`, `TOU-GS-2`, `TOU-GS-3` | General Service |
| `TOU-PA-2`, `TOU-PA-3` | Public Authority |
| `TOU-8ST`, `TOU-8P`, `TOU-8S` | General Service 8 PM (Super TOU / Primary / Secondary) |

Each tariff × location combination is an OpenADR 3 program. PG&E programs are named `<RATE>-<9-digit-circuit-id>` (e.g. `EELEC-013532223`). SCE programs are named `<RATE>-<substation>` (e.g. `TOU-PRIME-Eagle Rock`).

### Computed tariffs (URDB) — 3 initial rate schedules

Published rate schedules from the [OpenEI Utility Rate Database](https://openei.org/wiki/Utility_Rate_Database). Prices are computed from the tariff definition — no external data feed needed. Coverage will expand to all California tariffs.

| Utility | Rate | Type | Program |
|---------|------|------|---------|
| **PG&E** | E-1 Region P | Residential TOU | `PGE-E-1 - Baseline Region P` |
| **SCE** | TOU-D-2 | Residential TOU | `SCE-Time-of-use Domestic: TOU-D-2` |
| **SDG&E** | DR-TOU Coastal | Residential TOU | `SDGE-DR- TOU- Coastal Baseline Region` |

One program per tariff. Unlike feed-based tariffs, computed tariffs don't vary by circuit or substation — the published rate schedule applies uniformly.

### GHG emissions (MOER)

| Region | Program | Description |
|--------|---------|-------------|
| `SGIP_CAISO_PGE` | `MOER-PGE` | Pacific Gas & Electric (CAISO) |
| `SGIP_CAISO_SCE` | `MOER-SCE` | Southern California Edison (CAISO) |
| `SGIP_CAISO_SDGE` | `MOER-SDGE` | San Diego Gas & Electric (CAISO) |
| `SGIP_LADWP` | `MOER-LADWP` | Los Angeles DWP |
| `SGIP_BANC_SMUD` | `MOER-SMUD` | Sacramento Municipal Utility District |
| `SGIP_BANC_P2` | `MOER-BANC` | Balancing Authority of Northern California |
| `SGIP_IID` | `MOER-IID` | Imperial Irrigation District |
| `SGIP_PACW` | `MOER-PACW` | PacifiCorp West |
| `SGIP_NVENERGY` | `MOER-NVE` | NV Energy |
| `SGIP_TID` | `MOER-TID` | Turlock Irrigation District |
| `SGIP_WALC` | `MOER-WALC` | Western Area Lower Colorado |

MOER programs publish hourly marginal operating emissions rate in **g CO2/kWh**, aggregated from native 5-minute [SGIP Signal](https://sgipsignal.com/) data. Each event contains 24 hourly intervals, same as pricing events.

**Total: 34 pricing tariffs + 11 GHG emissions regions** (31 feed-based + 3 computed + 11 GHG)

## Finding your program

**For feed-based tariffs (GridX):** You need two things:

1. **Rate schedule** — which tariff applies to your service (e.g. `EELEC` for PG&E residential, `TOU-PRIME` for SCE residential). See the tariff table above.
2. **Circuit or substation** — which part of the grid you're on. For PG&E, this is a 9-digit circuit ID identifying your distribution feeder. For SCE, it's a substation name.

The combination determines your program name: `EELEC-013532223` or `TOU-PRIME-Eagle Rock`.

**How to find your circuit or substation:** If you don't know which circuit or substation serves your location, you can browse the full list in [circuits.md](circuits.md) — it includes the substation name, division, and region for every PG&E circuit, and all 46 SCE substation names. PG&E customers on the Dynamic Rate Pilot can find their circuit ID on their billing statement or through the [Priicer community](https://forum.priicer.com/t/pg-e-dynamic-pilot-california/33).

**For computed tariffs (URDB):** Just find the program by name — no circuit needed. These programs use the tariff name directly (e.g. `PGE-E-1 - Baseline Region P`).

## How to access

### Documentation & API reference

| URL | What |
|-----|------|
| [/docs](https://price.grid-coordination.energy/docs) | User guide (this document) |
| [/api](https://price.grid-coordination.energy/api) | Interactive API reference — browse endpoints, see schemas, try requests |
| [/openapi.json](https://price.grid-coordination.energy/openapi.json) | OpenAPI specification (JSON) for code generators and tooling |

### HTTPS (REST API)

```
https://price.grid-coordination.energy/openadr3/3.1.0
```

Standard OpenADR 3.1.0 endpoints: `/programs`, `/events`, `/subscriptions`, `/notifiers`. Read-only, no authentication required.

**Extension: `programName` lookup.** This server supports `GET /programs?programName=<name>` for direct program lookup by name, returning 0 or 1 results. This avoids paginating the full program list. We have [proposed this as a standard addition](https://github.com/oadr3-org/specification/issues/418) to the OpenADR 3 specification.

### MQTT (push notifications)

| Transport | URL | Port |
|---|---|---|
| **MQTT over TLS (recommended)** | `mqtts://mqtt.grid-coordination.energy` | 8883 |
| **Plain MQTT** | `tcp://mqtt.grid-coordination.energy` | 1883 |

Anonymous subscribe to `openadr3/3.1.0/events/#` for real-time price updates. No authentication required. See [mqtt-notifications.md](mqtt-notifications.md) for details.

### Broker discovery

The broker URLs are also available programmatically:

```bash
curl -s https://price.grid-coordination.energy/openadr3/3.1.0/notifiers | python3 -m json.tool
```

## Data model

### Programs

One program per tariff (computed) or per tariff × location (feed-based):

| Field | Feed-based (GridX) | Computed (URDB) |
|-------|-------------------|-----------------|
| `programName` | `EELEC-013532223` | `PGE-E-1 - Baseline Region P` |
| `programLongName` | `PG&E EELEC — circuit 013532223` | `Pacific Gas & Electric Co E-1 - Baseline Region P (TOU, URDB)` |
| `payloadDescriptors[0].payloadType` | `PRICE` | `PRICE` |
| `payloadDescriptors[0].units` | `kWh` | `kWh` |
| `payloadDescriptors[0].currency` | `USD` | `USD` |

### Events

One event per program per day, with 24 hourly intervals:

| Field | Pricing Example | GHG Example |
|-------|----------------|-------------|
| `eventName` | `EELEC-013532223-2026-04-12` | `MOER-PGE-2026-04-12` |
| `programID` | UUID linking to the program | UUID linking to the program |
| `intervals[n].id` | Hour of day (0 = midnight, 23 = 11 PM) | Hour of day (0 = midnight, 23 = 11 PM) |
| `intervals[n].payloads[0].type` | `PRICE` | `GHG` |
| `intervals[n].payloads[0].values[0]` | `0.02622` ($/kWh) | `661.2` (g CO2/kWh) |

### Forward window and history

**Feed-based prices (GridX)**: Fetched on startup and every hour. In practice, CAISO Day-Ahead Market data is available for **today + ~2 days** (day-ahead prices are typically published by ~4:30 PM PST). Historical prices go back to approximately August 2024 for PG&E and July 2025 for SCE.

**Computed prices (URDB)**: Generated on startup and refreshed daily. Events cover **today + 2 days**. No historical backfill — the tariff definition is the source of truth, and prices are computed on demand for the forward window.

**GHG emissions**: MOER data is fetched hourly from SGIP Signal. Today's event fills in progressively as 5-minute data becomes available. Historical data goes back approximately 30 days from initial deployment.

## Getting started

See [getting-started.md](getting-started.md) for a quick walkthrough using `curl`.

## Tutorials

Step-by-step guides for connecting with different client libraries:

| Language | Library | Tutorial |
|----------|---------|----------|
| Python | [python-oa3-client](https://github.com/grid-coordination/python-oa3-client) | [tutorial-python.md](tutorial-python.md) |
| Python | [openadr3-client (ElaadNL)](https://github.com/ElaadNL/openadr3-client) | [tutorial-python-elaadnl.md](tutorial-python-elaadnl.md) |
| Clojure | [clj-oa3-client](https://github.com/grid-coordination/clj-oa3-client) | [tutorial-clojure.md](tutorial-clojure.md) |
| Rust | [openleadr-rs (LF Energy)](https://github.com/OpenLEADR/openleadr-rs) | [tutorial-rust.md](tutorial-rust.md) |

Each tutorial covers listing programs, finding a circuit, fetching 24-hour prices, and charting. No authentication is required.

## MQTT notifications

Real-time push notifications for price updates. See [mqtt-notifications.md](mqtt-notifications.md).

## License

[MIT License](LICENSE) - Copyright (c) 2026 Clark Communications Corporation
