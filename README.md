# Comic-Con India Parking System — Database Design

A database design for a large multi-zone event parking facility at Comic-Con India. The system tracks vehicles entering across multiple days, assigns parking spots based on vehicle type and access category, manages parking sessions with entry/exit timestamps, issues tickets, and records payments.

---

## Business Context

This is not a simple parking entry-exit tracker. Key characteristics that shaped the design:

- **Multi-zone, multi-level facility** — the venue has named zones (Zone A, Zone B, EV Zone, VIP Zone) and each zone has multiple levels
- **Vehicle categories matter** — a bike needs a different spot than an SUV or an EV vehicle
- **Reserved parking categories exist** — VIP guests, cosplayers with props, exhibitors, staff, and EV charging vehicles each get designated spots
- **One vehicle can visit multiple times** — the same car can enter on Day 1, exit, and re-enter on Day 2; each entry is a separate session
- **One parking spot can serve many vehicles over time** — spot reuse across sessions is a core requirement
- **Ticket and session are separate** — a ticket is issued at entry; the session tracks the full duration including exit time and calculated fee
- **Payment connects to the session** — fee is calculated based on duration when the vehicle exits

---

## Schema Overview

The database has **10 entities**:

| Entity | Purpose |
|---|---|
| `vehicle_categories` | Master list of vehicle types — bike, car, SUV, cab, EV |
| `vehicles` | Individual vehicle records with license plate and owner info |
| `access_categories` | Special access types — VIP, exhibitor, cosplayer, staff, general |
| `parking_zones` | Top-level zones in the venue — Zone A, EV Zone, VIP Zone etc. |
| `parking_levels` | Levels within each zone — Level 1, Level 2 etc. |
| `spot_categories` | Types of spots — general, reserved, EV charging, oversized |
| `parking_spots` | Individual parking spots with availability and reservation status |
| `parking_sessions` | Entry/exit tracking per vehicle per visit — the core record |
| `parking_tickets` | Ticket issued at entry, linked to session and access category |
| `payments` | Payment record per session — amount, method, status |

---

## Key Design Decisions

### 1. Zone → Level → Spot hierarchy
The physical structure is: Zone contains Levels, Levels contain Spots. This three-tier hierarchy means availability can be queried at any level — "how many spots are free in Zone B Level 2?" works cleanly without scanning the entire facility.

### 2. `vehicle_categories` and `spot_categories` are separate master tables
Vehicle type (bike, SUV, EV) and spot type (general, reserved, EV charging) are independent catalogs. The system matches them at session creation — a bike category vehicle gets assigned a spot from a spot category that allows bikes.

### 3. `access_categories` handles all reserved parking
VIP guests, exhibitors, cosplayers with props, staff, and EV vehicles all have different access levels with a `priority_level` field. A parking spot can be `reserved_for_access_id` pointing to the specific access category it's designated for.

### 4. `parking_sessions` is the core business event
Every vehicle entry creates a new session. The session stores entry time, exit time (null while active), duration (calculated on exit), session status, and event day. This allows the same vehicle to have multiple sessions across event days and the same spot to appear in many sessions over time.

### 5. `parking_tickets` are separate from sessions
A ticket is issued at the gate the moment the vehicle enters. It carries the ticket number, issue time, and access category. The session is the operational record tracking duration. Separating them means the gate system issues a ticket immediately without waiting for exit data.

### 6. `is_available` on `parking_spots`
A real-time availability flag on each spot. Set to `false` when a session starts, flipped back to `true` when the session ends. Allows the system to answer "which spots are currently free" instantly.

### 7. Payment rate stored per session
`rate_per_hour` is stored on the payment record so different zones or event days can have different rates. Amount is calculated as `rate_per_hour × duration_minutes / 60` when exit is recorded.

---

## Parking System Workflow

```
Vehicle arrives at gate
      ↓
System checks vehicle_category + access_category
      ↓
Suitable available spot assigned (zone → level → spot)
      ↓
Parking session created — entry_time recorded, spot marked unavailable
      ↓
Parking ticket issued — ticket_number, access_id, session_id
      ↓
Vehicle parks
      ↓
Vehicle exits — exit_time recorded, duration calculated
      ↓
Session status → completed, spot marked available again
      ↓
Payment calculated and recorded
```

---

## Relationships

```
vehicle_categories   ||--o{   vehicles              : classifies
vehicles             ||--o{   parking_sessions      : enters via
parking_zones        ||--o{   parking_levels        : contains
parking_levels       ||--o{   parking_spots         : holds
spot_categories      ||--o{   parking_spots         : typed by
access_categories    ||--o{   parking_spots         : reserved for
parking_spots        ||--o{   parking_sessions      : used in
parking_sessions     ||--||   parking_tickets       : generates
vehicles             ||--o{   parking_tickets       : issued to
access_categories    ||--o{   parking_tickets       : categorized by
parking_sessions     ||--||   payments              : billed via
parking_tickets      ||--o{   payments              : linked to
```

---

## Entities and Attributes

### vehicle_categories
| Column | Type | Notes |
|---|---|---|
| category_id | int PK | |
| name | varchar | bike / car / SUV / cab / EV |
| description | varchar | |
| size_class | varchar | small / medium / large — for spot matching |

### vehicles
| Column | Type | Notes |
|---|---|---|
| vehicle_id | int PK | |
| category_id | int FK | → vehicle_categories |
| license_plate | varchar | Unique identifier |
| owner_name | varchar | |
| owner_phone | varchar | |
| ev_charging_required | boolean | Flags EV vehicles needing charging spots |
| registered_at | datetime | First seen at venue |

### access_categories
| Column | Type | Notes |
|---|---|---|
| access_id | int PK | |
| name | varchar | general / VIP / exhibitor / cosplayer / staff / EV |
| description | text | |
| priority_level | int | Higher = higher priority for spot allocation |

### parking_zones
| Column | Type | Notes |
|---|---|---|
| zone_id | int PK | |
| name | varchar | Zone A / EV Zone / VIP Zone / Staff Zone |
| description | text | |
| zone_type | varchar | general / reserved / ev_charging / overflow |
| total_levels | int | |

### parking_levels
| Column | Type | Notes |
|---|---|---|
| level_id | int PK | |
| zone_id | int FK | → parking_zones |
| level_number | int | 1, 2, 3... |
| total_spots | int | |
| description | varchar | Ground floor / Basement etc. |

### spot_categories
| Column | Type | Notes |
|---|---|---|
| spot_category_id | int PK | |
| name | varchar | general / reserved / ev_charging / oversized / cosplayer |
| description | text | |
| is_reserved | boolean | Requires special access |
| allowed_vehicle_size | varchar | small / medium / large / any |

### parking_spots
| Column | Type | Notes |
|---|---|---|
| spot_id | int PK | |
| level_id | int FK | → parking_levels |
| spot_category_id | int FK | → spot_categories |
| spot_number | varchar | A1-001, B2-045 etc. |
| is_ev_charging | boolean | Has EV charging point |
| is_available | boolean | Real-time availability flag |
| is_reserved | boolean | Pre-reserved spot |
| reserved_for_access_id | int FK | → access_categories (null for general spots) |

### parking_sessions
| Column | Type | Notes |
|---|---|---|
| session_id | int PK | |
| vehicle_id | int FK | → vehicles |
| spot_id | int FK | → parking_spots |
| entry_time | datetime | When vehicle entered |
| exit_time | datetime | When vehicle exited (null while active) |
| duration_minutes | int | Calculated on exit |
| session_status | varchar | active / completed / overstay / abandoned |
| event_day | int | Day 1 / Day 2 / Day 3 of Comic-Con |

### parking_tickets
| Column | Type | Notes |
|---|---|---|
| ticket_id | int PK | |
| session_id | int FK | → parking_sessions (one-to-one) |
| vehicle_id | int FK | → vehicles |
| ticket_number | varchar | Unique printed ticket code |
| issued_at | datetime | When ticket was printed at gate |
| ticket_type | varchar | entry / vip / exhibitor / ev / staff |
| access_id | int FK | → access_categories |

### payments
| Column | Type | Notes |
|---|---|---|
| payment_id | int PK | |
| session_id | int FK | → parking_sessions |
| ticket_id | int FK | → parking_tickets |
| amount | decimal | Final calculated amount |
| rate_per_hour | decimal | Rate applied for this session |
| payment_method | varchar | cash / card / UPI / FASTag |
| payment_status | varchar | pending / paid / failed / waived |
| transaction_ref | varchar | UPI / card reference |
| paid_at | datetime | |

---

## Real-World Scenarios Supported

| Scenario | How it's handled |
|---|---|
| Same car enters Day 1, exits, re-enters Day 2 | Two separate `parking_sessions` for same `vehicle_id` |
| Same spot reused by 5 vehicles across the day | Multiple `parking_sessions` reference same `spot_id` |
| VIP guest gets reserved spot | Spot has `reserved_for_access_id` → VIP; ticket has matching `access_id` |
| EV vehicle needs charging spot | `ev_charging_required` on vehicle + `is_ev_charging` on spot for matching |
| Cosplayer with props needs wider spot | `access_categories` → cosplayer; `spot_category` → oversized |
| Track currently parked vehicles | Query `parking_sessions` where `session_status = active` |
| Check free spots in Zone B Level 2 | Query `parking_spots` where `level_id = Zone B L2` AND `is_available = true` |
| Calculate parking fee on exit | `amount = rate_per_hour × duration_minutes / 60` |

---

## Tools Used

- **Eraser.io** — ERD diagram design
- **Eraser Entity Relationship Diagram** syntax for schema definition

---
## ER Diagram

> The ER diagram is included in this repository as:

 <p align="center">
  <img src="Comic-Con Parking System.png" width="800"/>
</p>
