# Seller console — supply, listings, payouts

`/api/v1/seller`. A **panel key or session token** works, and `seller_status`
must be `active` for non-admin users. Each route also rechecks resource ownership;
a `403` can indicate inactive/revoked seller consent **or** an ownership/access
check, so inspect the caller status and requested resource rather than assuming
one cause.

```bash
H=(-H "Authorization: Bearer $SUBTOKEN_PANEL_AUTH" -H 'Accept: application/json')
```

## Consent

```bash
# Required before anything else, and before you can BUY (see marketplace.md)
curl -sS "${H[@]}" -X POST "$SUBTOKEN_API/seller/consent" | jq '.data'

# Confirm
curl -sS "${H[@]}" "$SUBTOKEN_API/auth/me" | jq '.data.seller_status'
```

`seller_status` moves `inactive` → `active`, or → `revoked`. There is no
user-facing endpoint to revoke it; that is an operator action. Consent is
mutually binding on buyers and sellers and creates no obligation to trade.

```bash
# Everything below needs this to be "active"
curl -sS "${H[@]}" "$SUBTOKEN_API/seller/accounts" |
  jq 'if .code == 0 then .data.items else "blocked: \(.message)" end'
```

Seller collections return `{ "items": [...] }` — **no `total`, no `pages`.**
There is no pagination; these are your own records.

## Proxies

A proxy you own can be bound to accounts you register. Accounts may only bind
proxies you own — re-checked server-side on every write.

```bash
curl -sS "${H[@]}" "$SUBTOKEN_API/seller/proxies" | jq '.data.items[] | {id, name, protocol, host, port, status}'

curl -sS "${H[@]}" -H 'Content-Type: application/json' \
  -X POST "$SUBTOKEN_API/seller/proxies" -d '{
    "name": "resi-us-1", "protocol": "socks5",
    "host": "198.51.100.10", "port": 1080,
    "username": "u", "password": "p"
  }' | jq '.data'

curl -sS "${H[@]}" -H 'Content-Type: application/json' \
  -X PATCH "$SUBTOKEN_API/seller/proxies/3" -d '{"password":"rotated"}' | jq '.data'

curl -sS "${H[@]}" -X DELETE "$SUBTOKEN_API/seller/proxies/3"
```

`PATCH` accepts `name`, `host`, `port`, `username`, `password` — not `protocol`.
Credentials come back stripped. `DELETE` fails while an account still binds the
proxy; unbind first.

## Accounts

### Register an API-key endpoint

```bash
curl -sS "${H[@]}" -H 'Content-Type: application/json' \
  -X POST "$SUBTOKEN_API/seller/accounts" -d '{
    "name": "my upstream pool A",
    "base_url": "https://api.example.com",
    "api_key": "sk-upstream-…",
    "concurrency": 10,
    "priority": 50,
    "notes": "primary",
    "proxy_id": 3
  }' | jq '.data | del(.api_key)'
```

| Field | Notes |
| --- | --- |
| `name` | Required. |
| `base_url` | Required. Your OpenAI-compatible endpoint. |
| `api_key` | Required. **Never echo this back** — it is your upstream credential. |
| `concurrency`, `priority` | Scheduling hints. Higher priority is preferred. |
| `notes` | Free text, private to you. |
| `proxy_id` | Must be a proxy you own. |

The response is a projection with credential material stripped — `id`, `name`,
`platform`, `type`, `base_url`, `status`, `schedulable`, `benchmark_status`,
`priority`, `concurrency`, `proxy_id`, `group_ids`, `updated_at`.

```bash
curl -sS "${H[@]}" "$SUBTOKEN_API/seller/accounts" |
  jq '.data.items[] | {id, name, status, schedulable, benchmark_status, group_ids}'

# Rotate credentials, or disable an account without deleting it
curl -sS "${H[@]}" -H 'Content-Type: application/json' \
  -X PATCH "$SUBTOKEN_API/seller/accounts/9" \
  -d '{"credentials":{"api_key":"sk-new-…"}}' | jq '.data | del(.api_key)'

curl -sS "${H[@]}" -H 'Content-Type: application/json' \
  -X PATCH "$SUBTOKEN_API/seller/accounts/9" -d '{"status":"disabled"}' | jq '.data.status'
```

`status` is `active` or `disabled`. Disabling stops routing without losing your
configuration — prefer it to deleting.

`benchmark_status` is `pending`, `passed`, `failed`, or `null` when the account
type is exempt (OAuth accounts generally are). A `pending` account is not yet
eligible for a live listing on deployments that gate on it.

```bash
# Benchmark detail and re-run
curl -sS "${H[@]}" "$SUBTOKEN_API/seller/accounts/9/benchmark" | jq '.data'
curl -sS "${H[@]}" -X POST "$SUBTOKEN_API/seller/accounts/9/rebenchmark" |
  jq '.data.benchmark_status'   # back to "pending" until the next catalog poll
```

`POST …/rebenchmark` only resets the gate to `pending`; the actual run happens
on the benchmark server's next poll, so it is not immediate.

### OAuth onboarding

Self-serve OAuth so the seller completes provider authorisation themselves.
Ownership is forced server-side to the authenticated user. For an account
purchased for a specific dashboard user, this is the preferred path; an active
OAuth account in the right group is not enough to prove user ownership.

Platforms on the allowlist: `antigravity`, `openai`, `gemini`, `grok`.

For Antigravity, use these endpoints in order:

```text
POST /api/v1/seller/oauth/antigravity/auth-url
POST /api/v1/seller/oauth/antigravity/exchange-code
POST /api/v1/seller/oauth/accounts
```

The auth-url request body may contain an optional `proxy_id`; if present, the
proxy must be owned by the authenticated user. Complete provider consent before
exchanging the code, then submit the exchange credentials to
`/seller/oauth/accounts` with `platform: "antigravity"`. The account-create
body accepts `name`, `platform`, `credentials`, optional `priority`, optional
`concurrency`, and optional user-owned `proxy_id`; it does **not** accept
`group_ids`. Group binding is a separate seller operation.
For this OAuth flow, use the same authenticated interactive API session as the rest
of the skill. Keep `auth_url`, `session_id`, `state`, the provider callback code,
and exchange credentials in process memory; do not print them, place the callback
code in shell command arguments/history, or log raw request/response bodies.

1. **Start auth.** For Antigravity, call
   `POST /api/v1/seller/oauth/antigravity/auth-url` with an optional
   `{"proxy_id": <your_proxy_id>}` body. Keep the returned `session_id`, `state`,
   and `auth_url` in memory. Open the URL only in the intended provider-authenticated
   browser session, using the configured egress.
2. **Complete consent and capture the callback.** Use the account the user
   intended to authorize. Assert the callback's `state` exactly matches the
   state returned by this auth-url request. Do not pick an older callback from
   browser history or reuse a code from another session.
3. **Exchange.** For Antigravity, send
   `{"session_id": <same session_id>, "state": <same state>, "code": <callback code>}`
   to `POST /api/v1/seller/oauth/antigravity/exchange-code`. Process the response
   in memory; it contains live access/refresh credentials. If the session or
   state is rejected/expired, start a fresh authorization rather than retrying
   the stale code.
4. **Create the owned account.** Send the exchange credentials to
   `POST /api/v1/seller/oauth/accounts` with `name`, `platform: "antigravity"`,
   optional `priority` / `concurrency`, and optional user-owned `proxy_id`.
   The server forces ownership to the authenticated user. Do not send
   `group_ids`; that is not part of the user endpoint contract. Inspect only the
   returned account ID/status, not credentials.
5. **Verify.** Re-read `GET /api/v1/seller/accounts` as the same user and confirm
   the new ID appears in the caller's owned account list. Account ownership
   does not by itself bind the account to an inference-key group; bind only via
   an eligible seller-owned group and verify the intended routing separately.

`session_id`, `state`, and `code` must all come from the same auth-url step —
the exchange is validated against them. OpenAI and Grok accept an explicit
`redirect_uri`; Gemini derives it server-side and additionally takes `project_id`,
`oauth_type`, and `tier_id`, which must be repeated identically at exchange time.

## Seller-owned groups and Smart Router membership

Seller-owned groups are **not directly consumer-bindable** and do not appear in
`/api/v1/groups/available`. The `is_public` field controls only whether their
accounts participate in the shared Subtoken Smart Router pool; it does not expose
the group for binding to ordinary consumer inference keys and is unrelated to
marketplace listings.

```bash
curl -sS "${H[@]}" "$SUBTOKEN_API/seller/groups" |
  jq '.data.items[] | {id, name, platform, rate_multiplier, is_public, status, account_count}'

curl -sS "${H[@]}" -H 'Content-Type: application/json' \
  -X POST "$SUBTOKEN_API/seller/groups" -d '{
    "name": "claude-pool", "description": "…",
    "platform": "anthropic", "rate_multiplier": 0.4, "is_public": false
  }' | jq '.data'

curl -sS "${H[@]}" -H 'Content-Type: application/json' \
  -X PATCH "$SUBTOKEN_API/seller/groups/5" -d '{"is_public":true}' | jq '.data.is_public'

curl -sS "${H[@]}" -X DELETE "$SUBTOKEN_API/seller/groups/5"
```

`is_public` is the **Smart Router membership** toggle: `true` contributes the
group's schedulable accounts to the shared routing pool; `false` excludes them.
Neither setting publishes your accounts in the marketplace directory.
`DELETE` fails while accounts are bound.

## Bind accounts to groups

```bash
curl -sS "${H[@]}" -H 'Content-Type: application/json' \
  -X POST "$SUBTOKEN_API/seller/accounts/9/groups" -d '{"group_id":5}'

curl -sS "${H[@]}" -X DELETE "$SUBTOKEN_API/seller/accounts/9/groups/5"
```

Both sides must be yours — your account, your group. This is the same repository
path the admin binding surface uses, so scheduler outbox semantics are
identical.

## Listings

Lifecycle: `draft` → `active` ⇄ `paused` → `delisted` (terminal).
`sold` / `rented_out` are also terminal.

```bash
curl -sS "${H[@]}" "$SUBTOKEN_API/seller/listings" |
  jq '.data.items[] | {id, account_id, kind, status, title, price_usd, rent_term_days, benchmark_status}'

# Create a draft over one owned active account
curl -sS "${H[@]}" -H 'Content-Type: application/json' \
  -X POST "$SUBTOKEN_API/seller/listings" -d '{
    "account_id": 9, "kind": "rent", "title": "Claude Max — 5x",
    "description": "…", "price_usd": 12, "rent_term_days": 30,
    "auto_renew": true
  }' | jq '.data'
```

| Field | Notes |
| --- | --- |
| `account_id` | Required. Must be an **active** account you own. |
| `kind` | `rent` or `sale`. **Immutable** after creation — delist and create a new one to change it. |
| `title`, `description` | Public text. |
| `price_usd` | USD. |
| `rent_term_days` | Rentals only (30 / 90 / 365). |
| `auto_renew` | Rentals only; must match what you offer. |

```bash
# Edits: price, text, term, renewal. Not kind, not account.
curl -sS "${H[@]}" -H 'Content-Type: application/json' \
  -X PATCH "$SUBTOKEN_API/seller/listings/12" -d '{"price_usd":15}' | jq '.data.price_usd'
```

Lifecycle transitions, each a separate call:

```bash
curl -sS "${H[@]}" -X POST "$SUBTOKEN_API/seller/listings/12/activate"   # draft  → active
curl -sS "${H[@]}" -X POST "$SUBTOKEN_API/seller/listings/12/pause"      # active → paused
curl -sS "${H[@]}" -X POST "$SUBTOKEN_API/seller/listings/12/resume"     # paused → active
curl -sS "${H[@]}" -X POST "$SUBTOKEN_API/seller/listings/12/delist"     # any    → delisted
```

`activate` publishes to the public directory. `delist` is **terminal** — there
is no un-delist; create a new listing instead. To take a listing out temporarily,
use `pause`.

Each transition returns the updated listing, so check `.status` rather than
assuming it moved.

## Earnings

```bash
# All time
curl -sS "${H[@]}" "$SUBTOKEN_API/seller/earnings" | jq '.data'

# Trailing N days (strict rate limiter)
curl -sS "${H[@]}" "$SUBTOKEN_API/seller/earnings?days=30" |
  jq '.data.items[] | {account_id, requests, billed_cost, upstream_cost}'
```

```json
{ "items": [ { "account_id": 9, "requests": 4210, "billed_cost": 184.2, "upstream_cost": 96.1 } ], "days": 30 }
```

`billed_cost` is what your served requests were charged to consumers;
`upstream_cost` is what they cost you. The difference is your gross margin
before the platform fee. This rollup comes from the usage attribution ledger,
so it lags real time slightly.

## Payouts

Payout requests accept `Idempotency-Key`. Generate one fresh key per logical
payout request and reuse it with the identical amount/note only when retrying
that same request. The shipped `idempotency.observe_only` default accepts
keyless requests without deduplication, while deployments may enable strict key
requirements. If a request was keyless or its outcome is otherwise uncertain,
inspect `GET /seller/payouts` and `GET /seller/payouts/balance` before submitting
another request; a duplicate withdrawal row may otherwise be invisible in
`available` until operator approval.

The payout ledger is **derived**: `billed × revenue_share − approved − paid`.
It is only enabled when the operator configures a revenue share; otherwise
`payouts_enabled` is `false` and requests are refused.

```bash
curl -sS "${H[@]}" "$SUBTOKEN_API/seller/payouts/balance" | jq '.data'
```

```json
{
  "billed_cost": 184.2, "revenue_share": 0.8,
  "earned": 147.36, "marketplace_proceeds": 96.0,
  "reserved": 24.0, "paid_out": 40.0, "available": 33.36,
  "payouts_enabled": true
}
```

`available` is the only number you may request. `reserved` is in-flight or
uncleared; `earned` is lifetime and includes money already paid out. If
`available` disagrees with your own arithmetic, trust this response — it applies
the live revenue share, which may differ from the `seller_share` on individual
orders (a per-seller override exists and is reported as `seller_revenue_share`
on your profile).

```bash
curl -sS "${H[@]}" "$SUBTOKEN_API/seller/payouts" |
  jq '.data.items[] | {id, amount, status, requested_at, reviewed_at, payout_reference}'

curl -sS "${H[@]}" -H 'Content-Type: application/json' \
  -H "Idempotency-Key: $PAYOUT_IDEMPOTENCY_KEY" \
  -X POST "$SUBTOKEN_API/seller/payouts" -d '{"amount":33.36,"note":"October"}' | jq '.data'

# Cancel your own pending request
curl -sS "${H[@]}" -X DELETE "$SUBTOKEN_API/seller/payouts/7" | jq '.data.status'
```

`amount` above `available` is refused. A payout moves to `requested` and is
settled by the operator — it is not instant. `DELETE` only works while pending.

## Gotchas

- Seller endpoints return `{items: [...]}` with no pagination. Do not send
  `page`/`page_size` and do not loop.
- **Credentials are write-only.** You supply `api_key` and OAuth tokens; the
  server never returns them. Rotating means `PATCH`ing new values — there is no
  read-back. Filter responses with `jq 'del(.. | .api_key?)'` before printing.
- `seller_status: "revoked"` locks every route here **and** blocks purchasing.
- Deleting a proxy fails while an account binds it; deleting a group fails
  while accounts are bound. Unbind first.
- `delist` is terminal. `pause` is the reversible option.
- `kind` on a listing cannot be changed. Delist and recreate.
- `rebenchmark` is asynchronous — it resets the gate, the run happens later.
- Group `is_public` is Smart Router membership, not a marketplace listing.
