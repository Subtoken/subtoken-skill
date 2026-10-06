# Key introspection (API key, `/v1/*`)

Three endpoints answer "what can this key do, what has it spent, what will it
cost me" using **only the API key itself** — no session token, no dashboard.
This is the same data the dashboard's public **API Key Usage** page
(`/key-usage`) renders, and the only Subtoken surface reachable without a
session.

These responses are **not** wrapped in the `{code, message, data}` envelope and
errors are OpenAI-shaped. See [auth.md](auth.md).

## `GET /v1/usage` — key usage, quota, and limits

The main endpoint. Quota, spend, rate-limit windows, expiry, plus per-day and
per-model breakdowns.

```bash
curl -sS -H "Authorization: Bearer $SUBTOKEN_API_KEY" \
  -H "Accept: application/json" \
  "$SUBTOKEN_GW/usage"
```

### Query parameters

| Parameter | Default | Notes |
| --- | --- | --- |
| `days` | `30` | `1`–`90`. Drives `daily_usage` and `model_stats`. Out of range is `400 invalid_request_error`. |
| `start_date` | 30 days ago | `YYYY-MM-DD`. Window for `model_stats`. |
| `end_date` | now | `YYYY-MM-DD`. |
| `timezone` | server default | IANA name. Sets the day boundary for `daily_usage`; send it or "today" may disagree with the dashboard. |

### Three response modes

`mode` tells you which shape you got. Do not assume `remaining` means the same
thing in each.

**`mode: "quota_limited"`** — the key has a quota or at least one rate limit
(`quota > 0`, or a 5h/1d/7d cap set).

```json
{
  "mode": "quota_limited",
  "isValid": true,
  "status": "active",
  "quota": { "limit": 50, "used": 12.34, "remaining": 37.66, "unit": "USD" },
  "remaining": 37.66,
  "unit": "USD",
  "rate_limits": [
    { "window": "5h", "limit": 10, "used": 3.2, "remaining": 6.8,
      "window_start": "2026-10-04T08:00:00Z", "reset_at": "2026-10-04T13:00:00Z" },
    { "window": "1d", "limit": 25, "used": 9.1, "remaining": 15.9 },
    { "window": "7d", "limit": 100, "used": 41.0, "remaining": 59.0 }
  ],
  "expires_at": "2026-12-01T00:00:00Z",
  "days_until_expiry": 58,
  "usage": { "…": "…" },
  "daily_usage": [ "…" ],
  "model_stats": [ "…" ]
}
```

`quota` and `rate_limits` appear **only** when configured, and `reset_at`
appears only while the window is live. Iterate, do not index:

```bash
curl -sS -H "Authorization: Bearer $SUBTOKEN_API_KEY" "$SUBTOKEN_GW/usage" |
  jq -r '.rate_limits[]? | "\(.window)\t\(.used)/\(.limit)\tresets \(.reset_at // "n/a")"'
```

`status` reflects the key's lifecycle: `active`, `inactive`, `quota_exhausted`,
`expired`. `isValid` is `true` for `active`, `quota_exhausted` and `expired` —
it means "this is a real key", **not** "this key may still make requests".
Judge callability from `status`, and confirm with `GET /v1/models`.

**`mode: "unrestricted"` (subscription group)** — the key's group is a
subscription, so there is no wallet to drain:

```json
{
  "mode": "unrestricted",
  "isValid": true,
  "planName": "Pro Monthly",
  "unit": "USD",
  "remaining": 42.5,
  "subscription": {
    "daily_usage_usd": 1.2, "weekly_usage_usd": 8.0, "monthly_usage_usd": 31.4,
    "daily_limit_usd": 5, "weekly_limit_usd": 30, "monthly_limit_usd": 120,
    "weekly_window_start": "2026-09-28T00:00:00Z",
    "expires_at": "2026-11-01T00:00:00Z"
  },
  "usage": { "…": "…" }, "daily_usage": [ "…" ], "model_stats": [ "…" ]
}
```

`remaining` is the **smallest** headroom across whichever of the daily, weekly
and monthly limits the group defines — so it can read `0` while monthly spend
looks tiny. Check the individual `*_usage_usd` / `*_limit_usd` pairs to see
which cap bound.

**`mode: "unrestricted"` (wallet balance)** — pay-as-you-go group:

```json
{
  "mode": "unrestricted",
  "isValid": true,
  "planName": "钱包余额",
  "remaining": 12.34,
  "unit": "USD",
  "balance": 12.34,
  "usage": { "…": "…" }, "daily_usage": [ "…" ], "model_stats": [ "…" ]
}
```

`planName` is the untranslated Chinese string for "wallet balance". Match on
`mode` plus the presence of `balance`, never on `planName`. `remaining` and
`balance` are the same number: the key owner's spendable balance.

### The shared `usage` block

Present in every mode. This is the answer to "what did this key cost".

```json
{
  "today":  { "requests": 42, "input_tokens": 120000, "output_tokens": 31000,
              "cache_creation_tokens": 8000, "cache_read_tokens": 90000,
              "total_tokens": 249000, "cost": 1.204, "actual_cost": 0.612 },
  "total":  { "requests": 3180, "…": "same shape" },
  "average_duration_ms": 1420,
  "rpm": 6.4,
  "tpm": 38200
}
```

`cost` is list price; `actual_cost` is what was actually deducted after the
group multiplier. **They differ.** Billing questions are about `actual_cost`;
comparing value against official pricing is about `cost`. `rpm`/`tpm` are
trailing five-minute averages.

### `daily_usage` and `model_stats`

`daily_usage` is one row per day over `days`, aligned to `timezone`:

```json
[ { "date": "2026-10-04", "requests": 42, "input_tokens": 120000,
    "output_tokens": 31000, "cache_read_tokens": 90000,
    "cache_write_tokens": 8000, "total_tokens": 249000,
    "cost": 1.204, "actual_cost": 0.612 } ]
```

`model_stats` breaks the window down per model. Both are best-effort: if the
underlying aggregate query fails, the field is **omitted** rather than null.
Check with `has("model_stats")`, not for a non-null value.

```bash
# Spend per model over 7 days
curl -sS -H "Authorization: Bearer $SUBTOKEN_API_KEY" \
  "$SUBTOKEN_GW/usage?days=7&timezone=UTC" |
  jq -r '.model_stats[]? | "\(.model)\t\(.actual_cost // .cost)"'
```

## `GET /v1/sub2api/billing` — effective rate multiplier

What the key is actually charged at, right now. Answers "why is my spend
different from list price" without reconstructing anything from usage logs.

```bash
curl -sS -H "Authorization: Bearer $SUBTOKEN_API_KEY" "$SUBTOKEN_GW/sub2api/billing" | jq
```

```json
{
  "object": "sub2api.key_billing",
  "schema_version": 1,
  "billing_scope": "…",
  "group_rate_multiplier": 0.5,
  "user_rate_multiplier": 0.4,
  "resolved_rate_multiplier": 0.4,
  "peak_rate_enabled": true,
  "peak_start": "…", "peak_end": "…", "peak_rate_multiplier": 0.8,
  "applied_peak_multiplier": null,
  "effective_rate_multiplier": 0.4,
  "timezone": "Asia/Shanghai",
  "observed_at": "2026-10-04T12:00:00Z"
}
```

How the multipliers compose — the last field is the one that matters:

1. `group_rate_multiplier` — the key's group default.
2. `user_rate_multiplier` — a per-user override, **present only when it differs
   from the group rate**. Absent means "no override", not "zero".
3. `resolved_rate_multiplier` — `user_rate_multiplier` when present, else the
   group rate.
4. `effective_rate_multiplier` — `resolved` further multiplied by the peak
   window's multiplier when the current time falls inside one.

So a `peak_rate_multiplier` of `0.8` is a **discount**, not a surcharge. Check
`applied_peak_multiplier` for non-null to know whether a peak window is active
right now, and read `timezone` before comparing against your own clock. The
response is `Cache-Control: no-store` — do not cache it across a peak boundary.

Fails `404 not_found_error` in simple run mode, and `403 permission_error` when
the key has no group assigned.

## `GET /v1/models` — what this key may call

```bash
curl -sS -H "Authorization: Bearer $SUBTOKEN_API_KEY" "$SUBTOKEN_GW/models" | jq -r '.data[].id'
```

The catalogue is **per key**: it reflects the key's group, the group's model
allowlist, and the accounts currently serving it. Two keys on different groups
see different lists. A model missing here is not callable with this key, no
matter what the model plaza advertises.

The response shape follows the group's platform — OpenAI and Grok groups get
`{"object": "list", "data": [{"id": …, "object": "model"}]}`, Anthropic-style
groups get the `{"data": [{"id", "type", "display_name"}], "has_more", …}`
form. Read the ids defensively:

```bash
curl -sS -H "Authorization: Bearer $SUBTOKEN_API_KEY" "$SUBTOKEN_GW/models" |
  jq -r '(.data // .models // [])[] | .id // .name'
```

`GET /v1/models/{model}` returns the single-model form of the same lookup.

Passing `?client_version=` switches to the ChatGPT Codex manifest format that
Codex CLI and the Codex app expect. Leave it off unless you are debugging a
Codex model picker — it is a different response schema, not a filter.

```bash
curl -sS -H "Authorization: Bearer $SUBTOKEN_API_KEY" \
  "$SUBTOKEN_GW/models?client_version=0.20.0" | jq 'keys'
```

## Triaging a key that "does not work"

Run these in order; the first non-200 names the problem.

```bash
K="Authorization: Bearer $SUBTOKEN_API_KEY"

# 1. Credential accepted at all?
curl -sS -o /dev/null -w '%{http_code}\n' -H "$K" "$SUBTOKEN_GW/usage"

# 2. Group assigned, key enabled, not expired?
curl -sS -H "$K" "$SUBTOKEN_GW/usage" | jq '{mode, status, isValid, days_until_expiry}'

# 3. Budget left?
curl -sS -H "$K" "$SUBTOKEN_GW/usage" | jq '{remaining, unit, rate_limits}'

# 4. Model actually in the key's catalogue?
curl -sS -H "$K" "$SUBTOKEN_GW/models" | jq -r '.data[]?.id' | grep -Fx 'claude-sonnet-4-5'
```

| Symptom | Cause |
| --- | --- |
| `401 INVALID_API_KEY` | Wrong or revoked key, or `status: inactive`. |
| `401 API_KEY_REQUIRED` | No credential header reached the route. |
| `400 api_key_in_query_deprecated` | Key in the query string; move it to a header. |
| `403 permission_error` on billing | Key has no group assigned. |
| `429` on a call that should be cheap | Invalid-credential abuse counter tripped after repeated bad keys. Wait it out; do not keep retrying. |
| `200` but the model 404s on inference | Model is not in the key's catalogue — see step 4. |
| `status: quota_exhausted` | Key hit its own `quota`. Raise or reset it in the dashboard. |
| Rate-limit window blocking | A 5h/1d/7d cap is exhausted; wait for `reset_at`. |

## Related panel endpoints

Two of these have panel equivalents that accept a panel key or session token and support
narrower filters — `GET /api/v1/user/api-keys/:id/usage/daily` and
`GET /api/v1/usage/dashboard/api-keys-usage`. See
[usage.md](usage.md).
