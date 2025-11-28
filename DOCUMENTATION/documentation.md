# TICKIFY API Reference (Samples Included)

Every section below documents the controllers under `com.oshayer.event_manager.*` with:
- Request parameters & bodies (tied to DTO classes)
- Domain entities touched during processing
- Sample JSON payloads/responses so you can test endpoints quickly

Use this file as the authoritative backend contract.

---

## Reading Guide & Legend

| Badge | Meaning |
| ----- | ------- |
| `Auth: Public` | No JWT required |
| `Auth: JWT (ROLE_X)` | Requires a valid JWT with the listed authority |
| `Entities` | Primary JPA entities read/written |
| `DTO` | Request/response class under `src/main/java/com/oshayer/event_manager/**/dto` |

### Common Error Envelope
```json
{
  "timestamp": "2024-05-14T09:15:22Z",
  "status": 400,
  "error": "Bad Request",
  "message": "Seat label already exists in this layout",
  "path": "/api/seat-layouts/123/seats"
}
```

### Suggested Workflow
1. `/api/auth/signup` → `/api/auth/verify` → `/api/auth/login`
2. `/api/users/event-manager|operator|event-checker`
3. `/api/venues` → `/api/venues/{venueId}/layouts` → `/api/seat-layouts/{layoutId}/seats`
4. `/api/events` → `/api/events/{id}/seats/sync` → `/api/events/{id}/seat-map`
5. `/api/holds` → `/api/payments/intents` → Stripe webhook → `/api/tickets/issue/{ticketId}`
6. `/api/admin/dashboard/*`, `/api/events/{id}/ticket-details`, `/api/holds/events/{eventId}`

---

## 1. Authentication (`/api/auth`)

### POST `/api/auth/signup`
- **DTO**: `SignupRequest`
- **Entities**: `UserEntity`
- **Request Body Fields**: `username`, `email`, `password` (all required)
- **Sample Request**
```json
{
  "username": "johndoe",
  "email": "john@example.com",
  "password": "Secret123!"
}
```
- **Sample Response**
```json
"Signup successful! Please verify your email."
```

### POST `/api/auth/login`
- **DTO**: `LoginRequest` ➝ `JwtResponse`
- **Entities**: `UserEntity` (read-only)
- **Sample Request**
```json
{
  "email": "john@example.com",
  "password": "Secret123!"
}
```
- **Sample Response**
```json
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "type": "Bearer"
}
```

### GET `/api/auth/verify`
- **Query Param**: `token` (String)
- **Entities**: `UserEntity`
- **Sample Request**: `/api/auth/verify?token=2f54e4d8-...`
- **Response**: `"Email verified successfully! You can now log in."`

### PUT `/api/auth/change-password`
- **DTO**: `ChangePasswordRequest`
- **Entities**: `UserEntity`
- **Sample Request**
```json
{
  "currentPassword": "OldSecret",
  "newPassword": "NewSecret!2024"
}
```
- **Response**: `"Password changed successfully"`

### POST `/api/auth/forgot-password`
- **DTO**: `ForgotPasswordRequest`
- **Sample Request**
```json
{
  "email": "john@example.com"
}
```
- **Response**: `"Password reset email sent"`

### POST `/api/auth/reset-password`
- **DTO**: `ResetPasswordRequest`
- **Sample Request**
```json
{
  "token": "b48b4bb7-...",
  "newPassword": "RecoveredPass!"
}
```
- **Response**: `"Password has been reset successfully"`

---

## 2. Users (`/api/users`)

### GET `/api/users/me`
- **Response DTO**: `UserResponse`
- **Entities**: `UserEntity`, `OrganizationEntity`
- **Sample Response (truncated)**
```json
{
  "id": "11111111-1111-1111-1111-111111111111",
  "username": "johndoe",
  "email": "john@example.com",
  "role": "ROLE_USER",
  "organizationName": "Awesome Org",
  "emailVerified": true,
  "createdAt": "2024-05-10T09:42:00Z"
}
```

### PUT `/api/users/me`
- **DTO**: `UpdateUserRequest`
- **Entities**: `UserEntity`
- **Sample Request**
```json
{
  "phone": "+1-555-0100",
  "fullName": "Johnathan Doe",
  "imageUrl": "https://cdn.app/avatar/john.png"
}
```

### POST `/api/users/org`
- **DTO**: `OrgUserCreateRequest`
- **Entities**: `UserEntity`, `OrganizationEntity`
- **Sample Request**
```json
{
  "email": "operator@example.com",
  "phone": "+1-555-0200",
  "fullName": "Opal Operator",
  "imageUrl": "https://cdn.app/op.png",
  "password": "StrongPass123"
}
```

### POST `/api/users/event-manager`
- **DTO**: `CreateEventManagerRequest`
- **Entities**: `UserEntity`
- **Sample Request**
```json
{
  "email": "manager@acme.org",
  "phone": "+1-555-0300",
  "firstName": "Maggie",
  "lastName": "Manager",
  "fullName": "Maggie Manager",
  "imageUrl": "https://cdn.app/mm.png",
  "password": "Manager!23",
  "username": "maggie.manager"
}
```

### DELETE `/api/users/event-manager/{userId}`
- **Path Param**: `userId` (UUID)
- **Response**: `204 No Content`

### POST `/api/users/operator`
- **DTO**: `OperatorCreateRequest`
- **Sample Request** identical to event manager but role `ROLE_OPERATOR`.

### POST `/api/users/event-checker`
- **DTO**: `EventCheckerCreateRequest`
- **Sample Request**
```json
{
  "username": "checker1",
  "password": "Check!234",
  "email": "checker@acme.org",
  "phone": "+1-555-0400",
  "firstName": "Chad",
  "lastName": "Checker",
  "fullName": "Chad Checker",
  "imageUrl": "https://cdn.app/chad.png"
}
```

### GET `/api/users/event-managers|operators|event-checkers`
- **Response**: `List<UserResponse>` filtered by role
- **Sample Response**
```json
[
  {
    "id": "22222222-2222-...",
    "username": "maggie.manager",
    "role": "ROLE_EVENT_MANAGER",
    "email": "manager@acme.org",
    "organizationName": "Acme Events"
  }
]
```

---

## 3. Admin Dashboard (`/api/admin/dashboard`)

Common Query Params: `from`, `to`, `eventId`, `venueId`, `expiringWithin`. DTO: `DashboardFilter`.

### GET `/overview`
- **Response DTO**: `AdminOverviewResponse`
- **Entities**: `EventEntity`, `TicketEntity`, `UserEntity`
- **Sample**
```json
{
  "events": {"total": 12, "live": 3, "upcoming": 6, "completed": 3},
  "tickets": {"pending": 25, "issued": 410, "used": 390},
  "revenue": {"gross": 82000.0, "refunded": 1500.0, "net": 80500.0}
}
```

### GET `/events`
- **Response DTO**: `List<EventPerformanceRow>`
- **Entities**: `EventEntity`, `TicketEntity`, `EventSeatEntity`
- **Sample Item**
```json
{
  "eventId": "...",
  "eventName": "Jazz Night",
  "ticketsSold": 320,
  "capacity": 500,
  "gross": 28000.0,
  "refunded": 500.0,
  "netRevenue": 27500.0,
  "sellThrough": 64.0,
  "available": 180
}
```

### GET `/sales-trend`
- **DTO**: `SalesTrendPoint`
- **Entities**: `TicketEntity`, `PaymentEntity`

### GET `/operations`
- **DTO**: `OperationsSummaryResponse` (sold/reserved/available seats, hold alerts)
- **Sample Response**
```json
{
  "sold": 320,
  "reserved": 45,
  "available": 135,
  "activeHolds": 8,
  "expiringSoon": 2,
  "alerts": [
    {
      "holdId": "hold-uuid",
      "buyerName": "Opal Operator",
      "expiresAt": "2024-05-15T10:10:00Z"
    }
  ]
}
```

---

## 4. Directory & Venues

### POST `/api/artists`
- **DTO**: `ArtistCreateRequest`
- **Entities**: `ArtistEntity`
- **Sample Request**
```json
{
  "name": "DJ Pulse",
  "description": "Electronic artist",
  "email": "pulse@music.com",
  "mobile": "+1-555-0500",
  "address": "123 Music Ave",
  "facebookLink": "https://facebook.com/djpulse",
  "instagramLink": "https://instagram.com/djpulse",
  "youtubeLink": "https://youtube.com/djpulse",
  "websiteLink": "https://djpulse.com",
  "imageUrl": "https://cdn.app/pulse.jpg"
}
```

### POST `/api/business-organizations`
- **DTO**: `BusinessOrganizationCreateRequest`
- **Entities**: `BusinessOrganizationEntity`
- **Sample Request**
```json
{
  "name": "Acme Events",
  "description": "Corporate partner",
  "email": "info@acme.com",
  "mobile": "+1-555-0600",
  "address": "42 Enterprise Rd",
  "facebookLink": "https://facebook.com/acme",
  "youtubeLink": "https://youtube.com/acme",
  "websiteLink": "https://acme.com",
  "imageUrl": "https://cdn.app/acme.png"
}
```

### POST `/api/venues`
- **DTO**: `EventVenueDTO`
- **Entities**: `EventVenue`
- **Sample Request**
```json
{
  "typeCode": "ARENA",
  "typeName": "Arena",
  "venueCode": "VEN-001",
  "venueName": "Metro Arena",
  "address": "500 Center Plaza",
  "email": "contact@metroarena.com",
  "phone": "+1-555-0700",
  "maxCapacity": "5000",
  "mapAddress": "https://maps.example.com/metro",
  "socialMediaLink": "https://instagram.com/metroarena",
  "websiteLink": "https://metroarena.com"
}
```

---

## 5. Seat Layout & Seat APIs

### POST `/api/venues/{venueId}/layouts`
- **DTO**: `SeatLayoutDTO`
- **Entities**: `SeatLayout`, `EventVenue`
- **Sample Request**
```json
{
  "typeCode": "THEATER",
  "typeName": "Theater",
  "layoutName": "Main Floor",
  "totalRows": 20,
  "totalCols": 30,
  "totalCapacity": 600,
  "isActive": true
}
```

### PUT `/api/seat-layouts/{id}/banquet`
- **DTO**: `BanquetLayoutDTO`
- **Entities**: `SeatLayout`, `SeatEntity`
- **Sample Request**
```json
{
  "tables": [
    {
      "label": "T1",
      "tierCode": "VIP",
      "x": 200,
      "y": 150,
      "chairs": [
        {"label": "T1-C1", "angle": 0, "offsetX": 0, "offsetY": 0},
        {"label": "T1-C2", "angle": 60, "offsetX": 0, "offsetY": 0}
      ]
    }
  ]
}
```

### POST `/api/seat-layouts/{layoutId}/seats`
- **DTO**: `SeatDTO`
- **Entities**: `SeatEntity`, `SeatLayout`
- **Sample Request**
```json
{
  "row": "A",
  "number": 1,
  "label": "A-1",
  "type": "REGULAR"
}
```

---

## 6. Events (`/api/events`)

### POST `/api/events`
- **DTOs**: `CreateEventRequest`, `CreateEventTicketTierRequest`
- **Entities**: `EventEntity`, `EventTicketTier`, `EventSeatEntity`, link tables
- **Sample Request (trimmed)**
```json
{
  "typeCode": "CONCERT",
  "typeName": "Concert",
  "eventCode": "EVT-2024-0001",
  "eventName": "Jazz Night",
  "eventDescription": "Live jazz",
  "eventStart": "2024-07-01T19:00:00Z",
  "eventEnd": "2024-07-01T22:00:00Z",
  "venueId": "...",
  "seatLayoutId": "...",
  "eventManager": "...",
  "eventOperator1": "...",
  "eventChecker1": "...",
  "ticketTiers": [
    {"tierCode": "VIP", "tierName": "VIP", "totalQuantity": 50, "price": 150.0, "cost": 60.0},
    {"tierCode": "GEN", "tierName": "General", "totalQuantity": 450, "price": 50.0, "cost": 15.0}
  ],
  "artistIds": ["..."],
  "sponsorIds": ["..."],
  "organizerIds": ["..."]
}
```

### GET `/api/events/{id}`
- **Response DTO**: `EventResponse`
- **Entities**: `EventEntity`, `EventTicketTier`, `EventVenue`
- **Sample Response**
```json
{
  "id": "event-uuid",
  "eventCode": "EVT-2024-0001",
  "eventName": "Jazz Night",
  "eventDescription": "Live jazz",
  "venueId": "venue-uuid",
  "venueName": "Metro Arena",
  "eventStart": "2024-07-01T19:00:00Z",
  "eventEnd": "2024-07-01T22:00:00Z",
  "ticketTiers": [
    {"tierCode": "VIP", "price": 150.0, "totalQuantity": 50},
    {"tierCode": "GEN", "price": 50.0, "totalQuantity": 450}
  ]
}
```

### GET `/api/events/{id}/ticket-details`
- **Response DTO**: `EventTicketDetailsResponse`
- **Entities**: `TicketEntity`, `ReservationHoldEntity`
- **Sample Response**
```json
{
  "eventId": "event-uuid",
  "ticketsIssued": 410,
  "ticketsUsed": 390,
  "ticketsRefunded": 5,
  "grossRevenue": 82000.0,
  "netRevenue": 80500.0
}
```

### GET `/api/events/{id}/seats`
- **Response DTO**: `List<EventSeatResponse>`
- **Entities**: `EventSeatEntity`, `SeatEntity`
- **Sample Response**
```json
[
  {
    "eventSeatId": "event-seat-uuid",
    "seatId": "seat-uuid",
    "label": "A-1",
    "tierCode": "VIP",
    "price": 150.0,
    "status": "AVAILABLE"
  }
]
```

### GET `/api/events/{id}/seat-map`
- **DTO**: `EventSeatMapResponse`
- **Entities**: `EventSeatEntity`, `SeatLayout`, `EventTicketTier`
- **Sample Response (excerpt)**
```json
{
  "eventId": "...",
  "layout": {"layoutName": "Main Floor", "totalRows": 20, "totalCols": 30},
  "tiers": [
    {"tierCode": "VIP", "price": 150.0, "totalQuantity": 50},
    {"tierCode": "GEN", "price": 50.0, "totalQuantity": 450}
  ],
  "seats": [
    {"eventSeatId": "...", "seatId": "...", "label": "A-1", "tierCode": "VIP", "price": 150.0, "status": "AVAILABLE"}
  ]
}
```

### POST `/api/events/{id}/seats/sync`
- **DTO**: `SeatInventorySyncRequest`
- **Entities**: `EventSeatEntity`, `SeatEntity`, `EventTicketTier`
- **Sample Request**
```json
{
  "tierCode": "GEN",
  "price": 45.0,
  "overwriteExisting": true,
  "removeMissing": false
}
```

### PUT `/api/events/{id}/seats/assignments`
- **DTO**: `SeatAssignmentUpdateRequest`
- **Entities**: `EventSeatEntity`, `SeatEntity`, `EventTicketTier`
- **Sample Request**
```json
{
  "seats": [
    {
      "eventSeatId": "...",
      "tierCode": "VIP",
      "price": 160.0,
      "status": "AVAILABLE"
    }
  ]
}
```

### PUT `/api/events/{id}`
- **DTO**: `UpdateEventRequest`
- **Entities**: `EventEntity`, `EventTicketTier`, link tables
- **Sample Request**
```json
{
  "eventName": "Jazz Night - Late Show",
  "eventEnd": "2024-07-02T00:00:00Z",
  "ticketTiers": [
    {"id": "...", "tierCode": "VIP", "price": 170.0, "totalQuantity": 60}
  ]
}
```

- **Entities**: `EventEntity`
### GET `/api/events`
- **Entities**: `EventEntity`
- **Query Params**: pageable (`page`, `size`, `sort`). Default `size=20`, `sort=eventStart`.
- **Sample Response**
```json
{
  "content": [
    {"id": "event-uuid", "eventName": "Jazz Night", "eventStart": "2024-07-01T19:00:00Z"}
  ],
  "pageable": {"pageNumber": 0, "pageSize": 20},
  "totalElements": 12,
  "totalPages": 1
}
```

---

## 7. Reservation Holds (`/api/holds`)

### POST `/api/holds`
- **DTO**: `HoldCreateRequest`
- **Entities**: `ReservationHoldEntity`, `EventSeatEntity`
- **Sample Request (explicit seats)**
```json
{
  "eventId": "...",
  "buyerId": "...",
  "seatIds": ["seat-uuid-1", "seat-uuid-2"],
  "expiresAt": "2024-05-15T10:15:00Z"
}
```
- **Sample Request (tiers)**
```json
{
  "eventId": "...",
  "tierSelections": [
    {"tierCode": "VIP", "quantity": 2},
    {"tierCode": "GEN", "quantity": 3}
  ],
  "expiresAt": "2024-05-15T10:15:00Z"
}
```
- **Sample Response**
```json
{
  "id": "hold-uuid",
  "status": "ACTIVE",
  "heldSeats": [
    {"seatId": "seat-uuid-1", "seatLabel": "A-1", "tierCode": "VIP"}
  ],
  "expiresAt": "2024-05-15T10:15:00Z"
}
```

### POST `/api/holds/release`
- **DTO**: `HoldReleaseRequest`
- **Entities**: `ReservationHoldEntity`, `EventSeatEntity`
- **Sample Request**
```json
{
  "holdId": "hold-uuid",
  "reason": "Customer canceled"
}
```

### POST `/api/holds/convert`
- **DTO**: `HoldConvertRequest`
- **Entities**: `ReservationHoldEntity`
- **Sample Request**
```json
{
  "holdId": "hold-uuid",
  "paymentId": "payment-uuid"
}
```

### GET `/api/holds/{holdId}`
- **Response**: `HoldResponse`
- **Entities**: `ReservationHoldEntity`
- **Sample Response**
```json
{
  "id": "hold-uuid",
  "eventId": "event-uuid",
  "status": "ACTIVE",
  "heldSeats": [
    {"seatId": "seat-uuid-1", "seatLabel": "A-1", "tierCode": "VIP"}
  ],
  "expiresAt": "2024-05-15T10:15:00Z"
}
```

### GET `/api/holds/events/{eventId}`
- **Response**: `List<HoldResponse>`
- **Entities**: `ReservationHoldEntity`
- **Sample Response**
```json
[
  {
    "id": "hold-uuid",
    "status": "ACTIVE",
    "heldSeats": [
      {"seatLabel": "A-1", "tierCode": "VIP"}
    ],
    "expiresAt": "2024-05-15T10:15:00Z"
  }
]
```

---

## 8. Tickets (`/api/tickets`)

### POST `/api/tickets/reserve`
- **DTO**: `TicketCreateRequest`
- **Entities**: `TicketEntity`, `EventSeatEntity`
- **Sample Request**
```json
{
  "eventId": "...",
  "seatId": "seat-uuid-1",
  "buyerId": "buyer-uuid",
  "reservedUntil": "2024-05-15T11:00:00Z",
  "holderName": "Alex Guest",
  "holderEmail": "alex@example.com"
}
```
- **Sample Response**
```json
{
  "id": "ticket-uuid",
  "status": "PENDING",
  "eventId": "...",
  "seatLabel": "A-1",
  "tierCode": "VIP",
  "qrCode": "7bb0...",
  "verificationCode": "AB12CD34",
  "reservedUntil": "2024-05-15T11:00:00Z"
}
```

### POST `/api/tickets/issue/{ticketId}`
- **Response**: `TicketResponse` with status `ISSUED`
- **Entities**: `TicketEntity`, `EventSeatEntity`, `EventTicketTier`

### POST `/api/tickets/checkin/{ticketId}`
- **DTO**: `TicketCheckInRequest`
- **Entities**: `TicketEntity`, `UserEntity`
- **Sample Request**
```json
{
  "checkerId": "checker-uuid",
  "gate": "Gate A"
}
```

### POST `/api/tickets/refund/{ticketId}`
- **DTO**: `TicketRefundRequest`
- **Entities**: `TicketEntity`, `EventSeatEntity`, `EventTicketTier`
- **Sample Request**
```json
{
  "refundAmount": 50.0
}
```

### GET `/api/tickets`
- **Query Params**: `eventId` or `buyerId`
- **Response**: `List<TicketResponse>`
- **Entities**: `TicketEntity`, `EventSeatEntity`
- **Sample Response**
```json
[
  {
    "id": "ticket-uuid",
    "eventId": "event-uuid",
    "buyerId": "buyer-uuid",
    "seatLabel": "A-1",
    "status": "ISSUED",
    "issuedAt": "2024-05-01T12:00:00Z"
  }
]
```

---

## 9. Payments (`/api/payments`)

### POST `/api/payments/intents`
- **DTOs**: `CreatePaymentIntentRequest` ➝ `CreatePaymentIntentResponse`
- **Sample Request**
```json
{
  "holdId": "hold-uuid",
  "currency": "usd",
  "customerEmail": "buyer@example.com",
  "description": "Hold payment for Jazz Night"
}
```
- **Sample Response**
```json
{
  "paymentId": "payment-uuid",
  "paymentIntentId": "pi_3Oo...",
  "clientSecret": "pi_3Oo_secret_abc",
  "amountCents": 32500,
  "currency": "usd",
  "status": "REQUIRES_PAYMENT_METHOD"
}
```

### POST `/api/payments/webhook`
- **Headers**: `Stripe-Signature`
- **Payload**: Raw Stripe event JSON (no DTO)
- **Entities**: `PaymentEntity`, `ReservationHoldEntity`, `EventSeatEntity`
- **Sample Payload (truncated)**
```json
{
  "id": "evt_1Pxxxx",
  "type": "payment_intent.succeeded",
  "data": {
    "object": {
      "id": "pi_3Oo...",
      "status": "succeeded",
      "metadata": {
        "paymentId": "payment-uuid",
        "holdId": "hold-uuid"
      }
    }
  }
}
```

---

## 10. Notifications & Tests

- **EmailService**: emits verification & reset links (`/api/auth/verify`, `/api/auth/reset-password`).
- **TestController**: `/api/test/ok`, `/api/test/slow`, `/api/test/error` for sanity checks.

---

## 11. Endpoint Role Matrix

| Endpoint Group | Required Auth |
| -------------- | ------------- |
| `/api/auth/*` | Public |
| `/api/users/me` | JWT (any authenticated user) |
| `/api/users/**` (create/delete/list) | JWT (`ROLE_ORG_ADMIN`) |
| `/api/admin/dashboard/*` | JWT (`ROLE_ORG_ADMIN`) |
| `/api/artists`, `/api/sponsors`, `/api/business-organizations`, `/api/venues` | JWT (`ROLE_ORG_ADMIN`) |
| `/api/seat-layouts/**`, `/api/seat-layouts/{layoutId}/seats` | JWT (`ROLE_ORG_ADMIN` or delegated event manager) |
| `/api/events/**` | JWT (`ROLE_ORG_ADMIN`/`ROLE_EVENT_MANAGER`) depending on op |
| `/api/holds/**`, `/api/tickets/**` | JWT (`ROLE_EVENT_MANAGER`/`ROLE_OPERATOR`) |
| `/api/payments/intents` | JWT (seller roles), `/api/payments/webhook` uses Stripe signature |
| `/api/test/*` | Public |

---

## 12. DTO Appendix

| DTO | Key Fields ( * = required ) | Java File |
| --- | -------------------------- | --------- |
| `SignupRequest` | `username*`, `email*`, `password*` | `auth/dto/SignupRequest.java` |
| `LoginRequest` | `email*`, `password*` | `auth/dto/LoginRequest.java` |
| `ChangePasswordRequest` | `currentPassword*`, `newPassword*` | `auth/dto/ChangePasswordRequest.java` |
| `ForgotPasswordRequest` | `email*` | `auth/dto/ForgotPasswordRequest.java` |
| `ResetPasswordRequest` | `token*`, `newPassword*` | `auth/dto/ResetPasswordRequest.java` |
| `UserResponse` | identity, role, org metadata | `users/dto/UserResponse.java` |
| `UpdateUserRequest` | `phone`, `fullName`, `imageUrl` | `users/dto/UpdateUserRequest.java` |
| `CreateEventManagerRequest` | `email*`, `password*`, `username*`, `firstName*`, `lastName*` | `users/dto/CreateEventManagerRequest.java` |
| `CreateEventRequest` | `typeCode*`, `typeName*`, `eventCode*`, `eventName*`, `eventStart*`, `eventEnd*`, `venueId*`, staffing UUIDs*, `ticketTiers*` | `events/dto/CreateEventRequest.java` |
| `CreateEventTicketTierRequest` | `tierCode*`, `tierName*`, `totalQuantity*`, `price*`, `cost*` | `events/dto/CreateEventTicketTierRequest.java` |
| `UpdateEventRequest` | optional updates for identity/schedule/venue/staff/tiers | `events/dto/UpdateEventRequest.java` |
| `SeatInventorySyncRequest` | `tierCode`, `price`, `overwriteExisting`, `removeMissing` | `events/dto/SeatInventorySyncRequest.java` |
| `SeatAssignmentUpdateRequest.SeatAssignment` | `eventSeatId?`, `seatId?`, `tierCode*`, `price`, `status` | `events/dto/SeatAssignmentUpdateRequest.java` |
| `EventVenueDTO` | venue metadata fields | `venues/dto/EventVenueDTO.java` |
| `SeatLayoutDTO` | `typeCode*`, `typeName*`, `layoutName*`, capacity metrics, `isActive*` | `seat/dto/SeatLayoutDTO.java` |
| `SeatDTO` | `row*`, `number*`, `label`, `type` | `seat/dto/SeatDTO.java` |
| `BanquetLayoutDTO` | tables & chairs geometry | `seat/dto/BanquetLayoutDTO.java` |
| `HoldCreateRequest` | `eventId*`, `buyerId`, `seatIds` xor `tierSelections`, `expiresAt*` | `ticketing/dto/HoldCreateRequest.java` |
| `HoldConvertRequest` | `holdId*`, `paymentId*` | `ticketing/dto/HoldConvertRequest.java` |
| `TicketCreateRequest` | `eventId*`, `seatId*`, `buyerId*`, `reservedUntil`, `holderName`, `holderEmail` | `ticketing/dto/TicketCreateRequest.java` |
| `TicketCheckInRequest` | `checkerId*`, `gate` | `ticketing/dto/TicketCheckInRequest.java` |
| `TicketRefundRequest` | `refundAmount*` | `ticketing/dto/TicketRefundRequest.java` |
| `CreatePaymentIntentRequest` | `holdId*`, `currency`, `customerEmail`, `description` | `payments/dto/CreatePaymentIntentRequest.java` |
| `CreatePaymentIntentResponse` | `paymentId`, `paymentIntentId`, `clientSecret`, `amountCents`, `status` | `payments/dto/CreatePaymentIntentResponse.java` |

---

## 13. Workflow Overview
1. Sign up/login to obtain JWT.
2. Org admin provisions staff accounts.
3. Create venues, layouts, and seats.
4. Create events and sync/assign seats.
5. Sell tickets via holds + Stripe payments + ticket issuance.
6. Monitor dashboards and holds.
7. Use EmailService + `/api/test/*` endpoints for support/testing.
