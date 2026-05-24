# Allo Inventory — Take-Home Exercise

A Next.js inventory and reservation system for multi-warehouse retail, with concurrency-safe stock holds.

## Running Locally

### Prerequisites
- Node.js 18+
- Hosted Postgres (Supabase, Neon, or Railway — all have free tiers)
- Redis (Upstash free tier works well)

### 1. Clone and install
```bash
git clone <repo-url>
cd allo-inventory
npm install
```

### 2. Set environment variables
```bash
cp .env.example .env.local
```
Edit `.env.local` with your DATABASE_URL, REDIS_URL, NEXTAUTH_URL, CRON_SECRET.

### 3. Run migrations and seed
```bash
npx prisma migrate dev --name init
npx prisma generate
npm run db:seed
```

### 4. Start
```bash
npm run dev
```

---

## Architecture

### Data Model
```
Product         — name, price, description, imageUrl
Warehouse       — name, location
StockLevel      — productId + warehouseId (unique), totalUnits, reservedUnits
Reservation     — productId, warehouseId, quantity, status (PENDING/CONFIRMED/RELEASED), expiresAt
```

`availableUnits = totalUnits - reservedUnits` — always computed, never stored.

When confirmed: both totalUnits and reservedUnits are decremented (permanent sale).
When released: only reservedUnits is decremented (units return to pool).

---

## Concurrency Safety

### The Problem
Two simultaneous requests for the last unit must yield exactly one 201 and one 409.

### The Solution: Distributed Lock + DB Transaction

**Layer 1 — Redis `SET NX EX` lock:**
Before touching the DB, each request acquires `lock:stock:{productId}:{warehouseId}`. Only one holder at a time. If taken, the second request gets a 429 rather than silently corrupting data.

**Layer 2 — Prisma `$transaction` with atomic increment:**
Inside the lock: read stock, verify availability, use `{ increment: quantity }` (atomic at DB level), double-check post-increment, create Reservation record.

This is belt-and-suspenders: the Redis lock prevents concurrent transactions for the same SKU from even starting; the DB transaction ensures atomicity as a second line of defence.

---

## Expiry Mechanism

### Production: Vercel Cron (vercel.json)
```json
{ "crons": [{ "path": "/api/cron/cleanup-reservations", "schedule": "* * * * *" }] }
```
Runs every minute. Finds PENDING reservations where `expiresAt < now`, decrements reservedUnits, sets status to RELEASED. Each reservation is re-checked inside its own transaction to prevent double-release on retries.

### Lazy Cleanup on Read
The `/confirm` endpoint also checks expiry on every call. If it finds an expired PENDING reservation, it releases it immediately and returns 410. This closes the gap if the cron is delayed.

---

## Bonus: Idempotency

`POST /api/reservations` and `POST /api/reservations/:id/confirm` both accept an `Idempotency-Key` header.

1. First request: processed normally, response cached in Redis for 24 hours under `idempotency:{key}`.
2. Retry with same key: cached response returned immediately with `X-Idempotent-Replayed: true`, no side effect re-run.

---

## Trade-offs & What I'd Do Differently

**Simplified:**
- No auth — a real system would attach reservations to user sessions
- Quantity hardcoded to 1 in the UI (API supports arbitrary quantities)
- No test suite — would add unit tests for locking logic and integration tests for the concurrent scenario

**Would add with more time:**
- `SELECT FOR UPDATE` as Redis fallback
- WebSocket push so stock counts update live on the listing page
- Graceful lock retry with exponential backoff instead of immediate 429
- Redis sorted set + sub-second background worker for tighter expiry guarantees (cron granularity = up to 59s lag after expiresAt)
