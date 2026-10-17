# Changelog

Changes visible to clients of the public price server, including corrections to this guide, newest first.

## 2026-10-16

- **MQTT documentation corrections.** The server's MQTT behavior is unchanged; these fix what the guide said about it.
  - Topics are case-sensitive and start with `OpenADR/3.1.0/`. The README and the Clojure, Python and Rust tutorials gave `openadr3/3.1.0/events/#`, which matches nothing.
  - Notifications are envelopes, `{"objectType", "operation", "object"}`, with the entity in `object`. [mqtt-notifications.md](mqtt-notifications.md) showed a bare entity, and its Python example printed `?` for every event name.
  - Examples now subscribe to a program's own topic, `OpenADR/3.1.0/programs/<programID>/events/+`. The global `OpenADR/3.1.0/events/#` carries every program's events.
- **Simulated demo programs.** Programs named `DEMO-*` (`DEMO-SIMULATED-FLEX-ALERT`, `DEMO-DOE-<site>`) may appear during demonstrations. They carry simulated data, not real grid signals. See [demo-programs.md](demo-programs.md).
- **Program units use the OpenADR 3.1.0 enumeration.** `payloadDescriptors[].units` is now `KWH` on all `PRICE` programs (was `kWh`) and `GHG` on all MOER programs (was `g/kWh`), the values in the spec's `units.schema.yaml`. Event payload values are unchanged: prices are still USD/kWh and MOER is still g CO2/kWh. A client that compares `units` to `kWh` or `g/kWh` needs updating. ([clj-price-server#11](https://github.com/grid-coordination/clj-price-server/issues/11))
