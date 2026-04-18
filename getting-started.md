# Getting Started

This guide gets you reading electricity prices from the price server in under 2 minutes using only `curl`.

## Base URL

```
https://price.grid-coordination.energy/openadr3/3.1.0
```

No authentication is required. All endpoints are read-only.

You can also explore the API interactively at [/api](https://price.grid-coordination.energy/api), or download the [OpenAPI spec](https://price.grid-coordination.energy/openapi.json) for use with code generators and other tools.

## 1. Choose your rate and circuit

To get prices, you need to identify two things:

1. **Rate schedule** — the tariff that applies to your service. For PG&E residential, that's `EELEC`. For SCE residential, `TOU-PRIME`, `TOU-D-49`, or `TOU-D-58`. See the [full tariff list](README.md#available-tariffs).
2. **Circuit (PG&E) or substation (SCE)** — which part of the distribution grid serves your location. See [circuits.md](circuits.md) for the complete list with location details.

These combine into a **program name**: `EELEC-013532223` (PG&E) or `TOU-PRIME-Eagle Rock` (SCE). Each program has one event per day with 24 hourly prices.

If you already know your program name, skip to step 3.

## 2. Find your program in the API

List programs to find the UUID for your program name:

```bash
curl -s https://price.grid-coordination.energy/openadr3/3.1.0/programs | python3 -m json.tool
```

The default page returns 50 programs. There are 492 total. Use `?skip=50` to paginate:

```bash
curl -s 'https://price.grid-coordination.energy/openadr3/3.1.0/programs?skip=50' | python3 -m json.tool
```

Filter for a specific program:

```bash
curl -s https://price.grid-coordination.energy/openadr3/3.1.0/programs \
  | python3 -c "
import json, sys
programs = json.load(sys.stdin)
for p in programs:
    if 'EELEC-013532223' in p['programName']:
        print(json.dumps(p, indent=2))
        break
"
```

Note the `id` field (a UUID) — you'll use it to query events.

## 3. Get today's prices

Fetch events for your program using the UUID from step 2:

```bash
PROGRAM_ID="<uuid-from-step-2>"
curl -s "https://price.grid-coordination.energy/openadr3/3.1.0/events?programID=$PROGRAM_ID" \
  | python3 -m json.tool
```

Each event covers one day and contains 24 intervals (one per hour). The interval `id` is the hour of day (0 = midnight local, 23 = 11 PM). The price is in `payloads[0].values[0]`, in USD/kWh.

## 4. Print a price table

```bash
PROGRAM_ID="<uuid-from-step-2>"
curl -s "https://price.grid-coordination.energy/openadr3/3.1.0/events?programID=$PROGRAM_ID" \
  | python3 -c "
import json, sys
events = json.load(sys.stdin)
for event in events[:1]:
    print(event['eventName'])
    print(f\"{'Hour':<8} {'Price ($/kWh)'}\")
    print('-' * 25)
    for interval in event['intervals']:
        hour = interval['id']
        price = interval['payloads'][0]['values'][0]
        print(f'{hour:02d}:00    \${price:.5f}')
"
```

Output:

```
EELEC-013532223-2026-04-12
Hour     Price ($/kWh)
-------------------------
00:00    $0.02622
01:00    $0.02500
02:00    $0.02441
...
17:00    $0.01955
18:00    $0.03180
...
23:00    $0.02893
```

## 5. Subscribe to updates via MQTT

For real-time price notifications, subscribe to the MQTT broker:

```bash
# Plain TCP
mosquitto_sub -h mqtt.grid-coordination.energy -p 1883 \
  -t 'OpenADR/3.1.0/events/#' -v

# TLS (Linux)
mosquitto_sub -h mqtt.grid-coordination.energy -p 8883 \
  --capath /etc/ssl/certs \
  -t 'OpenADR/3.1.0/events/#' -v

# TLS (macOS)
mosquitto_sub -h mqtt.grid-coordination.energy -p 8883 \
  --cafile /etc/ssl/cert.pem \
  -t 'OpenADR/3.1.0/events/#' -v
```

See [mqtt-notifications.md](mqtt-notifications.md) for more details.

## Next steps

- **Python tutorial:** [tutorial-python.md](tutorial-python.md) — full walkthrough with charting
- **Python (ElaadNL) tutorial:** [tutorial-python-elaadnl.md](tutorial-python-elaadnl.md)
- **Clojure tutorial:** [tutorial-clojure.md](tutorial-clojure.md) — REPL-driven with Vega-Lite charts
- **Rust tutorial:** [tutorial-rust.md](tutorial-rust.md) — using openleadr-rs
