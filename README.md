# Price Server User Guide

Public documentation for the [Grid Coordination](https://grid-coordination.energy) price server — an [OpenADR 3.1.0](https://www.openadr.org/) VTN serving hourly California marginal electricity prices from the CAISO Day-Ahead Market via [GridX](https://www.gridx.com/).

## What is the price server?

The price server publishes real-time and historical electricity prices for **PG&E** and **SCE** rate schedules as OpenADR 3 programs and events. Each program represents a specific rate schedule × distribution circuit (PG&E) or substation (SCE) combination. Each event contains 24 hourly price intervals for one day.

Prices are marginal cost in USD/kWh from the CAISO Day-Ahead Market, composed through GridX CalFUSE rate calculations.

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
| | | **Total** | **492 programs** |

PG&E programs are named `<RATE>-<9-digit-circuit-id>` (e.g. `EELEC-013532223`). SCE programs are named `<RATE>-<substation>` (e.g. `TOU-PRIME-Eagle Rock`).

## Finding your program

To get prices relevant to you, you need two things:

1. **Rate schedule** — which tariff applies to your service (e.g. `EELEC` for PG&E residential, `TOU-PRIME` for SCE residential). See the tariff table above.
2. **Circuit or substation** — which part of the grid you're on. For PG&E, this is a 9-digit circuit ID identifying your distribution feeder. For SCE, it's a substation name.

The combination determines your program name: `EELEC-013532223` or `TOU-PRIME-Eagle Rock`.

**How to find your circuit or substation:** If you don't know which circuit or substation serves your location, you can browse the full list in [circuits.md](circuits.md) — it includes the substation name, division, and region for every PG&E circuit, and all 46 SCE substation names. PG&E customers on the Dynamic Rate Pilot can find their circuit ID on their billing statement or through the [Priicer community](https://forum.priicer.com/t/pg-e-dynamic-pilot-california/33).

## How to access

### Interactive API reference

Browse and test all endpoints in your browser:

```
https://price.grid-coordination.energy/api
```

The interactive reference (powered by [Scalar](https://github.com/scalar/scalar)) lets you explore endpoints, see schemas, and make live requests against the server. The underlying OpenAPI specification is also available directly:

```
https://price.grid-coordination.energy/openapi.json
```

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

| Field | Example |
|-------|---------|
| `eventName` | `EELEC-013532223-2026-04-12` |
| `programID` | UUID linking to the program |
| `intervals[n].id` | Hour of day (0 = midnight, 23 = 11 PM) |
| `intervals[n].payloads[0].type` | `PRICE` |
| `intervals[n].payloads[0].values[0]` | `0.02622` ($/kWh) |

### Forward window and history

The price server fetches a **7-day forward window** (today + 6 days) from GridX on startup and every hour. Day-ahead prices are typically available by ~4:30 PM PST. Historical prices go back to approximately August 2024 for PG&E and July 2025 for SCE.

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
