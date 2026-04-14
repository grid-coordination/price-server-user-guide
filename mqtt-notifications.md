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

Subscribe to `ALL` to receive all event operations, or pick specific ones.

| Discovery endpoint | Description |
|----------|-------------|
| `/notifiers/mqtt/topics/programs` | All program notifications |
| `/notifiers/mqtt/topics/programs/{programID}` | Single program update/delete |
| `/notifiers/mqtt/topics/programs/{programID}/events` | Events for a specific program |
| `/notifiers/mqtt/topics/events` | All event notifications |

## Message format

Each message is a JSON object representing the OpenADR 3 entity that was created, updated, or deleted. For example, an event CREATE notification:

```json
{
  "objectType": "EVENT",
  "id": "b9729ed1-3748-444e-83c0-c895f27749bf",
  "programID": "b54d3d1f-bc87-4e47-bbc1-2b95958283fc",
  "eventName": "EELEC-013532223-2026-04-12",
  "intervals": [
    {"id": 0, "payloads": [{"type": "PRICE", "values": [0.02622]}]},
    {"id": 1, "payloads": [{"type": "PRICE", "values": [0.02500]}]},
    "..."
  ]
}
```

## Quick start

### mosquitto_sub (CLI)

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

### Python (paho-mqtt)

```bash
pip install paho-mqtt
```

```python
import paho.mqtt.client as mqtt
import ssl, json

def on_connect(client, userdata, flags, rc, props=None):
    print(f"Connected (rc={rc})")
    client.subscribe("OpenADR/3.1.0/events/#")

def on_message(client, userdata, msg):
    event = json.loads(msg.payload)
    print(f"{msg.topic}: {event.get('eventName', '?')}")

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
  {"OpenADR/3.1.0/events/#" 1}
  (fn [topic meta payload]
    (println topic ":" (String. payload))))
```

For TLS, use `ssl://mqtt.grid-coordination.energy:8883`.

## When are notifications published?

The price server's poll loop runs **every 60 minutes**. On each cycle it fetches the forward price window (today through today+6 days) from GridX for all 492 programs and creates or updates events for any new price data. You'll typically see a burst of notifications once per hour as fresh prices flow in.

On server startup, the initial forward-window fetch also triggers notifications. Historical backfill events are published on first creation.

## Subscription persistence

Subscriptions are session-scoped. If the broker restarts, clients need to reconnect and re-subscribe. Prices are re-published on the next poll cycle, so no data is lost — just delayed until the next hourly refresh.
