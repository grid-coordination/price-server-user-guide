# Price Server User Guide

Public documentation for the [Grid Coordination](https://grid-coordination.energy) price server — an [OpenADR 3.1.0](https://www.openadr.org/) VTN serving hourly California electricity prices and GHG emissions data.

## What is the price server?

The price server publishes real-time and historical electricity prices for **PG&E** and **SCE** rate schedules, plus hourly marginal GHG emissions intensity (MOER) for 11 California grid regions, as OpenADR 3 programs and events.

- **Prices**: Marginal cost in USD/kWh from the CAISO Day-Ahead Market via [GridX](https://www.gridx.com/) CalFUSE
- **GHG emissions**: Marginal Operating Emissions Rate (MOER) in g CO2/kWh from [SGIP Signal](https://sgipsignal.com/) (operated by WattTime for the CPUC)

## Available tariffs

| Utility | Rate | Type | Programs |
|---------|------|------|----------|
| **PG&E** | `EELEC` | Standard residential electric | 59 (one per distribution feeder) |
| **PG&E** | `BEV1` | Commercial EV charging | 59 |
| **PG&E** | `BEV2P` | Business EV 2 Primary | 59 |
| **PG&E** | `BEV2S` | Business EV 2 Secondary | 59 |
| **PG&E** | `B6` | Small commercial | 59 |
| **PG&E** | `B19P` | Medium commercial Primary | 59 |
| **SCE** | `TOU-PRIME` | Residential time-of-use premium | 46 (one per substation) |
| **SCE** | `TOU-D-49` | Residential time-of-use 4–9 PM peak | 46 |
| **SCE** | `TOU-D-58` | Residential time-of-use 5–8 PM peak | 46 |
| | | **Subtotal** | **492 pricing programs** |

PG&E programs are named `<RATE>-<9-digit-circuit-id>` (e.g. `EELEC-013532223`). SCE programs are named `<RATE>-<substation>` (e.g. `TOU-PRIME-Eagle Rock`).

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

**Total: 503 programs** (492 pricing + 11 GHG emissions)

## Finding your program

To get prices relevant to you, you need two things:

1. **Rate schedule** — which tariff applies to your service (e.g. `EELEC` for PG&E residential, `TOU-PRIME` for SCE residential). See the tariff table above.
2. **Circuit or substation** — which part of the grid you're on. For PG&E, this is a 9-digit circuit ID identifying your distribution feeder. For SCE, it's a substation name.

The combination determines your program name: `EELEC-013532223` or `TOU-PRIME-Eagle Rock`.

**How to find your circuit or substation:** If you don't know which circuit or substation serves your location, you can browse the full list in [circuits.md](circuits.md) — it includes the substation name, division, and region for every PG&E circuit, and all 46 SCE substation names. PG&E customers on the Dynamic Rate Pilot can find their circuit ID on their billing statement or through the [Priicer community](https://forum.priicer.com/t/pg-e-dynamic-pilot-california/33).

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

One program per rate schedule × circuit/substation:

| Field | PG&E Example | SCE Example |
|-------|-------------|-------------|
| `programName` | `EELEC-013532223` | `TOU-PRIME-Eagle Rock` |
| `programLongName` | `PG&E EELEC — circuit 013532223` | `SCE TOU-PRIME — substation Eagle Rock` |
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

**Prices**: The price server fetches a **7-day forward window** (today + 6 days) from GridX on startup and every hour. Day-ahead prices are typically available by ~4:30 PM PST. Historical prices go back to approximately August 2024 for PG&E and July 2025 for SCE.

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
