# Tutorial: reading prices with python-oa3-client

This tutorial walks you through installing [python-oa3-client](https://github.com/grid-coordination/python-oa3-client), connecting it to the live price server, and reading California electricity prices.

**Estimated time:** 5 minutes. You'll need Python 3.11+ and `pip`.

## What you'll build

By the end, you'll have a script that:

1. Lists all programs in the price server (PG&E + SCE tariffs)
2. Looks up a specific circuit's program
3. Fetches today's hourly prices (24 intervals on a normal day; 23 or 25 on DST transitions) and prints a table
4. Charts those prices with matplotlib

## The price server

A public OpenADR 3 VTN serving hourly California marginal electricity prices from the CAISO Day-Ahead Market, published via [GridX](https://www.gridx.com/). Covers California utility rate schedules from PG&E and SCE, plus GHG emissions for California grid regions. Base URL:

```
https://price.grid-coordination.energy/openadr3/3.1.0
```

No authentication is required.

## 1. Install

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install 'python-oa3-client>=0.4.0' matplotlib tabulate
```

`matplotlib` is only needed for the chart at the end. `tabulate` is for pretty-printing tables. Version **0.4.0** or newer is required — it pulls in `python-oa3 0.3.0+` which exposes datetimes as zone-aware `pendulum.DateTime` end-to-end (matching the OpenADR spec's UTC `Z` form on the wire).

## 2. Connect and list programs

Save this as `list_programs.py`:

```python
from openadr3_client import VenClient
from tabulate import tabulate

URL = "https://price.grid-coordination.energy/openadr3/3.1.0"

with VenClient(url=URL, token="anonymous",
               user_agent="my-app/1.0 (contact@example.com)") as ven:
    programs = ven.programs()

# programs are coerced Pydantic Program objects
rows = [[p.program_name, p.id] for p in programs]
print(tabulate(rows, headers=["Program", "ID"], tablefmt="simple"))
print(f"\n{len(programs)} programs")
```

Run it:

```bash
python list_programs.py
```

You'll see output like:

```
Program          ID
---------------  ------------------------------------
EELEC-012041131  2d3e1839-f8cb-4a1b-aa6c-c1168297d5cc
EELEC-012640401  6711667c-2b1f-42ad-ae08-c118f41dd0f3
EELEC-013532223  b54d3d1f-bc87-4e47-bbc1-2b95958283fc
...

50 programs
```

> **Note:** You'll see 50 programs per page (the OpenADR 3 pagination limit). There are far more than 50 programs available — paginate with `ven.programs(skip=50, limit=50)` to get the next page of 50.
>
> All client methods (`programs()`, `find_program_by_name()`, `poll_events()`) return coerced [Pydantic](https://docs.pydantic.dev/) models — access fields as attributes, not dict keys.
>
> The `user_agent` parameter (new in 0.3.0) identifies your application in the server's access logs. It's optional but encouraged — helps the operator understand who's using the service.

## 3. Look up a specific circuit

Each `programName` follows the pattern `EELEC-<circuitId>`. Let's look up `EELEC-013532223` (a feeder near Lakewood substation, Bay Area):

```python
with VenClient(url=URL, token="anonymous",
               user_agent="my-app/1.0") as ven:
    prog = ven.find_program_by_name("EELEC-013532223")
    print(f"Program: {prog.program_name}")
    print(f"  ID:    {prog.id}")
    if prog.payload_descriptors:
        pd = prog.payload_descriptors[0]
        print(f"  Type:  {pd.payload_type}")
        print(f"  Units: {pd.units}")
        print(f"  Ccy:   {pd.currency}")
```

All fields are accessed as attributes — `prog.program_name`, `pd.payload_type`, etc. As of 0.3.0, `payload_descriptors` returns typed `EventPayloadDescriptor` models (not raw dicts).

## 4. Fetch today's hourly prices

Save as `show_prices.py`:

```python
from datetime import date
from decimal import Decimal
from openadr3_client import VenClient
from tabulate import tabulate

URL = "https://price.grid-coordination.energy/openadr3/3.1.0"
CIRCUIT = "EELEC-013532223"

with VenClient(url=URL, token="anonymous", user_agent="my-app/1.0") as ven:
    events = ven.poll_events(CIRCUIT)

if not events:
    print(f"No events for {CIRCUIT}")
    raise SystemExit(1)

# Each event covers one day. Pick the event for today.
today = date.today().isoformat()
event = next((e for e in events if today in e.event_name), events[0])

# interval.id is the hour-of-day in the program zone (0..23 normally; 0..22 on PT
# spring-forward, 0..24 on fall-back). interval.interval_period.start is the canonical
# UTC Z instant. payload[0].values[0] is the price.
rows = []
for interval in event.intervals:
    hour = interval.id
    price = interval.payloads[0].values[0]
    rows.append([f"{hour:02d}:00", f"${price:.5f}/kWh"])

print(f"{event.event_name}\n")
print(tabulate(rows, headers=["Hour", "Price"], tablefmt="simple"))

prices = [float(i.payloads[0].values[0]) for i in event.intervals]
print(f"\nmin: ${min(prices):.5f}   max: ${max(prices):.5f}   avg: ${sum(prices)/len(prices):.5f}")
```

Output:

```
EELEC-013532223-2026-04-11

Hour    Price
------  -------------
00:00   $0.03024/kWh
01:00   $0.02955/kWh
02:00   $0.02941/kWh
...
17:00   $0.01955/kWh
18:00   $0.03180/kWh
...
23:00   $0.02893/kWh

min: $0.01955   max: $0.04080   avg: $0.02755
```

> **Note:** The client returns coerced entity objects (`event.event_name`, `interval.id`, etc.). If you prefer the raw JSON shape, use `ven.api.events(programID=...)` and `ven.api.get_programs()` instead of the coerced helpers.

## 5. Chart the prices

Save as `chart_prices.py`:

```python
from datetime import date
import matplotlib.pyplot as plt
from openadr3_client import VenClient

URL = "https://price.grid-coordination.energy/openadr3/3.1.0"
CIRCUIT = "EELEC-013532223"

with VenClient(url=URL, token="anonymous", user_agent="my-app/1.0") as ven:
    events = ven.poll_events(CIRCUIT)

today = date.today().isoformat()
event = next((e for e in events if today in e.event_name), events[0])

hours  = [i.id for i in event.intervals]
prices = [float(i.payloads[0].values[0]) for i in event.intervals]

fig, ax = plt.subplots(figsize=(10, 5))
ax.bar(hours, prices, color="steelblue", edgecolor="navy")
ax.set_xlabel("Hour of day (local)")
ax.set_ylabel("Price ($/kWh)")
ax.set_title(f"{event.event_name} — PG&E EELEC marginal price")
ax.set_xticks(range(0, 24, 2))
ax.grid(axis="y", alpha=0.3)
ax.set_axisbelow(True)

# Annotate the cheapest hour
min_idx = prices.index(min(prices))
ax.annotate(f"cheapest\n${prices[min_idx]:.4f}",
            xy=(hours[min_idx], prices[min_idx]),
            xytext=(hours[min_idx], max(prices) * 0.7),
            ha="center",
            arrowprops=dict(arrowstyle="->", color="red"))

plt.tight_layout()
plt.savefig("prices.png", dpi=100)
print("Saved prices.png")
plt.show()
```

## 6. What about tomorrow (and the day after)?

GridX publishes CAISO Day-Ahead Market prices as they become available (~4:30 PM PST). In practice, data is available for **today + ~2 days**. The price server fetches on startup and every poll cycle, so if you call `ven.poll_events("EELEC-013532223")`, you'll typically get events for today, tomorrow, and the day after.

Here's a version of the price-fetching code that shows today and tomorrow side-by-side when both are available:

```python
from datetime import date, timedelta
from openadr3_client import VenClient
from tabulate import tabulate

URL = "https://price.grid-coordination.energy/openadr3/3.1.0"
CIRCUIT = "EELEC-013532223"

with VenClient(url=URL, token="anonymous", user_agent="my-app/1.0") as ven:
    events = ven.poll_events(CIRCUIT)

# Event names look like "EELEC-013532223-2026-04-11" — the last three segments
# are the ISO date. Index events by that.
def event_date(e):
    return "-".join(e.event_name.rsplit("-", 3)[-3:])

by_date = {event_date(e): e for e in events}

today = date.today().isoformat()
tomorrow = (date.today() + timedelta(days=1)).isoformat()

today_event    = by_date.get(today)
tomorrow_event = by_date.get(tomorrow)

def prices(event):
    return [float(i.payloads[0].values[0]) for i in event.intervals] if event else None

t_prices = prices(today_event)
n_prices = prices(tomorrow_event)

rows = []
for hour in range(24):
    row = [f"{hour:02d}:00"]
    row.append(f"${t_prices[hour]:.5f}" if t_prices else "  —  ")
    row.append(f"${n_prices[hour]:.5f}" if n_prices else "  —  ")
    rows.append(row)

print(tabulate(rows, headers=["Hour", f"Today ({today})", f"Tomorrow ({tomorrow})"],
               tablefmt="simple"))
```

If tomorrow's data isn't yet published by GridX, the second column will be all dashes.

> **Historical prices:** The price server archives older prices going back as far as GridX has data (~August 2024 for PG&E, ~July 2025 for SCE). Historical events are available via the same `poll_events()` call — iterate over the returned list and filter by date.

## 7. Fetch GHG emissions data

The price server also publishes hourly marginal GHG emissions (MOER) for 11 California grid regions. The API works identically to pricing — just a different program name and payload type.

```python
# Find the MOER-PGE program (GHG emissions for PG&E territory)
moer_prog = ven.find_program_by_name("MOER-PGE")
print(f"GHG program: {moer_prog.program_name}")
print(f"  payload: {moer_prog.payload_descriptors[0].payload_type}")  # "GHG"
print(f"  units:   {moer_prog.payload_descriptors[0].units}")          # "g/kWh"

# Fetch today's emissions
moer_events = ven.poll_events("MOER-PGE")
moer_today = moer_events[0]

print(f"\n{moer_today.event_name}")
print(f"{'Hour':<8} {'Emissions (g CO2/kWh)'}")
print("-" * 35)
for interval in moer_today.intervals:
    hour = interval.id
    ghg = float(interval.payloads[0].values[0])
    print(f"{hour:02d}:00    {ghg:8.1f}")
```

Available MOER programs: `MOER-PGE`, `MOER-SCE`, `MOER-SDGE`, `MOER-LADWP`, `MOER-SMUD`, `MOER-BANC`, `MOER-IID`, `MOER-PACW`, `MOER-NVE`, `MOER-TID`, `MOER-WALC`.

> **Tip:** Combine price and emissions data to optimize for both cost *and* carbon. Low prices often coincide with high solar generation and low emissions (midday), while high prices and high emissions tend to occur during evening peak hours.

## Going further

- **Paginate beyond the first 50**: `ven.programs(skip=50, limit=50)`, `skip=100`, etc. Programs are grouped by tariff — PG&E rates first, then SCE, then MOER.
- **PG&E tariffs**: residential (`EELEC`), EV (`BEV1`, `BEV2P`, `BEV2S`), commercial (`B6`, `B10P`/`B10S`, `B19P`/`B19S`, `B20P`/`B20S`), agricultural (`AG-A1`, `AG-A2`, `AGBP`/`AGBS`, `AGCP`/`AGCS`) — each served across PG&E distribution feeders.
- **SCE tariffs**: residential TOU, EV, general service, public authority — each served across SCE substations. SCE prices are significantly higher (~$0.18/kWh vs PG&E's ~$0.03/kWh).
- **GHG emissions**: 11 MOER programs covering California and neighboring grid regions. Values in g CO2/kWh. See [README.md#ghg-emissions-moer](README.md#ghg-emissions-moer) for the full list.
- **Subscribe to price updates via MQTT**: the broker at `mqtt.grid-coordination.energy` supports anonymous subscribe on `openadr3/3.1.0/events/#`. See [mqtt-notifications.md](mqtt-notifications.md) for details.
- **User-Agent**: set `user_agent="your-app/1.0"` when creating the client so the server operator can identify your application in access logs.

## Full script

All examples combined as a single runnable script:

```python
#!/usr/bin/env python3
"""Read PG&E EELEC prices from the price server."""
from datetime import date, timedelta
import matplotlib.pyplot as plt
from openadr3_client import VenClient
from tabulate import tabulate

URL = "https://price.grid-coordination.energy/openadr3/3.1.0"
CIRCUIT = "EELEC-013532223"

with VenClient(url=URL, token="anonymous", user_agent="my-app/1.0") as ven:
    # 1. List programs
    programs = ven.programs()
    print(f"{len(programs)} programs (first 5):")
    for p in programs[:5]:
        print(f"  {p.program_name}")
    print()

    # 2. Look up our circuit (returns a coerced Program model)
    prog = ven.find_program_by_name(CIRCUIT)
    print(f"Found: {prog.program_name}  (id: {prog.id})")
    if prog.payload_descriptors:
        pd = prog.payload_descriptors[0]
        print(f"  {pd.payload_type} in {pd.currency}/{pd.units}")
    print()

    # 3. Fetch all events for this circuit
    events = ven.poll_events(CIRCUIT)

# Index events by ISO date (parsed from the event name)
def event_date(e):
    return "-".join(e.event_name.rsplit("-", 3)[-3:])

by_date = {event_date(e): e for e in events}

today    = date.today().isoformat()
tomorrow = (date.today() + timedelta(days=1)).isoformat()
today_ev    = by_date.get(today)
tomorrow_ev = by_date.get(tomorrow)

def hourly_prices(event):
    return [float(i.payloads[0].values[0]) for i in event.intervals] if event else None

t_prices = hourly_prices(today_ev)
n_prices = hourly_prices(tomorrow_ev)

# Today vs tomorrow table
rows = []
for h in range(24):
    row = [f"{h:02d}:00"]
    row.append(f"${t_prices[h]:.5f}" if t_prices else "  —  ")
    row.append(f"${n_prices[h]:.5f}" if n_prices else "  —  ")
    rows.append(row)

print(tabulate(rows, headers=["Hour", f"Today ({today})",
                              f"Tomorrow ({tomorrow})"], tablefmt="simple"))

if t_prices:
    print(f"\ntoday  min/max/avg: "
          f"${min(t_prices):.5f} / ${max(t_prices):.5f} / ${sum(t_prices)/24:.5f}")
if n_prices:
    print(f"tomorrow min/max/avg: "
          f"${min(n_prices):.5f} / ${max(n_prices):.5f} / ${sum(n_prices)/24:.5f}")

# Chart — today + tomorrow side by side when both are available
if t_prices:
    fig, ax = plt.subplots(figsize=(11, 5))
    width = 0.4
    hours = list(range(24))
    ax.bar([h - width/2 for h in hours], t_prices, width,
           label=f"Today ({today})", color="steelblue", edgecolor="navy")
    if n_prices:
        ax.bar([h + width/2 for h in hours], n_prices, width,
               label=f"Tomorrow ({tomorrow})", color="orange", edgecolor="darkorange")
    ax.set(xlabel="Hour of day (local)", ylabel="Price ($/kWh)",
           title=f"{CIRCUIT} — PG&E EELEC marginal price")
    ax.set_xticks(range(0, 24, 2))
    ax.grid(axis="y", alpha=0.3)
    ax.set_axisbelow(True)
    ax.legend()
    plt.tight_layout()
    plt.savefig("prices.png", dpi=100)
    print(f"\nSaved chart to prices.png")
```
