# DRWALA Rental Platform
## Product, API, data, and interaction specification

> This document defines the complete rental flow for customers, inventory managers, and operations administrators. The current prototype can use the same contracts with in-memory state; production should replace the repository layer with a database and object storage.

## 1. Product scope

DRWALA is a camera and production-equipment rental platform with three connected surfaces:

1. **Customer catalog** — browse equipment, inspect a full detail page, select a date range, review price and availability, and create a booking.
2. **Operations dashboard** — manage overview metrics, bookings, inventory, tracking, customers, and audit logs from a persistent sidebar.
3. **Inventory handover workflow** — an inventory manager records condition with required photographs when equipment is handed over and returned. The audit trail links the booking, asset, operator, timestamps, notes, and evidence images.

Every state change must be visible in the relevant dashboard tab and in the customer booking details view.

## 2. Roles and permissions

| Role | Permissions |
|---|---|
| Customer | Browse public equipment, view details, check availability, create bookings, view own booking details, cancel within policy, view payment and pickup instructions. |
| Inventory manager | View assigned bookings, prepare equipment, upload handover and return photos, record condition notes, change operational status, flag damage. |
| Operations admin | Full dashboard access, manage equipment, approve or reject inventory changes, update booking status, assign staff, view tracking and all audit records. |
| System | Calculates price and availability, generates booking IDs, timestamps events, validates transitions, and appends immutable audit events. |

Authorization should be enforced on the server for every protected endpoint. Never trust a role or price sent by the browser.

## 3. Recommended architecture

```text
Next.js App Router
  ├─ Public catalog and equipment detail routes
  ├─ Customer booking routes
  ├─ Protected operations dashboard routes
  ├─ Route handlers /api/v1/*
  └─ Server-side service layer
       ├─ EquipmentService
       ├─ AvailabilityService
       ├─ BookingService
       ├─ InventoryService
       ├─ TrackingService
       └─ AuditService

Database: PostgreSQL
Object storage: private bucket for condition photos
Queue/event stream: optional for notifications and tracking updates
```

Use a repository/service boundary so the prototype can use dummy state while production can use PostgreSQL without changing UI contracts. All dates are ISO 8601 strings and all money values are integer minor units plus a currency code. For the current Indian pricing UI, return `currency: "INR"` and `amountMinor` in paise.

## 4. Domain entities

### Equipment

```ts
type Equipment = {
  id: string
  sku: string
  name: string
  slug: string
  brand: string
  category: "camera" | "lens" | "gimbal" | "audio" | "lighting" | "accessory"
  description: string
  shortDescription: string
  specifications: Record<string, string>
  dailyRate: Money
  depositAmount: Money
  images: AssetImage[]
  stockCount: number
  availableCount: number
  status: "available" | "reserved" | "maintenance" | "retired"
  condition: "excellent" | "good" | "fair" | "needs_review"
  rating?: number
  createdAt: string
  updatedAt: string
}
```

### Physical asset

Equipment catalog entries may represent multiple physical units.

```ts
type Asset = {
  id: string
  equipmentId: string
  serialNumber: string
  barcode?: string
  status: "available" | "reserved" | "checked_out" | "in_transit" | "maintenance" | "lost"
  condition: "excellent" | "good" | "fair" | "damaged"
  currentLocation?: Location
  lastInspectionAt?: string
  createdAt: string
  updatedAt: string
}
```

### Booking

```ts
type Booking = {
  id: string
  bookingNumber: string
  customerId: string
  status: "draft" | "pending" | "confirmed" | "ready_for_pickup" | "picked_up" | "in_use" | "return_pending" | "returned" | "completed" | "cancelled" | "rejected"
  startDate: string
  endDate: string
  pickupAt?: string
  returnAt?: string
  lineItems: BookingLineItem[]
  pricing: PricingSummary
  pickupLocation: Location
  notes?: string
  createdAt: string
  updatedAt: string
}
```

```ts
type BookingLineItem = {
  id: string
  equipmentId: string
  assetId?: string
  equipmentName: string
  quantity: number
  dailyRate: Money
  days: number
  subtotal: Money
}
```

### Evidence and audit event

```ts
type Evidence = {
  id: string
  kind: "handover_condition" | "return_condition" | "damage" | "identity" | "other"
  storageKey: string
  url?: string
  contentType: string
  checksum?: string
  capturedAt: string
  capturedBy: string
}

type AuditEvent = {
  id: string
  entityType: "booking" | "equipment" | "asset" | "user"
  entityId: string
  action: string
  actorId: string
  actorName: string
  metadata: Record<string, unknown>
  evidence: Evidence[]
  createdAt: string
}
```

## 5. State machines

### Booking state transitions

```text
draft -> pending -> confirmed -> ready_for_pickup -> picked_up -> in_use
in_use -> return_pending -> returned -> completed
pending -> rejected
pending|confirmed|ready_for_pickup -> cancelled
```

Only the server may transition a booking. The UI should disable actions that are not valid for the current state. Each transition creates an audit event and refreshes overview counts, booking lists, equipment availability, tracking, and the customer booking view.

### Asset state transitions

```text
available -> reserved -> checked_out -> in_transit -> available
checked_out -> maintenance
maintenance -> available
any active state -> lost
```

### Handover and return workflow

1. Manager opens a booking in `ready_for_pickup`.
2. Manager verifies customer identity and scans/selects the assigned asset.
3. Manager captures at least one handover photo. Recommended: front, rear, serial number, and accessories.
4. Manager enters condition notes and confirms handover.
5. Server validates the photo upload, changes booking to `picked_up`, changes asset to `checked_out`, and writes audit events.
6. On return, manager captures return photos before accepting the equipment.
7. Manager records missing accessories, damage, and notes.
8. Server changes booking to `returned` or `return_pending_review`, changes the asset to `available` or `maintenance`, and writes immutable return evidence.

A handover or return cannot be submitted without evidence unless an admin explicitly records an override reason.

## 6. API conventions

Base URL: `/api/v1`

Headers:

```http
Authorization: Bearer <session-token>
Content-Type: application/json
Idempotency-Key: <unique-key-for-mutating-request>
```

Success envelope:

```json
{
  "data": {},
  "meta": { "requestId": "req_123" }
}
```

Paginated envelope:

```json
{
  "data": [],
  "meta": {
    "requestId": "req_123",
    "page": 1,
    "pageSize": 20,
    "total": 84,
    "hasNextPage": true
  }
}
```

Error envelope:

```json
{
  "error": {
    "code": "BOOKING_DATE_UNAVAILABLE",
    "message": "The selected equipment is unavailable for these dates.",
    "fieldErrors": {}
  },
  "meta": { "requestId": "req_123" }
}
```

Use `400` for validation, `401` for missing session, `403` for permissions, `404` for missing records, `409` for availability or transition conflicts, and `422` for semantically invalid input.

## 7. Public catalog and camera APIs

### Fetch equipment catalog

`GET /api/v1/equipment`

Query parameters:

```text
category=camera
search=sony
brand=Sony
status=available
startDate=2026-10-10
endDate=2026-10-13
sort=popular|price_asc|price_desc|rating
page=1&pageSize=24
```

Response:

```json
{
  "data": [
    {
      "id": "eq_fx3",
      "name": "Sony FX3",
      "slug": "sony-fx3",
      "brand": "Sony",
      "category": "camera",
      "shortDescription": "Compact full-frame cinema camera.",
      "dailyRate": { "amountMinor": 250000, "currency": "INR" },
      "availability": { "available": true, "availableCount": 5, "requestedQuantity": 1 },
      "rating": 4.9,
      "thumbnail": { "url": "/camera-hero.png", "alt": "Sony FX3 camera" }
    }
  ],
  "meta": { "page": 1, "pageSize": 24, "total": 1, "hasNextPage": false }
}
```

### Fetch a single camera/equipment detail

`GET /api/v1/equipment/:equipmentId`

Response must include the complete detail page payload: description, specifications, included accessories, images, deposit, cancellation policy, rating, stock, related equipment, blackout dates, and availability summary.

```json
{
  "data": {
    "id": "eq_fx3",
    "name": "Sony FX3",
    "description": "Compact full-frame cinema camera with exceptional low-light performance...",
    "specifications": { "sensor": "Full-frame", "resolution": "4K 120p", "mount": "E-mount" },
    "includedItems": ["Body", "Battery", "Charger", "2 x memory card"],
    "dailyRate": { "amountMinor": 250000, "currency": "INR" },
    "depositAmount": { "amountMinor": 1000000, "currency": "INR" },
    "availability": { "availableCount": 5, "blockedDates": ["2026-10-18"] },
    "bookingPolicy": { "minimumDays": 1, "cancellationHours": 48 },
    "relatedEquipment": []
  }
}
```

The customer detail route should be a real page route such as `/equipment/:slug`, not only a modal. It should contain a gallery, title, description, specifications, price summary, date-range calendar, quantity, add-ons, pickup information, and a persistent booking summary.

## 8. Availability and calendar APIs

### Check availability

`GET /api/v1/equipment/:equipmentId/availability?startDate=2026-10-10&endDate=2026-10-13&quantity=1`

Response:

```json
{
  "data": {
    "equipmentId": "eq_fx3",
    "startDate": "2026-10-10",
    "endDate": "2026-10-13",
    "days": 3,
    "quantity": 1,
    "available": true,
    "availableCount": 5,
    "blockedDates": [],
    "conflicts": []
  }
}
```

The calendar sends a request after the user selects a valid date range. The server recalculates availability again during booking creation to prevent race conditions. Never rely only on the calendar's client-side disabled dates.

## 9. Customer booking flow

### Step 1: Create a booking draft

`POST /api/v1/bookings`

Request:

```json
{
  "items": [{ "equipmentId": "eq_fx3", "quantity": 1 }],
  "startDate": "2026-10-10",
  "endDate": "2026-10-13",
  "pickupLocationId": "loc_mumbai_01",
  "notes": "Documentary shoot"
}
```

Server actions:

1. Authenticate the customer.
2. Validate dates, minimum rental duration, quantity, and pickup location.
3. Lock/check availability in a transaction.
4. Recalculate daily rate, number of billable days, deposit, taxes, discounts, and total.
5. Create a `pending` or `draft` booking.
6. Create `booking.created` audit event.

Response:

```json
{
  "data": {
    "id": "book_1024",
    "bookingNumber": "DRW-1024",
    "status": "pending",
    "startDate": "2026-10-10",
    "endDate": "2026-10-13",
    "pricing": {
      "rental": { "amountMinor": 750000, "currency": "INR" },
      "deposit": { "amountMinor": 1000000, "currency": "INR" },
      "tax": { "amountMinor": 135000, "currency": "INR" },
      "total": { "amountMinor": 885000, "currency": "INR" }
    }
  }
}
```

### Confirm booking

`POST /api/v1/bookings/:bookingId/confirm`

Request may include customer contact, billing information, and payment reference. The server rechecks availability and amount. On success, status becomes `confirmed`, reserved assets are assigned, and the customer receives confirmation.

### Fetch customer booking details

`GET /api/v1/bookings/:bookingId`

Customers can only retrieve their own bookings. Admins and assigned managers can retrieve operational details.

Response includes:

- Booking number and status timeline.
- Equipment and assigned serial numbers when available.
- Rental dates, pickup and return instructions.
- Full pricing and deposit status.
- Current tracking/location state.
- Handover/return evidence visible according to role.
- Cancellation and support actions.

### List customer bookings

`GET /api/v1/me/bookings?status=active&page=1&pageSize=20`

### Cancel booking

`POST /api/v1/bookings/:bookingId/cancel`

Request:

```json
{ "reason": "Plans changed" }
```

The service applies the cancellation policy, releases the reservation, updates equipment availability, and creates an audit event.

## 10. Overview/dashboard API

`GET /api/v1/dashboard/overview?from=2026-10-01&to=2026-10-31`

This powers the dashboard home without making the UI calculate business metrics.

```json
{
  "data": {
    "revenue": { "amountMinor": 184500000, "currency": "INR", "changePercent": 12.4 },
    "activeBookings": 28,
    "equipmentUtilizationPercent": 74,
    "availableEquipment": 46,
    "maintenanceEquipment": 5,
    "pendingHandover": 7,
    "pendingReturns": 3,
    "bookingTrend": [{ "date": "2026-10-01", "count": 8, "revenueMinor": 4200000 }],
    "topEquipment": [{ "equipmentId": "eq_fx3", "name": "Sony FX3", "rentals": 18 }],
    "recentActivity": []
  }
}
```

Dashboard cards should link to filtered views. For example, clicking `Pending handover` opens Bookings with `status=ready_for_pickup`, and clicking `Maintenance` opens Inventory filtered by `maintenance`.

## 11. Booking management API

### List all bookings

`GET /api/v1/admin/bookings?status=confirmed&from=2026-10-01&to=2026-10-31&search=DRW-1024&page=1&pageSize=20`

### Update booking status

`PATCH /api/v1/admin/bookings/:bookingId/status`

```json
{ "status": "ready_for_pickup", "note": "Kit checked and packed" }
```

Allowed transitions are checked by the server. Every change appears in the booking timeline and audit log.

### Assign physical asset

`POST /api/v1/admin/bookings/:bookingId/assets`

```json
{ "assignments": [{ "equipmentId": "eq_fx3", "assetId": "asset_fx3_004" }] }
```

## 12. Inventory APIs

### Get inventory register

`GET /api/v1/inventory?status=available&category=camera&search=FX3&page=1&pageSize=25`

Response includes catalog data plus each physical asset, serial number, current status, condition, booking assignment, last inspection, and location.

### Add new equipment catalog item

`POST /api/v1/admin/equipment`

Request:

```json
{
  "name": "Canon EOS R5 C",
  "brand": "Canon",
  "category": "camera",
  "description": "8K hybrid cinema camera.",
  "specifications": { "resolution": "8K RAW", "mount": "RF" },
  "dailyRate": { "amountMinor": 320000, "currency": "INR" },
  "depositAmount": { "amountMinor": 1500000, "currency": "INR" },
  "stockCount": 3,
  "assets": [
    { "serialNumber": "R5C-001", "condition": "excellent" }
  ]
}
```

The add-equipment form should be a full page or sheet with fields for name, brand, category, description, specs, price, deposit, quantity, serial numbers, condition, images, and published status. On submit, display the created equipment in the inventory list immediately and update overview counts.

### Update equipment

`PATCH /api/v1/admin/equipment/:equipmentId`

Supports catalog fields, pricing, status, and condition. Price changes must not rewrite historical booking prices.

### Add physical asset

`POST /api/v1/admin/equipment/:equipmentId/assets`

### Change asset status

`PATCH /api/v1/admin/assets/:assetId/status`

```json
{ "status": "maintenance", "reason": "Sensor inspection required" }
```

### Upload inventory/equipment image

`POST /api/v1/uploads`

Use a signed upload URL for object storage. The client uploads directly, then sends the returned `storageKey` to the equipment or evidence endpoint. Validate MIME type, size, ownership, and checksum.

## 13. Handover and return evidence APIs

### Request upload URL

`POST /api/v1/bookings/:bookingId/evidence/upload-url`

```json
{ "kind": "handover_condition", "fileName": "front.jpg", "contentType": "image/jpeg", "size": 2400000 }
```

Response:

```json
{ "data": { "uploadUrl": "signed-url", "storageKey": "evidence/book_1024/front.jpg", "expiresAt": "2026-10-10T10:00:00Z" } }
```

### Submit handover

`POST /api/v1/bookings/:bookingId/handover`

```json
{
  "assetId": "asset_fx3_004",
  "evidenceIds": ["ev_001", "ev_002", "ev_003"],
  "condition": "excellent",
  "notes": "Body, battery, charger, and cards verified.",
  "accessories": [{ "name": "Battery", "present": true }]
}
```

Server validates that the manager is authorized, evidence belongs to the booking, at least one photo exists, and the booking is `ready_for_pickup`. It then changes the booking to `picked_up`, the asset to `checked_out`, and appends `handover.completed`.

### Submit return

`POST /api/v1/bookings/:bookingId/return`

```json
{
  "assetId": "asset_fx3_004",
  "evidenceIds": ["ev_010", "ev_011", "ev_012"],
  "condition": "good",
  "damageFound": false,
  "notes": "Returned on time. All accessories present.",
  "accessories": [{ "name": "Battery", "present": true }]
}
```

If damage is found, require `damageDescription`, set booking to `return_pending`, set asset to `maintenance`, and notify an admin. The evidence must remain immutable after submission.

## 14. Audit log APIs

### List audit logs

`GET /api/v1/admin/audit-logs?entityType=booking&entityId=book_1024&action=handover.completed&from=2026-10-01&to=2026-10-31&page=1&pageSize=50`

Response:

```json
{
  "data": [
    {
      "id": "audit_001",
      "action": "handover.completed",
      "entityType": "booking",
      "entityId": "book_1024",
      "actorName": "Ananya Rao",
      "metadata": { "assetId": "asset_fx3_004", "condition": "excellent" },
      "evidence": [{ "id": "ev_001", "kind": "handover_condition", "url": "signed-view-url" }],
      "createdAt": "2026-10-10T10:42:00Z"
    }
  ]
}
```

### Get entity timeline

`GET /api/v1/admin/audit-logs/:entityType/:entityId`

The detail page should show a chronological timeline with actor, timestamp, action, note, and evidence thumbnail. Audit events are append-only; do not expose an edit or delete action.

## 15. Tracking API

Tracking is operational visibility for assets and active bookings. It can use dummy coordinates in the prototype and real scanner/GPS events later.

### Current tracking map

`GET /api/v1/admin/tracking?status=active&region=mumbai`

```json
{
  "data": {
    "updatedAt": "2026-10-10T10:45:00Z",
    "units": [
      {
        "assetId": "asset_fx3_004",
        "bookingId": "book_1024",
        "equipmentName": "Sony FX3",
        "status": "in_use",
        "location": { "lat": 19.076, "lng": 72.8777, "label": "Mumbai studio" },
        "lastSeenAt": "2026-10-10T10:42:00Z",
        "customerName": "Rohan Mehta"
      }
    ]
  }
}
```

### Asset tracking history

`GET /api/v1/admin/tracking/:assetId/history?from=...&to=...`

The tracking screen should display a map region, status legend, count cards, selected unit details, last-seen time, and linked booking. For the demo, use deterministic dummy positions and make map markers clickable.

## 16. Customers API

`GET /api/v1/admin/customers?search=rohan&status=active&page=1&pageSize=25`

`GET /api/v1/admin/customers/:customerId`

Customer details include contact information, booking history, lifetime value, outstanding deposits, verification status, and support notes. Personal information is role-protected and should not appear in public catalog responses.

## 17. Notifications and integrations

Recommended event names:

```text
booking.created
booking.confirmed
booking.cancelled
booking.ready_for_pickup
booking.picked_up
booking.returned
booking.completed
equipment.created
equipment.updated
asset.status_changed
handover.completed
return.submitted
return.damage_flagged
```

Events can trigger email/SMS/WhatsApp notifications, but event creation must not block the core database transaction. Use an outbox table or queue for reliable delivery.

## 18. Frontend route structure

```text
/
├─ /equipment
├─ /equipment/[slug]
├─ /booking/[bookingId]
├─ /account/bookings
└─ /admin
   ├─ /overview
   ├─ /bookings
   ├─ /inventory
   ├─ /tracking
   ├─ /customers
   └─ /audit-logs
```

The existing prototype can keep a single-page demo entry point, but the target implementation should promote the equipment detail view and booking details to real routes. Use query parameters for dashboard filters so tabs are deep-linkable and browser navigation works.

## 19. UI-to-API interaction map

| UI action | API call | State updates |
|---|---|---|
| Load catalog | `GET /equipment` | Equipment cards, filter counts, availability badges. |
| Open camera | `GET /equipment/:id` | Detail route data, gallery, specs, calendar. |
| Select dates | `GET /equipment/:id/availability` | Calendar conflicts, days, rental subtotal. |
| Confirm booking | `POST /bookings` then confirm | Booking state, customer booking list, dashboard counts, availability. |
| View booking | `GET /bookings/:id` | Timeline, pricing, assigned equipment, pickup/return state. |
| Open overview | `GET /dashboard/overview` | KPI cards, chart, activity, links to filtered tabs. |
| Open bookings | `GET /admin/bookings` | Table, filters, status transitions, booking drawer/page. |
| Open inventory | `GET /inventory` | Asset register, serials, condition and status. |
| Add equipment | `POST /admin/equipment` | New catalog row, stock totals, activity event. |
| Upload handover evidence | upload URL + `POST /handover` | Booking picked up, asset checked out, audit timeline. |
| Submit return | upload URL + `POST /return` | Booking returned/completed or damage review, asset availability. |
| Open audit logs | `GET /admin/audit-logs` | Immutable event timeline and evidence viewer. |
| Open tracking | `GET /admin/tracking` | Map markers, active units, last-seen panel. |

## 20. Prototype state implementation

Until a backend is connected, keep one top-level state store rather than independent tab state:

```ts
type RentalStore = {
  equipment: Equipment[]
  assets: Asset[]
  bookings: Booking[]
  auditLogs: AuditEvent[]
  trackingUnits: TrackingUnit[]
  currentUser: User
}
```

Use service-shaped functions even for dummy data:

```ts
getEquipment(filters)
getEquipmentById(id)
checkAvailability(id, range)
createBooking(input)
getBooking(id)
listBookings(filters)
addEquipment(input)
submitHandover(input)
submitReturn(input)
listAuditLogs(filters)
getOverview(range)
getTracking(filters)
```

Each mutation must update the same store and append an audit event. Do not show a success toast as the only result. After a successful action, navigate or render the updated resource, show its status badge, and refresh dependent lists. Toasts are supplementary feedback only.

For demo uploads, use an actual file input and store object URLs in state. Revoke object URLs when the item is removed. Production must replace object URLs with signed private storage URLs.

## 21. Validation and security checklist

- Recalculate pricing and availability on the server.
- Validate `startDate < endDate`, minimum rental days, quantity, and date format.
- Prevent overlapping reservations with a transaction or exclusion constraint.
- Use idempotency keys for booking creation, payment, handover, and return submissions.
- Scope customer queries by authenticated `customerId`.
- Enforce manager/admin authorization on inventory and audit endpoints.
- Never accept arbitrary `actorId`, status transitions, prices, or evidence ownership from the client.
- Validate upload size, MIME type, image dimensions, and malware policy.
- Keep evidence private and serve short-lived signed view URLs.
- Redact personal data from logs and client errors.
- Add rate limits to authentication, availability, booking, and upload endpoints.
- Audit every mutation with actor, timestamp, before/after status, and request ID.
- Use soft deletion/retirement for equipment so historical bookings remain readable.

## 22. Recommended database tables

```text
users
customer_profiles
roles
locations
equipment
assets
equipment_images
asset_status_events
bookings
booking_items
booking_asset_assignments
booking_status_events
payments
deposits
evidence_files
audit_events
tracking_events
notifications
outbox_events
```

Important indexes:

```text
bookings(customer_id, created_at desc)
bookings(status, start_date, end_date)
booking_items(equipment_id)
assets(equipment_id, status)
audit_events(entity_type, entity_id, created_at desc)
tracking_events(asset_id, created_at desc)
```

Use a uniqueness or exclusion strategy to prevent two confirmed bookings from reserving the same physical asset over overlapping date ranges.

## 23. Definition of done

The rental site is complete when:

- A customer can browse, open a real equipment detail page, select dates in a calendar, see availability and total pricing, confirm a booking, and open booking details.
- The booking appears in the admin bookings view with the correct status and dates.
- An admin can add equipment through a complete form, see it in inventory, and open its detail page.
- An inventory manager can select a booking, upload handover photos, submit condition notes, and see the booking move to `picked_up`.
- The manager can upload return photos, submit condition, and see the asset become available or maintenance based on damage.
- Audit logs show every important mutation with actor, timestamps, notes, and evidence.
- Tracking displays dummy or real asset positions linked to bookings.
- Overview cards, bookings, inventory, tracking, customers, and audit logs all read from the same source of truth.
- Invalid actions are disabled or rejected with a clear inline error; a toast alone is never the only confirmation.
- Refreshing a real production page preserves data through the backend; the demo store may be replaced by a persistence adapter later.

## 24. Example end-to-end happy path

```text
Customer opens /equipment/sony-fx3
  -> GET equipment detail
Customer selects Oct 10–13
  -> GET availability
Customer confirms
  -> POST booking
  -> booking DRW-1024 is pending/confirmed
  -> equipment availability decreases
  -> customer sees /booking/DRW-1024
Operations opens Bookings
  -> GET admin bookings
Admin marks ready for pickup
  -> PATCH booking status
Manager opens booking
  -> POST upload URLs
  -> uploads handover photos
  -> POST handover
  -> booking picked_up, asset checked_out
  -> audit event and tracking unit created
Customer returns equipment
Manager uploads return photos
  -> POST return
  -> booking returned/completed
  -> asset available or maintenance
  -> audit log updated
Overview refreshes
  -> active bookings, utilization, available stock, and activity update
```

This contract keeps catalog, detail, calendar, bookings, inventory, tracking, overview, and audit logging connected as one rental workflow instead of isolated demo screens.