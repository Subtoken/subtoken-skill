# Usage, spend, and errors

Panel endpoints under `/api/v1`. A **panel key or session token** works. The
whole `/usage` group sits behind the strict rate limiter — expect to page rather
than sweep, and to back off on `429`.

```bash
H=(-H "Authorization: Bearer $SUBTOKEN_PANEL_AUTH" -H 'Accept: application/json')
TZ='timezone=UTC'
```

## The two cost numbers

Every usage figure comes in a pair, and mixing them up is the most common
reporting error:

| Field | Meaning |
| --- | --- |
| `cost` / `total_cost` | **List price** — what the same tokens would cost at official provider rates. |
| `actual_cost` / `total_actual_cost` | **What you were actually charged**, after the group multiplier, any per-user override, and any peak window. |

Invoices, balance deltas, and "what did I spend" questions are all
`actual_cost`. "What did I save versus paying the provider directly" is
`cost - actual_cost`.

`rate_multiplier` on an individual log row is the multiplier that produced
`actual_cost`. If it disagrees with `GET /v1/sub2api/billing`, the log is older
than the current rate — rates change, logs are immutable.

## Raw logs

### `GET /api/v1/usage`

```bash
curl -sS "${H[@]}" \
  "$SUBTOKEN_API/usage?page=1&page_size=50&start_date=2026-10-01&end_date=2026-10-04&$TZ" |
  jq '.data.items[] | {created_at, model, input_tokens, output_tokens, actual_cost, duration_ms}'
```

Filters:

| Parameter | Notes |
| --- | --- |
| `page`, `page_size` | Standard pagination; `page_size` caps at 100. |
| `start_date`, `end_date` | `YYYY-MM-DD`, inclusive. |
| `timezone` | IANA name. Sets the day boundary — send it or "today" may not match the dashboard. |
| `api_key_id`, `group_id`, `model` | Narrow to one key, group, or model. |
| `request_type` | `unknown`, `sync`, `stream`, `ws_v2`, `cyber`, `live`. |
| `stream` | `true` / `false`. |
| `billing_mode`, `billing_type` | Per-token vs per-request billing splits. |
| `native_compaction_v2` | Filter the compaction path. |
| `sort_by`, `sort_order` | Server-side; `sort_order` is `asc` or `desc`. |

A log row is dense. The fields that answer real questions:

```json
{
  "id": 918273, "created_at": "2026-10-04T11:02:33Z",
  "model": "claude-sonnet-4-5", "request_id": "req_…",
  "api_key_id": 42, "group_id": 12, "account_id": null, "subscription_id": null,
  "inbound_endpoint": "/v1/messages", "upstream_endpoint": "/v1/messages",
  "input_tokens": 4210, "output_tokens": 812,
  "cache_creation_tokens": 0, "cache_read_tokens": 20480,
  "total_cost": 0.0913, "actual_cost": 0.0457, "rate_multiplier": 0.5,
  "request_type": "stream", "stream": true,
  "duration_ms": 4310, "first_token_ms": 620,
  "billing_mode": "per_token",
  "user_agent": "claude-cli/1.0.98", "ip_address": "203.0.113.7"
}
```

`account_id` is the upstream account that served the request and `group_id` is
the group billed — both useful when reconciling against seller earnings. Image
generation adds `image_count`, `image_size`, and `image_*_cost` fields.

### `GET /api/v1/usage/:id` — one log

```bash
curl -sS "${H[@]}" "$SUBTOKEN_API/usage/918273" | jq '.data'
```

## Aggregates

### `GET /api/v1/usage/stats`

```bash
# Named period
curl -sS "${H[@]}" "$SUBTOKEN_API/usage/stats?period=today&$TZ" | jq '.data'

# Explicit range, optionally scoped to one key
curl -sS "${H[@]}" \
  "$SUBTOKEN_API/usage/stats?start_date=2026-09-01&end_date=2026-09-30&api_key_id=42&$TZ" | jq '.data'
```

`period` accepts `today`, `week`, `month`, `year`; omit it when passing
`start_date`/`end_date`.

```json
{
  "period": "today",
  "total_requests": 412,
  "total_input_tokens": 1204000,
  "total_output_tokens": 318000,
  "total_cache_tokens": 900000,
  "total_cache_read_tokens": 820000,
  "total_cache_creation_tokens": 80000,
  "total_tokens": 2422000,
  "total_cost": 7.41, "total_actual_cost": 3.70,
  "average_duration_ms": 1380,
  "models": { "claude-sonnet-4-5": 512 },
  "endpoints": [ { "endpoint": "/v1/messages", "requests": 390, "total_tokens": 2100000, "cost": 6.2, "actual_cost": 3.1 } ]
}
```

`total_cache_tokens` is the sum of read + creation. `models` is a
`{model: request_count}` map, not spend. `endpoints` is an array; the
`upstream_endpoints` and `endpoint_paths` variants exist alongside it.

## Dashboard rollups

Pre-aggregated for the dashboard. Prefer these over paging raw logs for charts
and totals.

```bash
# Headline cards: totals, today, average duration, rpm/tpm
curl -sS "${H[@]}" "$SUBTOKEN_API/usage/dashboard/stats" | jq '.data | {total_actual_cost, today_actual_cost, rpm, tpm, average_duration_ms}'

# Per-platform split
curl -sS "${H[@]}" "$SUBTOKEN_API/usage/dashboard/stats" | jq '.data.by_platform'
```

### `GET /api/v1/usage/dashboard/trend`

```bash
curl -sS "${H[@]}" \
  "$SUBTOKEN_API/usage/dashboard/trend?start_date=2026-09-01&end_date=2026-10-01&granularity=day&$TZ" |
  jq '.data | {start_date, end_date, granularity, points: (.trend | length)}'
```

Returns `{trend: [...], start_date, end_date, granularity}`. Each point is
`{date, requests, input_tokens, output_tokens, cache_creation_tokens,
cache_read_tokens, total_tokens, cost, actual_cost}`.

`granularity` is `day` or `hour`; it is echoed back, so trust the response over
your request when they disagree.

Filters: `start_date`, `end_date`, `granularity`, `api_key_id`, `model`,
`group_id`, `request_type`, `stream`, `native_compaction_v2`, `billing_type`,
`billing_mode`, `timezone`.

### `GET /api/v1/usage/dashboard/models`

```bash
curl -sS "${H[@]}" "$SUBTOKEN_API/usage/dashboard/models?start_date=2026-09-01&end_date=2026-09-30&$TZ" |
  jq -r '.data.models | sort_by(-.actual_cost)[] | "\(.model)\t\(.requests)\t\(.actual_cost)"'
```

Returns `{models: [{model, requests, input_tokens, output_tokens,
cache_creation_tokens, cache_read_tokens, total_tokens, cost, actual_cost}],
start_date, end_date}`. `account_cost` exists on this struct but is
admin-only and never populated for a user call.

### `GET /api/v1/usage/dashboard/snapshot-v2`

One round trip for trend + model + group breakdowns. Cheapest way to build a
full dashboard.

```bash
curl -sS "${H[@]}" \
  "$SUBTOKEN_API/usage/dashboard/snapshot-v2?include_trend=true&include_model_stats=true&include_group_stats=true&$TZ" |
  jq '.data | {generated_at, granularity, trend_points: (.trend | length), models: (.models | length), groups: (.groups | length)}'
```

The `include_*` flags are **query parameters** on a `GET`. Omitted sections are
**absent**, not empty, so guard with `has()`:

```bash
jq '.data | {trend: (.trend // []), models: (.models // []), groups: (.groups // [])}'
```

`groups[]` is `{group_id, group_name, requests, total_tokens, cost,
actual_cost}`.

## Per-key usage

```bash
# One key, daily rows, 1-90 days (strict limiter)
curl -sS "${H[@]}" "$SUBTOKEN_API/user/api-keys/42/usage/daily?days=30&$TZ" |
  jq '.data | {days, start_date, end_date, last: .items[-1]}'
```

Rows are `{date, requests, input_tokens, output_tokens, cache_read_tokens,
cache_write_tokens, total_tokens, cost, actual_cost}`. Note this struct says
`cache_write_tokens` while the rest of the API says `cache_creation_tokens` —
same quantity, different name.

```bash
# Today + total for several keys at once
curl -sS "${H[@]}" -H 'Content-Type: application/json' \
  -X POST "$SUBTOKEN_API/usage/dashboard/api-keys-usage" -d '{"api_key_ids":[42,43,44]}' |
  jq '.data.stats | to_entries[] | {key_id: .key, today: .value.today_actual_cost, total: .value.total_actual_cost}'
```

## Errors

Failed requests are logged separately from billed ones — they cost nothing but
they explain gaps in your usage table.

```bash
curl -sS "${H[@]}" \
  "$SUBTOKEN_API/usage/errors?page=1&page_size=50&start_date=2026-10-01&category=rate_limit&$TZ" |
  jq '.data.items[] | {created_at, model, status_code, category, message}'
```

Filters: `page`, `page_size`, `start_date`, `end_date`, `timezone`, `model`,
`status_code`, `category`, `api_key_id`, `sort_by`, `sort_order`.

Rows carry `category`, `status_code`, `message`, `key_name`, `key_deleted`,
`client_ip`, `group_name`, and `user_agent`.

```bash
# Full upstream error body for one failure
curl -sS "${H[@]}" "$SUBTOKEN_API/usage/errors/5150" | jq -r '.data.error_body'
```

`key_deleted: true` means the request used a key you have since deleted — the
row survives the key. `GET /api/v1/usage/errors/:id` adds `error_body` and
`upstream_status_code`; `upstream_status_code` is how you tell "Subtoken
rejected it" from "the provider rejected it".

```bash
# Failure rate by status code today
curl -sS "${H[@]}" "$SUBTOKEN_API/usage/errors?page=1&page_size=100&$TZ" |
  jq '[.data.items[] | .status_code] | group_by(.) | map({code: .[0], n: length})'
```

## Worked examples

**What did I spend today, per key?**

Prefer requesting this endpoint with all relevant key IDs in the REPL, then
inspect the returned map. The loop shown here is a read-only fallback for
deployments without the batch usage endpoint; run it only after reviewing the
key IDs and keep it out of mutation workflows.

```bash
for id in $(curl -sS "${H[@]}" "$SUBTOKEN_API/keys?page_size=100" | jq -r '.data.items[].id'); do
  printf '%s\t' "$id"
  curl -sS "${H[@]}" "$SUBTOKEN_API/user/api-keys/$id/usage/daily?days=1&$TZ" |
    jq -r '.data.items[0].actual_cost // 0'
done
```

**Savings versus paying providers directly, this month:**

```bash
curl -sS "${H[@]}" \
  "$SUBTOKEN_API/usage/stats?start_date=2026-10-01&end_date=2026-10-31&$TZ" |
  jq '.data | {list_price: .total_cost, charged: .total_actual_cost,
               saved: (.total_cost - .total_actual_cost)}'
```

**Export the filtered table as CSV** (what the dashboard's **Export CSV**
button produces):

```bash
{
  echo 'date,model,requests,input,output,cache_read,cache_create,total_tokens,list_cost,charged'
  curl -sS "${H[@]}" \
    "$SUBTOKEN_API/usage/dashboard/models?start_date=2026-10-01&end_date=2026-10-31&$TZ" |
    jq -r '.data.models[] |
      [.date // "", .model, .requests, .input_tokens, .output_tokens,
       .cache_read_tokens, .cache_creation_tokens, .total_tokens, .cost, .actual_cost] | @csv'
} > usage-2026-10.csv
```

**Explain a spend spike on one key:**

```bash
K=42
curl -sS "${H[@]}" "$SUBTOKEN_API/user/api-keys/$K/usage/daily?days=7&$TZ" |
  jq -r '.data.items[] | "\(.date)\t\(.requests)\t\(.actual_cost)"'
curl -sS "${H[@]}" \
  "$SUBTOKEN_API/usage?api_key_id=$K&start_date=$(date -u -d '2 days ago' +%F)&end_date=$(date -u +%F)&page_size=100&$TZ" |
  jq -r '.data.items | group_by(.model)[] | "\(.[0].model)\t\(length) reqs\t\(map(.actual_cost) | add)"'
```
