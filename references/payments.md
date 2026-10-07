# Recharge, plans, and orders

Panel endpoints under `/api/v1/payment`. A **panel key or session token** works
for user payment routes, except the two `public` routes at the bottom which are
anonymous. Payment webhooks are provider callbacks, not user API operations.

```bash
H=(-H "Authorization: Bearer $SUBTOKEN_PANEL_AUTH" -H 'Accept: application/json')
```

**This is the money-moving surface.** Creating an order, requesting a refund,
and confirming payment all have consequences. Print the amount, method, and
order id and get explicit confirmation before calling a write.

## Read the configuration before spending

```bash
curl -sS "${H[@]}" "$SUBTOKEN_API/payment/config" | jq '.data'
```

```json
{
  "payment_enabled": true,
  "min_amount": 5, "max_amount": 1000,
  "daily_limit": 500, "max_pending_orders": 3,
  "order_timeout_minutes": 30,
  "balance_disabled": false,
  "balance_recharge_multiplier": 1,
  "subscription_usd_to_cny_rate": 7.2,
  "enabled_payment_types": ["alipay", "wxpay", "stripe"],
  "help_text": "", "help_image_url": "",
  "stripe_publishable_key": "pk_live_…"
}
```

Check first, because each of these refuses the order:

| Field | Refuses when |
| --- | --- |
| `payment_enabled` | `false` — the operator has switched payments off entirely. |
| `min_amount` / `max_amount` | Your amount is outside the window. |
| `daily_limit` | You have already recharged that much today. |
| `max_pending_orders` | You have too many unsettled orders. |
| `order_timeout_minutes` | How long you have to finish paying before `EXPIRED`. |
| `enabled_payment_types` | Your chosen `payment_type` is not in the list. |

`balance_recharge_multiplier` and `subscription_usd_to_cny_rate` are conversion
factors, not prices — a `0` rate means that path is disabled.

```bash
# Per-method limits, fees, and today's remaining allowance
curl -sS "${H[@]}" "$SUBTOKEN_API/payment/limits" |
  jq '.data | {global_min, global_max, methods: (.methods | to_entries | map({key, value: .value}))}'
```

```json
{
  "global_min": 5, "global_max": 1000,
  "methods": {
    "alipay": { "display_name": "Alipay", "daily_limit": 500, "daily_used": 40,
                "daily_remaining": 460, "single_min": 5, "single_max": 1000,
                "fee_rate": 0, "available": true }
  }
}
```

`daily_remaining` is the number that stops a second attempt today.
`fee_rate` is a fraction (`0.06` = 6%), and `available: false` means the method
is configured but currently unusable — usually a daily cap or an inactive
provider.

`GET /api/v1/payment/checkout-info` returns methods, plans, limits, help text,
and Stripe key in **one** call. Prefer it when you are building a checkout
flow; it saves three round trips and is what the dashboard uses.

## Subscription plans

```bash
curl -sS "${H[@]}" "$SUBTOKEN_API/payment/plans" |
  jq '.data[] | select(.for_sale) | {id, name, price, validity_days, validity_unit, daily_limit_usd, monthly_limit_usd, rate_multiplier}'
```

A plan bundles a group, a price, a validity period, and optional spend caps.
`for_sale: false` plans are listed but not purchasable — filter on it. The cap
fields are `null` when the plan's group sets no such limit.

```bash
curl -sS "${H[@]}" "$SUBTOKEN_API/payment/checkout-info" | jq '.data.plans | length'
```

## Creating an order

Order creation has no documented `Idempotency-Key` contract. If the request
times out or its response is lost, check `GET /payment/orders/my` for a matching
pending order before creating another. Compare amount, payment method, and
creation time; do not assume a network timeout means the order was not created.

```bash
curl -sS "${H[@]}" -H 'Content-Type: application/json' \
  -X POST "$SUBTOKEN_API/payment/orders" -d '{
    "amount": 50,
    "payment_type": "alipay",
    "order_type": "balance"
  }' | jq '.data'
```

| Field | Required | Notes |
| --- | --- | --- |
| `amount` | yes | USD credited to balance. For `order_type: "subscription"` this is ignored in favour of `plan_id`. |
| `payment_type` | yes | One of `enabled_payment_types`. `alipay` / `wxpay` produce a QR code; `alipay_direct` / `wxpay_direct` redirect; `stripe`, `easypay`, `airwallex` behave per provider. |
| `order_type` | yes | `balance` (top up) or `subscription` (buy a plan). |
| `plan_id` | for subscriptions | From `/payment/plans`. |
| `return_url` | no | Where the provider sends the browser after payment. |
| `payment_source` | no | Client hint. |
| `openid`, `wechat_resume_token`, `is_mobile` | no | WeChat Pay specifics — see below. |

The response tells you how to collect payment:

```json
{
  "order_id": 5012,
  "amount": 50, "pay_amount": 50, "fee_rate": 0, "currency": "USD",
  "expires_at": "2026-10-04T12:30:00Z",
  "result_type": "order_created",
  "payment_type": "alipay",
  "out_trade_no": "ST2026…",
  "pay_url": "https://…", "qr_code": "weixin://…", "client_secret": "…",
  "resume_token": "…"
}
```

`result_type` changes what you must do next:

| `result_type` | What it means | Next step |
| --- | --- | --- |
| `order_created` | Normal order. | Present `qr_code` or send the browser to `pay_url`. |
| `oauth_required` | WeChat needs the browser to authorise first. | Open `oauth.authorize_url`, then re-create with `openid` + `wechat_resume_token`. |
| `jsapi_ready` | WeChat JSAPI payload ready. | Hand `jsapi` / `jsapi_payload` to the client SDK. |

`amount` is what you receive; `pay_amount` is what the provider charges
(`pay_amount = amount × (1 + fee_rate)`). They differ when a fee applies.

`qr_code` may be a `weixin://` deep link or a base64 image. `pay_url` is
sometimes empty for QR flows — check both before assuming you have nothing to
show the user.

## Completing and tracking an order

Payment completes asynchronously: the provider webhook credits the balance.
Your job is to poll and confirm.

```bash
# Ask the server to re-check with the provider
curl -sS "${H[@]}" -H 'Content-Type: application/json' \
  -X POST "$SUBTOKEN_API/payment/orders/verify" -d '{"out_trade_no":"ST2026…"}' |
  jq '.data | {id, status, amount, paid_at, completed_at}'
```

```bash
# Or just read it back
curl -sS "${H[@]}" "$SUBTOKEN_API/payment/orders/5012" | jq '.data.status'
curl -sS "${H[@]}" "$SUBTOKEN_API/payment/orders/my?page=1&page_size=20&status=PENDING" |
  jq '.data.items[] | {id, amount, pay_amount, status, expires_at}'
```

`verify` is safe to repeat — it reconciles state, it does not charge. Use it
instead of guessing from the browser redirect.

Statuses:

```
PENDING → PAID → RECHARGING → COMPLETED
   ↓          ↓
EXPIRED   CANCELLED / FAILED
   ↓
REFUND_REQUESTED → REFUNDING → REFUND_PENDING → PARTIALLY_REFUNDED → REFUNDED
                                                              ↘ REFUND_FAILED
```

`PAID` means the provider took the money; `RECHARGING`/`COMPLETED` mean balance
was credited. Credit is not guaranteed at `PAID` — poll until `COMPLETED`.

The polling loop is an optional helper. In an agent task, request the order
status interactively, inspect it, wait as appropriate, then make the next
request. Stop when the response shows a terminal status; keep ambiguous payment
states visible rather than hiding them inside an unattended loop.

## Cancelling and refunding

```bash
# Cancel an unpaid order
curl -sS "${H[@]}" -X POST "$SUBTOKEN_API/payment/orders/5012/cancel" | jq '.data.status'

# Ask for a refund on a completed order
curl -sS "${H[@]}" -H 'Content-Type: application/json' \
  -X POST "$SUBTOKEN_API/payment/orders/5012/refund-request" -d '{"reason":"Bought by mistake"}' |
  jq '.data.status'
```

Only `PENDING` orders can be cancelled, and only `COMPLETED` ones can have a
refund requested — everything else is a `4xx`.

Check whether your method even supports refunds first:

```bash
curl -sS "${H[@]}" "$SUBTOKEN_API/payment/orders/refund-eligible-providers" |
  jq '.data.provider_instance_ids'
```

An empty list means no method on this deployment supports refunds; do not file a
request that cannot succeed. A refund request is a **request**, not a refund —
it moves the order to `REFUND_REQUESTED` and a human or the provider settles
it. It is not instant and not guaranteed.

## Public order lookups (no session token)

Two routes work without authentication, for a browser returning from a payment
provider with no session.

```bash
# By trade number — legacy compatibility path
curl -sS -H 'Content-Type: application/json' \
  -X POST "$SUBTOKEN_API/payment/public/orders/verify" -d '{"out_trade_no":"ST2026…"}' | jq '.data'

# By signed resume token — preferred
curl -sS -H 'Content-Type: application/json' \
  -X POST "$SUBTOKEN_API/payment/public/orders/resolve" -d '{"resume_token":"…"}' | jq '.data'
```

Both return a minimal projection:

```json
{ "out_trade_no": "ST2026…", "status": "COMPLETED",
  "paid": true, "created_at": "…", "expires_at": "…" }
```

`resume_token` comes back from `POST /payment/orders` and is signed, so prefer
it. Never log or persist it — it grants unauthenticated access to that order's
status.

## Provider webhooks

`/api/v1/payment/webhook/*` (`easypay`, `alipay`, `wxpay`, `stripe`,
`airwallex`) are for payment providers, authenticated by provider signature
rather than a Subtoken credential. Do not call them; a forged call can credit
balance without payment.

## Worked example: top up and confirm

```bash
# 1. Is this amount even allowed today?
curl -sS "${H[@]}" "$SUBTOKEN_API/payment/limits" |
  jq --argjson amt 50 '.data.methods | to_entries[] |
      select($amt >= .value.single_min and $amt <= .value.single_max) |
      {method: .key, fee: .value.fee_rate, remaining_today: .value.daily_remaining}'

# 2. Create the order
order=$(curl -sS "${H[@]}" -H 'Content-Type: application/json' \
  -X POST "$SUBTOKEN_API/payment/orders" \
  -d '{"amount":50,"payment_type":"alipay","order_type":"balance"}')
echo "$order" | jq '.data | {order_id, pay_amount, expires_at, result_type}'
echo "$order" | jq -r '.data.qr_code // .data.pay_url'   # present this to the payer

# 3. After they pay, reconcile
curl -sS "${H[@]}" -H 'Content-Type: application/json' \
  -X POST "$SUBTOKEN_API/payment/orders/verify" \
  -d "{\"out_trade_no\":$(echo "$order" | jq '.data.out_trade_no')}" | jq '.data.status'

# 4. Confirm the balance landed
curl -sS "${H[@]}" "$SUBTOKEN_API/auth/me" | jq '.data.balance'
```
