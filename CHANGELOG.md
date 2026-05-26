# Vortex Direct API — Changelog

Version history for the Vortex Direct API public specification.

This changelog covers wire-visible changes only — endpoint shapes, field names, error codes, retry policy, authentication, and other contract-level behavior. Documentation-only edits (typo fixes, clarifying prose, new examples) are bundled into PATCH releases without individual entries.

The spec follows [Semantic Versioning](https://semver.org/):

- **PATCH** (`1.0.0` → `1.0.1`) — clarifications, no wire change. Partners require no action.
- **MINOR** (`1.0.x` → `1.1.0`) — backwards-compatible additions (new optional fields, new error codes added to the closed set, new endpoints). Partners are notified out-of-band; existing integrations continue unchanged.
- **MAJOR** (`1.x.x` → `2.0.0`) — breaking changes. Minimum 6-month deprecation notice; Vortex runs both versions in parallel during the migration window.

---

## v1.0.0 — 2026-05-20

Initial public release of the Vortex Direct API.

### Endpoints

Two-directional wire contract over HTTP+JSON with mutual HMAC-SHA256 authentication:

- **`/v2/operator.Launcher/Real`** — operator → Vortex; create real-money session and obtain launch URL.
- **`/v2/operator.Launcher/Demo`** — operator → Vortex; create demo (play-money) session. Test-flagged partners only.
- **`/v2/operator.Catalog/List`** — operator → Vortex; discovery of games available to the partner.
- **`/v2/operator.Round/Details`** — operator → Vortex; regulatory bet-history lookup.
- **`/v2/provider.Player/Balance`** — Vortex → operator; balance read and session validation.
- **`/v2/provider.Round/Transaction`** — Vortex → operator; bet (debit) or win (credit) — exactly one per call.
- **`/v2/provider.Round/Rollback`** — Vortex → operator; refund of a previously processed bet.

### Authentication

- HMAC-SHA256 over raw request body bytes; hex digest in `X-REQUEST-SIGN`.
- Mutual signing: both requests and responses are signed and verified — partners MUST sign every response (auth-stage 401 envelopes excepted, see Section 2).
- 5-minute nonce-replay protection on inbound `request_uuid`.
- `X-Vortex-Version: 1.0.0` advertised on every request and response.

### Idempotency

- `transaction_uuid` is the per-money-movement idempotency key, sticky across retries.
- `request_uuid` regenerates per attempt for HTTP correlation and nonce dedup.
- Partners MUST retain the dedup record (request `transaction_uuid` → response body) durably for ≥ 25 hours.

### Retry policy

- `{bet}` envelope: single attempt, no retry.
- `{win}` and `{rollback}` envelopes: long-tail schedule `1m → 3m → 5m → 1h → 6h → 12h` (≈ 24 h budget) when `retry_class != KNOWN_ERROR`.
- `/Player/Balance`: TIMEOUT class triggers three immediate retries before falling back to the long-tail schedule.

### Error catalog

Closed string-enum of error codes with HTTP status, retry class, and suspend-player flag. See Section 4 for the full catalog. Notable invariants:

- `DUPLICATE_TRANSACTION` (HTTP 409) is **terminal** on Vortex side — partners MUST replay the original success response on retried `transaction_uuid` rather than returning 409.
- `TRANSACTION_ALREADY_REFUNDED` (HTTP 409) on `/Round/Rollback` is treated as successful no-op (idempotent replay).

### Out of scope for v1.0.0

- Atomic `{bet, win}` envelope on `/Round/Transaction` — not supported. Bet and win MUST arrive as separate HTTP calls linked by `win.reference_transaction_uuid === bet.transaction_uuid`.

---
