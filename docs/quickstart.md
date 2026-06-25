# Delopt API — Developer Quickstart

## Authentication

Every request (except `/health` and `/demo/optimize`) requires your API key in the header:

```
x-api-key: <your-key>
```

Contact `support@transopt.io` to get one.

---

## Basic Workflow

### 1. Check your depot configuration

```
GET /depots/
```

Returns your depot(s) — note the `depot_location` (lat/lon) and your `max_vehicles` limit. You'll need the depot coordinates for the next step.

### 2. Submit an optimization request

```
POST /optimize
```

Build a request body with three arrays:

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
| `dur_optimization` | How long the engine runs (e.g. `PT5S`). Longer = better routes, more latency. | `PT1S` |
| `max_deliveries_per_vehicle` | Max concurrent deliveries per vehicle (1–2). | `1` |

### 3. Read the response

The response is a `Dispatch` object with three arrays:

- **`orders`** — each order with its assigned `vehicle_id`, `delivery_id`, `dt_depart`, and `dt_arrive`
- **`vehicles`** — each vehicle's departure and return times
- **`deliveries`** — the actual delivery runs, grouping `order_ids` per vehicle trip

---

## Try it without an API key

`POST /demo/optimize` accepts the same request body but requires no authentication. Good for integration testing before you have a key.

---

## Tips

- All timestamps are ISO 8601 with timezone (e.g. `2026-06-25T10:00:00+00:00`)
- `dt_prepared` should be ≥ `dt_received`; the engine won't dispatch before the order is ready
- Vehicle `lat`/`lon` are optional — if omitted, the depot location is assumed as the starting point
- `dt_return` on a vehicle tells the engine the vehicle is busy until that time
