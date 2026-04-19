# Tutorial: reading prices with clj-oa3-client

This tutorial walks you through installing [clj-oa3-client](https://github.com/grid-coordination/clj-oa3-client), connecting it to the live price server, and reading PG&E EELEC prices.

**Estimated time:** 5 minutes. You'll need Java 21+ and the [Clojure CLI](https://clojure.org/guides/install_clojure).

## What you'll build

By the end, you'll have a REPL session that:

1. Lists all programs in the price server (one per PG&E circuit)
2. Looks up a specific circuit's program
3. Fetches today's 24 hourly prices and prints a table
4. Renders a Vega-Lite chart of those prices in the browser

## The price server

A public OpenADR 3.1.0 VTN serving hourly PG&E EELEC marginal prices from the CAISO Day-Ahead Market, published via [GridX](https://www.gridx.com/). One program per PG&E distribution circuit (~59 circuits). Base URL:

```
https://price.grid-coordination.energy/openadr3/3.1.0
```

No authentication is required.

## 1. Set up a project

Create a new directory and a minimal `deps.edn`:

```bash
mkdir pge-prices && cd pge-prices
```

```clojure
;; deps.edn
{:deps {energy.grid-coordination/clj-oa3-client {:mvn/version "0.3.3"}
        metosin/oz                              {:mvn/version "2.0.0-alpha5"}}
 :aliases
 {:repl {:main-opts ["-m" "nrepl.cmdline"]
         :extra-deps {nrepl/nrepl {:mvn/version "1.6.0"}}}}}
```

> [oz](https://github.com/metasoarous/oz) is a Clojure wrapper around [Vega-Lite](https://vega.github.io/vega-lite/) for interactive charts. It's optional — used only for the chart at the end.

Start a REPL:

```bash
clj
```

## 2. Connect and list programs

```clojure
(require '[com.stuartsierra.component :as component]
         '[openadr3.client.ven :as ven]
         '[openadr3.client.base :as base])

(def URL "https://price.grid-coordination.energy/openadr3/3.1.0")

(def my-ven
  (component/start
    (ven/ven-client {:url        URL
                     :token      "anonymous"
                     :user-agent "my-app/1.0 (contact@example.com)"})))
```

The `:user-agent` parameter identifies your application in the server's access logs. It's optional but encouraged.

Now list the programs. The Clojure client returns coerced entity maps with namespaced keywords:

```clojure
(def programs (base/programs my-ven))

(count programs)
;=> 50

(->> programs
     (take 5)
     (mapv :openadr.program/name))
;=> ["EELEC-012041131" "EELEC-012640401" "EELEC-013532223"
;    "EELEC-013921103" "EELEC-014052103"]
```

> **Note:** You'll see 50 programs per page (the OpenADR 3 pagination limit). There are 492 total — 6 PG&E rates × 59 circuits + 3 SCE rates × 46 substations. PG&E program names follow the pattern `EELEC-<9-digit-circuit-id>`; SCE programs use `TOU-PRIME-<substation-name>`.

A simple tabular print:

```clojure
(defn print-table [headers rows]
  (let [widths (mapv (fn [i] (apply max (count (nth headers i))
                                        (map #(count (str (nth % i))) rows)))
                     (range (count headers)))
        fmt    (fn [row] (apply str (interpose "  "
                                     (map (fn [w v] (format (str "%-" w "s") (str v)))
                                          widths row))))]
    (println (fmt headers))
    (println (apply str (interpose "  " (map #(apply str (repeat % "-")) widths))))
    (doseq [r rows] (println (fmt r)))))

(print-table
  ["Program" "ID"]
  (->> (take 5 programs)
       (mapv (juxt :openadr.program/name :openadr/id))))
```

Output:

```
Program          ID
---------------  ------------------------------------
EELEC-012041131  2d3e1839-f8cb-4a1b-aa6c-c1168297d5cc
EELEC-012640401  6711667c-2b1f-42ad-ae08-c118f41dd0f3
EELEC-013532223  b54d3d1f-bc87-4e47-bbc1-2b95958283fc
EELEC-013921103  98e3c431-e023-42cf-ae5c-ad5481bf2cd3
EELEC-014052103  ee0efa62-f407-44a5-98b2-b6da0fe7fc97
```

## 3. Look up a specific circuit

The client has a `resolve-program-id` helper that caches name→ID lookups:

```clojure
(def pid (ven/resolve-program-id my-ven "EELEC-013532223"))
pid
;=> "b54d3d1f-bc87-4e47-bbc1-2b95958283fc"
```

## 4. Fetch today's hourly prices

```clojure
(def events (ven/poll-events my-ven {:program-id pid}))

(count events)
;=> 1
```

There's one event per circuit per day. Each event has 24 intervals (one per hour):

```clojure
(def event (first events))

(keys event)
;=> (:openadr/id :openadr/created :openadr/modified :openadr/object-type
;    :openadr.event/program-id :openadr.event/name :openadr.event/intervals)

(:openadr.event/name event)
;=> "EELEC-013532223-2026-04-11"

(count (:openadr.event/intervals event))
;=> 24
```

Extract hour/price pairs:

```clojure
(defn interval-price [interval]
  (get-in interval [:openadr.interval/payloads 0 :openadr.payload/values 0]))

(def hourly
  (->> (:openadr.event/intervals event)
       (mapv (fn [i] [(:openadr.interval/id i) (interval-price i)]))))

(take 3 hourly)
;=> ([0 0.030244M] [1 0.030244M] [2 0.028550M])
```

Note the prices are `BigDecimal`s — the Clojure client preserves exact decimal precision through the whole pipeline.

Print a formatted table:

```clojure
(print-table
  ["Hour" "Price/kWh"]
  (mapv (fn [[h p]] [(format "%02d:00" h)
                     (format "$%.5f" (double p))])
        hourly))

;; Hour   Price/kWh
;; -----  ---------
;; 00:00  $0.03024
;; 01:00  $0.03024
;; 02:00  $0.02855
;; ...
;; 13:00  $0.01239
;; 14:00  $0.01265
;; ...

(let [prices (mapv (comp double second) hourly)]
  {:min (apply min prices)
   :max (apply max prices)
   :avg (/ (reduce + prices) (count prices))})
;=> {:min 0.01239, :max 0.03365, :avg 0.023828}
```

The classic "duck curve": prices bottom out midday when abundant solar floods the CAISO grid, peak in the early morning.

## 5. Chart the prices with oz

[oz](https://github.com/metasoarous/oz) wraps Vega-Lite and serves charts to a live browser window:

```clojure
(require '[oz.core :as oz])

(oz/start-server!)
;; => server starts and a browser tab opens at http://localhost:10666

(def chart
  {:data     {:values (mapv (fn [[h p]] {:hour h :price (double p)}) hourly)}
   :mark     {:type "bar" :color "steelblue"}
   :encoding {:x {:field "hour"  :type "ordinal" :title "Hour of day"}
              :y {:field "price" :type "quantitative"
                  :title "Price ($/kWh)"}
              :tooltip [{:field "hour"  :type "ordinal"}
                        {:field "price" :type "quantitative" :format ".5f"}]}
   :title    (:openadr.event/name event)
   :width    700
   :height   350})

(oz/view! chart)
```

The browser tab updates live. Hover over any bar to see the exact price.

If you've loaded both today and tomorrow (see section 7 below), you can render them as a grouped bar chart in long format:

```clojure
(def combined-data
  (concat
    (map-indexed (fn [h p] {:hour h :price p :day "today"})    (or t-prices []))
    (map-indexed (fn [h p] {:hour h :price p :day "tomorrow"}) (or n-prices []))))

(oz/view!
  {:data     {:values combined-data}
   :mark     {:type "bar"}
   :encoding {:x       {:field "hour"  :type "ordinal" :title "Hour of day"}
              :y       {:field "price" :type "quantitative" :title "Price ($/kWh)"}
              :xOffset {:field "day"   :type "nominal"}
              :color   {:field "day"   :type "nominal"
                        :scale {:domain ["today" "tomorrow"]
                                :range  ["steelblue" "orange"]}}}
   :title    (str "EELEC-013532223 — " today-iso " vs " tomorrow-iso)
   :width    800
   :height   350})
```

## 6. Clean up

```clojure
(component/stop my-ven)
(oz/stop-server!)
```

## 7. Tomorrow (and the day after)

GridX publishes day-ahead and week-ahead prices as they become available (~4:30 PM PST for day-ahead). The price server fetches a **7-day forward window** — today plus the next 6 days — on startup and every poll cycle. So a single `ven/poll-events` call typically returns events covering today, tomorrow, and several days into the future.

Here's a version that groups the events by date and prints today and tomorrow side by side:

```clojure
(require '[tick.core :as t])

(defn event-date
  "Extract the ISO date from an event name like EELEC-013532223-2026-04-11."
  [e]
  (let [parts (clojure.string/split (:openadr.event/name e) #"-")
        n     (count parts)]
    (clojure.string/join "-" (drop (- n 3) parts))))

(def events (ven/poll-events my-ven {:program-id pid}))

(def by-date (into {} (map (juxt event-date identity)) events))

(def today-iso    (str (t/today)))
(def tomorrow-iso (str (t/tomorrow)))

(def today-event    (get by-date today-iso))
(def tomorrow-event (get by-date tomorrow-iso))

(defn hourly-prices [event]
  (when event
    (->> (:openadr.event/intervals event)
         (mapv interval-price)
         (mapv double))))

(def t-prices (hourly-prices today-event))
(def n-prices (hourly-prices tomorrow-event))

;; Today vs tomorrow table
(print-table
  ["Hour" (str "Today (" today-iso ")") (str "Tomorrow (" tomorrow-iso ")")]
  (mapv (fn [h]
          [(format "%02d:00" h)
           (if t-prices (format "$%.5f" (nth t-prices h)) "  —  ")
           (if n-prices (format "$%.5f" (nth n-prices h)) "  —  ")])
        (range 24)))
```

If tomorrow's data hasn't been published yet (which sometimes happens early in the day before GridX posts the day-ahead run), `tomorrow-event` will be nil and the third column will be all dashes.

> **Historical prices:** The price server archives older prices going back as far as GridX has data (~August 2024 for PG&E, ~July 2025 for SCE). Historical events come back in the same `poll-events` call — filter `by-date` by the dates you want.

## Going further

- **All 503 programs**: paginate via the raw API: `(base/search-programs my-ven {:skip 50})`, `{:skip 100}`, etc.
- **Other PG&E rates**: `BEV1` (commercial EV), `BEV2P`/`BEV2S`, `B6` (small commercial), `B19P` (medium commercial primary). Same 59 circuits per rate.
- **SCE rates**: `TOU-PRIME`, `TOU-D-49`, `TOU-D-58` — 46 substations each. SCE prices are ~$0.18/kWh vs PG&E's ~$0.03/kWh.
- **GHG emissions**: `MOER-PGE`, `MOER-SCE`, `MOER-SDGE`, and 8 other regional programs publish hourly marginal GHG emissions in g CO2/kWh. Fetch the same way as prices: `(ven/poll-events my-ven {:program-id (ven/resolve-program-id my-ven "MOER-PGE")})`. See [README.md#ghg-emissions-moer](README.md#ghg-emissions-moer).
- **Raw HTTP access**: `base/programs` returns coerced entities. For the raw wire format use `base/get-programs` (returns the HTTP response map).
- **Subscribe to price updates via MQTT**: the broker at `mqtt.grid-coordination.energy` supports anonymous subscribe on `openadr3/3.1.0/events/#`. See [mqtt-notifications.md](mqtt-notifications.md). Use `ven/add-mqtt` and `ven/subscribe`.
- **User-Agent**: set `:user-agent "your-app/1.0"` when creating the client so the server operator can identify your application in access logs.

## Full script

Copy this into a file `prices.clj` and run with `clj prices.clj`:

```clojure
(require '[com.stuartsierra.component :as component]
         '[openadr3.client.ven :as ven]
         '[openadr3.client.base :as base]
         '[tick.core :as t]
         '[oz.core :as oz])

(def URL "https://price.grid-coordination.energy/openadr3/3.1.0")
(def CIRCUIT "EELEC-013532223")

(defn interval-price [i]
  (get-in i [:openadr.interval/payloads 0 :openadr.payload/values 0]))

(defn event-date
  "Extract the ISO date from an event name like EELEC-013532223-2026-04-11."
  [e]
  (let [parts (clojure.string/split (:openadr.event/name e) #"-")
        n     (count parts)]
    (clojure.string/join "-" (drop (- n 3) parts))))

(defn hourly-prices [event]
  (when event
    (->> (:openadr.event/intervals event)
         (mapv interval-price)
         (mapv double))))

(let [my-ven (component/start (ven/ven-client {:url URL :token "anonymous"
                                                :user-agent "my-app/1.0"}))]
  (try
    ;; 1. List programs
    (let [progs (base/programs my-ven)]
      (println (count progs) "programs (first 5):")
      (doseq [p (take 5 progs)]
        (println " " (:openadr.program/name p)))
      (println))

    ;; 2. Resolve a circuit and poll events
    (let [pid      (ven/resolve-program-id my-ven CIRCUIT)
          events   (ven/poll-events my-ven {:program-id pid})
          by-date  (into {} (map (juxt event-date identity)) events)
          today    (str (t/today))
          tomorrow (str (t/tomorrow))
          t-event  (get by-date today)
          n-event  (get by-date tomorrow)
          t-prices (hourly-prices t-event)
          n-prices (hourly-prices n-event)]
      (println "Found:" CIRCUIT " (id:" pid ")")
      (println (count events) "events available")
      (println)

      ;; 3. Today vs tomorrow table
      (println (format "  Hour   %-22s %-22s"
                       (str "Today (" today ")")
                       (str "Tomorrow (" tomorrow ")")))
      (println "  -----  ----------------------  ----------------------")
      (doseq [h (range 24)]
        (println (format "  %02d:00  %-22s %-22s"
                         h
                         (if t-prices (format "$%.5f" (nth t-prices h)) "  —")
                         (if n-prices (format "$%.5f" (nth n-prices h)) "  —"))))

      (when t-prices
        (println)
        (println "today  min/max/avg:"
                 (format "$%.5f / $%.5f / $%.5f"
                         (apply min t-prices)
                         (apply max t-prices)
                         (/ (reduce + t-prices) 24.0))))
      (when n-prices
        (println "tmrrw  min/max/avg:"
                 (format "$%.5f / $%.5f / $%.5f"
                         (apply min n-prices)
                         (apply max n-prices)
                         (/ (reduce + n-prices) 24.0))))

      ;; 4. Grouped bar chart — today + tomorrow
      (when (or t-prices n-prices)
        (oz/start-server!)
        (oz/view!
          {:data     {:values (concat
                                (when t-prices
                                  (map-indexed (fn [h p] {:hour h :price p :day "today"})
                                               t-prices))
                                (when n-prices
                                  (map-indexed (fn [h p] {:hour h :price p :day "tomorrow"})
                                               n-prices)))}
           :mark     {:type "bar"}
           :encoding {:x       {:field "hour"  :type "ordinal" :title "Hour of day"}
                      :y       {:field "price" :type "quantitative" :title "Price ($/kWh)"}
                      :xOffset {:field "day"   :type "nominal"}
                      :color   {:field "day"   :type "nominal"
                                :scale {:domain ["today" "tomorrow"]
                                        :range  ["steelblue" "orange"]}}}
           :title    (str CIRCUIT " — " today " vs " tomorrow)
           :width 800 :height 350})
        (println "\nChart rendered at http://localhost:10666")))

    (finally
      (component/stop my-ven))))
```
