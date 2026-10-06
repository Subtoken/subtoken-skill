---
name: subtoken
description: Work with the Subtoken user APIs at subtoken.net - inspect and manage your own API keys, usage, balance, groups, subscriptions, recharge orders, redeem codes, marketplace purchases, seller listings and payouts, and account security (TOTP, passkeys, email bindings). Also covers the API-key-authenticated gateway introspection endpoints (/v1/usage, /v1/models, /v1/sub2api/billing). Use when the user mentions Subtoken, subtoken.net, their Subtoken API key, Subtoken usage or spend, Subtoken balance or recharge, buying or selling accounts on the Subtoken marketplace, or wants to script anything on the Subtoken dashboard. Not for Sub2API admin APIs.
---

# Subtoken user APIs

Subtoken is a hosted AI gateway and account marketplace. This skill covers the
**user-facing** surface only — the APIs behind the dashboard at `subtoken.net`.
Admin APIs are a different surface and out of scope.

## Two hosts, two credentials

Subtoken splits its API in two, and there are two kinds of key. Getting this
wrong is the single most common failure, so check it first.

| Surface | Base URL | Credential | Header |
| --- | --- | --- | --- |
| Gateway (inference + key introspection) | `https://subtoken.net/v1` | **Inference key** (`sk-…`) from **API Keys** in the dashboard | `Authorization: Bearer sk-…` |
| Panel (account, usage, payments, marketplace) | `https://subtoken.net/api/v1` | **Panel key** (`skp-…`) or **session token** | `Authorization: Bearer skp-…` or `<jwt>` |

**Each key type works on exactly one surface, and says so by name on the
other.** This is the fastest way to tell which credential you are holding:

| Sent to | Result |
| --- | --- |
| `sk-…` on `/api/v1` | `401 INFERENCE_KEY_ON_PANEL` |
| `skp-…` on `/v1` | `403 PANEL_KEY_CANNOT_INFER` |

The prefix (`sk-` vs `skp-`) is cosmetic — a custom key can wear either — so
treat those two error codes, not the prefix, as authoritative.

**Prefer a panel key over a session token for ordinary panel automation.** A
session token must be scraped out of browser localStorage, expires, and is bound
to your IP and browser. A panel key is created in the dashboard and does not
expire by default. The trade is that a panel key cannot change your identity —
see [Credential 3](#credential-3--the-panel-key-apiv1-everything-but-identity)
in [references/auth.md](references/auth.md).

## Operating mode: interactive API REPL

For agent-driven API work, use a persistent, REPL-like session by default. Keep
the base URLs, chosen credential, response objects, and returned IDs in session
state. Make one request, inspect its status/reason/data, then decide what to do
next from the actual response. A persistent Python `requests.Session()`, shell
session, or equivalent HTTP REPL is suitable.

Do not generate standalone deterministic scripts, batch runners, or chains of
uninspected mutations unless the user asks for them. This matters most for
money, credentials, account security, and seller/marketplace operations. Read
the target first, show the relevant IDs and state, perform one write, and
inspect its response before continuing. The curl examples below and in the
references are request shapes to run interactively; safe read-only batch
helpers may still be useful where they are explicitly identified.

Set up once per shell:

```bash
export SUBTOKEN_BASE_URL='https://subtoken.net'
export SUBTOKEN_API="${SUBTOKEN_BASE_URL}/api/v1"
export SUBTOKEN_GW="${SUBTOKEN_BASE_URL}/v1"

# Inference key — dashboard → API Keys → Key Type: Inference → copy at creation.
export SUBTOKEN_API_KEY='sk-…'

# Panel key — dashboard → API Keys → Key Type: Panel → copy at creation.
# Preferred for ordinary panel API work; falls back to the session token below.
export SUBTOKEN_PANEL_KEY='skp-…'

# Session token — only needed for identity changes (password, TOTP, passkeys,
# identity bindings). Read it out of the browser while signed in:
#   devtools → Application → Local Storage → https://subtoken.net → auth_token
export SUBTOKEN_JWT='eyJhbGciOi…'

# One credential for the panel: a panel key if you have one, else the JWT.
export SUBTOKEN_PANEL_AUTH="${SUBTOKEN_PANEL_KEY:-$SUBTOKEN_JWT}"
```

Read [references/auth.md](references/auth.md) before anything else: envelope
shape, how to obtain each credential, error codes, pagination, rate limits.

## Start here

```bash
# Which key is this, and what may it call? (API key only)
curl -sS -H "Authorization: Bearer $SUBTOKEN_API_KEY" "$SUBTOKEN_GW/models" | jq '.data[].id'

# Key quota, spend, and rate-limit windows (API key only)
curl -sS -H "Authorization: Bearer $SUBTOKEN_API_KEY" "$SUBTOKEN_GW/usage" | jq

# Effective billing multiplier for this key (API key only)
curl -sS -H "Authorization: Bearer $SUBTOKEN_API_KEY" "$SUBTOKEN_GW/sub2api/billing" | jq

# Who am I, and what is my balance? (panel key or session token)
curl -sS -H "Authorization: Bearer $SUBTOKEN_PANEL_AUTH" "$SUBTOKEN_API/auth/me" | jq '.data | {email, balance, concurrency, seller_status}'

# Example request shape: create a second inference key, inspect the response,
# retain the plaintext, then rotate the consumer and delete the old key.
curl -sS -X POST "$SUBTOKEN_API/keys" \
  -H "Authorization: Bearer $SUBTOKEN_PANEL_KEY" -H 'Content-Type: application/json' \
  -d '{"name":"ci-2026-07","group_id":1,"expires_in_days":90}' | jq '.data | {id, key, key_type}'
```

## Surface map

Read the reference file for the surface you need; each is a curl cookbook with
individual request shapes, response examples, and the gotchas that bite.

| Surface | Reference | Credential |
| --- | --- | --- |
| Key introspection, model discovery, effective rate | [references/key-introspection.md](references/key-introspection.md) | inference key |
| Profile, API keys, groups, channels, redeem, subscriptions, announcements, affiliate | [references/account.md](references/account.md) | panel key or session token |
| Usage logs, stats, dashboard rollups, per-key daily | [references/usage.md](references/usage.md) | panel key or session token |
| Recharge orders, plans, refunds, public order resolve | [references/payments.md](references/payments.md) | panel key or session token |
| Marketplace directory and buying | [references/marketplace.md](references/marketplace.md) | anonymous / panel key / session token |
| Seller console: supply, listings, earnings, payouts | [references/seller.md](references/seller.md) | panel key or session token |
| Password, TOTP, step-up, passkeys, identity bindings, sessions | [references/security.md](references/security.md) | **session token only** |
| Anonymous read-only: settings, public stats, model plaza, health | [references/public.md](references/public.md) | none |

"Panel key" means a panel key works; a session token also works everywhere a
panel key does. Only the security surface is session-token-only.

## Workflow

1. **Identify the credential you have.** Inference key → the introspection
   surface only. Panel key → the whole panel API except identity changes.
   Session token → everything. No credential → the public surface.
2. **Read before writing.** In the same REPL, `GET` the resource, confirm the
   id and current state, then issue one mutation and inspect its response.
3. **Prefer reversible changes.** Disable a key (`status: inactive`) over
   deleting it. Pause a listing over delisting it.
4. **Re-read after writing** to confirm the server's state, not your assumption.
5. **Never print secrets.** Keys, session tokens, passwords, `custom_key`, seller
   `api_key` credentials, OAuth tokens. Filter before echoing: pipe through
   `jq 'del(.. | .api_key?)'` or print only ids and names.
6. **A key's plaintext is returned exactly once**, in the `POST /keys` response.
   Every later read is masked (`skp-012345…cdef`). Capture it at creation and
   store it yourself — you cannot retrieve it again.
7. **`key_type` is immutable.** To change a key's class, create a new key of the
   wanted type, verify it, then delete the old one.

## Safety notes

- **Money moves on several of these endpoints.** Creating a payment order,
  buying a marketplace listing, requesting a payout, and redeeming a code all
  spend balance irreversibly. Print the amount, currency, and target before
  calling, and get explicit confirmation.
- **Marketplace purchases are idempotent only by nonce.** `POST
  /marketplace/listings/:id/orders` takes a `nonce`; reusing it returns the
  original order instead of charging twice. Always generate a fresh nonce per
  attempt and never reuse one across listings.
- **Registration and login are captcha-gated.** `subtoken.net` runs Turnstile,
  so `POST /api/v1/auth/login` returns `400 TURNSTILE_VERIFICATION_FAILED`
  without a `turnstile_token`. Headless login is not the normal path — reuse a
  browser session token. Registration additionally requires an invitation code
  when the operator enables it.
- **Feature flags surface as 404, not 403.** A disabled feature (model plaza,
  marketplace, a payment provider) answers `404`. Check
  `GET /api/v1/settings/public` before concluding an endpoint does not exist.
- **Rate limits are real.** Usage logs, dashboard rollups, seller earnings and
  several per-account reads use the strict limiter. Page through list
  endpoints instead of asking for `page_size=1000`, and back off on `429`.
- **Session tokens expire and refresh tokens rotate.** On `401 TOKEN_EXPIRED`,
  either refresh with the refresh token or re-read a token from the browser. A
  refresh token is single-use — a replay revokes the session.
- **Admin endpoints are not in this skill.** `/api/v1/admin/*` and
  `/api/v1/admin/payment/*` need an admin credential and are covered by a
  separate admin skill.
