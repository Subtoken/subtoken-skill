# Account, keys, groups, and entitlements

Panel endpoints under `/api/v1`. A **panel key or session token** works for
ordinary account, key, group, usage, redeem, subscription, and affiliate
operations. Identity-changing routes remain session-token-only; see
[security.md](security.md). Every response is the `{code, message, data}`
envelope; see [auth.md](auth.md).

```bash
H=(-H "Authorization: Bearer $SUBTOKEN_PANEL_AUTH" -H 'Accept: application/json')
```

In the examples below `$H` stands for those two headers plus, on writes,
`-H 'Content-Type: application/json'`.

## Identity

### `GET /api/v1/auth/me` — the current user

The richest single read: identity, balance, concurrency, seller state, and
every bound auth provider.

```bash
curl -sS "${H[@]}" "$SUBTOKEN_API/auth/me" | jq '.data'
```

```json
{
  "id": 1, "email": "you@example.com", "email_bound": true, "username": "you",
  "role": "user", "balance": 12.5, "frozen_balance": 0, "concurrency": 5,
  "rpm_limit": 0, "seller_status": "", "status": "active", "allowed_groups": null,
  "balance_notify_enabled": false, "total_recharged": 0,
  "linuxdo_bound": false, "oidc_bound": false, "wechat_bound": false,
  "dingtalk_bound": false,
  "identities": { "email": { "bound": true, "can_unbind": false, "…": "…" } },
  "identity_bindings": { "…": "…" },
  "created_at": "…", "updated_at": "…"
}
```

Fields that decide what you may do next:

| Field | Meaning |
| --- | --- |
| `balance` | Spendable now. `frozen_balance` is held by async batch jobs and is **not** spendable. |
| `concurrency` | Max simultaneous requests across all your keys. |
| `rpm_limit` | Account-level RPM cap; `0` = unlimited. The group's own limit wins when set. |
| `seller_status` | `inactive`, `active`, or `revoked`. `active` unlocks `/seller/*` **and** is required to buy on the marketplace. |
| `status` | `active` or `disabled`. |
| `allowed_groups` | `null` means every non-exclusive group. An array restricts you to those ids. |
| `role` | `user` or `admin`. There is no `seller` role; seller rights flow from `seller_status`. |

`seller_status` is `""` (empty string) on accounts that never touched the
seller console — treat that as `inactive`, not as a distinct state.

`GET /api/v1/user/profile` returns the same profile object; use it when you
want the user-scoped handler rather than the auth one.

## API keys

### `GET /api/v1/keys` — list

```bash
curl -sS "${H[@]}" "$SUBTOKEN_API/keys?page=1&page_size=100" | jq '.data.items[] | {id, name, key_type, status, group_id, quota, quota_used}'
```

| Parameter | Notes |
| --- | --- |
| `page`, `page_size` | Standard pagination. |
| `search` | Matches name. |
| `status` | `active`, `inactive`, `quota_exhausted`, `expired`. |
| `key_type` | `inference` or `panel`. |
| `group_id` | Filter by bound group. |
| `sort_by`, `sort_order` | e.g. `sort_by=name&sort_order=asc`. |

`key` comes back **masked** (`sk-abcdef…wxyz`). The full value is returned only
by the create response. The dashboard can use a newly-created key while its
secret remains in that browser tab's memory; a later list/detail response never
reveals it. If you no longer have the secret, rotate the key — you cannot read
it back.

Per-key live counters come along on each row: `usage_5h` / `usage_1d` /
`usage_7d`, `window_*_start`, and `reset_5h_at` / `reset_1d_at` /
`reset_7d_at`.

### `POST /api/v1/keys` — create

```bash
curl -sS "${H[@]}" -H 'Content-Type: application/json' \
  -X POST "$SUBTOKEN_API/keys" -d '{
    "name": "laptop Claude Code",
    "group_id": 12,
    "quota": 25,
    "expires_in_days": 90,
    "rate_limit_1d": 10,
    "rate_limit_5h": 5,
    "ip_whitelist": ["203.0.113.7"]
  }' | jq '.data'
```

Only `name` is required.

Set `key_type` to `inference` (the default) for `/v1/*` model requests or to
`panel` for `/api/v1/*` account automation. Panel keys are server-generated,
cannot use `custom_key`, do not take a group, and ignore spend quota/rate-limit
fields. They still honor expiry and IP allow/deny lists. `key_type` cannot be
changed later; rotate by creating a replacement of the other type.

| Field | Type | Notes |
| --- | --- | --- |
| `name` | string | Label only. |
| `group_id` | int \| null | Decides which models the key can call and at what price. `null` = no group, and the key is then **rejected by every `/v1/*` route**. |
| `custom_key` | string | Bring your own key text. One-shot, create-only. |
| `ip_whitelist` | string[] | Only these addresses may use the key. |
| `ip_blacklist` | string[] | These addresses may not. |
| `quota` | number | Lifetime spend cap in USD. `0` or absent = unlimited. |
| `expires_in_days` | number | Omit for "never". |
| `rate_limit_5h` / `_1d` / `_7d` | number | Per-window spend caps in USD. `0`/absent = that window uncapped. |

**This is the one response that returns the full `key`.** Capture it before
doing anything else — the list will only ever show it masked.

```bash
curl -sS "${H[@]}" -H 'Content-Type: application/json' \
  -X POST "$SUBTOKEN_API/keys" -d '{"name":"ci","group_id":12}' \
  | jq -r '.data.key' > ~/.config/subtoken/api-key-ci && chmod 600 ~/.config/subtoken/api-key-ci
```

### `PUT /api/v1/keys/:id` — update

```bash
curl -sS "${H[@]}" -H 'Content-Type: application/json' \
  -X PUT "$SUBTOKEN_API/keys/42" -d '{"status":"inactive"}' | jq '.data.status'
```

Send only the fields you are changing.

| Field | Notes |
| --- | --- |
| `name`, `group_id` | Rebinding a group changes which models the key can call and its price. |
| `status` | `active` or `inactive` only. This is the safe way to cut a key off. |
| `quota` | New cap in USD. `0` = unlimited. **Omit the field** to leave it alone — sending `null` means "no change", not "clear". |
| `expires_at` | ISO timestamp. Same omit-vs-`null` rule. |
| `reset_quota` | `true` zeroes `quota_used`. The only way to reuse an exhausted key without editing the cap. |
| `rate_limit_5h` / `_1d` / `_7d` | New caps. |
| `reset_rate_limit_usage` | `true` clears the 5h/1d/7d counters. |
| `ip_whitelist`, `ip_blacklist` | Replaces the list. |

**Rotating a key** is create-then-swap, not update: create the new key, move
your tools, verify, then delete the old one. There is no way to re-read a key's
value.

### `DELETE /api/v1/keys/:id`

```bash
curl -sS "${H[@]}" -X DELETE "$SUBTOKEN_API/keys/42" | jq '.data.message'
```

Irreversible. Prefer `status: inactive` when the goal is "stop using this
temporarily".

### Other key reads

```bash
# One key
curl -sS "${H[@]}" "$SUBTOKEN_API/keys/42" | jq '.data'

# That key's daily spend, 1-90 days (strict rate limiter)
curl -sS "${H[@]}" "$SUBTOKEN_API/user/api-keys/42/usage/daily?days=30&timezone=UTC" | jq '.data.items[-1]'

# Today + total for a batch of your own keys (POST because it takes a list)
curl -sS "${H[@]}" -H 'Content-Type: application/json' \
  -X POST "$SUBTOKEN_API/usage/dashboard/api-keys-usage" -d '{"api_key_ids":[42,43]}' | jq '.data.stats'
```

## Groups and channels

A key's **group** decides which models it can call and the multiplier applied to
list price. Read the catalogue before creating keys.

```bash
# Groups you may bind (public, explicitly allowed, or subscription-backed)
curl -sS "${H[@]}" "$SUBTOKEN_API/groups/available" | jq '.data[] | {id, name, platform, subscription_type, rate_multiplier, is_exclusive}'

# Your personal multiplier overrides, keyed by group id
curl -sS "${H[@]}" "$SUBTOKEN_API/groups/rates" | jq '.data'
```

`/groups/rates` returns a `{group_id: multiplier}` map, or `null` when you have
no overrides. Your effective rate is the override when present, else the
group's `rate_multiplier`. `GET /v1/sub2api/billing` resolves this for a
specific key including any peak window — see
[key-introspection.md](key-introspection.md).

`is_exclusive: true` means the group is granted to specific users rather than
open; if a group you want is missing from `/groups/available` you either lack
the grant or it is exclusive.

```bash
# Channels: providers plus the models and pricing you can reach through them
curl -sS "${H[@]}" "$SUBTOKEN_API/channels/available" | jq '.data[] | {name, platforms: [.platforms[] | {platform, models: (.supported_models | length)}]}'
```

## Redeem codes

```bash
# Redeem (spends the code; case-sensitive, one-shot)
curl -sS "${H[@]}" -H 'Content-Type: application/json' \
  -X POST "$SUBTOKEN_API/redeem" -d '{"code":"GIFT-ABC-123"}' | jq '.data'
```

```json
{ "message": "…", "type": "balance", "value": 10, "new_balance": 22.5 }
```

`type` decides the effect: `balance` credits `new_balance`, `admin_concurrency`
credits `new_concurrency`, subscription codes activate a group instead. Check
the matching `new_*` field — it is absent for types that do not produce one.

A redeemed code cannot be reused; a second attempt fails. Confirm the exact
code string before posting, including case.

```bash
# History — bare array, no pagination envelope
curl -sS "${H[@]}" "$SUBTOKEN_API/redeem/history" | jq '.data[] | {code, type, value, used_at}'
```

## Subscriptions

```bash
curl -sS "${H[@]}" "$SUBTOKEN_API/subscriptions"        | jq '.data[] | {id, group_id, status, expires_at}'
curl -sS "${H[@]}" "$SUBTOKEN_API/subscriptions/active"  | jq '.data | length'
curl -sS "${H[@]}" "$SUBTOKEN_API/subscriptions/summary" | jq '.data'
```

`/subscriptions/summary` is the dashboard rollup — `active_count` plus per
subscription `daily_progress`, `weekly_progress`, `monthly_progress`,
`days_remaining`. Use it for "how close am I to a cap".

```bash
# Which cap is actually binding?
curl -sS "${H[@]}" "$SUBTOKEN_API/subscriptions/progress" | jq '.data[] |
  {id: .subscription_id, days_left: .days_remaining,
   caps: [ .daily, .weekly, .monthly | select(. != null) | {used, limit, pct: .percentage, reset_in: .reset_in_seconds} ]}'
```

Each period is `null` when the group defines no such limit — that is not zero
usage, it is an absent cap. `percentage` is a percentage, not a fraction.

`GET /api/v1/subscriptions/:id/progress` narrows to one subscription.

## Announcements

```bash
curl -sS "${H[@]}" "$SUBTOKEN_API/announcements" | jq '.data[] | {id, title, published_at}'
curl -sS "${H[@]}" "$SUBTOKEN_API/announcements?unread_only=1" | jq '.data | length'

# Mark one read
curl -sS "${H[@]}" -X POST "$SUBTOKEN_API/announcements/12/read" | jq '.data.message'
```

Bare array, no pagination. `unread_only=1` (not `true`) is what the dashboard
sends.

## Affiliate

```bash
# Your referral code, invitees, and rebates
curl -sS "${H[@]}" "$SUBTOKEN_API/user/aff" | jq '.data'

# Move unclaimed rebate quota into spendable balance
curl -sS "${H[@]}" -X POST "$SUBTOKEN_API/user/aff/transfer" | jq '.data'
```

The transfer moves balance and is not reversible. Read `/user/aff` first to see
what is actually pending.

## Platform quotas

Per-provider rate and spend ceilings across everything you own, as opposed to
one key's limits:

```bash
curl -sS "${H[@]}" "$SUBTOKEN_API/user/platform-quotas" | jq '.data'
```

## Owned accounts (Claude reset credits)

Applies only if you have bought or rented a Claude account on the marketplace.

```bash
curl -sS "${H[@]}" "$SUBTOKEN_API/user/owned-accounts/claude-reset" | jq '.data'

curl -sS "${H[@]}" "$SUBTOKEN_API/user/accounts/9/claude/reset-credits" | jq '.data'

# Redeem a credit — audited, and Idempotency-Key is required
curl -sS "${H[@]}" -H 'Content-Type: application/json' -H "Idempotency-Key: redeem-$(date +%s)-9" \
  -X POST "$SUBTOKEN_API/user/accounts/9/claude/reset-credits/redeem" | jq '.data'
```

Ownership is re-checked per request: you only ever see your own accounts. The
redeem call requires an `Idempotency-Key` header — the server picks the grant,
so a retry without a fresh key risks redeeming twice. Both routes are on the
strict rate limiter.

## Profile updates

```bash
curl -sS "${H[@]}" -H 'Content-Type: application/json' \
  -X PUT "$SUBTOKEN_API/user" -d '{"username":"new-name"}' | jq '.data.username'

curl -sS "${H[@]}" -H 'Content-Type: application/json' \
  -X PUT "$SUBTOKEN_API/user" -d '{
    "balance_notify_enabled": true,
    "balance_notify_threshold": 5
  }' | jq '.data.balance_notify_enabled'
```

Exactly four fields are honoured: `username`, `avatar_url`,
`balance_notify_enabled`, `balance_notify_threshold`. Send only what you mean
to change — the handler binds the body loosely, so an unrecognised field is
**silently ignored rather than rejected**. `email` and `concurrency` are
accepted by the wire format and then dropped; balance, role, and concurrency
have no user-facing write path at all.

To change your login email, use the binding flow in
[security.md](security.md). To add alert recipients, use the notify-email
routes there — `balance_notify_extra_emails` sent here does nothing.
