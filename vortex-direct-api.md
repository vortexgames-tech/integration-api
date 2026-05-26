# Vortex Direct API — Public Specification

**Version**: `1.0.0`
**Last updated**: 2026-05-20
**Audience**: backend engineering teams on the operator/wallet side integrating with the Vortex Direct API.
**Changelog**: see [`CHANGELOG-vortex-direct-api.md`](./CHANGELOG-vortex-direct-api.md) for version history.

---

## Table of contents

1. **Overview** — launch-URL flow, key concepts
2. **Authentication** — HMAC-SHA256, `X-REQUEST-SIGN`, worked example
3. **Endpoints** — inventory + request/response shapes
4. **Errors** — closed-set catalog with HTTP status + retry class
5. **Envelope schema** — DTO sketches
6. **SLA & versioning** — timeouts, retry schedule, throttle, `X-Vortex-Version`

---

## Section 1 — Overview

Vortex Direct is the Direct-API contract operator wallets use to integrate Vortex games. The contract is a small set of HTTP+JSON endpoints with mutual HMAC-SHA256 authentication, decimal-string money on the wire, and an explicit retry taxonomy on every error response.

The audience is the operator-partner engineering team integrating against the contract — typically backend engineers building (a) endpoints we call (`/v2/provider.*`) on the wallet side, and (b) clients that call our endpoints (`/v2/operator.*`) to create sessions and fetch round history. The spec assumes generic web-platform fluency (HTTP, JSON, HMAC) and is language- and framework-agnostic.

**Launch-URL flow** (high-level):

```
+----------------+      1. POST /v2/operator.Launcher/Real         +-----------------+
|                | ----------------------------------------------> |                 |
|  Operator      |      { account_id, currency, game_id, ... }     |  Vortex API     |
|  Wallet        |                                                 |  (Vortex)       |
|  (partner)     | <---------------------------------------------- |                 |
|                |      2. { launch_url, session_id }              |                 |
+----------------+                                                 +-----------------+
        |                                                                    ^
        | 3. redirect player browser to launch_url                           |
        v                                                                    |
+----------------+                                                           |
|  Player        |  4. browser loads game (WebSocket + REST)                 |
|  Browser       | --------------------------------------------------------> |
+----------------+                                                           |
        ^                                                                    |
        |                                                                    |
        |                  5. POST /v2/provider.Round/Transaction (bet)      |
        |                  6. POST /v2/provider.Round/Transaction (win)      |
        |                  7. POST /v2/provider.Player/Balance (any time)    |
        |                  8. POST /v2/provider.Round/Rollback (refund)      |
        |              <-------------------------------------------------    |
+----------------+                                                           |
|  Operator      | <------------------------------------------------------- /
|  Wallet        |
+----------------+
```

Key concepts:

- **Mutual HMAC**: every request (in both directions) carries an `X-REQUEST-SIGN` header with `hex(HMAC-SHA256(rawBody, shared_secret))`. Responses are also signed. See Section 2.
- **Dual UUID**: `request_uuid` is the per-HTTP-attempt correlation id (regenerated on retry); `transaction_uuid` is the per-money-movement idempotency key (sticky across retries). See Section 5.
- **Decimal-string amounts**: all wire amounts are JSON strings with explicit decimal point in major units, e.g. `"100.00"` (USD/EUR), `"100"` (JPY), `"1.500"` (BHD). The number of fractional digits is **fixed by the currency's ISO-4217 minor-unit exponent** — see "Currency precision" below. The receiver maps to internal minor units via that exponent.
- **Retry taxonomy**: every error response carries `meta.retry_class ∈ {KNOWN_ERROR, UNKNOWN_ERROR, TIMEOUT}` driving the caller's retry decision.
- **One handler, two modes**: the single `/v2/provider.Round/Transaction` endpoint accepts EXACTLY ONE of `{bet}` (debit only) or `{win}` (credit only) per call. The partner implements one handler that branches on which field is present. Atomic `{bet, win}` is **not supported** — bet and win MUST arrive as separate HTTP calls, linked by `win.reference_transaction_uuid === bet.transaction_uuid`.

### Currency precision (MUST read — money-corruption risk)

Every wire amount (`bet.amount`, `win.amount`, `balance`) is a decimal string whose **fraction length is exactly the ISO-4217 minor-unit exponent of `currency`**. The partner MUST decode/encode using the same exponent. Decoding with the wrong exponent silently mis-scales money by 10×/100× and is the single highest-severity integration bug after idempotency.

| ISO-4217 minor units | Fraction digits | Wire example | Examples (non-exhaustive) |
|---|---|---|---|
| 0 | none (`"100"`) | `"1500"` | JPY, KRW, VND, CLP, ISK, UGX, PYG, XOF, XAF, RWF, BIF, DJF, GNF, KMF, MGA*, VUV, XPF |
| 2 (default) | `"100.00"` | `"1500.00"` | USD, EUR, GBP, BRL, INR, CNY, TRY, PLN, ZAR, MXN, and the large majority of currencies |
| 3 | `"100.000"` | `"1500.000"` | BHD, IQD, JOD, KWD, OMR, TND, LYD |

\* MGA/MRU are technically base-5 minor units; treat as exponent 2 on the wire.

**Hard rules:**

1. **Use the ISO-4217 table as the source of truth**, not a hardcoded short list. A partner wallet that knows only the common 2-decimal currencies and falls back to "2" for everything will under-credit every 3-decimal player by 10× and over-/under-scale every 0-decimal player by 100×. Maintain the full ISO-4217 minor-unit map (it is small and stable — the official list rarely changes).
2. **A currency you do not have a precision for MUST be hard-rejected** with `CURRENCY_NOT_ALLOWED` (Section 4), never silently defaulted to 2. Silent default is money corruption; an explicit reject is a safe, observable failure. Vortex applies this rule symmetrically — an unknown `currency` is rejected with `CURRENCY_NOT_ALLOWED` before any amount is parsed.
3. **Round-trip identity**: `decode(encode(minorUnits, currency), currency) === minorUnits` for every currency you accept. Test this per currency before going live.
4. Vortex emits canonical form: fraction padded to exactly the currency's exponent (`"100.00"`, never `"100"` for USD; `"100"`, never `"100.0"` for JPY). Partners SHOULD emit canonical form too. Vortex accepts any `^[0-9]+(\.[0-9]+)?$` form on input, but fraction digits **beyond** the currency's exponent are **truncated toward zero (NOT rounded)** — e.g. `"1.999"` for EUR is read as `1.99`, not `2.00`. Emitting canonical form avoids both the truncation-vs-rounding ambiguity and HMAC-byte mismatches.

---

## Environments

Vortex maintains two environments:

| Environment | Purpose |
|---|---|
| **Sandbox** | End-to-end integration testing. Fully isolated from production — separate database, no real-money flows, partner can request a state reset from the Vortex integration manager. |
| **Production** | Live real-money traffic. |

Base URLs, `partner_id`, and `api_secret` are issued out-of-band by the Vortex integration manager — sandbox credentials at onboarding, production credentials at go-live sign-off. The spec deliberately does not bake URLs in; each deployment cluster receives its own.

**Recommended integration order:**

1. Implement HMAC sign/verify and reproduce the worked example in Section 2 against your own implementation before any wire traffic.
2. Implement inbound endpoints in this order: `POST /v2/provider.Player/Balance` → `POST /v2/provider.Round/Transaction` → `POST /v2/provider.Round/Rollback`.
3. Implement outbound calls to Vortex (`/v2/operator.Launcher/*`, `/v2/operator.Catalog/List`, `/v2/operator.Round/Details`).
4. Run the Vortex sandbox integration suite (provided at onboarding) end-to-end against sandbox credentials.
5. Production credential issuance and go-live sign-off.

---

## Section 2 — Authentication

All requests in both directions are authenticated with HMAC-SHA256 over the **raw request body bytes**, keyed by the shared API secret. The hex digest is sent in the `X-REQUEST-SIGN` HTTP header. **Both requests and responses MUST be signed** — Vortex signs every response (success and error envelopes alike) and verifies every response received from a partner. A missing or mismatched `X-REQUEST-SIGN` on a partner response is rejected with `INVALID_SIGNATURE` (KNOWN_ERROR) — there is no opt-out, and signing is a launch-blocker, not a future requirement.

**Required headers on every call**:

```
X-REQUEST-SIGN: <hex HMAC-SHA256(rawBody, shared_api_secret)>
X-Vortex-Version: 1.0.0
Content-Type: application/json
```

**Byte-preservation rule** (inline MUST):

> The HMAC is computed over the **raw bytes received on the wire**, NOT over a re-serialized form of the parsed JSON object. Frameworks that auto-parse JSON before middleware sees the body MUST be configured to retain the raw body buffer for HMAC verification. Re-serializing (e.g. `JSON.stringify(req.body)`) before verifying WILL cause signature mismatches on any request whose original encoding differs from your serializer's output (key ordering, whitespace, number formatting, unicode escapes). This is the single most common integration bug.

**Worked example** — self-verify your HMAC implementation against this concrete vector before integrating:

```
Secret (32-byte hex):
  f3a9b2c4d6e8f0a2b4c6d8e0f2a4b6c8d0e2f4a6b8c0d2e4f6a8b0c2d4e6f8a0

Raw body (163 UTF-8 bytes — note exact key order, no whitespace, no trailing newline):
  {"request_uuid":"550e8400-e29b-41d4-a716-446655440000","partner_id":"vortex-direct-001","account_id":"player-abc-123","session_id":"sess-xyz-789","currency":"EUR"}

Expected X-REQUEST-SIGN (hex HMAC-SHA256):
  d4cd1dc7dfb56f04fed2cce2d7ba2fc7db88fec57e427559c1dfcb0a56fa990b
```

Reproduce in Node.js:

```js
const crypto = require('crypto');
const secret = 'f3a9b2c4d6e8f0a2b4c6d8e0f2a4b6c8d0e2f4a6b8c0d2e4f6a8b0c2d4e6f8a0';
const body = '{"request_uuid":"550e8400-e29b-41d4-a716-446655440000","partner_id":"vortex-direct-001","account_id":"player-abc-123","session_id":"sess-xyz-789","currency":"EUR"}';
crypto.createHmac('sha256', secret).update(body).digest('hex');
// → 'd4cd1dc7dfb56f04fed2cce2d7ba2fc7db88fec57e427559c1dfcb0a56fa990b'
```

The HMAC key is the secret string as-is (UTF-8 bytes), NOT hex-decoded. Both Node's `createHmac('sha256', secret)` and the OpenSSL `openssl dgst -sha256 -hmac <secret>` CLI accept the secret as a text string and use its raw bytes; if you hex-decode the secret first you will get a different digest. Verify by reproducing the digest above before sending real traffic.

If your client produces a different digest:

1. **Check byte preservation** — see the byte-preservation MUST above; the #1 cause is `JSON.stringify(req.body)` (or any re-serialization of the parsed object) before computing the HMAC.
2. **Check the secret encoding** — pass the secret string directly to your HMAC routine. Hex-decoding it changes the result.
3. **Check the digest encoding** — output must be lowercase hex (64 chars), not base64.

**Nonce / replay protection**: Vortex deduplicates inbound `request_uuid` values within a 5-minute window. A replayed `request_uuid` within the window yields `INVALID_NONCE` (HTTP 401). Partners SHOULD implement equivalent dedup on inbound calls from us.

**Auth-stage error responses are UNSIGNED.** When an inbound call fails HMAC verification, nonce dedup, or partner-not-found lookup (`INVALID_SIGNATURE`, `INVALID_NONCE` — both HTTP 401), the error envelope is returned WITHOUT an `X-REQUEST-SIGN` header. The signing key is not yet established at that stage (the request didn't authenticate), so signing would be meaningless. All POST-auth responses — success and business-error envelopes alike — are signed. Partners verifying response signatures MUST treat absent `X-REQUEST-SIGN` as expected on 401 auth-stage responses and as a mismatch error on every other response.

---

## Section 3 — Endpoints inventory

The contract is two-directional. Endpoints starting with `/v2/operator.*` are exposed BY Vortex and called BY the partner. Endpoints starting with `/v2/provider.*` are exposed BY the partner and called BY Vortex.

| Endpoint | Direction | Method | Purpose |
|---|---|---|---|
| `/v2/operator.Launcher/Real` | partner → Vortex | POST | Create real-money game session, return launch URL |
| `/v2/operator.Launcher/Demo` | partner → Vortex | POST | Create demo (play-money) session — test-flagged partners only |
| `/v2/operator.Catalog/List` | partner → Vortex | POST | Get available games (for partner catalog UI) |
| `/v2/operator.Round/Details` | partner → Vortex | POST | Get round outcome (player dispute resolution / regulatory bet-history) |
| `/v2/provider.Player/Balance` | Vortex → partner | POST | Read player balance / validate the active session (`session_id`) |
| `/v2/provider.Round/Transaction` | Vortex → partner | POST | Bet only OR win only (one per call; atomic bet+win is not supported) |
| `/v2/provider.Round/Rollback` | Vortex → partner | POST | Refund a previously processed bet |

### 3.1 Partner → Vortex (partner calls these)

#### `POST /v2/operator.Launcher/Real`
Create a real-money game session and return a launch URL the partner redirects the player browser to.

Request:
```json
{
  "request_uuid": "550e8400-e29b-41d4-a716-446655440000",
  "partner_id": "vortex-direct-001",
  "game_id": "cricket_crash",
  "currency": "EUR",
  "token": "<operator-issued session token>",
  "account_id": "player-abc-123",
  "balance": "1500.00",
  "username": "JohnDoe",
  "locale": "en",
  "platform": "desktop",
  "exit_url": "https://operator.example/lobby",
  "deposit_url": "https://operator.example/cashier"
}
```

Required: `request_uuid`, `partner_id`, `game_id`, `currency`, `token`, `account_id`, `balance` (decimal string, in major units per ISO-4217 precision — `"1500.00"` for EUR, `"1500"` for JPY). Optional: `username`, `locale`, `platform`, `exit_url`, `deposit_url`.

Success response:
```json
{
  "session_id": "sess-xyz-789",
  "launch_url": "https://game.example/?gameId=cricket_crash&sessionId=sess-xyz-789"
}
```

Errors (per Section 4 envelope): `GAME_NOT_AVAILABLE` (400), `CURRENCY_NOT_ALLOWED` (400), `WRONG_PARAMETERS` (400), `INVALID_SIGNATURE` (401), `INVALID_NONCE` (401), `PARTNER_UNAVAILABLE` (503), `INTERNAL_ERROR` (500). Token / account-state errors (`INVALID_TOKEN`, `EXPIRED_TOKEN`, `USER_DISABLED`, `USER_NOT_FOUND`) reach Launcher only when surfaced by the partner's own `Player/Balance` response during launch-time verification, and propagate per Section 4.

#### `POST /v2/operator.Launcher/Demo`
Create a demo (play-money) session — same request shape and same response shape as `Launcher/Real`. Only partners flagged as test on our side can use this endpoint; production partners hitting `/Demo` receive `WRONG_PARAMETERS`. Whether you have a test-flagged or production partner record is communicated out-of-band at onboarding.

Demo is **not** a no-auth shortcut: Vortex still calls your `POST /v2/provider.Player/Balance` to validate the `token` and read the authoritative balance before returning a `launch_url`. Errors from that round-trip surface in the Launcher response per the standard error envelope. Implement `/Player/Balance` to accept the same token your partner-side launcher would pass to `/Launcher/Demo`.

#### `POST /v2/operator.Catalog/List`
Returns the catalog of games available to this partner.

Request:
```json
{
  "request_uuid": "550e8400-e29b-41d4-a716-446655440000",
  "partner_id": "vortex-direct-001"
}
```

Response:
```json
{
  "games": [
    {"game_id": "cricket_crash", "name": "Cricket Crash", "currency_codes": ["EUR", "USD"], "min_bet": "0.10", "max_bet": "500.00"}
  ]
}
```
`min_bet` / `max_bet` are decimal strings, matching wire convention everywhere else. **Limits are quoted in the game's primary currency** (the first entry in `currency_codes`); partners are responsible for converting to other currencies if displayed in a multi-currency UI.

#### `POST /v2/operator.Round/Details`
Given `{request_uuid, partner_id, account_id, round_id}` returns round outcome with **plural transaction arrays** (a round may contain N transactions historically).

> **Scope (read carefully):** Round/Details is scoped to **partner + round**, not to a single account. A multiplayer round (e.g. `cricket_crash`) contains bets from many players of the same partner; the response returns **all bets on that round that belong to the requesting partner**, regardless of the `account_id` in the request. Other partners' bets are excluded (`WRONG_PARAMETERS` if the round contains no bet for the requesting partner). `account_id` is accepted on the request for forward compatibility and correlation; it does NOT narrow the result. If a partner needs per-account history it should filter the returned arrays client-side by joining against its own ledger.
```json
{
  "round_id": "round-12345",
  "game_id": "cricket_crash",
  "status": "FINISHED",
  "bet_transactions":  [{"transaction_uuid": "...", "amount": "100.00", "type": "bet", "created_at": "..."}],
  "win_transactions":  [{"transaction_uuid": "...", "reference_transaction_uuid": "...", "amount": "250.00", "type": "win", "created_at": "..."}],
  "crash_point": 2.50,
  "duration_ms": 8721,
  "started_at": "...",
  "finished_at": "..."
}
```
Plural naming makes the distinction explicit vs the singular `bet`/`win` on `/Round/Transaction` (which represents a SINGLE call).

`status` is one of:

| Value | Meaning |
|---|---|
| `STARTED` | Round created, betting not yet open. |
| `BETS_OPEN` | Round accepts new bets. |
| `BETS_CLOSED` | Bets closed; round is computing/playing out. |
| `RESULT` | Result resolved; payouts being applied. |
| `FINISHED` | Round fully closed; all transactions settled. `finished_at` is populated. |

For not-yet-finished rounds the `duration_ms` field is computed against the current time and `finished_at` is omitted. For non-crash games `crash_point` is omitted. If the round exists but has zero bets yet, both transaction arrays are returned empty — not an error.

### 3.2 Vortex → partner (partner implements these)

These are the endpoints the partner MUST implement on their wallet.

#### `POST /v2/provider.Player/Balance`
Read-only balance check **and active-session validation**. Retry semantics follow the standard `retry_class` taxonomy described in Section 6.

> **Session identity (read carefully):** The `BalanceEnvelope` carries `account_id` + `session_id`, **not** the raw operator launch token. The launch token is consumed once by `operator.Launcher/*` to mint the session; from then on `session_id` is the identity for every `/v2/provider.*` call. On `Player/Balance` the partner MUST validate that `session_id` is a live, non-expired session for `account_id`, and reject with the appropriate session/account error (`SESSION_NOT_FOUND` / `EXPIRED_TOKEN` / `USER_DISABLED` / `USER_NOT_FOUND`, see Section 4) otherwise. There is no token field on this envelope by design — do not build token-based validation here; validate the session.

Request envelope: `BalanceEnvelope` (see Section 5).
```json
{
  "request_uuid": "550e8400-e29b-41d4-a716-446655440000",
  "partner_id":  "vortex-direct-001",
  "account_id":  "player-abc-123",
  "session_id":  "sess-xyz-789",
  "currency":    "EUR"
}
```
Response: `BalanceResponse` — `{"balance": "1500.00"}`.

#### `POST /v2/provider.Round/Transaction`

A single endpoint with **two modes**, distinguished by which of `bet` / `win` is present in the body. The envelope is identical across modes; only the `finished` flag and the bet/win sub-object differ. **Exactly one** of `bet` or `win` MUST be present per call — atomic `{bet, win}` is **not supported**; sending both is rejected with `WRONG_PARAMETERS`.

Common envelope (sent on every call):
```json
{
  "request_uuid": "<uuid>",
  "partner_id":   "vortex-direct-001",
  "account_id":   "player-abc-123",
  "session_id":   "sess-xyz-789",
  "currency":     "EUR",
  "game_id":      "cricket_crash",
  "round_id":     "round-12345",
  "finished":     true | false
  // exactly one of {bet} or {win} — see below
}
```

> **Why `account_id` AND `session_id` (and `currency`) on every call?** `session_id` identifies the active session; `account_id` is the player on the partner side; `currency` is the active account currency. They are sent together so the partner can cross-check that the `session_id` truly belongs to `account_id` and that the wire `currency` matches `account.currency` — defense-in-depth against a stolen/replayed session being misused on the wrong account, or a money-movement arriving in the wrong currency. The partner SHOULD reject a request whose `(session_id, account_id, currency)` triple does not match its records (`SESSION_NOT_FOUND` / `USER_NOT_FOUND` / `CURRENCY_NOT_ALLOWED` as appropriate).

**Bet** (debit only — round start, `finished: false`):
```json
"bet": {"transaction_uuid": "bet-tx-uuid-001", "amount": "100.00"}
```

**Win** (credit only — round close, `finished: true`):
```json
"win": {
  "transaction_uuid":           "win-tx-uuid-002",
  "reference_transaction_uuid": "bet-tx-uuid-001",
  "amount":                     "250.00"
}
```

The win envelope's `reference_transaction_uuid` MUST equal the `bet.transaction_uuid` of the originating debit. This is how Vortex links the credit back to its bet across the two separate HTTP calls — the partner SHOULD reject a `win` whose `reference_transaction_uuid` does not match a previously-accepted bet on the same `(account_id, round_id)`.

Response: `TransactionResponse` — mirrors the request shape; only the field for the transaction actually processed (`bet` or `win`) is returned. See Section 5.

Retry policy:
- `{bet}` — **single attempt, no retry** (we never double-debit).
- `{win}` — long-tail retry (`1m → 3m → 5m → 1h → 6h → 12h`, ~24 h) when `retry_class != KNOWN_ERROR`.

#### `POST /v2/provider.Round/Rollback`
Refund of a previously processed bet. Separate endpoint to make the operational intent obvious. Long-tail retry schedule.

```json
{
  "request_uuid": "...",
  "partner_id": "vortex-direct-001",
  "account_id": "player-abc-123",
  "session_id": "sess-xyz-789",
  "currency": "EUR",
  "game_id": "cricket_crash",
  "round_id": "round-12345",
  "transaction_uuid": "rollback-uuid-001",
  "reference_transaction_uuid": "bet-tx-uuid-001"
}
```
Rolling back a `bet` also rolls back any `win` that referenced it (cascade is the partner's responsibility — we only target the bet).

---

## Section 4 — Error catalog

Every error response (HTTP 4xx or 5xx) follows the shape:

```json
{
  "code": "invalid_argument",
  "msg": "Player has not enough funds",
  "meta": {
    "api_code": "INSUFFICIENT_FUNDS",
    "retry_class": "KNOWN_ERROR",
    "balance": "50.00"
  }
}
```

- `code` is the top-level wire class: `"invalid_argument"` for business errors (HTTP 4xx) or `"internal"` for transport/server errors (HTTP 5xx).
- `meta.api_code` is a closed string enum (the full set is enumerated in the catalog below). Partners MUST treat any unknown `api_code` value as `INTERNAL_ERROR` for forward-compat.
- `meta.retry_class` drives the caller's retry decision: `KNOWN_ERROR` → abort, no retry; `UNKNOWN_ERROR` → retry per schedule; `TIMEOUT` → retry per schedule (see Section 6 for the per-endpoint timing rules).
- `meta.balance` is optional decimal string — partner SHOULD include for `INSUFFICIENT_FUNDS` so we can show accurate value to player.

### Full catalog

| api_code | http_status | retry_class | suspend_player | When to use |
|---|---|---|---|---|
| `INVALID_TOKEN` | 401 | KNOWN_ERROR | true | Session token is malformed, unknown, or tampered. |
| `EXPIRED_TOKEN` | 401 | KNOWN_ERROR | true | Session token has expired; player must re-authenticate. |
| `SESSION_NOT_FOUND` | 404 | KNOWN_ERROR | true | Session id is not recognized (revoked, never existed). |
| `USER_DISABLED` | 403 | KNOWN_ERROR | true | Player account is disabled by partner (self-exclusion, fraud hold). |
| `USER_NOT_FOUND` | 404 | KNOWN_ERROR | true | `account_id` does not exist on partner side. |
| `INSUFFICIENT_FUNDS` | 402 | KNOWN_ERROR | false | Player balance below requested bet amount. Include `meta.balance`. |
| `BET_EXCEEDED_LIMIT` | 400 | KNOWN_ERROR | false | Bet amount exceeds per-bet limit configured for this player. |
| `LOSS_EXCEEDED_LIMIT` | 400 | KNOWN_ERROR | false | Cumulative losses exceed responsible-gaming limit. |
| `PARTNER_LIMIT_REACHED` | 400 | KNOWN_ERROR | false | Partner-level aggregate cap reached (e.g. daily payout ceiling). |
| `GAME_NOT_AVAILABLE` | 400 | KNOWN_ERROR | false | Game is unknown, deactivated, in maintenance, or suspended. |
| `GAME_NOT_AVAILABLE_IN_COUNTRY` | 400 | KNOWN_ERROR | false | Geo-restriction: player's country is not in the game's allowed list. |
| `CURRENCY_NOT_ALLOWED` | 400 | KNOWN_ERROR | false | `currency` is not supported by this game/partner combination. |
| `ROUND_ALREADY_CLOSED` | 409 | KNOWN_ERROR | false | Operating on a round whose state forbids the requested transition. |
| `TRANSACTION_NOT_FOUND` | 404 | KNOWN_ERROR | false | `reference_transaction_uuid` (rollback / win) points to no known bet. |
| `DUPLICATE_TRANSACTION` | 409 | KNOWN_ERROR | false | A retried `transaction_uuid` whose original result the partner cannot replay. **Terminal on the Vortex side** — a 409 on a legitimate retry strands the player payout (Vortex cannot auto-recover). Partners MUST instead return the original success (HTTP 200 echo) — see "Idempotent replay" below. |
| `TRANSACTION_ALREADY_REFUNDED` | 409 | KNOWN_ERROR | false | Returned by the partner on `/Round/Rollback` for a `reference_transaction_uuid` that has already been rolled back. Vortex treats this as a successful idempotent replay (no-op) — see "Considered-successful on idempotent replay" below. |
| `INVALID_SIGNATURE` | 401 | KNOWN_ERROR | false | `X-REQUEST-SIGN` did not match HMAC of raw body. |
| `INVALID_NONCE` | 401 | KNOWN_ERROR | false | `request_uuid` was seen in the last 5 minutes (replay). |
| `WRONG_PARAMETERS` | 400 | KNOWN_ERROR | false | Schema validation failure, missing required field, unknown enum value, or version mismatch. |
| `PARTNER_UNAVAILABLE` | 503 | UNKNOWN_ERROR | false | Partner side is unhealthy / downstream wallet failure. Triggers retry per the Section 6 schedule; sustained failures cause Vortex to back off until your endpoint recovers. |
| `INTERNAL_ERROR` | 500 | UNKNOWN_ERROR | false | Opaque server-side failure on either side. Retry per schedule. |
| `REQUEST_TIMED_OUT` | 504 | TIMEOUT | false | Request did not complete within the call budget. Retry per TIMEOUT schedule. |

**Note on suspend_player**: when Vortex receives an error with `suspend_player: true` (the 5 player/session codes), the player's session is invalidated; the partner must issue a new launch URL before that player can play again.

### Idempotent replay on duplicate `transaction_uuid`

When the partner receives a `/Round/Transaction` or `/Round/Rollback` request whose `transaction_uuid` was already processed in the dedup window (≥ 25h retention), the partner **MUST return the original success response** — `HTTP 200` with the same `TransactionResponse` body it returned the first time (same `balance`, same `bet.id` / `win.id` ledger identifiers).

This is the standard idempotency contract: same `transaction_uuid` → same response, regardless of network retries.

> **Do NOT respond with `DUPLICATE_TRANSACTION` (HTTP 409) for a retried `transaction_uuid`.** The wire-level error envelope (`ErrorResponse`) does not carry ledger identifiers, so a 409 cannot convey the original result. Vortex treats `DUPLICATE_TRANSACTION` as **terminal**: a 409 on a legitimate retry strands a real player payout — Vortex cannot auto-recover it. This is why the dedup record MUST retain the **original response body** (see "Idempotency window" → durability requirement), not merely an "already-seen" flag.

### Considered-successful on idempotent replay

A small number of error codes are treated by Vortex as a successful no-op when they arrive on the retry path of a money-moving call. This avoids stranding a transaction whose underlying state change has already happened on the partner side.

| Endpoint | Code | Vortex behavior on this code |
|---|---|---|
| `POST /v2/provider.Round/Rollback` | `TRANSACTION_ALREADY_REFUNDED` (409) | Treated as success — the referenced bet was already refunded, no further action needed. Vortex does NOT alert ops or stage manual intervention; the long-tail rollback retry loop exits cleanly. |

Partners SHOULD return this code on `/Round/Rollback` when the `reference_transaction_uuid` points to a bet that is already in a refunded state in the partner ledger. Do NOT return it on `/Round/Transaction` — for retried bets/wins follow the standard idempotent-replay contract (HTTP 200 echo of the original response).

---

## Section 5 — Envelope schema (DTO sketch)

Field types use the convention `field: type [required|optional] // description`. Decimal regex: `^[0-9]+(\.[0-9]+)?$` (no exponent, no thousands separator, no leading sign). The regex only constrains *syntax* — the **fraction length is governed by the currency's ISO-4217 exponent**, see Section 1 → "Currency precision". The regex passing does NOT mean the scale is correct.

> **`request_uuid` validation regimes.** Vortex emits a strict RFC 4122 UUID for every outbound `request_uuid` on `provider.*` calls. All inbound `operator.*` endpoints (`Launcher/*`, `Catalog/List`, `Round/Details`) accept an opaque ≤128-character string for `request_uuid` so partners with non-UUID correlation-id conventions can integrate. Partners SHOULD emit RFC 4122 UUIDs everywhere — it is the only form that round-trips without ambiguity.

> **`transaction_uuid` / `reference_transaction_uuid`.** These are opaque strings, not enforced UUIDs. Any partner-stable identifier scheme works (e.g. `"bet-tx-uuid-001"` or a database row id). Stickiness across retries is the invariant — see Section 4 → "Idempotent replay on duplicate `transaction_uuid`".

### `TransactionEnvelope` (request body for `POST /v2/provider.Round/Transaction`)

```
TransactionEnvelope:
  request_uuid:   uuid     [required] // HTTP correlation; regenerated per attempt
  partner_id:     string   [required] // partner identifier
  account_id:     string   [required] // player on partner side
  session_id:     string   [required] // partner-issued session id
  currency:       string   [required] // ISO-4217 3-letter code (e.g. "EUR", "USD", "JPY", "BHD")
  game_id:        string   [required] // game identifier (e.g. "cricket_crash")
  round_id:       string   [required] // round identifier on Vortex side
  finished:       boolean  [required] // true when this call closes the round
  bet:            BetTransaction  [optional] // present for {bet} only
  win:            WinTransaction  [optional] // present for {win} only
  // class-level invariant: EXACTLY ONE of (bet, win) MUST be present (XOR — never both, never neither)
```

### `BetTransaction`

```
BetTransaction:
  transaction_uuid: string  [required] // idempotency key (sticky across retries)
  amount:           string  [required, matches decimal regex] // e.g. "100.00"
```

### `WinTransaction`

```
WinTransaction:
  transaction_uuid:           string  [required] // idempotency key for the win
  reference_transaction_uuid: string  [required] // linkage to originating bet
  amount:                     string  [required, matches decimal regex]
```

### `RollbackEnvelope` (request body for `POST /v2/provider.Round/Rollback`)

```
RollbackEnvelope:
  request_uuid:               uuid    [required]
  partner_id:                 string  [required]
  account_id:                 string  [required]
  session_id:                 string  [required]
  currency:                   string  [required] // ISO-4217 3-letter code
  game_id:                    string  [required]
  round_id:                   string  [required]
  transaction_uuid:           string  [required] // idempotency key for the rollback itself
  reference_transaction_uuid: string  [required] // bet being rolled back
```

### `BalanceEnvelope` (request body for `POST /v2/provider.Player/Balance`)

```
BalanceEnvelope:
  request_uuid: uuid    [required]
  partner_id:   string  [required]
  account_id:   string  [required]
  session_id:   string  [required]
  currency:     string  [required] // ISO-4217 3-letter code
```

### `TransactionResponse` (success response for `/Round/Transaction` and `/Round/Rollback`)

```
TransactionResponse:
  balance: string                    [required, matches decimal regex] // post-tx balance
  bet:     TransactionResponseEntry  [optional] // returned when request carried `bet`
  win:     TransactionResponseEntry  [optional] // returned when request carried `win`

TransactionResponseEntry:
  id:               string [required] // partner-side wallet ledger id
  transaction_uuid: string [required] // echo of request transaction_uuid
```

### `BalanceResponse` (success response for `/Player/Balance`)

```
BalanceResponse:
  balance: string [required, matches decimal regex]
```

### `ErrorResponse` (4xx/5xx — shared by all endpoints)

```
ErrorResponse:
  code: string [required, enum: "invalid_argument" | "internal"]
  msg:  string [required] // human-readable, for logs only (do NOT show to player)
  meta: ErrorResponseMeta [required]

ErrorResponseMeta:
  api_code:    string [required] // closed enum — see catalog in Section 4
  retry_class: string [required, enum: "KNOWN_ERROR" | "UNKNOWN_ERROR" | "TIMEOUT"]
  balance:     string [optional, matches decimal regex] // SHOULD include for INSUFFICIENT_FUNDS
```

---

## Section 6 — SLA & versioning

### Timeouts

| Direction | Endpoint | Timeout |
|---|---|---|
| Vortex → partner | all `/v2/provider.*` | 5 s |
| Partner → Vortex | all `/v2/operator.*` | 5 s |

Partners should treat 5 s as the hard upper bound — slower responses are abandoned and re-driven per the retry schedule below.

### Retry schedule

Driven by `meta.retry_class` in the error envelope (Section 4).

| Class | Behavior |
|---|---|
| `KNOWN_ERROR` | Abort. The error is final (insufficient funds, wrong params, already-finished round, etc.). |
| `UNKNOWN_ERROR` | Retry per the long-tail schedule: `1m → 3m → 5m → 1h → 6h → 12h` (≈ 24 h total budget). After the budget Vortex stops auto-retrying and the money movement is flagged Vortex-side for reconciliation. |
| `TIMEOUT` | On `/Player/Balance` (read-only, low-latency): first three retries are sent without delay so a momentarily-slow partner can recover; subsequent attempts fall back to the long-tail schedule. On `/Round/Transaction` (win) and `/Round/Rollback` (money-moving): the full long-tail schedule applies from attempt 1 — sub-minute settlement on transient timeouts is not guaranteed. |

Single-attempt rule: any envelope that contains a `bet` field is sent **once** — we never double-debit. Only `{win}` envelopes and rollback envelopes are eligible for retries.

### Throttle

Vortex applies a rate limit of **≈50 requests/s short window (1 s bucket)** to inbound `/v2/operator.*`, keyed by **source IP** (not per `partner_id`). A partner calling from a single egress IP shares the bucket across all its traffic; partners behind multiple egress IPs effectively get higher throughput per IP. Bursts above the limit yield HTTP 429 with `meta.api_code = INTERNAL_ERROR` and `retry_class = UNKNOWN_ERROR`. Back off per the `UNKNOWN_ERROR` schedule (see "Retry schedule" above).

### Idempotency window

Partners MUST retain the `transaction_uuid` dedup record (the originating request's response body) for **at least 25 hours** so that any Vortex retry within that window receives the original response (HTTP 200 with the same `TransactionResponse` body). The 25 h floor covers the chargeback dispute window. After the window the partner MAY release the dedup slot. See "Idempotent replay on duplicate `transaction_uuid`" above for the contract.

**Durability requirement (critical — read carefully):**

The dedup record MUST be **durable for the full 25 h window**, not stored only in an in-memory or volatile cache (e.g. an in-memory store with no persistence, or one that may evict entries under memory pressure). The window is wall-clock, not best-effort:

- Vortex's win/rollback retry schedule spans **up to ~24 h** (`1m → 3m → 5m → 1h → 6h → 12h`). A money-movement that fails transiently can legitimately be retried with the **same** `transaction_uuid` many hours after the first attempt.
- Vortex-side reconciliation may re-drive the same `transaction_uuid` near the **upper edge** of the 25 h window.

If the dedup record is lost before 25 h elapses (cache flushed, instance restarted without persistence, key evicted), the partner will treat a legitimate retry as a **brand-new** money movement and **double-credit / double-debit the player**. This is the single highest-severity contract violation in the protocol — there is no Vortex-side guard that can detect a partner that silently forgot a `transaction_uuid`.

Recommended implementation: persist the dedup record (request `transaction_uuid` → response body) in the same durable store as the wallet ledger (the DB row that recorded the money movement is itself a sufficient idempotency key — look it up by `transaction_uuid` before applying). A separate volatile cache MAY front it for latency but MUST NOT be the system of record.

### `X-Vortex-Version` header

Every request and response carries `X-Vortex-Version: <major>.<minor>.<patch>`. The current version is `1.0.0`.

Semantic rules (SemVer-aligned):

- Header is OPTIONAL on inbound (missing → assume `1.0.0`).
- If a value IS present, it MUST be in the supported set; an unsupported version yields `WRONG_PARAMETERS`.
- **PATCH** (e.g. `1.0.0` → `1.0.1`): clarifications, documentation fixes, no wire-level change — partners require no action.
- **MINOR** (e.g. `1.0.x` → `1.1.0`): backwards-compatible additions (new optional fields, new error codes added to the closed set, new endpoints) — partners are notified out-of-band, but existing integrations continue to work unchanged.
- **MAJOR** (e.g. `1.x.x` → `2.0.0`): breaking changes (field removed or semantically changed, endpoint URL changed, error code semantics changed). Partners receive a minimum 6-month deprecation notice; Vortex runs both versions in parallel during the migration window.

Repeated header (HTTP/2 field repetition, misconfigured proxy) is rejected with `WRONG_PARAMETERS` rather than silently bypassed — defense in depth against version-pinning attacks.

### TLS

All endpoints in both directions MUST be served over TLS 1.2 or newer. Plaintext HTTP is rejected at the load-balancer layer.

---

## Support

- **Integration & technical questions** during onboarding and live operation — your dedicated Vortex integration manager. Contact details are issued in the welcome packet at the start of onboarding.
- **Production incidents** — the 24/7 partner-support channel established at go-live sign-off.
- **Spec corrections / clarifications** — file a request via your integration manager; substantive corrections land in the next patch release (see Versioning above).

## License

This specification is provided to existing and prospective Vortex partners for the sole purpose of implementing the Vortex Direct API integration. Redistribution, public republication, or use outside of an active or prospective partnership requires prior written consent from Vortex Games.

Vortex® and the Vortex Games logo are trademarks of Vortex Games. All other trademarks are the property of their respective owners.

---

*Vortex Direct API Specification — v1.0.0 — © Vortex Games. All rights reserved.*

