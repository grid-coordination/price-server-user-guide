# Changelog

Changes to the public price server that are visible to clients, newest first.

## 2026-10-16

- **Program units use the OpenADR 3.1.0 enumeration.** `payloadDescriptors[].units` is now `KWH` on all `PRICE` programs (was `kWh`) and `GHG` on all MOER programs (was `g/kWh`), the values in the spec's `units.schema.yaml`. Event payload values are unchanged: prices are still USD/kWh and MOER is still g CO2/kWh. A client that compares `units` to `kWh` or `g/kWh` needs updating. ([clj-price-server#11](https://github.com/grid-coordination/clj-price-server/issues/11))
