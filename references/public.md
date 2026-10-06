# Public and read-only endpoints

Everything here is a read. Most needs **no credential at all**, which makes
this the right surface for orientation and health checks.

```bash
# No auth header at all in this file unless a line says otherwise.
```

A **404 on any of these means the feature is switched off**, not that you got
the path wrong. Check the flags in `/api/v1/settings/public` first.

## `GET /health` — liveness

Outside the `/api/v1` prefix entirely.

```bash
curl -sS "$SUBTOKEN_BASE_URL/health"
```

```json
{ "status": "ok", "region": "default" }
```

No envelope, no auth. `region` identifies the serving region in a multi-region
deployment — use it to confirm which instance answered before debugging
inconsistency between calls.

```bash
# Poll until healthy
until curl -fsS "$SUBTOKEN_BASE_URL/health" > /dev/null; do sleep 2; done && echo up
```

## `GET /api/v1/settings/public` — feature flags

**Read this first on any new deployment.** One unauthenticated call that tells
you what is enabled and what to expect.

```bash
curl -sS "$SUBTOKEN_API/settings/public" | jq '.data'
```

```json
{
  "registration_enabled": true,
  "email_verify_enabled": true,
  "invitation_code_enabled": true,
  "promo_code_enabled": false,
  "password_reset_enabled": false,
  "totp_enabled": true,
  "passkey_enabled": true,
  "model_plaza_enabled": false,
  "model_plaza_require_auth": false,
  "backend_mode_enabled": false,
  "channel_monitor_mode": "v1",
  "turnstile_site_key": "0x4AAAAAAE5rwFA1T3vbpDWV",
  "site_name": "Subtoken",
  "site_subtitle": "Premium AI Accounts Market / LLM APIs at Fractional Cost"
}
```

Flags that change what you can do:

| Flag | Effect when false / off |
| --- | --- |
| `registration_enabled` | `POST /auth/register` is refused. |
| `email_verify_enabled` | Registration needs no email code. |
| `invitation_code_enabled` | Registration **requires** an invitation code. |
| `promo_code_enabled` | `validate-promo-code` is refused. |
| `password_reset_enabled` | Forgot/reset password are refused. |
| `totp_enabled` | TOTP routes are refused; step-up is unavailable. |
| `passkey_enabled` | Passkey ceremonies are refused. |
| `model_plaza_enabled` | `GET /model-plaza` → `404`. |
| `model_plaza_require_auth` | Plaza needs a session token too. |
| `backend_mode_enabled` | The panel is admin-only; every non-admin panel route answers `403`. |
| `channel_monitor_mode` | `v1` or `v2` — selects which monitor routes exist. |

`turnstile_site_key` being **present** means captcha is configured on the auth
entry points, so headless login needs a `turnstile_token`. See
[auth.md](auth.md).

`backend_mode_enabled: true` is the important one: it means this deployment is a
management backend, not a public product, and the user-facing surface in this
skill is closed to non-admins.

## `GET /api/v1/public/stats` — platform metrics

Anonymous, and the same numbers the landing page shows.

```bash
curl -sS "$SUBTOKEN_API/public/stats" | jq '.data'
```

```json
{
  "tokens_routed": 918273645,
  "models_supported": 142,
  "median_ttft_ms": 820,
  "saved_this_month": 41230.55,
  "models":    [ { "model": "claude-sonnet-4-5", "requests": 120411, "tokens": 412000000,
                   "ttft_p50_ms": 760, "tps": 48.2, "success_rate": 0.997,
                   "cache_rate": 0.71, "deepswe": 41.2 } ],
  "providers": [ { "name": "…", "platform": "anthropic", "requests": 51200,
                   "share": 0.42, "ttft_p50_ms": 810, "success_rate": 0.996 } ],
  "routes":    [ { "model": "claude-sonnet-4-5", "provider": "…", "platform": "anthropic",
                   "requests": 21000, "share": 0.55, "success_rate": 0.998 } ],
  "generated_at": "2026-10-04T12:00:00Z"
}
```

Everything is a **30-day rolling window** except `tokens_routed` (all-time) and
`saved_this_month` (month to date). `models`, `providers`, `routes`, and
`generated_at` are **absent** until the collector has produced a snapshot — test
with `has()`, not for a non-empty array.

`routes[]` is the (model, provider) pair view: who serves what, and how well.
Use it to judge whether a model is well covered before you bind a key to it.

```bash
# Most reliable models by success rate, minimum volume
curl -sS "$SUBTOKEN_API/public/stats" |
  jq '[.data.models[]? | select(.requests > 1000)] | sort_by(-.success_rate) | .[:5]
      | .[] | "\(.model)\t\(.success_rate)\tttft \(.ttft_p50_ms)ms"'
```

`success_rate` and `share` are fractions (`0.997`, not `99.7`). `null` means no
sample yet.

## `GET /api/v1/model-plaza` — price catalogue

Per-group model pricing, plus LiteLLM official reference prices. Anonymous
unless `model_plaza_require_auth` is on.

```bash
curl -sS "$SUBTOKEN_API/model-plaza" |
  jq '.data.groups[] | {id, name, platform, rate_multiplier,
                        user_rate_multiplier, is_exclusive, models: (.models | length)}'
```

Currently **disabled on subtoken.net** (`model_plaza_enabled: false`) and answers
`404`. Each group carries `models[]` with `pricing` (what you pay),
`official_pricing` (the provider's list price), and optionally
`long_context_basis`, `time_pricing`, and the peak-rate fields.

`user_rate_multiplier` is present only when you sent a session token **and** the
operator set a per-user rate for that group. Its absence means "use
`rate_multiplier`", not zero.

This is a **catalogue of what exists**, not what your key can call. For that,
use `GET /v1/models` with the key itself — see
[key-introspection.md](key-introspection.md). A model in the plaza may not be in
your key's group.

## `GET /api/v1/marketplace/listings` — anonymous directory

Documented in full in [marketplace.md](marketplace.md). The one thing to know
here: it works with no credential, and sending a session token enriches the
response with viewer fields.

## `GET /api/v1/setup/status`

```bash
curl -sS "$SUBTOKEN_BASE_URL/setup/status" | jq '.data'
# { "needs_setup": false, "step": "completed" }
```

Outside the `/api/v1` prefix. `needs_setup: true` means the instance is waiting
on initial configuration.

## Channel monitors (session token, read-only)

Availability and latency for the channels serving your groups. **Needs a
session token** despite being pure reads. Which routes exist depends on
`channel_monitor_mode`.

### v1 (current on subtoken.net)

```bash
H=(-H "Authorization: Bearer $SUBTOKEN_PANEL_AUTH" -H 'Accept: application/json')

curl -sS "${H[@]}" "$SUBTOKEN_API/channel-monitors" |
  jq '.data.items[] | {id, name, provider, group_name, primary_model,
                       primary_status, primary_latency_ms, availability_7d,
                       extra: (.extra_models | length)}'

curl -sS "${H[@]}" "$SUBTOKEN_API/channel-monitors/3/status" | jq '.data'
```

Each item is `{id, name, provider, group_name, primary_model, primary_status,
primary_latency_ms, primary_ping_latency_ms, availability_7d, extra_models[],
timeline[], latest_quota?}`.

`primary_status` and `extra_models[].status` are the live verdict; `timeline[]`
is the recent history (`status`, `latency_ms`, `ping_latency_ms`, `checked_at`).
`latest_quota` appears only when the operator enables quota display — the server
strips it otherwise, so treat absence as "not published".

### v2

Only mounted when `channel_monitor_mode` is `v2`; otherwise the group is
inactive and these fail the feature gate.

```bash
curl -sS "${H[@]}" "$SUBTOKEN_API/channel-monitor-v2/dimensions" | jq '.data'
curl -sS "${H[@]}" "$SUBTOKEN_API/channel-monitor-v2/snapshot"  | jq '.data'
curl -sS "${H[@]}" "$SUBTOKEN_API/channel-monitor-v2/models"     | jq '.data'
curl -sS "${H[@]}" "$SUBTOKEN_API/channel-monitor-v2/matrix"     | jq '.data'
curl -sS "${H[@]}" "$SUBTOKEN_API/channel-monitor-v2/errors"     | jq '.data'
curl -sS "${H[@]}" "$SUBTOKEN_API/channel-monitor-v2/users"      | jq '.data'
```

All six are on the strict rate limiter.

## Orientation in one call

Optional read-only overview helper. For normal agent work, run the individual
requests in a persistent REPL and inspect each response as it arrives.

```bash
set -a; . ./subtoken.env; set +a   # SUBTOKEN_BASE_URL, SUBTOKEN_PANEL_AUTH
H=(-H "Authorization: Bearer $SUBTOKEN_PANEL_AUTH" -H 'Accept: application/json')

echo "== health =="
curl -sS "$SUBTOKEN_BASE_URL/health" | jq -c .

echo "== flags =="
curl -sS "$SUBTOKEN_API/settings/public" |
  jq -c '{registration_enabled, invitation_code_enabled, totp_enabled,
          passkey_enabled, model_plaza_enabled, backend_mode_enabled,
          channel_monitor_mode}'

echo "== me =="
curl -sS "${H[@]}" "$SUBTOKEN_API/auth/me" |
  jq -c '.data | {email, balance, concurrency, seller_status, status}'

echo "== key count =="
curl -sS "${H[@]}" "$SUBTOKEN_API/keys?page_size=100" |
  jq -c '{total: .data.total, active: [.data.items[] | select(.status == "active")] | length}'

echo "== spend today =="
curl -sS "${H[@]}" "$SUBTOKEN_API/usage/stats?period=today&timezone=UTC" |
  jq -c '.data | {requests: .total_requests, charged: .total_actual_cost}'
```

## Not public, despite appearances

| Route | Why it needs a credential |
| --- | --- |
| `/api/v1/user/*`, `/keys`, `/groups`, `/usage`, `/redeem`, `/subscriptions` | Panel key or session token; identity mutations are session-only. |
| `/api/v1/payment/*` | Panel key or session token. The two `public` sub-routes are the exception. |
| `/api/v1/seller/*` | Panel key or session token **and** `seller_status: active`. |
| `/v1/*` | API key. |
| `/api/v1/admin/*` | Admin credential — out of scope for this skill. |
| `/api/v1/payment/webhook/*` | Provider signature. Never call these. |
