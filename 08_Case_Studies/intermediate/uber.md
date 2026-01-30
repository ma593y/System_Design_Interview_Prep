# Uber / Lyft Ride-Sharing System Design

## 1. Problem Statement

Design a ride-sharing platform like Uber or Lyft that connects riders with nearby drivers
in real-time. The system must handle millions of concurrent rides, track driver locations
continuously, match riders to optimal drivers, calculate dynamic pricing, and process
payments -- all with low latency and high reliability.

**Core Functionality:**
- Riders request rides by specifying pickup and dropoff locations
- System matches riders with the best available nearby driver
- Real-time tracking of driver location during the ride
- Dynamic (surge) pricing based on supply and demand
- Trip lifecycle management (request, match, pickup, ride, dropoff, payment)
- ETA calculation for driver arrival and trip duration

**Real-World Examples:**
- Uber
- Lyft
- Grab (Southeast Asia)
- Ola (India)
- DiDi (China)

---

## 2. Requirements Clarification

### Functional Requirements

| Requirement | Description |
|-------------|-------------|
| Ride Request | Rider specifies pickup/dropoff, sees ETA and price |
| Driver Matching | Match rider with optimal nearby driver |
| Real-Time Tracking | Track driver location, show on rider's map |
| Pricing | Calculate fare with distance, time, surge multiplier |
| Trip Management | Full lifecycle: request -> match -> ride -> complete |
| ETA Calculation | Estimate arrival time and trip duration |
| Payment | Process payment at trip end (card, wallet) |
| Rating | Rider and driver rate each other post-trip |

### Non-Functional Requirements

| Requirement | Target |
|-------------|--------|
| High Availability | 99.99% uptime |
| Low Latency | Match a driver within 30 seconds |
| Real-Time Updates | Location updates every 3-4 seconds |
| Scalability | Support 20M daily rides, 5M concurrent drivers |
| Consistency | Trip and payment data must be strongly consistent |
| Global | Operate in 70+ countries with localized behavior |

### Out of Scope
- Uber Eats / food delivery
- Autonomous vehicle integration
- Driver onboarding and background checks
- Regulatory compliance per jurisdiction

---

## 3. Capacity Estimation

### Assumptions

```
- 100 million registered riders
- 5 million registered drivers
- 20 million rides per day
- 2 million concurrent active drivers at peak
- Driver sends location update every 4 seconds
- Average ride duration: 15 minutes
- Average ETA requests before ride: 3 (user checks price/ETA)
```

### Traffic Estimates

```
Ride Requests:
- Rides per day: 20 million
- Rides per second: 20M / 86,400 ≈ 231 rides/sec
- Peak (3x): ~700 rides/sec

Location Updates (from drivers):
- Active drivers: 2 million
- Update frequency: every 4 seconds
- Location updates per second: 2M / 4 = 500,000 updates/sec
- This is the HIGHEST throughput component

ETA Requests:
- 20M rides/day * 3 ETA checks = 60M ETA requests/day
- ETA requests per second: 60M / 86,400 ≈ 694 req/sec
- Peak: ~2,100 req/sec

Matching Operations:
- Same as ride requests: ~231/sec, peak ~700/sec
- Each match queries nearby drivers: ~50 candidates evaluated
```

### Storage Estimates

```
Location data (hot, in-memory):
- Per driver location: 8 bytes lat + 8 bytes lng + 8 bytes driver_id
  + 8 bytes timestamp = 32 bytes
- 2M active drivers * 32 bytes = 64 MB (fits in memory easily)

Trip history (persistent):
- Per trip record: ~1 KB (route polyline, fare, timestamps, etc.)
- Daily: 20M trips * 1 KB = 20 GB/day
- Annual: 20 GB * 365 = 7.3 TB/year

Location history (for analytics):
- 500K updates/sec * 32 bytes * 86,400 sec = 1.38 TB/day
- Compressed and archived after 7 days
- Keep 7 days hot: ~9.7 TB
```

### Bandwidth Estimates

```
Location updates (inbound from drivers):
- 500,000 updates/sec * 32 bytes = 16 MB/s inbound

Location broadcasts (outbound to riders tracking):
- ~1M active rides, each rider gets driver location every 4 sec
- 1M / 4 * 32 bytes = 8 MB/s outbound

Map tile requests:
- Served from CDN, not counting toward backend bandwidth
```

---

## 4. High-Level Design

```
┌──────────────────────────────────────────────────────────────────────┐
│                     Clients (Rider App / Driver App)                 │
└──────────┬──────────────────────────────────┬────────────────────────┘
           │ REST/HTTP                         │ WebSocket
           │ (ride request, ETA)               │ (location updates,
           │                                   │  real-time tracking)
┌──────────▼──────────┐             ┌──────────▼──────────┐
│   API Gateway       │             │  WebSocket Gateway  │
│   (REST endpoints)  │             │  (persistent conn)  │
└──────────┬──────────┘             └──────────┬──────────┘
           │                                   │
     ┌─────┼─────────────────┬─────────────────┼──────────┐
     │     │                 │                 │          │
┌────▼───┐ │ ┌──────────┐ ┌─▼──────────┐ ┌────▼───┐ ┌────▼───────┐
│ Ride   │ │ │ Pricing  │ │ Location   │ │ Trip   │ │ Payment    │
│Request │ │ │ Service  │ │ Service    │ │Service │ │ Service    │
│Service │ │ │          │ │            │ │        │ │            │
└───┬────┘ │ └────┬─────┘ └─────┬──────┘ └───┬────┘ └────┬───────┘
    │      │      │             │             │           │
    │  ┌───▼──────▼──┐   ┌─────▼──────┐      │           │
    │  │  Matching   │   │ Geospatial │      │           │
    │  │  Service    │   │ Index      │      │           │
    │  │             │   │ (QuadTree/ │      │           │
    │  │             │   │  Geohash)  │      │           │
    │  └─────────────┘   └────────────┘      │           │
    │                                        │           │
┌───▼────────────────────────────────────────▼───────────▼──────────┐
│                       Data Layer                                   │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────────┐  │
│  │  Redis   │  │ Postgres │  │  Kafka   │  │  Map/Routing     │  │
│  │(Location │  │ (Trips,  │  │ (Events, │  │  Service         │  │
│  │  Cache)  │  │  Users,  │  │  Logs)   │  │  (OSRM/Google)   │  │
│  │          │  │  Payments│  │          │  │                  │  │
│  └──────────┘  └──────────┘  └──────────┘  └──────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
```

### Ride Request Flow (End-to-End)

```
Rider                   System                          Driver
  │                       │                               │
  │ 1. Request Ride       │                               │
  │   (pickup, dropoff)   │                               │
  │──────────────────────▶│                               │
  │                       │                               │
  │                       │ 2. Calculate price & ETA      │
  │                       │    (Pricing + Routing)        │
  │                       │                               │
  │ 3. Show price & ETA   │                               │
  │◀──────────────────────│                               │
  │                       │                               │
  │ 4. Confirm ride       │                               │
  │──────────────────────▶│                               │
  │                       │                               │
  │                       │ 5. Find nearby drivers        │
  │                       │    (Geospatial query)         │
  │                       │                               │
  │                       │ 6. Rank & select best driver  │
  │                       │    (Matching algorithm)       │
  │                       │                               │
  │                       │ 7. Send ride offer            │
  │                       │───────────────────────────────▶│
  │                       │                               │
  │                       │ 8. Driver accepts             │
  │                       │◀───────────────────────────────│
  │                       │                               │
  │ 9. Driver matched!    │                               │
  │   (driver info, ETA)  │                               │
  │◀──────────────────────│                               │
  │                       │                               │
  │ 10. Real-time tracking│  Location updates every 4s    │
  │◀ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ │◀ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─│
  │                       │                               │
  │ 11. Driver arrives    │ 11. Arrived at pickup         │
  │◀──────────────────────│◀───────────────────────────────│
  │                       │                               │
  │                       │ 12. Trip started              │
  │                       │◀───────────────────────────────│
  │                       │                               │
  │                       │ 13. Trip ended                │
  │                       │◀───────────────────────────────│
  │                       │                               │
  │                       │ 14. Calculate final fare      │
  │                       │     Process payment           │
  │                       │                               │
  │ 15. Receipt + Rating  │  15. Earnings + Rating        │
  │◀──────────────────────│───────────────────────────────▶│
```

---

## 5. API Design

### Ride Request

```
POST /api/v1/rides/estimate
Headers:
  Authorization: Bearer <token>
Body:
{
  "pickup": {"lat": 37.7749, "lng": -122.4194},
  "dropoff": {"lat": 37.3382, "lng": -121.8863},
  "ride_type": "UberX",
  "payment_method_id": "pm_12345"
}

Response 200:
{
  "estimate_id": "est_abc123",
  "ride_type": "UberX",
  "estimated_fare": {
    "min": 28.50,
    "max": 35.00,
    "currency": "USD",
    "surge_multiplier": 1.3
  },
  "estimated_pickup_eta_minutes": 4,
  "estimated_trip_duration_minutes": 42,
  "estimated_distance_miles": 35.2,
  "fare_breakdown": {
    "base_fare": 2.50,
    "distance_charge": 18.70,
    "time_charge": 8.40,
    "surge_charge": 5.50,
    "booking_fee": 2.75
  },
  "expires_at": "2024-01-25T10:35:00Z"
}
```

### Confirm Ride

```
POST /api/v1/rides
Headers:
  Authorization: Bearer <token>
Body:
{
  "estimate_id": "est_abc123",
  "pickup": {"lat": 37.7749, "lng": -122.4194, "address": "123 Market St"},
  "dropoff": {"lat": 37.3382, "lng": -121.8863, "address": "456 Santa Clara St"},
  "ride_type": "UberX",
  "payment_method_id": "pm_12345"
}

Response 201:
{
  "ride_id": "ride_xyz789",
  "status": "matching",
  "created_at": "2024-01-25T10:30:00Z"
}
```

### Driver Location Update

```
POST /api/v1/drivers/location
Headers:
  Authorization: Bearer <driver_token>
Body:
{
  "lat": 37.7752,
  "lng": -122.4180,
  "heading": 45,
  "speed": 25.5,
  "accuracy": 5.0,
  "timestamp": 1706177400000
}

Response 200:
{
  "status": "ok"
}
```

### Trip Status Updates (WebSocket)

```
WebSocket: wss://ws.uber.com/v1/rides/{ride_id}/track

Server -> Client messages:
{
  "type": "driver_location",
  "data": {
    "lat": 37.7752,
    "lng": -122.4180,
    "heading": 45,
    "eta_minutes": 3,
    "updated_at": "2024-01-25T10:31:00Z"
  }
}

{
  "type": "status_change",
  "data": {
    "ride_id": "ride_xyz789",
    "status": "driver_arrived",
    "timestamp": "2024-01-25T10:34:00Z"
  }
}

{
  "type": "trip_completed",
  "data": {
    "ride_id": "ride_xyz789",
    "final_fare": 31.25,
    "distance_miles": 34.8,
    "duration_minutes": 40,
    "route_polyline": "encoded_polyline_string"
  }
}
```

---

## 6. Database Schema

### Rides Table (PostgreSQL, sharded by city)

```
Table: rides
┌─────────────────────┬──────────────┬──────────────────────────────┐
│ Column              │ Type         │ Notes                        │
├─────────────────────┼──────────────┼──────────────────────────────┤
│ ride_id             │ UUID (PK)    │ Globally unique              │
│ rider_id            │ BIGINT       │ FK to users                  │
│ driver_id           │ BIGINT       │ FK to users (null until match│
│ ride_type           │ VARCHAR(16)  │ UberX, UberXL, Black, etc.   │
│ status              │ VARCHAR(16)  │ matching/accepted/arrived/   │
│                     │              │ in_progress/completed/       │
│                     │              │ cancelled                    │
│ pickup_lat          │ DECIMAL(9,6) │ Pickup latitude              │
│ pickup_lng          │ DECIMAL(9,6) │ Pickup longitude             │
│ pickup_address      │ VARCHAR(256) │ Human-readable address       │
│ dropoff_lat         │ DECIMAL(9,6) │ Dropoff latitude             │
│ dropoff_lng         │ DECIMAL(9,6) │ Dropoff longitude            │
│ dropoff_address     │ VARCHAR(256) │ Human-readable address       │
│ estimated_fare      │ DECIMAL(10,2)│ Pre-trip estimate            │
│ actual_fare         │ DECIMAL(10,2)│ Final calculated fare        │
│ surge_multiplier    │ DECIMAL(3,2) │ Surge pricing factor         │
│ distance_miles      │ DECIMAL(8,2) │ Actual trip distance         │
│ duration_seconds    │ INT          │ Actual trip duration         │
│ city_id             │ INT          │ City identifier (shard key)  │
│ payment_method_id   │ VARCHAR(64)  │ Payment method used          │
│ route_polyline      │ TEXT         │ Encoded route path           │
│ requested_at        │ TIMESTAMP    │ When ride was requested      │
│ matched_at          │ TIMESTAMP    │ When driver was matched      │
│ started_at          │ TIMESTAMP    │ When trip started            │
│ completed_at        │ TIMESTAMP    │ When trip ended              │
│ cancelled_at        │ TIMESTAMP    │ When/if cancelled            │
│ cancellation_reason │ VARCHAR(64)  │ Why cancelled                │
└─────────────────────┴──────────────┴──────────────────────────────┘
Indexes:
  - PRIMARY KEY (ride_id)
  - INDEX idx_rider (rider_id, requested_at DESC)
  - INDEX idx_driver (driver_id, requested_at DESC)
  - INDEX idx_status (city_id, status)
  - INDEX idx_city_time (city_id, requested_at DESC)
```

### Drivers Table

```
Table: drivers
┌─────────────────────┬──────────────┬──────────────────────────────┐
│ Column              │ Type         │ Notes                        │
├─────────────────────┼──────────────┼──────────────────────────────┤
│ driver_id           │ BIGINT (PK)  │ Unique driver ID             │
│ user_id             │ BIGINT       │ FK to users                  │
│ vehicle_id          │ BIGINT       │ FK to vehicles               │
│ license_number      │ VARCHAR(32)  │ Driving license              │
│ status              │ VARCHAR(16)  │ online/offline/on_trip        │
│ city_id             │ INT          │ Primary operating city       │
│ rating              │ DECIMAL(2,1) │ Average rating               │
│ total_rides         │ INT          │ Completed ride count         │
│ ride_types[]        │ VARCHAR[]    │ Eligible ride types          │
│ created_at          │ TIMESTAMP    │ Registration date            │
└─────────────────────┴──────────────┴──────────────────────────────┘
```

### Driver Location (Redis - In-Memory Geospatial)

```
Redis Geospatial Index:
  Key: drivers:city:{city_id}
  Type: GEOADD / GEOSEARCH

  GEOADD drivers:city:42 -122.4194 37.7749 "driver_12345"
  GEOSEARCH drivers:city:42 FROMLONLAT -122.4194 37.7749 BYRADIUS 5 km ASC COUNT 20

  Additional per-driver metadata in hash:
  Key: driver:info:{driver_id}
  Fields: status, heading, speed, last_updated, ride_type, rating
```

### Surge Pricing Table

```
Table: surge_zones
┌─────────────────────┬──────────────┬──────────────────────────────┐
│ Column              │ Type         │ Notes                        │
├─────────────────────┼──────────────┼──────────────────────────────┤
│ zone_id             │ INT (PK)     │ Geohash-based zone           │
│ city_id             │ INT          │ City identifier              │
│ geohash             │ VARCHAR(8)   │ Geohash of zone center       │
│ surge_multiplier    │ DECIMAL(3,2) │ Current surge factor         │
│ demand_count        │ INT          │ Recent ride requests         │
│ supply_count        │ INT          │ Available drivers            │
│ updated_at          │ TIMESTAMP    │ Last recalculation           │
└─────────────────────┴──────────────┴──────────────────────────────┘
Updated every 2 minutes based on real-time supply/demand.
```

---

## 7. Deep Dive

### 7.1 Geospatial Indexing for Driver Location

The core challenge: given a rider's location, quickly find the nearest available drivers.

**Approach 1: Geohash**

```
Geohash encodes (lat, lng) into a string. Nearby locations share prefixes.

Example:
  San Francisco: 9q8yyk (precision 6 = ~1.2 km x 0.6 km cell)

  Geohash grid:
  ┌────────┬────────┬────────┐
  │ 9q8yym │ 9q8yyt │ 9q8yyw │
  ├────────┼────────┼────────┤
  │ 9q8yyk │ 9q8yys │ 9q8yyv │  <-- Rider is in 9q8yyk
  ├────────┼────────┼────────┤
  │ 9q8yyh │ 9q8yyn │ 9q8yyq │
  └────────┴────────┴────────┘

  To find nearby drivers:
  1. Compute rider's geohash: 9q8yyk
  2. Get 8 neighboring cells: 9q8yym, 9q8yyt, 9q8yys, ...
  3. Query all drivers in these 9 cells
  4. Sort by actual distance, return nearest N

  Redis implementation:
  - GEOADD stores drivers with coordinates
  - GEOSEARCH returns drivers within radius
  - O(log N + M) where N = total drivers, M = results

Pros: Simple, built into Redis, fast
Cons: Edge effects at cell boundaries (solved by querying neighbors)
```

**Approach 2: QuadTree**

```
QuadTree recursively divides 2D space into 4 quadrants.
Split a cell when it contains > K drivers (e.g., K = 100).

                    ┌─────────────────────┐
                    │     Root Cell        │
                    │   (entire city)      │
                    └──────────┬──────────┘
               ┌───────┬───────┼───────┬──────┐
               │       │       │       │      │
          ┌────▼──┐┌───▼──┐┌───▼──┐┌───▼──┐   │
          │  NW   ││  NE  ││  SW  ││  SE  │   │
          │(dense)││      ││      ││      │   │
          └───┬───┘└──────┘└──────┘└──────┘   │
         ┌────┼────┐                           │
    ┌────▼┐┌─▼──┐┌▼───┐┌────┐                 │
    │NW-NW││NW-NE││NW-SW││NW-SE│               │
    └─────┘└─────┘└─────┘└─────┘               │

  Dense areas (downtown) -> deeper tree (smaller cells)
  Sparse areas (suburbs) -> shallower tree (larger cells)

  Search: traverse from root, find leaf containing rider's location,
          expand to neighboring leaves until enough candidates found.

Pros: Adapts to density, efficient for non-uniform distribution
Cons: More complex to implement, harder to distribute across servers
```

**Approach 3: S2 Geometry (Google's approach)**

```
S2 maps the sphere to a cube face, then uses a Hilbert curve
to linearize 2D space into 1D (S2 Cell IDs).

Benefits:
- No edge effects (works on sphere, not flat plane)
- Hierarchical cells at 30 levels of precision
- S2 cell ID is a 64-bit integer (easy to index)
- Used by Uber's H3 hexagonal grid system

Level 12 cell ≈ 3.31 km^2 (good for urban areas)
Level 14 cell ≈ 0.21 km^2 (good for dense downtown)

Driver index:
  Key: s2cell:{cell_id}
  Value: set of driver_ids

Query:
  1. Get S2 cell covering rider's location at desired level
  2. Get neighboring cells at same level
  3. Fetch all drivers in those cells
  4. Filter by distance, rank by suitability
```

**Our Choice: Geohash in Redis + S2 for advanced queries**

```
Primary: Redis GEOSEARCH (simple, fast, handles 500K updates/sec)
Secondary: S2-based index for advanced features (polygon search,
           surge zone computation, heat maps)
```

### 7.2 Ride Matching Algorithm

```
When a rider requests a ride, the matching service must find the
optimal driver. This is NOT simply "nearest driver."

Matching Score = w1 * ProximityScore
              + w2 * ETAScore
              + w3 * DriverRatingScore
              + w4 * AcceptanceRateScore
              + w5 * RideTypeEligibility

Step-by-step matching process:

1. CANDIDATE GENERATION
   ┌──────────────────────────────────────────────────┐
   │ Query geospatial index:                          │
   │ - Radius: start at 2 km, expand to 5 km if few  │
   │ - Filter: status = "online" (not on trip)        │
   │ - Filter: ride_type eligible (UberX, Black, etc.)│
   │ - Result: ~20-50 candidate drivers               │
   └──────────────────────────────────────────────────┘

2. ETA CALCULATION (for each candidate)
   ┌──────────────────────────────────────────────────┐
   │ For each candidate driver:                       │
   │ - Query routing service for ETA to pickup point  │
   │ - Account for current traffic conditions         │
   │ - Batch API call to routing service              │
   │   (20 origins -> 1 destination)                  │
   └──────────────────────────────────────────────────┘

3. SCORING
   ┌──────────────────────────────────────────────────┐
   │ For each candidate:                              │
   │                                                  │
   │ proximity_score = 1.0 - (distance / max_radius)  │
   │ eta_score = 1.0 - (eta_minutes / 15.0)           │
   │ rating_score = driver_rating / 5.0               │
   │ acceptance_score = acceptance_rate                │
   │                                                  │
   │ total_score = 0.4 * eta_score                    │
   │             + 0.3 * proximity_score               │
   │             + 0.2 * rating_score                  │
   │             + 0.1 * acceptance_score              │
   └──────────────────────────────────────────────────┘

4. OFFER TO DRIVER
   ┌──────────────────────────────────────────────────┐
   │ Send ride offer to top-scoring driver             │
   │ - Driver has 15 seconds to accept/decline         │
   │ - If declined or timeout: offer to next driver    │
   │ - If 3 drivers decline: expand radius and retry   │
   │ - If no driver found in 60s: notify rider,        │
   │   suggest waiting or different ride type           │
   └──────────────────────────────────────────────────┘

Matching timeline:
  t=0s    Rider confirms ride
  t=0.5s  Geospatial query returns 30 candidates
  t=1.5s  ETA batch computed for all 30
  t=2.0s  Scoring complete, offer sent to driver #1
  t=5.0s  Driver #1 accepts
  t=5.5s  Rider notified: "Driver matched, ETA 4 min"
  Total: ~5 seconds average
```

### 7.3 Surge Pricing (Dynamic Pricing)

```
Surge pricing balances supply (drivers) and demand (riders) in real-time.

Supply-Demand Calculation (per zone, every 2 minutes):

  demand = count(ride_requests in zone in last 5 min)
  supply = count(available_drivers in zone)

  ratio = demand / max(supply, 1)

  Surge multiplier lookup:
  ┌───────────────┬────────────────────┐
  │ Demand/Supply │ Surge Multiplier   │
  ├───────────────┼────────────────────┤
  │ 0.0 - 1.0    │ 1.0x (no surge)    │
  │ 1.0 - 1.5    │ 1.2x               │
  │ 1.5 - 2.0    │ 1.5x               │
  │ 2.0 - 3.0    │ 1.8x               │
  │ 3.0 - 4.0    │ 2.2x               │
  │ 4.0+         │ 2.5x - 3.0x (cap)  │
  └───────────────┴────────────────────┘

Zone Definition:
  City divided into hexagonal zones (H3 grid, resolution 7)
  Each hexagon ≈ 5.16 km^2

  ┌──────┐ ┌──────┐ ┌──────┐
  │ 1.0x │ │ 1.5x │ │ 1.0x │
  │      ├─┤      ├─┤      │
  └──┬───┘ └──┬───┘ └──┬───┘
  ┌──┴───┐ ┌──┴───┐ ┌──┴───┐
  │ 1.2x │ │ 2.5x │ │ 1.8x │   <-- Downtown concert = high surge
  │      ├─┤      ├─┤      │
  └──┬───┘ └──┬───┘ └──┬───┘
  ┌──┴───┐ ┌──┴───┐ ┌──┴───┐
  │ 1.0x │ │ 1.2x │ │ 1.0x │
  │      ├─┤      ├─┤      │
  └──────┘ └──────┘ └──────┘

Architecture:
  ┌─────────────┐     ┌──────────────┐     ┌──────────────┐
  │ Ride Request│────▶│ Kafka Topic  │────▶│ Surge        │
  │ Events      │     │ (ride_events)│     │ Calculator   │
  └─────────────┘     └──────────────┘     │ (Flink job)  │
                                           └──────┬───────┘
  ┌─────────────┐                                 │
  │ Driver      │     ┌──────────────┐     ┌──────▼───────┐
  │ Location    │────▶│ Kafka Topic  │────▶│ Updates surge│
  │ Events      │     │(driver_avail)│     │ multiplier   │
  └─────────────┘     └──────────────┘     │ per zone     │
                                           └──────┬───────┘
                                                  │
                                           ┌──────▼───────┐
                                           │ Redis Cache  │
                                           │ surge:{zone} │
                                           │ = multiplier │
                                           └──────────────┘

  Pricing Service reads surge from Redis when calculating fares.
```

---

## 8. Scaling Discussion

### Location Service Scaling

```
Challenge: 500,000 location updates per second

Solution: Partitioned by city/region

  ┌─────────────────────────────────────────────────┐
  │          Location Update Router                  │
  │  (routes by city_id to correct partition)        │
  └───────────┬─────────────┬───────────────────────┘
              │             │             │
   ┌──────────▼──┐  ┌──────▼──────┐  ┌──▼──────────┐
   │ Redis Shard │  │ Redis Shard │  │ Redis Shard │
   │ NYC         │  │ SF Bay Area │  │ London      │
   │ 50K upd/sec │  │ 30K upd/sec │  │ 25K upd/sec │
   └─────────────┘  └─────────────┘  └─────────────┘

  Each city is independently scalable.
  Large cities may span multiple shards.

  Single Redis instance handles ~100K GEOADD/sec.
  Cluster mode for cities with more active drivers.
```

### WebSocket Connection Scaling

```
Challenge: Millions of concurrent WebSocket connections

  ┌────────────────────────────────────────────────────────┐
  │                WebSocket Architecture                   │
  │                                                        │
  │  ┌──────────┐  ┌──────────┐  ┌──────────┐             │
  │  │ WS Node  │  │ WS Node  │  │ WS Node  │  ... N nodes│
  │  │  50K     │  │  50K     │  │  50K     │             │
  │  │ conns    │  │ conns    │  │ conns    │             │
  │  └────┬─────┘  └────┬─────┘  └────┬─────┘             │
  │       └──────────────┼──────────────┘                  │
  │                      │                                 │
  │               ┌──────▼──────┐                          │
  │               │ Redis Pub/  │                          │
  │               │ Sub Channel │                          │
  │               │             │                          │
  │               │ ride:{id}   │                          │
  │               └─────────────┘                          │
  └────────────────────────────────────────────────────────┘

  When driver sends location update:
  1. Location Service updates Redis geospatial index
  2. Publishes to Redis channel: ride:{ride_id}
  3. WS node subscribed to that channel receives update
  4. WS node pushes to connected rider client

  Sticky sessions: rider and driver connections can be on
  different WS nodes. Redis Pub/Sub bridges them.
```

### Multi-Region Deployment

```
┌─────────────────────┐    ┌─────────────────────┐
│   US Region          │    │   EU Region          │
│                      │    │                      │
│  US cities:          │    │  EU cities:          │
│  NYC, SF, LA, etc.   │    │  London, Paris, etc. │
│                      │    │                      │
│  Independent stack:  │    │  Independent stack:  │
│  - API servers       │    │  - API servers       │
│  - Location Redis    │    │  - Location Redis    │
│  - Trip DB           │    │  - Trip DB           │
│  - Matching service  │    │  - Matching service  │
└──────────┬───────────┘    └──────────┬───────────┘
           │                           │
           └─────────┬─────────────────┘
                     │
           ┌─────────▼──────────┐
           │  Global Services   │
           │  - User DB         │
           │  - Payment gateway │
           │  - Analytics       │
           └────────────────────┘

Each region is self-contained for latency-sensitive operations.
Global services replicated with eventual consistency.
```

---

## 9. Trade-offs

### Matching Strategy

```
┌─────────────────┬─────────────────────┬───────────────────────┐
│ Aspect          │ Greedy (first fit)  │ Batch (global optimal)│
├─────────────────┼─────────────────────┼───────────────────────┤
│ Latency         │ ~5 seconds          │ ~30-60 seconds        │
│ Optimality      │ Local optimum       │ Global optimum        │
│ User experience │ Fast match          │ Better match quality  │
│ Implementation  │ Simple              │ Complex (Hungarian    │
│                 │                     │ algorithm or similar) │
│ Fairness        │ First-come-first    │ Can optimize for      │
│                 │ served              │ overall system        │
├─────────────────┼─────────────────────┼───────────────────────┤
│ Decision        │ Greedy for standard requests. Batch         │
│                 │ matching for shared rides (UberPool) where  │
│                 │ route overlap optimization is important.    │
└─────────────────┴─────────────────────────────────────────────┘
```

### Location Update Frequency

```
┌──────────────┬────────────┬──────────────┬──────────────────┐
│ Frequency    │ Throughput │ Accuracy     │ Battery Impact   │
├──────────────┼────────────┼──────────────┼──────────────────┤
│ Every 1s     │ 2M/sec    │ Excellent    │ High             │
│ Every 4s     │ 500K/sec  │ Good         │ Moderate         │
│ Every 10s    │ 200K/sec  │ Fair         │ Low              │
│ Every 30s    │ 67K/sec   │ Poor         │ Very Low         │
├──────────────┼────────────┼──────────────┼──────────────────┤
│ Decision: 4 seconds -- good balance of accuracy and        │
│ efficiency. Increase to 1-2s during active trip for        │
│ smoother rider tracking experience.                        │
└────────────────────────────────────────────────────────────┘
```

### ETA Calculation

```
┌──────────────────┬───────────────────┬────────────────────┐
│ Aspect           │ Simple (distance) │ Routing-based      │
├──────────────────┼───────────────────┼────────────────────┤
│ Accuracy         │ Poor (20-30% off) │ Good (5-10% off)   │
│ Computation cost │ Negligible        │ 5-20ms per query   │
│ Traffic-aware    │ No                │ Yes                │
│ Road-aware       │ No (straight line)│ Yes (actual roads) │
├──────────────────┼───────────────────┼────────────────────┤
│ Decision: Routing-based ETA using OSRM or Google Maps     │
│ Directions API with real-time traffic data. Cache popular  │
│ routes. Use ML model trained on historical trip data for   │
│ improved accuracy.                                         │
└───────────────────────────────────────────────────────────┘
```

---

## 10. Failure Scenarios & Handling

### Scenario 1: Driver App Crashes Mid-Trip

```
Problem: Driver's phone crashes, stops sending location updates.

Detection: No location update for > 30 seconds during active trip.

Handling:
1. Send push notification to driver: "Are you still on the trip?"
2. If driver reopens app within 2 minutes:
   - Resume normal tracking
   - Interpolate missing route using last known location + road network
3. If driver doesn't respond in 5 minutes:
   - Mark trip as "interrupted"
   - Contact rider via automated call/SMS
   - Offer rider option to end trip at last known location
   - Calculate partial fare based on completed distance
4. Safety: if rider presses "emergency" button, contact authorities
   with last known driver location
```

### Scenario 2: Payment Fails After Trip

```
Problem: Credit card charge declined after trip completion.

Handling:
1. Trip still completes normally (don't strand the rider)
2. Retry payment 3 times with exponential backoff
3. If still failing, notify rider to update payment method
4. Grace period: 72 hours to resolve
5. If unresolved: suspend rider account, send to collections
6. Driver always gets paid regardless of rider payment status

Architecture:
  Trip Completes -> Payment Queue (Kafka)
                 -> Payment Worker attempts charge
                 -> Success: mark paid, settle with driver
                 -> Failure: enqueue retry with backoff
                 -> Max retries: flag for manual review
```

### Scenario 3: No Drivers Available

```
Handling:
1. Expand search radius gradually (2km -> 5km -> 10km)
2. If still no drivers after 60 seconds:
   - Notify rider: "No drivers available right now"
   - Offer to notify when driver becomes available
   - Suggest alternative: different ride type, nearby pickup point
3. Increase surge pricing to attract more drivers to the area
4. Send notifications to offline drivers near the area:
   "High demand in your area. Go online to earn more."
```

### Scenario 4: Database Failover During Active Rides

```
Problem: Trip DB primary goes down during rush hour.

Handling:
1. Automatic failover to replica (< 30 seconds with RDS Multi-AZ)
2. In-flight ride state cached in Redis (survives DB failover)
3. Trip events buffered in Kafka (replayed after recovery)
4. Matching service uses in-memory state (not affected)
5. Location service uses Redis (not affected)

Critical data (ride state) flow:
  Driver action -> Kafka event -> Trip Service -> DB + Redis
                                                  |
  If DB down:                                     |
  Driver action -> Kafka event -> Trip Service -> Redis (cache)
                                                  |
  When DB recovers:                               |
  Kafka replay -> Trip Service -> DB (backfill)
```

### Scenario 5: GPS Drift / Inaccurate Location

```
Problem: Driver GPS shows wrong location (building interference,
         tunnel, etc.)

Handling:
1. Kalman filter smooths GPS jitter on client side
2. Server-side validation:
   - If driver "teleports" > 1 km in 4 seconds: reject update
   - If speed > 200 mph: reject as GPS error
3. Map matching: snap GPS coordinates to nearest road
4. Dead reckoning: use speed + heading to interpolate position
   during GPS blackouts (tunnels, parking garages)
5. Fare calculation uses map-matched route, not raw GPS
```

---

## 11. Monitoring & Alerting

### Key Metrics

```
┌─────────────────────────────┬────────────────────────────────┐
│ Metric                      │ Alert Threshold                │
├─────────────────────────────┼────────────────────────────────┤
│ Matching latency p99        │ > 15s (warn), > 30s (crit)    │
│ Matching success rate       │ < 95% (warn), < 90% (crit)    │
│ Location update throughput  │ Drop > 20% (crit)             │
│ Active WebSocket connections│ Drop > 10% in 5 min (crit)    │
│ ETA accuracy (actual vs est)│ > 30% deviation (warn)        │
│ Surge pricing accuracy      │ Manual review if > 5x          │
│ Trip completion rate        │ < 90% (warn), < 85% (crit)    │
│ Payment failure rate        │ > 2% (warn), > 5% (crit)      │
│ Driver cancellation rate    │ > 15% (warn), > 25% (crit)    │
│ Rider wait time p95         │ > 10 min (warn)                │
│ API latency p99             │ > 500ms (warn), > 2s (crit)   │
│ Location staleness          │ > 30s without update (warn)    │
│ Redis memory usage          │ > 80% per shard (warn)         │
│ Kafka consumer lag          │ > 10K messages (warn)          │
└─────────────────────────────┴────────────────────────────────┘
```

### Real-Time Operations Dashboard

```
City-Level Dashboard (per city):
┌─────────────────────────────────────────┐
│  San Francisco - Live Operations        │
│                                         │
│  Active Rides:  12,453                  │
│  Available Drivers:  8,921              │
│  Riders Waiting:  342                   │
│  Avg Wait Time:  3.2 min               │
│  Surge Zones:  4 of 28 zones active    │
│  Max Surge:  2.1x (Downtown)           │
│                                         │
│  [Heat Map showing supply/demand]       │
│  [Real-time matching success rate]      │
│  [ETA accuracy chart]                   │
└─────────────────────────────────────────┘
```

---

## 12. Interview Variations

### Common Follow-Up Questions

**Q1: "How would you handle shared rides (UberPool/Lyft Shared)?"**
```
A: Key differences from standard rides:
1. Batch matching: wait 30-60 seconds to collect riders
   heading in similar directions
2. Route optimization: find optimal pickup/dropoff ordering
   (traveling salesman variant for 2-3 riders)
3. Detour tolerance: rider accepts up to X% longer trip
4. Dynamic pricing: shared rides cheaper per person
5. Real-time route adjustment when new rider joins mid-trip
6. Matching uses route overlap score, not just proximity
```

**Q2: "How would you calculate ETA more accurately?"**
```
A: ML-based ETA prediction:
1. Historical data: same route, same time of day, same day of week
2. Real-time inputs: current traffic, road closures, weather
3. Features: distance, number of turns, road types, traffic signals
4. Model: gradient-boosted trees trained on billions of past trips
5. Ensemble: blend routing engine ETA with ML prediction
6. Feedback loop: compare predicted vs actual, retrain weekly
7. Special events: concerts, sports games, holidays (manual overrides)
```

**Q3: "How would you prevent fraud (fake rides, driver manipulation)?"**
```
A: Multi-layer fraud detection:
1. GPS spoofing detection: cross-reference with cell tower triangulation
2. Trip anomaly detection: impossibly short/long trips, circular routes
3. Collusion detection: same rider-driver pairs repeatedly
4. Device fingerprinting: flag multiple accounts from same device
5. ML model: trained on labeled fraud examples
6. Real-time scoring: flag trips with fraud score > threshold
7. Manual review queue for flagged trips
```

**Q4: "How do you handle driver location in areas with no cellular coverage?"**
```
A:
1. Client-side location buffering: store GPS points locally
2. When connectivity resumes, batch upload buffered locations
3. Server interpolates missing points using road network
4. Fare calculated on full route (including offline segment)
5. Trip state preserved locally on both driver and rider phones
6. Fallback: use Wi-Fi triangulation or last known location
```

**Q5: "How would you design the system for autonomous vehicles?"**
```
A: Key changes:
1. No driver app -- vehicle reports directly via API
2. Higher frequency location updates (10 Hz for safety)
3. Vehicle health monitoring (battery, sensors, tires)
4. Remote operations center for edge cases (stuck vehicle)
5. Geofenced operating areas (AV-approved roads only)
6. Passenger verification (PIN code, face recognition)
7. Ride handoff protocol when AV reaches its boundary
```

---

## Summary: Key Design Decisions

```
┌─────────────────────┬──────────────────────────────────────┐
│ Component           │ Decision                             │
├─────────────────────┼──────────────────────────────────────┤
│ Geospatial index    │ Redis GEOSEARCH (primary)            │
│ Location updates    │ WebSocket, 4-second interval         │
│ Matching algorithm  │ Greedy scoring (ETA + proximity)     │
│ Surge pricing       │ Real-time Flink job, 2-min windows   │
│ Trip database       │ PostgreSQL sharded by city           │
│ Event streaming     │ Kafka (ride events, location events) │
│ ETA calculation     │ Routing engine + ML model            │
│ Real-time tracking  │ WebSocket + Redis Pub/Sub            │
│ Payment processing  │ Async with retry and reconciliation  │
│ Deployment          │ Multi-region, city-level isolation   │
└─────────────────────┴──────────────────────────────────────┘
```
