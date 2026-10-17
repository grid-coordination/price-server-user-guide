# Simulated demo programs

Programs whose names start with `DEMO-` carry **simulated** data for demonstrations and VEN testing. They are not real grid signals: no utility, CAISO or grid operator issued them. Every event on them has an `eventName` starting with `SIMULATED-`, and every text value starts with `SIMULATED:`.

Demo programs exist only while a demo or test is running, so most of the time `GET /programs` lists none. Like every event on the server, a demo event belongs to its program: a client that polls `/events?programID=<id>` or subscribes to `OpenADR/3.1.0/programs/<id>/events/+` for its own programs never receives demo events. They appear only on the unfiltered paths (`GET /events` without `programID`, and the global `OpenADR/3.1.0/events/+` topics), which carry every program's events.

## Programs

| Program | Payload type | Units | What it simulates |
|---|---|---|---|
| `DEMO-SIMULATED-FLEX-ALERT` | `ALERT_FLEX_ALERT` | none | A CAISO Flex Alert or Energy Emergency Alert |
| `DEMO-DOE-<site>` | `IMPORT_CAPACITY_LIMIT`, `EXPORT_CAPACITY_LIMIT`, `IMPORT_CAPACITY_SUBSCRIPTION`, `EXPORT_CAPACITY_SUBSCRIPTION` | `KW` | A dynamic operating envelope for one demo site |

Both programs set `programLongName` and a `PROGRAM_LONG_NAME` attribute that say the program is simulated.

## Simulated Flex Alert events

One program carries alerts for every CAISO region; the region is in the event's text and, by default, its `eventName`. Each event has one interval spanning the alert window, with one string value:

```json
{
  "eventName": "SIMULATED-fa-NORTHERN_CA-20261017T1600",
  "priority": 4,
  "intervalPeriod": {"start": "2026-10-17T23:00:00Z", "duration": "PT5H"},
  "intervals": [{"id": 0, "payloads": [{"type": "ALERT_FLEX_ALERT",
    "values": ["SIMULATED: not a real Flex Alert. FLEX_ALERT NORTHERN_CA 2026-10-17 16:00 PDT to 2026-10-17 21:00 PDT"]}]}]
}
```

Regions are `CAISO_GRID`, `NORTHERN_CA`, `SOUTHERN_CA` and `VEA`. The text names the simulated alert level, and `priority` follows it (a lower number is more severe):

| Alert level | `priority` |
|---|---|
| `FLEX_ALERT` | 4 |
| `EEA_WATCH` | 3 |
| `EEA_1` | 2 |
| `EEA_2` | 1 |
| `EEA_3` | 0 |

A cancelled alert is deleted.

## Simulated DOE events

A schedule is one event with consecutive intervals. Each interval has its own `intervalPeriod` and one value for each direction published (import, export or both), in kW, sent as a JSON number with a decimal point. The event's own `intervalPeriod` spans the whole schedule. A schedule may also set `randomizeStart` (for example `PT5M`) on the event's `intervalPeriod`.

```json
{
  "eventName": "SIMULATED-doe-demo1-20261017T0000",
  "priority": 1,
  "intervalPeriod": {"start": "2026-10-17T07:00:00Z", "duration": "PT1H"},
  "intervals": [
    {"id": 0, "intervalPeriod": {"start": "2026-10-17T07:00:00Z", "duration": "PT30M"},
     "payloads": [{"type": "IMPORT_CAPACITY_LIMIT", "values": [10.0]},
                  {"type": "EXPORT_CAPACITY_LIMIT", "values": [4.0]}]},
    {"id": 1, "intervalPeriod": {"start": "2026-10-17T07:30:00Z", "duration": "PT30M"},
     "payloads": [{"type": "IMPORT_CAPACITY_LIMIT", "values": [8.0]},
                  {"type": "EXPORT_CAPACITY_LIMIT", "values": [6.0]}]}
  ]
}
```

A DOE program can also carry a fallback event: one long interval with either `*_CAPACITY_SUBSCRIPTION` values (the site's subscribed capacity) or `*_CAPACITY_LIMIT` values with no `priority`. In OpenADR 3 a lower `priority` number is a higher priority. Schedules carry `priority` 1 by default and fallbacks carry none, so a VEN that ranks an event without a priority below any numbered event applies the schedule where the two overlap.

## Finding and following them

```bash
curl -s "https://price.grid-coordination.energy/openadr3/3.1.0/programs?programName=DEMO-SIMULATED-FLEX-ALERT"
curl -s "https://price.grid-coordination.energy/openadr3/3.1.0/events?programID=<id>"
```

Without `dateStart`/`dateEnd`, `/events` returns only events that overlap the next 2 days. Events are polled and pushed the same way as price events. Over MQTT, subscribe to `OpenADR/3.1.0/programs/<id>/events/+` for one demo program; see [mqtt-notifications.md](mqtt-notifications.md). Event times come back in UTC.
