# Event Ticketing API – Design

## Goal

A REST API for selling seats to events. Admins create events with a seat map.
Users browse events, hold seats for a limited time, confirm the booking with a
mock payment, and cancel it if needed. The main technical goal is to make sure
the same seat can never be sold twice, even when many users try to book it at
the same moment.

## Scope

In scope:

- Registration and login with JWT
- Admin-only event creation with generated seats
- Public event and seat listing
- Seat hold, confirmation, cancellation and automatic expiry
- Consistent error responses and API documentation (Swagger UI)

Out of scope, on purpose:

- Real payments (confirmation is a mock that always succeeds)
- Refresh tokens, password reset and email notifications
- Frontend
- Multiple price categories per event (one seat price per event)

## Tech stack

| Technology | Why |
|---|---|
| Java 21 | Current LTS; records for DTOs |
| Spring Boot 4.1 | Standard for Java backends; fast setup |
| Spring Web, Bean Validation | REST endpoints and request validation |
| Spring Data JPA (Hibernate) | Entity mapping and optimistic locking with `@Version` |
| PostgreSQL | Reliable relational database with strong transaction support |
| Flyway | Versioned, reviewable schema changes instead of `ddl-auto=update` |
| Spring Security, OAuth2 Resource Server | Stateless JWT authentication (HS256) |
| springdoc-openapi | Swagger UI generated from the code |
| JUnit 5, Mockito, Testcontainers | Unit tests and integration tests against a real PostgreSQL |
| Docker Compose | One command to run the database and the app locally |
| GitHub Actions | Runs `mvn verify` on every push and pull request |

## Architecture

The application is a modular monolith: one deployable app, split into
packages by feature instead of by layer.

```
com.kagan.ticketing
├── user      registration, login, JWT
├── event     events and seats
├── booking   hold, confirm, cancel, expiry
└── common    error handling, security config, shared utilities
```

Each module has its own controller, service, repository, entities and DTOs.
Rules:

- Controllers handle HTTP only. Business rules live in services.
- Responses are DTOs (records), never entities.
- A module calls another module through its service, not its repository.
- Schema changes are made only with Flyway migrations.
  Hibernate runs with `ddl-auto=validate`.
- `spring.jpa.open-in-view` is disabled, so lazy loading outside a
  transaction fails fast instead of running hidden queries.

## Data model

```
User 1 ──── * Booking * ──── 1 Event
                1                1
                │                │
                *                *
               Seat ─────────────┘
```

**users**

| Column | Type | Notes |
|---|---|---|
| id | bigint | PK |
| email | varchar | unique, stored lowercase |
| password_hash | varchar | BCrypt |
| role | varchar | `USER` or `ADMIN` |
| created_at | timestamptz | |

**events**

| Column | Type | Notes |
|---|---|---|
| id | bigint | PK |
| title | varchar | |
| venue_name | varchar | |
| starts_at | timestamptz | must be in the future when created |
| seat_price | numeric(10,2) | one price for every seat |
| row_count | int | 1–26 (rows are labelled A–Z) |
| seats_per_row | int | 1–50 |
| created_at | timestamptz | |

**seats**

| Column | Type | Notes |
|---|---|---|
| id | bigint | PK |
| event_id | bigint | FK → events |
| label | varchar | for example `A1`, `C12`; unique per event |
| status | varchar | `AVAILABLE`, `HELD`, `BOOKED` |
| booking_id | bigint | FK → bookings, null when available |
| version | bigint | optimistic locking |

**bookings**

| Column | Type | Notes |
|---|---|---|
| id | bigint | PK |
| user_id | bigint | FK → users |
| event_id | bigint | FK → events |
| status | varchar | `HELD`, `CONFIRMED`, `CANCELLED`, `EXPIRED` |
| expires_at | timestamptz | hold deadline |
| total_price | numeric(10,2) | seat count × seat price |
| created_at | timestamptz | |
| version | bigint | optimistic locking |

Enums are stored as strings, not ordinals. All `@ManyToOne` relations are
`LAZY`; queries that need related data use `join fetch`.

Seats are generated when the event is created: `row_count × seats_per_row`
rows in one transaction.

## API

Base path: `/api/v1`

| Method | Path | Access | Description |
|---|---|---|---|
| POST | `/auth/register` | public | Create a `USER` account |
| POST | `/auth/login` | public | Return a JWT |
| POST | `/events` | `ADMIN` | Create an event and its seats |
| GET | `/events` | public | List upcoming events (paged) |
| GET | `/events/{id}` | public | Event details |
| GET | `/events/{id}/seats` | public | Seat map with statuses |
| POST | `/bookings` | `USER` | Hold seats for 10 minutes |
| POST | `/bookings/{id}/confirm` | owner | Confirm with mock payment |
| POST | `/bookings/{id}/cancel` | owner | Cancel a held or confirmed booking |
| GET | `/bookings/me` | `USER` | Current user's bookings (paged) |

Authentication:

- Login returns a signed JWT (HS256) valid for 60 minutes. The token carries
  the user id as subject and the role as a claim.
- The signing secret comes from the `JWT_SECRET` environment variable and is
  never committed.
- Registration always creates a `USER`. The single admin account is created
  at startup from `ADMIN_EMAIL` and `ADMIN_PASSWORD` if it does not exist.

## Booking flow and concurrency

### Holding seats

`POST /bookings` with `eventId` and a list of `seatIds`:

1. Validate the request: 1–10 seats, no duplicates.
2. Reject if the event has already started.
3. Load the seats and check that all of them belong to the event and are
   `AVAILABLE`. If any is not, reject the whole request.
4. Create a `HELD` booking with `expires_at = now + 10 minutes` and mark every
   seat `HELD`, linked to the booking.
5. Flush inside the same transaction.

The request is all-or-nothing: either every seat is held, or none is.

### Two users, one seat

Each seat row has a `version` column (`@Version`). Hibernate updates a seat
with `WHERE id = ? AND version = ?`. When two requests read the same
available seat, the first update succeeds and increases the version. The
second update matches zero rows, Hibernate throws an optimistic locking
exception, the transaction rolls back, and the client gets
`409 SEAT_UNAVAILABLE`.

Optimistic locking is chosen over pessimistic locking (`SELECT ... FOR
UPDATE`) because conflicts are rare for most seats, and it does not keep
rows locked while a request is running. The loser gets a clear error and can
pick another seat; there is no automatic retry.

### Confirming

`POST /bookings/{id}/confirm`:

- Only the owner can confirm. Another user's booking returns `404`, not
  `403`, so the API does not reveal which booking ids exist.
- The booking must be `HELD` and `expires_at` must be in the future.
  Otherwise the request fails with `409 BOOKING_EXPIRED` or
  `409 INVALID_BOOKING_STATE`.
- The mock payment always succeeds. Seats become `BOOKED`, the booking becomes
  `CONFIRMED`.

### Cancelling

`POST /bookings/{id}/cancel`:

- Allowed for `HELD` and `CONFIRMED` bookings of the owner, until the event
  starts.
- Seats return to `AVAILABLE` and are unlinked. The booking becomes
  `CANCELLED`.

### Expiry

A `@Scheduled` job runs every 30 seconds:

1. Find `HELD` bookings whose `expires_at` has passed.
2. Expire each booking in its own transaction: seats return to `AVAILABLE`,
   the booking becomes `EXPIRED`.

Each booking is processed separately so one failure does not roll back the
others. The per-booking method lives in a separate bean, because Spring's
`@Transactional` does not apply to calls inside the same class.

If a user confirms at the same moment the job expires the booking, the
booking's `version` makes one of them fail. The confirm endpoint also checks
`expires_at` itself, so a late confirmation is rejected even before the job
runs.

Time comes from an injected `Clock` bean, so tests can control "now".

## Error handling

Errors use the RFC 9457 Problem Details format (`ProblemDetail`) with an
extra `code` field that clients can rely on.

```json
{
  "type": "about:blank",
  "title": "Conflict",
  "status": 409,
  "detail": "Seat A3 is no longer available",
  "code": "SEAT_UNAVAILABLE"
}
```

| Status | Code | When |
|---|---|---|
| 400 | `VALIDATION_FAILED` | Invalid request body or parameters |
| 401 | `UNAUTHORIZED` | Missing or invalid token |
| 401 | `INVALID_CREDENTIALS` | Wrong email or password |
| 403 | `FORBIDDEN` | Valid token, wrong role |
| 404 | `EVENT_NOT_FOUND` | Unknown event |
| 404 | `BOOKING_NOT_FOUND` | Unknown booking, or someone else's booking |
| 409 | `EMAIL_ALREADY_USED` | Registration with an existing email |
| 409 | `SEAT_UNAVAILABLE` | A requested seat is held, booked or taken concurrently |
| 409 | `EVENT_ALREADY_STARTED` | Booking or cancelling after the event start |
| 409 | `BOOKING_EXPIRED` | Confirming after the hold deadline |
| 409 | `INVALID_BOOKING_STATE` | Action not allowed in the booking's current status |

## Testing

| Test type | What it proves |
|---|---|
| Unit tests (Mockito) | Business rules in services: seat limit, event start, booking states |
| `@WebMvcTest` | Status codes, validation errors and JSON shape of controllers |
| Integration tests (Testcontainers) | The full app against real PostgreSQL, including Flyway migrations and security |
| Concurrency test | 20 threads book the same seat: exactly 1 succeeds, 19 get `409` |

Locking and migrations are tested against a real database on purpose: a
mocked repository cannot prove that `@Version` works.

GitHub Actions runs `mvn verify` for every push and pull request.

## Trade-offs

- **Expired holds can stay visible for up to 30 seconds.** The job runs
  every 30 seconds, so a seat may still show `HELD` after its deadline.
  Confirmation checks `expires_at` directly, so this never lets a late
  confirmation through.
- **No distributed lock for the scheduled job.** With several app instances,
  each would run the job. Optimistic locking keeps the data correct, but work
  is duplicated. A tool like ShedLock would fix this; it is not needed for a
  single instance.
- **The database is the bottleneck under very high load.** Every hold is a
  database transaction. A very popular event would need a queue or a
  cache-based reservation layer, which is beyond this project.
- **No automatic retry on conflict.** A user who loses a race gets `409` and
  chooses again, which is simpler and honest about what happened.

## Delivery plan

Work is delivered in small commits, each building and passing tests:

1. Project setup
2. Design document
3. PostgreSQL with Docker Compose and Flyway
4. CI with GitHub Actions
5. Global error handling with Problem Details
6. User registration
7. JWT login
8. Admin event creation with seat generation
9. Public event and seat queries
10. Seat hold
11. Concurrency test
12. Booking confirmation
13. Cancellation and user's bookings
14. Hold expiry job
15. OpenAPI documentation
16. Dockerfile and full Docker Compose setup
17. README
