# MQTT Notifications

The price server publishes real-time OpenADR 3 notifications via MQTT. When an event is created or updated (i.e. new prices arrive), a JSON message is published to the corresponding MQTT topic. Subscribe to receive push notifications instead of polling the REST API.

## Broker

| Transport | URL | Port |
|---|---|---|
| **MQTT over TLS (recommended)** | `mqtts://mqtt.grid-coordination.energy` | 8883 |
| **Plain MQTT** | `tcp://mqtt.grid-coordination.energy` | 1883 |

The TLS endpoint uses a standard certificate — clients validate it automatically via their system trust store (no cert pinning or custom CA needed).

**No authentication is required to subscribe.** Anonymous connections can subscribe to any topic under the OpenADR 3.1.0 hierarchy.

## Discovering the broker

The broker URLs are also available programmatically via the VTN's `/notifiers` endpoint:

```bash
curl -s https://price.grid-coordination.energy/openadr3/3.1.0/notifiers | python3 -m json.tool
```

```json
{
    "WEBHOOK": false,
    "MQTT": {
        "URIS": [
            "tcp://mqtt.grid-coordination.energy:1883",
            "mqtts://mqtt.grid-coordination.energy:8883"
        ],
        "serialization": "JSON",
        "authentication": {
            "method": "ANONYMOUS"
        }
    }
}
```

## Discovering topics

Use the topic discovery endpoints to find the MQTT topics to subscribe to:

```bash
# All event topics
curl -s https://price.grid-coordination.energy/openadr3/3.1.0/notifiers/mqtt/topics/events | python3 -m json.tool

# Events for a specific program
PROGRAM_ID="<uuid>"
curl -s "https://price.grid-coordination.energy/openadr3/3.1.0/notifiers/mqtt/topics/programs/$PROGRAM_ID/events" | python3 -m json.tool

# All program topics
curl -s https://price.grid-coordination.energy/openadr3/3.1.0/notifiers/mqtt/topics/programs | python3 -m json.tool
```

Example response:

```json
{
    "topics": {
        "CREATE": "OpenADR/3.1.0/events/create",
        "UPDATE": "OpenADR/3.1.0/events/update",
        "DELETE": "OpenADR/3.1.0/events/delete",
        "ALL":    "OpenADR/3.1.0/events/+"
    }
}
```

Subscribe to `ALL` to receive all event operations, or pick specific ones. A client that follows particular programs should subscribe to each program's events topic; the global `OpenADR/3.1.0/events/+` topics carry the events of every program on the server, including [simulated demo programs](demo-programs.md).

| Discovery endpoint | Description |
|----------|-------------|
| `/notifiers/mqtt/topics/programs` | All program notifications |
| `/notifiers/mqtt/topics/programs/{programID}` | Single program update/delete |
| `/notifiers/mqtt/topics/programs/{programID}/events` | Events for a specific program |
| `/notifiers/mqtt/topics/events` | All event notifications |

## Message format

Each message is a JSON envelope: `objectType` and `operation` (`CREATE`, `UPDATE` or `DELETE`) describe the change, and `object` holds the OpenADR 3 entity as created, updated or deleted. For example, an event CREATE notification:

```json
{
  "objectType": "EVENT",
  "operation": "CREATE",
  "object": {
    "objectType": "EVENT",
    "id": "b9d3c37c-ea50-391b-bd5a-a7e583a9aa17",
    "programID": "cd63ca2d-f107-49cb-881d-6e9160b060f2",
    "eventName": "MOER-PGE-2026-10-19",
    "intervalPeriod": {"start": "2026-10-19T07:00:00Z", "duration": "PT24H"},
    "intervals": [
      {"id": 0, "intervalPeriod": {"start": "2026-10-19T07:00:00Z", "duration": "PT1H"},
       "payloads": [{"type": "GHG", "values": [440.04]}]},
      "..."
    ]
  }
}
```

## Quick start

### mosquitto_sub (CLI)

Replace `<programID>` with your program's `id` from `GET /programs?programName=<name>`.

```bash
# Plain TCP
mosquitto_sub -h mqtt.grid-coordination.energy -p 1883 \
  -t 'OpenADR/3.1.0/programs/<programID>/events/+' -v

# TLS (Linux)
mosquitto_sub -h mqtt.grid-coordination.energy -p 8883 \
  --capath /etc/ssl/certs \
  -t 'OpenADR/3.1.0/programs/<programID>/events/+' -v

# TLS (macOS)
mosquitto_sub -h mqtt.grid-coordination.energy -p 8883 \
  --cafile /etc/ssl/cert.pem \
  -t 'OpenADR/3.1.0/programs/<programID>/events/+' -v
```

### Python (paho-mqtt)

```bash
pip install paho-mqtt
```

```python
import paho.mqtt.client as mqtt
import ssl, json

PROGRAM_ID = "<programID>"

def on_connect(client, userdata, flags, rc, props=None):
    print(f"Connected (rc={rc})")
    client.subscribe(f"OpenADR/3.1.0/programs/{PROGRAM_ID}/events/+")

def on_message(client, userdata, msg):
    note = json.loads(msg.payload)
    event = note["object"]
    print(f"{msg.topic}: {note['operation']} {event.get('eventName', '?')}")

client = mqtt.Client(
    client_id="my-price-listener",
    callback_api_version=mqtt.CallbackAPIVersion.VERSION2,
)
# For TLS (port 8883):
client.tls_set(cert_reqs=ssl.CERT_REQUIRED)
client.connect("mqtt.grid-coordination.energy", 8883, 60)

# For plain TCP (port 1883), omit tls_set() and use port 1883 instead.

client.on_connect = on_connect
client.on_message = on_message
client.loop_forever()
```

### Clojure (machine_head / Paho Java)

```clojure
(require '[clojurewerkz.machine-head.client :as mh])

(def client
  (mh/connect "tcp://mqtt.grid-coordination.energy:1883"
              {:client-id "my-price-listener"}))

(mh/subscribe client
  {"OpenADR/3.1.0/programs/<programID>/events/+" 1}
  (fn [topic meta payload]
    (println topic ":" (String. payload))))
```

For TLS, use `ssl://mqtt.grid-coordination.energy:8883`.

## When are notifications published?

The price server's poll loop runs **every 60 minutes**. On each cycle it fetches the latest prices from GridX for all feed-based tariffs and creates or updates events for any new price data. In practice, CAISO Day-Ahead Market data covers today + ~2 days. You'll typically see a burst of notifications once per hour as fresh prices flow in.

On server startup, the initial forward-window fetch also triggers notifications. Historical backfill events are published on first creation.

## Subscription persistence

Subscriptions are session-scoped. If the broker restarts, clients need to reconnect and re-subscribe. Prices are re-published on the next poll cycle, so no data is lost — just delayed until the next hourly refresh.
