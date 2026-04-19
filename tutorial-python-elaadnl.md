# Tutorial: reading prices with openadr3-client (ElaadNL)

This tutorial walks you through using [openadr3-client](https://github.com/ElaadNL/openadr3-client), the ElaadNL OpenADR 3 Python client library, to connect to the live price server and read California electricity prices.

**Estimated time:** 5 minutes. You'll need Python 3.12+ and `pip`.

> **Note on authentication:** The ElaadNL client currently requires OAuth credentials in its factory. Our price server has no auth (Phase 1). This tutorial includes a workaround. We've filed [ElaadNL/openadr3-client#87](https://github.com/ElaadNL/openadr3-client/issues/87) requesting anonymous VTN support.

## What you'll build

By the end, you'll have a script that:

1. Lists all programs in the price server (PG&E + SCE tariffs)
2. Fetches today's 24 hourly prices for a specific circuit
3. Prints a formatted price table

## The price server

A public OpenADR 3.1.0 VTN serving hourly California marginal electricity prices from the CAISO Day-Ahead Market via [GridX](https://www.gridx.com/). Covers 9 rate schedules across PG&E (59 circuits) and SCE (46 substations) — 492 programs total. Base URL:

```
https://price.grid-coordination.energy/openadr3/3.1.0
```

No authentication is required.

## 1. Install

```bash
python3.12 -m venv .venv
source .venv/bin/activate
pip install openadr3-client
```

> **Requires Python 3.12+.** Earlier versions are not supported by the library.

## 2. Connect to the price server

The ElaadNL client's factory requires OAuth parameters. Since our VTN is unauthenticated, we create the client normally then replace the authenticated session with a plain `requests.Session()`:

```python
import requests
from openadr3_client.ven.http_factory import VirtualEndNodeHttpClientFactory
from openadr3_client.version import OADRVersion

URL = "https://price.grid-coordination.energy/openadr3/3.1.0"

def create_anonymous_ven(url: str):
    """Create a VEN client for an unauthenticated VTN.

    Workaround for ElaadNL/openadr3-client#87 — the library doesn't
    yet support anonymous connections natively.
    """
    ven = VirtualEndNodeHttpClientFactory.create_http_ven_client(
        vtn_base_url=url,
        client_id="anonymous",
        client_secret="anonymous",
        token_url=f"{url}/auth/token",
        version=OADRVersion.OADR_310,
    )
    # Replace the OAuth session with a plain session
    plain = requests.Session()
    ven.programs.session = plain
    ven.events.session = plain
    return ven

ven = create_anonymous_ven(URL)
```

> **When ElaadNL/openadr3-client#87 is resolved**, this workaround won't be needed. The factory will accept `None` for credentials and use an unauthenticated session directly.

## 3. List programs

```python
programs = ven.programs.get_programs(target=None, pagination=None)
print(f"{len(programs)} programs (first 10):")
for p in programs[:10]:
    print(f"  {p.program_name}")
```

Output:

```
50 programs (first 10):
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

> **Note:** The default page size is 50. There are 492 programs total (6 PG&E rates × 59 circuits + 3 SCE rates × 46 substations). Use the `pagination` parameter to page through them.

The ElaadNL models are Pydantic v2 with strong typing. Each program has:

```python
p = programs[0]
print(f"Name: {p.program_name}")
print(f"ID:   {p.id}")
if p.payload_descriptor:
    pd = p.payload_descriptor[0]
    print(f"Type: {pd.payload_type}")  # PRICE
    print(f"Unit: {pd.units}")         # KWH
```

## 4. Fetch hourly prices

```python
# Find a specific circuit's program
target_name = "EELEC-013532223"
program = next(p for p in programs if p.program_name == target_name)
print(f"Program: {program.program_name} (id: {program.id})")

# Get events for this program
events = ven.events.get_events(
    target=None,
    pagination=None,
    program_id=program.id,
)
print(f"{len(events)} events")

# Show the first event's prices
event = events[0]
print(f"\n{event.event_name}")
print(f"{'Hour':<8} {'Price ($/kWh)'}")
print("-" * 25)

if event.intervals:
    for interval in event.intervals:
        hour = interval.id
        if interval.payloads:
            price = interval.payloads[0].values[0]
            print(f"{hour:02d}:00    ${price:.5f}")
```

Output:

```
Program: EELEC-013532223 (id: b54d3d1f-bc87-4e47-bbc1-2b95958283fc)
50 events

EELEC-013532223-2026-04-11
Hour     Price ($/kWh)
-------------------------
00:00    $0.03024
01:00    $0.03024
02:00    $0.02855
...
13:00    $0.01239
14:00    $0.01265
...
23:00    $0.02724
```

## 5. Model details

The ElaadNL library uses frozen Pydantic v2 models with rich typing:

```python
# Event fields
event.event_name          # str: "EELEC-013532223-2026-04-11"
event.id                  # str: UUID
event.programID           # str: UUID of the parent program
event.intervals           # tuple[Interval, ...] | None

# Interval fields
interval.id               # int: hour of day (0-23)
interval.payloads         # tuple[EventPayload, ...]
interval.interval_period  # IntervalPeriod | None

# Payload fields
payload.type              # EventPayloadType: PRICE
payload.values            # tuple[float, ...]
```

Intervals are ordered by hour (0 = midnight local, 23 = 11 PM local). Prices are marginal cost in USD/kWh from the CAISO Day-Ahead Market.

## Going further

- **Other rates**: PG&E serves EELEC, BEV1, BEV2P, BEV2S, B6, B19P. SCE serves TOU-PRIME, TOU-D-49, TOU-D-58. Paginate with `pagination` to find them.
- **GHG emissions**: 11 MOER programs (`MOER-PGE`, `MOER-SCE`, etc.) publish hourly marginal GHG emissions in g CO2/kWh. Fetch the same way as prices — find the program, get events, read interval payloads. See [README.md#ghg-emissions-moer](README.md#ghg-emissions-moer).
- **Pandas DataFrame support**: `pip install 'openadr3-client[pandas]'` adds event interval conversion to DataFrames — great for analysis.
- **MQTT notifications**: `pip install 'openadr3-client[mqtt]'` adds MQTT support. The price server's broker is at `mqtt.grid-coordination.energy` (ports 1883/8883). See [mqtt-notifications.md](mqtt-notifications.md).
- **Historical data**: events go back ~600 days for PG&E, ~200 days for SCE. Page through events to access history.

## About openadr3-client (ElaadNL)

[openadr3-client](https://github.com/ElaadNL/openadr3-client) is developed by [ElaadNL](https://elaad.nl/en/) as part of the Dutch smart charging ecosystem. It supports both OpenADR 3.0.1 and 3.1.0, with typed Pydantic v2 models, OAuth2 client credentials auth, and optional pandas/MQTT integrations. Published on [PyPI](https://pypi.org/project/openadr3-client/).

## Full script

```python
#!/usr/bin/env python3
"""Read PG&E EELEC prices using the ElaadNL openadr3-client."""
import requests
from openadr3_client.ven.http_factory import VirtualEndNodeHttpClientFactory
from openadr3_client.version import OADRVersion

URL = "https://price.grid-coordination.energy/openadr3/3.1.0"

# Workaround for anonymous VTN (ElaadNL/openadr3-client#87)
ven = VirtualEndNodeHttpClientFactory.create_http_ven_client(
    vtn_base_url=URL,
    client_id="anonymous",
    client_secret="anonymous",
    token_url=f"{URL}/auth/token",
    version=OADRVersion.OADR_310,
)
plain = requests.Session()
ven.programs.session = plain
ven.events.session = plain

# 1. List programs
programs = ven.programs.get_programs(target=None, pagination=None)
print(f"{len(programs)} programs (first 5):")
for p in programs[:5]:
    print(f"  {p.program_name}")

# 2. Find a circuit and get events
target = next(p for p in programs if p.program_name == "EELEC-013532223")
events = ven.events.get_events(target=None, pagination=None, program_id=target.id)
print(f"\n{len(events)} events for {target.program_name}")

# 3. Print today's prices
if events:
    event = events[0]
    print(f"\n{event.event_name}")
    print(f"{'Hour':<8} {'Price ($/kWh)'}")
    print("-" * 25)
    if event.intervals:
        for interval in event.intervals:
            if interval.payloads:
                price = interval.payloads[0].values[0]
                print(f"{interval.id:02d}:00    ${price:.5f}")
```
