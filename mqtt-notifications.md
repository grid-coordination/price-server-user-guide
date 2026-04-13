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

## Topics

The VTN publishes to the standard OpenADR 3.1.0 topic hierarchy:

```
openadr3/3.1.0/programs/CREATE
openadr3/3.1.0/programs/UPDATE
openadr3/3.1.0/programs/DELETE
openadr3/3.1.0/events/CREATE
openadr3/3.1.0/events/UPDATE
openadr3/3.1.0/events/DELETE
openadr3/3.1.0/subscriptions/CREATE
openadr3/3.1.0/subscriptions/UPDATE
openadr3/3.1.0/subscriptions/DELETE
```

To receive all notifications:

```
openadr3/3.1.0/#
```

To receive only event notifications (price updates):

```
openadr3/3.1.0/events/#
```

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
  -t 'openadr3/3.1.0/events/#' -v

# TLS (Linux)
mosquitto_sub -h mqtt.grid-coordination.energy -p 8883 \
  --capath /etc/ssl/certs \
  -t 'openadr3/3.1.0/events/#' -v

# TLS (macOS)
mosquitto_sub -h mqtt.grid-coordination.energy -p 8883 \
  --cafile /etc/ssl/cert.pem \
  -t 'openadr3/3.1.0/events/#' -v
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
    client.subscribe("openadr3/3.1.0/events/#")

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
  {"openadr3/3.1.0/events/#" 1}
  (fn [topic meta payload]
    (println topic ":" (String. payload))))
```

For TLS, use `ssl://mqtt.grid-coordination.energy:8883`.

## When are notifications published?

The price server's poll loop runs **every 60 minutes**. On each cycle it fetches the forward price window (today through today+6 days) from GridX for all 492 programs and creates or updates events for any new price data. You'll typically see a burst of notifications once per hour as fresh prices flow in.

On server startup, the initial forward-window fetch also triggers notifications. Historical backfill events are published on first creation.

## Subscription persistence

Subscriptions are session-scoped. If the broker restarts, clients need to reconnect and re-subscribe. Prices are re-published on the next poll cycle, so no data is lost — just delayed until the next hourly refresh.
