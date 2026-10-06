# Marketplace — browsing and buying

`/api/v1/marketplace`. The directory is **anonymously readable**; buying needs
a panel key or session token. The buyer must have accepted marketplace consent
and have `seller_status: active`.

```bash
H=(-H "Authorization: Bearer $SUBTOKEN_PANEL_AUTH" -H 'Accept: application/json')
```

A disabled marketplace answers `404` on every route — the operator's switch,
not a missing endpoint. Check `GET /api/v1/settings/public` if you get one.

## Two kinds of listing

| `kind` | Meaning | After purchase |
| --- | --- | --- |
| `rent` | Temporary control for a fixed term (30 / 90 / 365 days). | The account works under your keys for the term, then returns to the seller. |
| `sale` | Permanent transfer of ownership. | The account is yours. |

The server dispatches on `listing.kind`; the same purchase endpoint handles
both.

## Browsing (no auth needed)

```bash
curl -sS "$SUBTOKEN_API/marketplace/listings?page=1&page_size=20" | jq '.data'
```

Sending the session token adds viewer-specific fields — `viewer_balance`,
`viewer_seller_status`, and per-listing `is_mine` / `purchased_by_me`. Use it
when you want to render buy buttons:

```bash
curl -sS "${H[@]}" "$SUBTOKEN_API/marketplace/listings?sort=price_asc" |
  jq '.data | {viewer_balance, viewer_seller_status, items: [.items[] | {id, kind, title, price_usd, is_mine, purchased_by_me}]}'
```

Filters:

| Parameter | Values |
| --- | --- |
| `q` | Free text over title, models, platform. |
| `kind` | `rent`, `sale`. |
| `platform` | e.g. `anthropic`, `openai`. |
| `type` | `api_key` (alias for the stored `apikey`) or `oauth`. |
| `min_price`, `max_price` | USD. |
| `sort` | `newest` (default), `price_asc`, `price_desc`, `benchmark`. |
| `page`, `page_size` | Standard. |

```json
{
  "items": [
    {
      "id": 88, "kind": "rent", "title": "Claude Max — 5x",
      "description": "…", "price_usd": 12, "rent_term_days": 30,
      "platform": "anthropic", "account_type": "oauth",
      "models": ["claude-sonnet-4-5", "claude-opus-4-1"],
      "benchmark_status": "passed",
      "seller_display_name": "t•••y", "listed_at": "2026-10-01T09:00:00Z"
    }
  ],
  "total": 42, "page": 1, "page_size": 20,
  "viewer_balance": 31.2, "viewer_seller_status": "active"
}
```

The projection is deliberately thin: **no account id, no credentials, no
endpoints, no seller user id.** `seller_display_name` is a username or a masked
email, never a raw address. `account_type` is the stored value (`apikey`,
`oauth`); the query filter accepts the friendlier alias `api_key`.

`benchmark_status` is the automated quality check:

| Value | Meaning |
| --- | --- |
| `passed` | Verified working. The good case. |
| `pending` | The check is still running. |
| `failed` | Did not pass — expect problems. |
| `null` | Not required for this account type (shown as "no benchmark required"). |

Treat `null` as *unverified*, not as *passed*.

## Buying

```bash
# Rent for 30 days with auto-renew
curl -sS "${H[@]}" -H 'Content-Type: application/json' \
  -X POST "$SUBTOKEN_API/marketplace/listings/88/orders" \
  -d '{"nonce":"3f2a9c10-6d1e-4a55-bb3f-0a1c2d3e4f50","auto_renew":true}' | jq '.data'
```

| Field | Notes |
| --- | --- |
| `nonce` | **Required.** Idempotency key, unique per purchase attempt. |
| `auto_renew` | Rentals only. Fixed at order time; cannot be changed later. Must match what the seller offers. |

```json
{
  "id": 1201, "listing_id": 88, "account_id": 9,
  "buyer_user_id": 1, "seller_user_id": 44,
  "kind": "rent", "status": "active",
  "amount_usd": 12, "seller_share": 0.8, "seller_proceeds": 9.6, "platform_fee": 2.4,
  "term_days": 30, "starts_at": "2026-10-04T12:00:00Z",
  "expires_at": "2026-11-03T12:00:00Z",
  "auto_renew": true, "created_at": "…", "updated_at": "…"
}
```

The full price comes out of your balance at the moment of purchase. There is no
pending state and no external payment step.

### Nonce discipline

**The nonce is what makes a double-click safe.** Reusing it returns the
original order instead of charging twice. So:

- Generate a **fresh** nonce for every new purchase attempt.
- Reuse the **same** nonce only when retrying the *same* purchase after a
  timeout or an ambiguous failure.
- Never reuse a nonce across different listings.

```bash
nonce=$(cat /proc/sys/kernel/random/uuid)
```

### Before you post

Four things block a purchase, and the server returns `4xx` for each:

| Blocker | Fix |
| --- | --- |
| Not signed in | `401` — authenticate. |
| Already accepted the marketplace rules | `POST /api/v1/seller/consent` first. |
| Insufficient balance | Top up — see [payments.md](payments.md). |
| Your own listing, or already purchased | `is_mine` / `purchased_by_me`; pick another. |

```bash
# Am I actually able to buy?
curl -sS "${H[@]}" "$SUBTOKEN_API/auth/me" |
  jq '{balance, seller_status, can_buy: (.seller_status == "active")}'
```

`seller_status` must be `active`. Consenting is a single call and does **not**
commit you to selling anything:

```bash
curl -sS "${H[@]}" -X POST "$SUBTOKEN_API/seller/consent" | jq '.data'
```

## After buying

The account is under your control immediately — nothing to claim, no key to
swap. It works with your existing keys.

- **Rentals** expire at `expires_at` unless auto-renew charged the next term. If
  renewal is off, or your balance cannot cover it, the account returns to the
  seller and the listing goes back up. You are emailed on renewal and expiry.
- **Sales** transfer ownership permanently.
- Claude accounts bought this way appear under
  `GET /api/v1/user/owned-accounts/claude-reset`, with their reset credits —
  see [account.md](account.md).

```bash
# What did I buy? Orders are readable per listing only via purchase, so track
# your own listing ids; the Claude panel lists sale purchases.
curl -sS "${H[@]}" "$SUBTOKEN_API/user/owned-accounts/claude-reset" | jq '.data'
```

There is no "my orders" endpoint on the buyer side. For rentals, keep the
`listing_id`s you purchased — the seller console shows the mirror view.

## Worked example: find and buy

```bash
# 1. Cheapest passed rentals on Anthropic OAuth accounts
curl -sS "$SUBTOKEN_API/marketplace/listings?kind=rent&platform=anthropic&type=oauth&sort=price_asc&page_size=10" |
  jq '.data.items[] | select(.benchmark_status == "passed") |
      {id, title, price_usd, rent_term_days, seller: .seller_display_name}'

# 2. Confirm preconditions
curl -sS "${H[@]}" "$SUBTOKEN_API/auth/me" |
  jq '{balance, seller_status}'   # need seller_status == "active"

# 3. Buy listing 88, idempotently
curl -sS "${H[@]}" -H 'Content-Type: application/json' \
  -X POST "$SUBTOKEN_API/marketplace/listings/88/orders" \
  -d '{"nonce":"3f2a9c10-6d1e-4a55-bb3f-0a1c2d3e4f50"}' |
  jq '.data | {id, status, amount_usd, starts_at, expires_at}'
```

For selling your own accounts, see [seller.md](seller.md).
