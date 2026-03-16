# Architecture Deep Dive

> This document is intended for technical reviewers, interviewers, or anyone curious about the engineering decisions behind Student Connect.

---

## Table of Contents

1. [Problem Statement](#1-problem-statement)
2. [System Overview](#2-system-overview)
3. [Sync Engine](#3-sync-engine)
4. [Security Architecture](#4-security-architecture)
5. [Authentication Flow](#5-authentication-flow)
6. [Background Scheduler](#6-background-scheduler)
7. [API Design Decisions](#7-api-design-decisions)
8. [Data Model](#8-data-model)
9. [Frontend Architecture](#9-frontend-architecture)

---

## 1. Problem Statement

Canvas LMS, Todoist, and Notion are three platforms that many students use daily — but they don't talk to each other. Students manually re-enter every due date from Canvas into their task manager of choice. This is:

- **Error-prone** — dates are mistyped or missed entirely
- **Time-consuming** — up to 20+ assignments per semester per course
- **Stale** — instructors frequently update due dates, and manual lists go out of sync

Student Connect solves this by acting as an automation bridge between these three APIs.

---

## 2. System Overview

```
User → React Frontend → Flask API → Canvas / Todoist / Notion APIs
                              ↕
                           MongoDB Atlas
                              ↕
                         APScheduler (background)
```

The Flask backend is the central hub. It:
- Authenticates users via stateless JWT sessions
- Accepts encrypted payloads from the frontend and decrypts them server-side
- Fetches assignments from Canvas, diffs them against the user's Todoist or Notion state
- Manages background sync jobs via APScheduler
- Sends email notifications after each sync

---

## 3. Sync Engine

**File:** `backend/src/sync.py`, `backend/src/utils_sync.py`

### The Core Challenge: Three Different Date Formats

| Platform | Date Format Example |
|---|---|
| Canvas | `2025-04-10T23:59:00Z` |
| Todoist | `2025-04-10` (date-only, user-local) |
| Notion | `2025-04-10T19:59:00-04:00` |

The sync engine normalizes all three using `utils_sync.py`:

```python
def is_upcoming(date_str):
    clean_date = str(date_str).replace('Z', '+00:00').replace(' ', 'T')
    input_dt = datetime.fromisoformat(clean_date)
    now = datetime.now(timezone.utc)
    if input_dt.tzinfo is None:
        input_dt = input_dt.replace(tzinfo=timezone.utc)
    return input_dt >= now
```

This handles:
- `Z` vs `+00:00` UTC indicators
- Missing `T` separator between date and time
- Naive datetime objects (no tzinfo)
- Todoist's date-only strings

### Course Filtering with `is_recent_date`

Canvas returns ALL courses a user has ever been enrolled in. The engine filters to only currently active courses by checking if the course start date falls within a 60-day window of the nearest expected semester start:

```python
def is_recent_date(date_input, window_days=60):
    today = datetime.today()
    years = [today.year, today.year - 1]
    months = [1, 6, 8]  # Jan (spring), Jun (summer), Aug (fall)
    candidates = [datetime(y, m, 1) for y in years for m in months]
    dt_reference = min(candidates, key=lambda d: abs((d - today).days))
    return abs((dt_input - dt_reference).days) <= window_days
```

### State-Comparison Algorithm (Deduplication)

For each upcoming Canvas assignment, the engine runs `checkdbsTodoist` or `checkdbsNotion`:

1. **Not in task manager** → Add as a new task
2. **Already exists, same due date** → Skip (no API call)
3. **Exists but due date changed** → Delete old task, create new task (counted as an "update")

This prevents the task manager from accumulating duplicate entries on repeated syncs.

### Timezone Handling

Todoist's API requires a due datetime in the user's local timezone. The sync engine:
1. First tries to get the timezone from the Todoist API response (`getTimeZone`)
2. Falls back to the stored timezone from login if the API call fails
3. Converts Canvas UTC datetimes to the user's local time using `pytz`

---

## 4. Security Architecture

**Files:** `backend/src/utils.py`, `backend/app.py`, `frontend/src/api/`

### Zero-Trust Encryption Model

The fundamental principle: **the server should never receive sensitive data in plaintext, even over HTTPS.**

```
Step 1 — On app load:
  Frontend fetches RSA public key from GET /api/public-key

Step 2 — On form submit (register, token link, login):
  Frontend uses node-forge to encrypt each sensitive field:
    encryptedValue = forge.pki.publicKeyFromPem(pubKey)
                         .encrypt(plaintext, 'RSA-OAEP')

Step 3 — Server receives hex-encoded ciphertext:
  private_key.decrypt(
      bytes.fromhex(encrypted),
      padding.OAEP(mgf=MGF1(SHA256), algorithm=SHA256)
  )

Step 4 — Storage:
  Decrypted value is immediately re-encrypted with the public key
  and stored in MongoDB as ciphertext. Plaintext never touches disk.

Step 5 — Sync time:
  Token is decrypted in-memory only for the duration of the sync.
```

### RSA Key Generation

If `private_key.pem` does not exist on startup, Flask generates a fresh keypair:

```python
private_key = rsa.generate_private_key(
    public_exponent=65537,
    key_size=2048,
    backend=default_backend()
)
```

The private key is stored encrypted with a 512-bit random password (`PRIVATE_PASS`), which is stored in `.env`. This means even if an attacker got the `.pem` file, they couldn't use it without `PRIVATE_PASS`.

### JWT Session Design

| Property | Value | Reason |
|---|---|---|
| Token location | HTTP-only cookie | JS cannot access; XSS-resistant |
| CSRF protection | Double-submit cookie | Blocks CSRF attacks |
| Access token expiry | 1 hour | Limits blast radius of theft |
| Refresh token expiry | 3 days | Balance between UX and security |
| Refresh token path | `/api/refresh` only | Token scoped — not sent on every request |
| Cookie security | `Secure=True, SameSite=None` | HTTPS only |

### Passwordless Authentication

Users never create a password. Login flow:
1. User enters email → server generates a 6-character alphanumeric code
2. Code is bcrypt-hashed and stored in MongoDB with a TTL
3. Code is emailed to the user
4. User enters code → server compares with bcrypt → issues JWT

This eliminates the most common attack vector: stolen/reused passwords.

---

## 5. Authentication Flow

```
Register:
  POST /api/register
  { name: RSA(name), email: RSA(email) }
  → User created in MongoDB (no password field)

Login Step 1:
  POST /api/code
  { email: RSA(email) }
  → 6-char code generated, bcrypt-hashed, stored, emailed

Login Step 2:
  POST /api/login
  { email: RSA(email), code: RSA(code), timezone: RSA(tz) }
  → Code validated → JWT access cookie + optional refresh cookie set

Protected Requests:
  All /api/* routes (except public-key, register, code, login)
  → @jwt_required() decorator validates HTTP-only cookie

Token Refresh:
  Axios interceptor catches 401 responses
  → POST /api/refresh (sends refresh cookie)
  → New access cookie set
  → Original request retried transparently
```

---

## 6. Background Scheduler

**File:** `backend/src/scheduler.py`

### Why APScheduler + MongoDB?

Standard in-memory schedulers lose all jobs when the server restarts. By using `MongoDBJobStore`, scheduled jobs persist in the database. On Flask startup, `_restore_schedules()` queries MongoDB for all users with `auto_sync: True` and re-registers their jobs.

```python
jobstores = {
    'default': MongoDBJobStore(
        database='Sync',
        collection='scheduler_jobs',
        client=db.client
    )
}
scheduler = BackgroundScheduler(jobstores=jobstores)
```

### Supported Intervals

| Option | Hours | Use Case |
|---|---|---|
| Daily | 24 | Active semester, frequent assignment updates |
| Every 3 days | 72 | Moderate course load |
| Weekly | 168 | Low-activity periods |

### Misfire Grace Time

Jobs are configured with `misfire_grace_time=3600` — if a job was supposed to fire but the server was down, APScheduler will run it up to 1 hour late rather than dropping it.

---

## 7. API Design Decisions

### Why cookies instead of `Authorization: Bearer` header?

HTTP-only cookies are invisible to JavaScript, which makes them immune to XSS-based token theft. `localStorage`-based JWT storage is a common anti-pattern. Using cookies with CSRF double-submit provides equivalent security with better XSS resilience.

### Why async email dispatch?

Sync emails are sent in a daemon thread to avoid blocking the API response:

```python
thread = threading.Thread(target=send_sync_email, args=(...))
thread.daemon = True
thread.start()
return jsonify(frontend_response), 200
```

This ensures the user gets their sync results immediately, while the email sends in the background.

### Why rate-limit sync?

Canvas, Todoist, and Notion all have API rate limits. The sync endpoint includes a server-side cooldown to prevent accidental hammering. The frontend displays a live countdown timer when the user is in cooldown.

---

## 8. Data Model

### User Document (MongoDB)

```json
{
  "_id": "ObjectId",
  "name": "<RSA ciphertext>",
  "email": "user@example.com",
  "code": "<bcrypt hash>",
  "code_expiry": "ISODate",
  "timezone": "America/New_York",
  "CToken": "<RSA ciphertext>",
  "TToken": "<RSA ciphertext>",
  "NToken": "<RSA ciphertext>",
  "NDatabase": "<RSA ciphertext>",
  "UseTToken": true,
  "url": "https://university.instructure.com",
  "last_sync": "ISODate",
  "auto_sync": true,
  "sync_interval": 24,
  "sync_history": [
    {
      "timestamp": "ISODate",
      "added": "3",
      "updated": "1",
      "service": "Todoist"
    }
  ]
}
```

**Design note:** All sensitive string fields (`name`, `CToken`, `TToken`, `NToken`, `NDatabase`) are stored as RSA ciphertext. The email address is stored in plaintext because it's the primary key used for lookups and is not a secret (it's also the login identifier).

---

## 9. Frontend Architecture

### Component Tree

```
App.jsx (Router + Public Key Fetch)
├── StartPage          — Landing / hero page
├── RegisterPage       — New account creation
├── LoginPage          — Email + code auth
│   └── axiosInterceptor.js (global JWT refresh)
├── Dashboard          — Main application view
│   ├── CanvasLink     — Canvas token input
│   ├── RecentActivity — Sync history table
│   └── ParticleBackground
└── SettingsPage       — Manage linked services
```

### RSA Encryption on the Frontend

```javascript
// Fetch public key on app load (App.jsx)
const res = await axios.get('/api/public-key');
const publicKey = forge.pki.publicKeyFromPem(res.data);

// Encrypt before sending (any form)
const encrypted = forge.util.encode64(
    publicKey.encrypt(plaintext, 'RSA-OAEP', {
        md: forge.md.sha256.create(),
        mgf1: { md: forge.md.sha256.create() }
    })
);
```

### JWT Interceptor

`axiosInterceptor.js` registers a global Axios response interceptor that:
1. Catches any `401` response
2. Attempts `POST /api/refresh` (sends the HTTP-only refresh cookie)
3. If successful, retries the original request transparently
4. If refresh also fails, propagates the error (triggers redirect to login)

This gives the user a seamless experience — their session silently refreshes without any visible interruption.
