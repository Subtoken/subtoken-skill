# Seller console — supply, listings, payouts

`/api/v1/seller`. A **panel key or session token** works, and `seller_status`
must be `active`. Every route re-checks consent and per-account ownership on
the server, so a `403` here means consent, not a bad id.

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
Ownership is forced server-side.

Platforms on the allowlist: `antigravity`, `openai`, `gemini`, `grok`.

```bash
# 1. Get an authorisation URL (optionally bound to one of your proxies)
sess=$(curl -sS "${H[@]}" -H 'Content-Type: application/json' \
  -X POST "$SUBTOKEN_API/seller/oauth/openai/auth-url" -d '{"proxy_id":3}')
echo "$sess" | jq '.data | {session_id, state, auth_url}'
```

```bash
# 2. Exchange the code the provider gave you
curl -sS "${H[@]}" -H 'Content-Type: application/json' \
  -X POST "$SUBTOKEN_API/seller/oauth/openai/exchange-code" -d "{
    \"session_id\": $(echo "$sess" | jq '.data.session_id'),
    \"state\":      $(echo "$sess" | jq '.data.state'),
    \"code\":       \"PASTE_CODE\",
    \"proxy_id\":   3
  }" | jq '.data | {token_type, expires_in}'   # tokens: never print
```

```bash
# 3. Register the authorised account
curl -sS "${H[@]}" -H 'Content-Type: application/json' \
  -X POST "$SUBTOKEN_API/seller/oauth/accounts" -d '{
    "name": "my ChatGPT account",
    "platform": "openai",
    "credentials": { "…": "tokens from step 2" },
    "concurrency": 3,
    "proxy_id": 3
  }' | jq '.data'
```

`session_id`, `state`, and `code` must all come from the same `auth-url` step —
the exchange is validated against them. OpenAI and Grok accept an explicit
`redirect_uri`; Gemini derives it server-side and additionally takes `project_id`,
`oauth_type`, and `tier_id`, which must be repeated **identically** at exchange
time.

The exchange response contains live access and refresh tokens. Pipe it straight
into the register step; never log it.

## Groups (private by default)

Seller-owned groups are **always private** — they never appear in
`/api/v1/groups/available`, and consumers cannot bind themselves to them.

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

`is_public` is the **Smart Router membership** toggle: `true` joins your
accounts to the shared routing pool, `false` keeps the group private to your own
keys. It is unrelated to the marketplace listing directory — public groups do
not publish your accounts for sale. `DELETE` fails while accounts are bound.

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

Payout requests have no documented `Idempotency-Key` contract. If a request
times out, inspect `GET /seller/payouts` and `GET /seller/payouts/balance`
before submitting again; the first request may already be reserved for review.

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
