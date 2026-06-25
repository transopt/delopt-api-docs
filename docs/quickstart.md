# Delopt API — Developer Quickstart

> Interactive API available at: [api.delopt.transopt.io](https://api.delopt.transopt.io)

## Authentication

Every request (except `/health` and `/demo/optimize`) requires your API key in the header:

```
x-api-key: <your-key>
```

Contact `support@transopt.io` to get one.

---

## Basic Workflow

### 1. Submit an optimization request

Use this to create the optimal delivery plan (dispatch) based on the active orders and the available vehicles.

```
POST /optimize
```

Build a request body with three arrays (depots, vehicles, orders):

```json
{
  "depots": [
    { "id": "d-0", "lat": 37.871487, "lon": 23.7646273 }
  ],
  "vehicles": [
    { "id": "v-0", "depot_id": "d-0", "capacity": 5 }
  ],
  "orders": [
    {
      "id": "o-100",
      "lat": 37.8644013,
      "lon": 23.7505317,
      "dt_received": "2026-06-25T09:57:35Z",
      "dt_prepared": "2026-06-25T09:59:35Z"
    }
  ]
}
```

Key query params:

| Parameter | Description | Default |
|---|---|---|
| `dur_optimization` | How long the engine runs. Longer = better routes, more latency. | `PT1S` |
| `max_deliveries_per_vehicle` | Max concurrent deliveries per vehicle (1–2). | `1` |

---

The response is a `Dispatch` object with three arrays:

- **`orders`** — each order with its assigned `vehicle_id`, `delivery_id`, `dt_depart`, and `dt_arrive`
- **`vehicles`** — each vehicle's departure and return times
- **`deliveries`** — the actual delivery runs, grouping `order_ids` per vehicle trip

### 2. Single vehicle departure (`POST /depart-vehicle`)

Use this just before a specific vehicle departs. It includes the expected travel time of the vehicle and the individual arrival times for each order based on the actual departure time of the vehicle.

The request body takes a single depot, a single vehicle, and the list of orders to be delivered by the vehicle:

```json
{
  "depot": { "id": "d-0", "lat": 37.871487, "lon": 23.7646273 },
  "vehicle": { "id": "v-0", "depot_id": "d-0", "capacity": 5 },
  "orders": [
    {
      "id": "o-100",
      "lat": 37.8644013,
      "lon": 23.7505317,
      "dt_received": "2026-06-25T09:57:35Z",
      "dt_prepared": "2026-06-25T09:59:35Z"
    }
  ]
}
```

Optionally pass a `delivery_id` query param to tag the result with your own external identifier:

```
POST /depart-vehicle?delivery_id=my-delivery-123
```

The response is a `VehicleDeparture` object:

- **`vehicle`** — the vehicle's scheduled `dt_depart` and estimated `dt_return`
- **`orders`** — the orders assigned to this delivery, each with `dt_depart`, `dt_arrive`, and `delivery_id`

---

## Try it without an API key

`POST /demo/optimize` accepts the same request body but requires no authentication. Good for integration testing before you have a key.

---

## Field Reference

### Depot

| Field | Required | Description |
|---|---|---|
| `id` | ✅ | Unique identifier for the depot (e.g. `"d-0"`) |
| `lat` | ✅ | Latitude in decimal degrees (`-90` to `90`) |
| `lon` | ✅ | Longitude in decimal degrees (`-180` to `180`) |

### Vehicle

| Field | Required | Description |
|---|---|---|
| `id` | ✅ | Unique identifier for the vehicle (e.g. `"v-0"`) |
| `depot_id` | ✅ | ID of the depot where the vehicle is stationed |
| `capacity` | ✅ | Number of orders the vehicle can carry simultaneously |
| `lat` | ➖ | Current latitude of the vehicle. Defaults to depot location if omitted |
| `lon` | ➖ | Current longitude of the vehicle. Defaults to depot location if omitted |
| `dt_return` | ➖ | When the vehicle is next free. The engine will not assign a route before this time |

### Order

| Field | Required | Description |
|---|---|---|
| `id` | ✅ | Unique reference for the order (e.g. `"o-100"`) |
| `lat` | ✅ | Delivery latitude in decimal degrees |
| `lon` | ✅ | Delivery longitude in decimal degrees |
| `dt_received` | ➖ | When the order was placed. ISO 8601 timestamp |
| `dt_prepared` | ➖ | When the order is ready for pickup. The engine won't dispatch before this time |
| `dt_arrive_at` | ➖ | Hard deadline — the order should arrive by this time |
| `dur_wait_target` | ➖ | Target maximum wait time from order placement to delivery. Default: `PT15M` |
| `priority` | ➖ | If `true`, the order is treated as urgent and departs as soon as possible. Default: `false` |
| `address` | ➖ | Human-readable delivery address. Passed through to the response, not used by the engine |
| `items` | ➖ | List of item identifiers or descriptions in the order. Passed through to the response |

---

## Field Conventions

### `dt_` fields — Timestamps

All fields prefixed with `dt_` are ISO 8601 timestamps and **must include a timezone**. UTC is recommended.

```
2026-06-25T10:00:00+00:00   ✅
2026-06-25T10:00:00Z        ✅
2026-06-25T10:00:00         ❌ missing timezone
```

### `dur_` fields — Durations

All fields prefixed with `dur_` represent a duration and accept either format interchangeably:

| Format | Example | Meaning |
|---|---|---|
| ISO 8601 duration | `PT30M` | 30 minutes |
| ISO 8601 duration | `PT1H30M` | 1 hour 30 minutes |
| Plain seconds | `1800` | 30 minutes |

---

## Tips

- `dt_prepared` should be ≥ `dt_received` — the engine won't dispatch before the order is ready
- Vehicle `lat`/`lon` are optional — if omitted, the depot location is assumed as the starting point
- `dt_return` on a vehicle tells the engine the vehicle is busy until that time
- The `factor_*` fields let you tune the optimisation priorities per order — useful when certain orders are more time-sensitive or perishable than others
