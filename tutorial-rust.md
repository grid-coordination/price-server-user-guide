# Tutorial: reading prices with openleadr-rs

This tutorial walks you through using [openleadr-rs](https://github.com/OpenLEADR/openleadr-rs), the LF Energy OpenLEADR Rust client library, to connect to the live price server and read California electricity prices.

**Estimated time:** 10 minutes. You'll need Rust 1.80+ and `cargo`.

## What you'll build

By the end, you'll have a Rust program that:

1. Lists all programs in the price server (PG&E + SCE tariffs)
2. Finds a specific circuit's program
3. Fetches today's 24 hourly prices and prints a formatted table
4. Shows today and tomorrow prices side by side

## The price server

A public OpenADR 3.1.0 VTN serving hourly California marginal electricity prices from the CAISO Day-Ahead Market, published via [GridX](https://www.gridx.com/). Covers 9 rate schedules across PG&E (59 circuits) and SCE (46 substations) — 492 programs total. Base URL:

```
https://price.grid-coordination.energy/openadr3/3.1.0/
```

No authentication is required.

## 1. Create a project

```bash
cargo new pge-prices
cd pge-prices
```

Edit `Cargo.toml`:

```toml
[package]
name = "pge-prices"
version = "0.1.0"
edition = "2021"

[dependencies]
openleadr-client = "0.2"
openleadr-wire = "0.2"
tokio = { version = "1", features = ["full"] }
```

## 2. Connect and list programs

Replace `src/main.rs`:

```rust
use openleadr_client::{Client, Filter, VirtualEndNode};

#[tokio::main]
async fn main() {
    // NOTE: trailing slash is required (url crate join semantics)
    let url = "https://price.grid-coordination.energy/openadr3/3.1.0/";
    let client = Client::<VirtualEndNode>::with_url(url.parse().unwrap(), None);

    // List programs — no auth needed, None for credentials
    let filter: Filter<&str> = Filter::None;
    let programs = client.get_program_list(filter).await.unwrap();

    println!("{} programs (first 10):", programs.len());
    for p in programs.iter().take(10) {
        println!("  {}", p.content().program_name);
    }
}
```

Run it:

```bash
cargo run
```

Output:

```
492 programs (first 10):
  EELEC-012041131
  EELEC-012640401
  EELEC-013532223
  EELEC-013921103
  EELEC-014052103
  EELEC-014321101
  EELEC-014592108
  EELEC-022011162
  EELEC-024011104
  EELEC-024040403
```

> **Note on `Client::<VirtualEndNode>`:** The openleadr-rs client is generic over the client role. `VirtualEndNode` provides read-only VEN operations (list programs, read events). `BusinessLogic` provides full CRUD. For reading prices, `VirtualEndNode` is the right choice.
>
> **Note on trailing slash:** The base URL must end with `/`. Without it, the `url` crate's `Url::join` drops the last path segment, causing 404 errors.

## 3. Find a specific program and get events

```rust
use openleadr_client::{Client, Filter, VirtualEndNode};

#[tokio::main]
async fn main() {
    let url = "https://price.grid-coordination.energy/openadr3/3.1.0/";
    let client = Client::<VirtualEndNode>::with_url(url.parse().unwrap(), None);

    // Find a specific circuit
    let programs = client.get_program_list(Filter::<&str>::None).await.unwrap();
    let program = programs
        .iter()
        .find(|p| p.content().program_name == "EELEC-013532223")
        .expect("Program not found");

    println!("Program: {} (id: {})", program.content().program_name, program.id());

    // Get events for this program
    let events = program.get_event_list(Filter::<&str>::None).await.unwrap();
    println!("{} events\n", events.len());

    // Show the first event's prices
    if let Some(event) = events.first() {
        let content = event.content();
        if let Some(name) = &content.event_name {
            println!("{name}");
        }

        if let Some(ref intervals) = content.intervals {
            println!("{:-<40}", "");
            println!("{:<8} {:<15}", "Hour", "Price ($/kWh)");
            println!("{:-<40}", "");

            for interval in intervals {
                if let Some(payload) = interval.payloads.first() {
                    if let Some(value) = payload.values.first() {
                        println!("{:02}:00    {:?}", interval.id, value);
                    }
                }
            }
        }
    }
}
```

Output:

```
Program: EELEC-013532223 (id: b54d3d1f-bc87-4e47-bbc1-2b95958283fc)
550 events

EELEC-013532223-2026-04-11
----------------------------------------
Hour     Price ($/kWh)
----------------------------------------
00:00    Number(0.030244)
01:00    Number(0.030244)
02:00    Number(0.028553)
...
13:00    Number(0.01239)
14:00    Number(0.012653)
...
23:00    Number(0.027242)
```

The prices are returned as `Value::Number(f64)`. The classic California duck curve is visible — low prices midday (solar), higher in the morning and evening.

## 4. Today and tomorrow

Events are named `<RATE>-<CIRCUIT>-<YYYY-MM-DD>`. To find today's and tomorrow's prices:

```rust
use openleadr_client::{Client, Filter, VirtualEndNode};
use chrono::{Local, Duration};

// Add to Cargo.toml: chrono = "0.4"

#[tokio::main]
async fn main() {
    let url = "https://price.grid-coordination.energy/openadr3/3.1.0/";
    let client = Client::<VirtualEndNode>::with_url(url.parse().unwrap(), None);

    let programs = client.get_program_list(Filter::<&str>::None).await.unwrap();
    let program = programs
        .iter()
        .find(|p| p.content().program_name == "EELEC-013532223")
        .unwrap();

    let events = program.get_event_list(Filter::<&str>::None).await.unwrap();

    let today = Local::now().format("%Y-%m-%d").to_string();
    let tomorrow = (Local::now() + Duration::days(1)).format("%Y-%m-%d").to_string();

    let today_event = events.iter().find(|e| {
        e.content().event_name.as_deref()
            .is_some_and(|n| n.ends_with(&today))
    });
    let tomorrow_event = events.iter().find(|e| {
        e.content().event_name.as_deref()
            .is_some_and(|n| n.ends_with(&tomorrow))
    });

    println!("{:<8} {:<18} {:<18}", "Hour",
             format!("Today ({})", &today),
             format!("Tomorrow ({})", &tomorrow));
    println!("{:-<50}", "");

    for hour in 0..24 {
        let get_price = |event: Option<&openleadr_client::EventClient<VirtualEndNode>>| -> String {
            event
                .and_then(|e| e.content().intervals.as_ref())
                .and_then(|intervals| intervals.get(hour))
                .and_then(|i| i.payloads.first())
                .and_then(|p| p.values.first())
                .map(|v| format!("{:?}", v))
                .unwrap_or_else(|| "  —".to_string())
        };
        println!("{:02}:00    {:<18} {:<18}",
                 hour,
                 get_price(today_event),
                 get_price(tomorrow_event));
    }
}
```

> **Note:** Add `chrono = "0.4"` to your `Cargo.toml` dependencies for the date handling.

## 5. What about other rates?

The price server serves 9 rate schedules:

| Utility | Rates | Circuits |
|---------|-------|----------|
| PG&E | EELEC, BEV1, BEV2P, BEV2S, B6, B19P | 59 feeders (9-digit IDs) |
| SCE | TOU-PRIME, TOU-D-49, TOU-D-58 | 46 substations (names like "Eagle Rock") |

To list SCE programs, paginate past the PG&E ones:

```rust
// SCE programs start around skip=354
// (after 6 PGE rates × 59 circuits)
```

Or filter the full list:

```rust
let sce_programs: Vec<_> = programs
    .iter()
    .filter(|p| p.content().program_name.starts_with("TOU-"))
    .collect();
println!("{} SCE programs", sce_programs.len());
```

## Going further

- **GHG emissions**: 11 MOER programs (`MOER-PGE`, `MOER-SCE`, etc.) publish hourly marginal GHG emissions in g CO2/kWh. Fetch the same way as prices — find the program, get events, read interval payloads. See [README.md#ghg-emissions-moer](README.md#ghg-emissions-moer).
- **MQTT notifications**: subscribe to `tcp://mqtt.grid-coordination.energy:1883` or `mqtts://:8883` topic `openadr3/3.1.0/events/#` for push updates. See [mqtt-notifications.md](mqtt-notifications.md).
- **Historical data**: events go back to ~August 2024 for PG&E, ~July 2025 for SCE. The `get_event_list` call returns all available events.
- **openleadr-rs docs**: full API documentation at [docs.rs/openleadr-client](https://docs.rs/openleadr-client/latest/openleadr_client/).

## About openleadr-rs

[openleadr-rs](https://github.com/OpenLEADR/openleadr-rs) is an [LF Energy](https://lfenergy.org/) project implementing OpenADR 3.1 in Rust. It includes both a VEN client library (used here) and a full VTN server implementation. Developed by [Tweede Golf](https://tweedegolf.nl/en) and sponsored by [ElaadNL](https://elaad.nl/en/).
