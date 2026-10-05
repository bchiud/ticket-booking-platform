# Ticket Booking Platform — High-Level System Design

A walkthrough of the classic "design a ticket booking system" problem.

## 1. Scope & Requirements

**Functional**

- Browse/search events (concerts, sports, theater) by name, city, date, artist
- View an event's venue, seat map, and availability
- **Book seats** — select seats, hold them, pay, confirm
- View booked tickets

**Non-functional (this is where it gets interesting)**

- **High read volume** — browsing/search dwarfs booking (~100:1 or worse)
- **Strong consistency on booking** — a seat must never be sold twice (double-booking is the cardinal sin)
- **High concurrency spikes** — Taylor Swift on-sale = millions hitting one event at t=0
- High availability for reads; correctness > availability for writes

The core tension: **reads want to be cached and eventually-consistent; bookings demand strict serialization on a hot,
contended resource.**

## 2. Core Entities

```
Event      → (name, date, venue_id, performer)
Venue      → (name, address, seat_map)
Seat       → (venue_id, section, row, number)
Ticket     → (event_id, seat_id, price, status, booking_id)   ← status: AVAILABLE / HELD / BOOKED
Booking    → (user_id, event_id, status, total)   ← its tickets link back via Ticket.booking_id
User, Payment
```

A **Ticket** is a (seat × event) instance — that's the unit you actually sell and lock.

## 3. High-Level Architecture

```
                    ┌─────────┐
   Clients ──CDN──▶ │  API GW │──▶ Load Balancer
                    └─────────┘
                         │
      ┌──────────────┬───┴────────┬──────────────┐
      ▼              ▼            ▼               ▼
  Search Svc    Event Svc    Booking Svc      Payment Svc
      │              │            │               │
  Elasticsearch   DB(read     DB(events,       Stripe/
   + cache        replicas)   tickets)         gateway
                                  │
                          Redis (seat holds/locks)
```

Split reads from writes into separate services so you can scale and optimize them independently.

## 4. The Read Path (browse & search)

- **Search Service** backed by **Elasticsearch** for full-text/faceted search (by artist, city, date range).
- **Event/availability reads** served from **read replicas + a cache (Redis/CDN)**. Seat maps and event metadata are
  cached aggressively.
- Availability counts can be *slightly* stale on the browse page — that's fine. Truth is enforced at booking time. This
  lets you serve the firehose of browsers cheaply.

> **As built:** this project serves browse/search with structured Postgres filtering (JPA
> Specifications over name / status / city / performer / date range), paginated and sortable
> (`?page` / `size` / `sort`), rather than a dedicated Elasticsearch service — see
> [ADR-0004](adr/0004-event-search-jpa-specifications.md). Elasticsearch, read replicas, and the
> CDN/cache tier above remain out-of-scope scale options.

## 5. The Write Path — Booking (the hard part)

This is a **reserve → pay → confirm** flow with a seat *hold*:

1. User selects seats → **Booking Service attempts to HOLD them** (e.g. 5–10 min TTL).
2. User has that window to pay.
3. On payment success → tickets flip to **BOOKED**. On timeout/failure → hold expires, seats return to AVAILABLE.

**How to enforce "no double-booking"** — a few options, in order of preference:

- **Database transaction with row locking** (`SELECT ... FOR UPDATE` on the ticket rows). The DB is the source of truth.
  Wrap select-availability + mark-held in one atomic transaction so two buyers can't grab the same seat. This is the
  safest default.
- **Redis distributed lock / atomic `SETNX` per seat** with a TTL for the hold — fast, and the TTL auto-releases
  abandoned holds without a cleanup job. Back it with the DB as the durable record.
- **Optimistic concurrency** (version column / conditional update) — the loser of a race just retries with fresh
  availability.

The hold TTL is the elegant trick: it prevents inventory from being locked forever by people who abandon checkout, and
it naturally handles crashes.

## 6. Handling the On-Sale Spike (Taylor Swift problem)

When millions hit one event simultaneously:

- **Virtual waiting queue** — admit users to the booking flow in controlled batches rather than letting all traffic hit
  the DB at once. This is Ticketmaster's actual "Smart Queue."
- **Rate limiting** per user/IP to blunt bots.
- The hot event's tickets are a contention hotspot → keep hold logic in **Redis** (in-memory, single-threaded atomic
  ops) to absorb the concurrency, with async write-through to the DB.

## 7. Data Storage Choices

- **Bookings/tickets/payments → relational DB (Postgres/MySQL)** — you need ACID transactions and strong consistency for
  money and seats.
- **Search → Elasticsearch.**
- **Holds & counters → Redis.**
- **Shard the transactional DB by event_id** — booking contention is naturally partitioned per event, so events scale
  independently.

## 8. Key Trade-off

> Reads are optimized for scale and tolerate staleness (cache/replicas/ES). Writes are optimized for correctness via a
> small strongly-consistent core (transactional DB + short-lived Redis holds). The waiting queue converts an unbounded
> concurrency spike into a manageable, throttled stream.

---

# Deep Dives

## 9. Seat-Hold Locking Logic

The booking flow is **reserve → pay → confirm**. The hold is a short-lived, exclusive claim on a ticket that must
survive races between concurrent buyers and clean itself up if the buyer disappears.

### 9.1 State machine per ticket

```
AVAILABLE ──hold()──▶ HELD ──confirm()──▶ BOOKED
    ▲                  │
    └──── expire() ────┘   (TTL elapsed or user cancelled)
```

Races come in two shapes, and each gets a different lock. **Contention for the shared seat pool** — two buyers `hold()`
the same seat — is handled *pessimistically* with row locks (§9.2). **Concurrent transitions on one already-owned
booking** — `pay()` racing `cancel()`, or confirming an already-expired hold — are handled *optimistically* with a
`version` column (§9.6). Both must be atomic.

### 9.2 Option A — DB transaction with row locking (source of truth)

Postgres example. `SELECT ... FOR UPDATE` locks the candidate rows so a concurrent transaction blocks until this one
commits.

```sql
BEGIN;

-- Lock the exact seats the user asked for. NOWAIT = fail fast instead of
-- queueing behind another buyer (better UX: "seat just taken, pick again").
SELECT id, status
FROM tickets
WHERE event_id = :event_id
  AND id = ANY (:seat_ids)
    FOR UPDATE NOWAIT;

-- Application checks every returned row is AVAILABLE; if not, ROLLBACK.

-- Insert the booking first so the held tickets can point at it (tickets.booking_id FK).
INSERT INTO bookings (id, user_id, event_id, status, expires_at)
VALUES (:booking_id, :user_id, :event_id, 'PENDING', now() + interval '8 minutes');

UPDATE tickets
SET status     = 'HELD',
    booking_id = :booking_id
WHERE id = ANY (:seat_ids)
  AND status = 'AVAILABLE';
-- guard: only flips rows still free
-- The hold window lives on the booking (bookings.expires_at); tickets carry no per-ticket
-- expiry column, so the sweep is booking-level (see §9.4).

-- Row count must equal len(seat_ids). If fewer, someone raced us → ROLLBACK.

COMMIT;
```

Pros: single source of truth, ACID, no reconciliation.
Cons: the `FOR UPDATE` lock is a contention point on a hot event — this is what the waiting queue and Redis front-line
exist to protect.

### 9.3 Option B — Redis atomic hold (front-line for hot events)

Use one key per seat; `SET NX` (set-if-absent) is atomic, and the TTL auto-releases abandoned holds with **no cleanup
job**. To hold multiple seats atomically (all-or-nothing), use a Lua script so the whole batch is one atomic operation:

```lua
-- KEYS = seat lock keys, ARGV[1] = user_id, ARGV[2] = ttl_seconds
-- Phase 1: verify every seat is free
for i, key in ipairs(KEYS) do
    if redis.call('EXISTS', key) == 1 then
        return { err = 'SEAT_TAKEN', seat = key }
    end
end
-- Phase 2: claim them all
for i, key in ipairs(KEYS) do
    redis.call('SET', key, ARGV[1], 'EX', ARGV[2])
end
return 'OK'
```

Redis is the fast gatekeeper; the DB is still written asynchronously as the durable record. If Redis says OK, persist
the hold to the DB; if the DB write fails, release the Redis keys.

> **Implemented: Option A (§9.2)** — a pessimistic row lock on the ticket rows in `BookingService.hold()`
> (`TicketRepository.findByIdIn`, `@Lock(PESSIMISTIC_WRITE)` + `lock.timeout=0`), which Hibernate 7 emits on
> Postgres as `SELECT … FOR NO KEY UPDATE NOWAIT`: still exclusive between two holds, but it doesn't block
> inserts that only reference the row by foreign key. Redis is
> used only for the queue and admission tokens (§11), not seat holds.
>
> **Option B (§9.3, Redis seat holds) is deferred, not replaced by §11** — they attack a hot event
> at different layers:
> - **§9.3 makes each hold *faster*** — in-memory `SETNX`, no DB row lock.
> - **§11 reduces the *number* of concurrent holds** — admission throttling, upstream of the hold.
>
> They're orthogonal. Here §11 thins the load enough that the §9.2 lock isn't a bottleneck, so §9.3
> isn't needed yet — it's the next step if one event ever outgrows the DB lock behind the queue.

### 9.4 Confirm & expiry

- **Confirm (after payment):** in one DB transaction, verify the booking is still `PENDING` **and**
  `expires_at > now()`, then flip tickets → `BOOKED` and booking → `CONFIRMED`. If the hold already expired,
  refund/abort — never confirm a stale hold.
- **Mark sold out:** after a confirm, the event flips to `SOLD_OUT` once no ticket remains that could still be sold or
  is mid-sale — i.e. no `AVAILABLE` **and** no `HELD` tickets (`BOOKED` and `CANCELLED` are both terminal). Checking "
  nothing sellable left" rather than "all BOOKED" keeps voided (`CANCELLED`) seats from blocking sold-out, and is a
  single `COUNT` query rather than a full ticket scan. The check first locks and re-reads the event row
  (`EntityManager.refresh(event, PESSIMISTIC_WRITE)`). Without the lock, two confirms for the last two seats each
  see the other's ticket still `HELD` under READ COMMITTED and neither flips the event (write skew); with it, the
  second waits for the first to commit and then counts 0. The re-read matters too: a plain locking query returns
  the copy this transaction already loaded, so a cancellation committed mid-confirm would be overwritten with
  `SOLD_OUT`.
- **Expiry sweep:** Redis TTL handles the in-memory locks automatically. For the DB, run a periodic job (or lazy check
  on read) that finds `PENDING` bookings whose `expires_at < now()` and, one transaction per booking, flips the
  booking → `EXPIRED` and its tickets back to `AVAILABLE`. The hold window lives on the **booking** (
  `bookings.expires_at`), so the sweep is booking-level — tickets carry no per-ticket expiry column.
  Both sides re-check: confirm re-checks `expires_at` so a late payment can't confirm a stale hold, and the sweep
  re-checks `status = PENDING` after reloading each booking so a payment that commits mid-sweep isn't expired (the
  reload reads the new `version`, so optimistic locking alone can't catch it).

### 9.5 Idempotency

Client retries and payment webhooks can fire twice, at two points:

- **Hold:** each booking attempt carries an **idempotency key** (`bookings.idempotency_key UNIQUE`); a duplicate hold
  returns the original booking instead of creating a second.
- **Pay:** a booking is paid **at most once**. `pay()` short-circuits — if the booking is already `CONFIRMED` it returns
  it unchanged (no second charge), and rejects non-payable states up front (never charge, then roll back). The invariant
  is backstopped in the DB by a **partial unique index** (`one SUCCEEDED payment per booking`), and a concurrent
  double-pay is caught by the `@Version` lock (§9.6). So a retried `pay` is idempotent — it returns the confirmed
  booking rather than double-charging or erroring. (A client-supplied idempotency key on the payment is the more general
  alternative; scoping to the booking is simpler and fits "a booking is paid once.")

### 9.6 Guarding booking-state transitions (optimistic locking)

`hold()` fights over a shared pool of seats, so it locks pessimistically (§9.2). But once a booking exists, `pay()`,
`cancel()`, and `confirm()` each mutate a **single booking the caller already owns** — there's no pool to contend for,
only the risk that two of them fire at once (e.g. a user cancels while a payment webhook confirms). Taking a pessimistic
row lock on every lifecycle call would be wasteful; instead the `bookings` row carries a `version` column (JPA
`@Version`). Every update is `... WHERE id = ? AND version = ?`: the first writer wins and bumps the version, the second
matches **zero rows** and its transaction is rejected and rolled back — including any ticket changes it made in the same
transaction.

**Rule of thumb:** pessimistic locking where callers contend for a *shared pool*; optimistic locking where they
transition a *single owned aggregate*.

The loser surfaces as a Spring `ObjectOptimisticLockingFailureException` (optimistic) or `CannotAcquireLockException` (
the pessimistic `FOR NO KEY UPDATE NOWAIT` loser). Both are subtypes of `ConcurrencyFailureException`, which
`ApiExceptionHandler` maps to **409 Conflict** ("modified concurrently, please retry") rather than a 500.

## 10. Database Schema

Transactional core in Postgres/MySQL.

```sql
CREATE TABLE venues
(
    id      BIGINT PRIMARY KEY,
    name    TEXT NOT NULL,
    address TEXT,
    city    TEXT
);

CREATE TABLE seats
( -- physical seats, one row per venue seat
    id          BIGINT PRIMARY KEY,
    venue_id    BIGINT REFERENCES venues (id),
    section     TEXT,
    row_label   TEXT,
    seat_number TEXT,
    UNIQUE (venue_id, section, row_lxabel, seat_number)
);

CREATE TABLE events
(
    id             BIGINT PRIMARY KEY,
    name           TEXT        NOT NULL,
    performer      TEXT,
    venue_id       BIGINT REFERENCES venues (id),
    starts_at      TIMESTAMPTZ NOT NULL,
    on_sale_at     TIMESTAMPTZ,
    status         TEXT    DEFAULT 'SCHEDULED', -- SCHEDULED / ON_SALE / SOLD_OUT / CANCELLED
    requires_queue BOOLEAN DEFAULT false        -- if true, booking requires a queue access token (§11)
);
CREATE INDEX idx_events_starts_at ON events (starts_at);

CREATE TABLE tickets
(                                                -- the sellable unit: seat × event
    id          BIGINT PRIMARY KEY,
    event_id    BIGINT REFERENCES events (id),
    seat_id     BIGINT REFERENCES seats (id),
    price_cents INT NOT NULL,
    status      TEXT DEFAULT 'AVAILABLE',        -- AVAILABLE / HELD / BOOKED
    booking_id  BIGINT REFERENCES bookings (id), -- the booking holding it; owns the link (see ADR-0003)
    UNIQUE (event_id, seat_id)                   -- a seat exists once per event
);
-- Hot path index:
CREATE INDEX idx_tickets_event_status ON tickets (event_id, status);

CREATE TABLE users
(
    id    BIGINT PRIMARY KEY,
    email TEXT UNIQUE NOT NULL,
    name  TEXT
);

CREATE TABLE bookings
(
    id              BIGINT PRIMARY KEY,
    user_id         BIGINT REFERENCES users (id),
    event_id        BIGINT REFERENCES events (id),
    -- a booking's tickets are found via tickets.booking_id (the FK is on the many side), not an
    -- array column here — see ADR-0003.
    status          TEXT            DEFAULT 'PENDING', -- PENDING / CONFIRMED / EXPIRED / CANCELLED
    total_cents     INT,
    idempotency_key TEXT UNIQUE,
    expires_at      TIMESTAMPTZ,
    version         BIGINT NOT NULL DEFAULT 0,         -- optimistic-lock counter (JPA @Version); see §9.6
    created_at      TIMESTAMPTZ     DEFAULT now()
);

CREATE TABLE payments
(
    id           BIGINT PRIMARY KEY,
    booking_id   BIGINT REFERENCES bookings (id),
    provider_ref TEXT, -- Stripe charge id, etc.
    amount_cents INT,
    status       TEXT, -- PENDING / SUCCEEDED / FAILED / REFUNDED
    created_at   TIMESTAMPTZ DEFAULT now()
);

-- at most one SUCCEEDED payment per booking (don't double-charge); partial so a booking can
-- still carry FAILED/REFUNDED rows alongside its SUCCEEDED one -- see §9.5
CREATE UNIQUE INDEX uq_payments_one_succeeded_per_booking
    ON payments (booking_id) WHERE status = 'SUCCEEDED';
```

**Notes**

- Pre-generating one `tickets` row per (seat, event) when an event goes on sale makes the seat map a simple query and
  gives you a concrete row to lock. For general-admission (no assigned seats), replace individual rows with an atomic *
  *counter** (`available_count`) decremented under a lock or via Redis.
- **Shard by `event_id`.** Contention is per-event, so events scale independently and a hot event's load stays isolated.
- A booking's tickets are linked by the **`tickets.booking_id`** foreign key (on the many side), not an array column on
  `bookings`. This makes the FK the single source of truth and enforces "a ticket belongs to at most one booking" at the
  DB layer — see [ADR-0003](adr/0003-booking-ticket-association.md). (In the DDL above,
  `tickets.booking_id REFERENCES bookings(id)` is a forward reference; in practice the constraint is added once both
  tables exist.)
- **`bookings.version`** is the optimistic-lock counter (JPA `@Version`). Lifecycle transitions (`pay`/`cancel`/
  `confirm`) update `... WHERE id = ? AND version = ?`, so a concurrent transition is rejected instead of silently
  overwriting — see §9.6. (Adding a `NOT NULL` column to an already-populated table needs a backfill migration;
  `ddl-auto=update` can't do it, which is one reason a real deploy uses Flyway/Liquibase rather than Hibernate DDL.)
- The **partial unique index** on `payments` enforces "at most one `SUCCEEDED` payment per booking" at the DB layer (see
  §9.5). It's *partial* (`WHERE status = 'SUCCEEDED'`) so a booking can still hold `FAILED`/`REFUNDED` rows — a plain
  `UNIQUE(booking_id)` would forbid those. Hibernate's `@Index` can't express a `WHERE` predicate, so this lives in
  `schema.sql` (and would be a Flyway migration in production).

## 11. Waiting-Queue Design (the on-sale spike)

Goal: when millions arrive at `on_sale_at`, protect the booking DB by converting the mob into a throttled stream — and
keep it fair (roughly first-come-first-served) and bot-resistant.

> **As built:** the queue is opt-in **per event** via the `events.requires_queue` flag. Only events
> with it set gate booking behind an access token; ordinary events book directly. Bot resistance at
> queue entry is a per-IP rate limit on the join endpoint (429 when exceeded); ticket-per-user caps
> are enforced at hold time.

### 11.1 Flow

```
                 ┌───────────────────────────┐
  User hits ────▶│  Queue Service (Redis)    │
  on-sale page   │  - assigns queue token    │
                 │  - position in sorted set │
                 └───────────┬───────────────┘
                             │ admits N users / sec
                             ▼
                     Booking Service ──▶ DB / Redis holds
```

1. On arrival, mint a **queue token** and add the user to a Redis **sorted set** scored by enqueue timestamp (→ FIFO-ish
   ordering).
2. The client polls (or holds a WebSocket) and is shown its position / estimated wait.
3. A **rate-controlled admitter** pops the front of the set at a fixed rate (e.g. "admit 500 users/sec" — tuned to what
   the booking DB can safely handle) and issues each a short-lived **access token** granting entry to the actual booking
   flow.
4. Only token-holders can call the Booking Service. No token → you're still in line. This caps concurrency on the DB
   regardless of how many people showed up.

### 11.2 Why this shape

- **Backpressure:** the admit rate is a dial matched to DB/Redis capacity, so the spike never reaches the transactional
  core at full force.
- **Fairness:** timestamp scoring approximates first-come-first-served, which users perceive as fair (vs. a random
  stampede).
- **Bot resistance:** combine with per-user/IP rate limits, CAPTCHA at entry, and account-age/verification checks. Cap
  tickets-per-user at hold time.
- **Graceful UX:** a position number and ETA beats timeouts and spinners.

### 11.3 Token & access-token expiry

The access token itself carries a TTL (e.g. 10 min). If an admitted user doesn't complete booking in that window, their
token expires, their holds are released (Redis TTL), and the freed capacity lets the admitter pull more people from the
queue. This keeps throughput steady even as some admitted users stall or leave.

### 11.4 Scaling the queue

- The Redis sorted set is small (a token + timestamp per user) and lives in memory — it absorbs millions of entries
  cheaply.
- Partition the queue **per event** so one blockbuster doesn't starve every other on-sale.
- The admitter is a simple, horizontally-stateless loop reading the rate limit from config, so you can retune live
  during an incident.

### 11.5 Escape hatch — skipping the queue when it's empty

The queue only earns its latency under contention. For a low-traffic event, making every arrival wait
for the next admit tick is bad UX — you'd wait seconds to book an event nobody else wants. So:

- **While the backlog is empty**, arrivals are fast-tracked straight to `ADMITTED` and skip the line.
- **Capped:** at most `admit-rate` arrivals skip per admit window; once that's hit — or a backlog
  forms — everyone else falls through to the normal queue, so a real spike is still throttled onto
  the DB. The hatch reopens once the window lapses and the backlog has drained.
- **Must be atomic:** concurrent arrivals race on "is the queue empty *and* are we under the cap?",
  so the check-and-admit is one atomic operation — otherwise two could both see an empty queue and
  over-admit.
- **One script, not two:** the admit decision and the enqueue are the same atomic step. Kept apart,
  a window-boundary counter reset could fast-track a latecomer ahead of an arrival that's mid-enqueue.
