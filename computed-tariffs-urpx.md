# URPX computed tariffs

The price server publishes the following 27 computed-tariff programs sourced from [URPX](https://lfenergy.org/projects/utility-rate-plan-exchange-urpx/) (Utility Rate Plan eXchange) — the LF Energy semantic standard for utility rate plans. Source filings live in the [`urpx-tariffs`](https://github.com/grid-coordination/urpx-tariffs) catalog.

Programs are listed by `programName`. Use `GET /programs?programName=<name>` to look one up, then `GET /events?programID=<id>` to fetch its hourly prices.

Most rate plans publish a single program. A few base plans are also offered with one or more **modifiers** applied — distinct programs whose prices include the modifier's adjustment (e.g. a low-income discount, a net-surplus export credit, an exchange/adjustment factor). The base plan is always published alongside any modifier-applied variants so consumers can choose whichever combination matches their account.

## PG&E (Pacific Gas & Electric)

| programName     | Description                                                                       |
| --------------- | --------------------------------------------------------------------------------- |
| `PGE-E-1`       | Residential Services                                                              |
| `PGE-E-ELEC`    | Residential Time-of-Use (Electric Home) Service                                   |
| `PGE-E-TOU-C`   | Residential Time-Of-Use (Peak Pricing 4–9 p.m. Every Day)                         |
| `PGE-E-TOU-D`   | Residential Time-Of-Use (Peak Pricing 5–8 p.m. Non-Holiday Weekdays)              |
| `PGE-EV2`       | Residential Time-Of-Use Service for Plug-In Electric Vehicle Customers            |

## SCE (Southern California Edison)

| programName             | Description                                                  |
| ----------------------- | ------------------------------------------------------------ |
| `SCE-D`                 | Schedule D Domestic Service                                  |
| `SCE-D-DEED-RESTRICTED` | Schedule D + Deed-Restricted Base Services Charge Discount   |

## SDG&E (San Diego Gas & Electric)

| programName            | Description                                                                                                              |
| ---------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| `SDGE-DR-SES`          | Schedule DR-SES Domestic Time-of-Use for Households with a Solar Energy System                                           |
| `SDGE-DR-SES-DRAH`     | Schedule DR-SES + Deed-Restricted Affordable Housing (DRAH) BSC Discount                                                 |
| `SDGE-EV-TOU-5`        | Schedule EV-TOU-5 Cost-Based Domestic Time-of-Use for Households with Electric Vehicles                                  |
| `SDGE-EV-TOU-5-DRAH`   | Schedule EV-TOU-5 + Deed-Restricted Affordable Housing (DRAH) BSC Discount                                               |
| `SDGE-TOU-DR2`         | Schedule TOU-DR2 Residential Time-of-Use                                                                                 |
| `SDGE-TOU-DR2-DRAH`    | Schedule TOU-DR2 + Deed-Restricted Affordable Housing (DRAH) BSC Discount                                                |
| `SDGE-TOU-ELEC`        | Schedule TOU-ELEC Domestic Time-of-Use for Households with Electric Vehicles, Energy Storage, or Electric Heat Pumps     |
| `SDGE-TOU-ELEC-DRAH`   | Schedule TOU-ELEC + Deed-Restricted Affordable Housing (DRAH) BSC Discount                                               |

## CPAU (City of Palo Alto Utilities)

| programName              | Description                                                                                       |
| ------------------------ | ------------------------------------------------------------------------------------------------- |
| `CPAU-E-1`               | Residential Electric Service                                                                      |
| `CPAU-E-1-E-EEC-1`       | Residential Electric Service + Export Electricity Compensation                                    |
| `CPAU-E-1-E-HRA`         | Residential Electric Service + Electric Hydro Rate Adjuster                                       |
| `CPAU-E-1-E-NSE-1`       | Residential Electric Service + Net Metering Net Surplus Electricity Compensation                  |
| `CPAU-E-1-TOU`           | Residential Electric Time of Use Service                                                          |
| `CPAU-E-1-TOU-E-HRA`     | Residential Electric Time of Use Service + Electric Hydro Rate Adjuster                           |
| `CPAU-E-1-TOU-E-NSE-1`   | Residential Electric Time of Use Service + Net Metering Net Surplus Electricity Compensation      |
| `CPAU-E-2`               | Residential Master-Metered and Small Non-Residential Electric Service                             |
| `CPAU-E-2-E-EEC-1`       | Residential Master-Metered / Small Non-Residential + Export Electricity Compensation              |
| `CPAU-E-2-E-HRA`         | Residential Master-Metered / Small Non-Residential + Electric Hydro Rate Adjuster                 |
| `CPAU-E-2-E-NSE-1`       | Residential Master-Metered / Small Non-Residential + Net Metering Net Surplus Electricity Compensation |

## LADWP (Los Angeles Department of Water and Power)

| programName  | Description                                       |
| ------------ | ------------------------------------------------- |
| `LADWP-R-1A` | Schedule R-1(A) Residential Service — Standard    |

## Naming convention

URPX programs use the rate plan's URPX **CURIE** (Compact URI) as the basis for `programName`:

- `<utility-prefix>:<rate-plan>` → `<UTILITY-PREFIX>-<RATE-PLAN>` (uppercase)
  - e.g. `pge:e-tou-c` → `PGE-E-TOU-C`
- For modifier-applied variants, the modifier's CURIE local-name is appended:
  - e.g. base `cpau:e-1-tou` + modifier `cpau:e-hra` → `CPAU-E-1-TOU-E-HRA`

This makes program names predictable from the underlying URPX filing without per-utility lookup tables.

## Adding more rate plans

The `urpx-tariffs` catalog contains many more rate plans than the price server currently publishes. The price server only publishes a rate plan once its URPX filing is marked `[modeling].complete = true` in `tariff.toml` — i.e. the modeling team has confirmed the filing is ready for downstream use. Modifier-applied variants are only published when both the base plan and the modifier are independently complete, so consumers never receive prices computed from a knowingly-incomplete filing.

To request publication of a specific rate plan, [open a Discussion](https://github.com/grid-coordination/price-server-user-guide/discussions).
