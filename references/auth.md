# Authentication, base URLs, and response shape

Everything in this skill hangs off these four facts. Read once, then go to the
surface reference.

## Base URLs

```bash
export SUBTOKEN_BASE_URL='https://subtoken.net'
export SUBTOKEN_API="${SUBTOKEN_BASE_URL}/api/v1"   # panel  (panel key or JWT)
export SUBTOKEN_GW="${SUBTOKEN_BASE_URL}/v1"         # gateway (inference key)

# One header value for the panel; a panel key is preferred over a session token.
export SUBTOKEN_PANEL_AUTH="${SUBTOKEN_PANEL_KEY:-$SUBTOKEN_JWT}"
```

For agent-driven work, use an interactive REPL-like session and send requests
one at a time. Keep response data and returned IDs in session state, inspect
each result, and base the next request on it. Batch helpers in this reference
are optional conveniences; do not turn a task into an unattended script or
batch of writes unless the user explicitly requests that.

Self-hosted deployments put the panel under the same origin. There is no
separate panel hostname.

## Credential 1 — the inference key (gateway only)

Generated in the dashboard under **API Keys → Create API Key** with
**Key Type: Inference**. Its value is shown once, in the create response; every
later read is masked.

```bash
export SUBTOKEN_API_KEY='sk-…'
```

Accepted headers, in the order the gateway tries them:

| Header | Notes |
| --- | --- |
| `Authorization: Bearer sk-…` | Preferred everywhere. |
| `x-api-key: sk-…` | Anthropic-style clients. |
| `x-goog-api-key: sk-…` | Gemini CLI compatibility. |

Passing the key as `?key=` or `?api_key=` is **rejected** with
`400 api_key_in_query_deprecated`. Keys in query strings leak into logs.

A key with **no group assigned** is rejected on every `/v1/*` route. Assign the
key to a group in the dashboard first.

A **panel key** sent to `/v1` is rejected earlier and more clearly, with
`403 PANEL_KEY_CANNOT_INFER`.

## Credential 3 — the panel key (`/api/v1`, everything but identity)

A panel key is the credential for scripting your own account over HTTP. Create
one under **API Keys → Create API Key** with **Key Type: Panel**; it starts with
`skp-` and its value is shown once, at creation.

```bash
export SUBTOKEN_PANEL_KEY='skp-…'
```

```bash
curl -sS -H "Authorization: Bearer $SUBTOKEN_PANEL_KEY" "$SUBTOKEN_API/auth/me" | jq
curl -sS -H "x-api-key: $SUBTOKEN_PANEL_KEY"           "$SUBTOKEN_API/keys"  | jq
```

Accepted headers are the same three as the gateway (`Authorization: Bearer`,
`x-api-key`, `x-goog-api-key`), and `?key=` / `?api_key=` are rejected the same
way — a panel key is a full-account credential, so it must never reach a URL.

**What it buys you over a session token:** no browser scraping, no expiry to
manage, no IP/User-Agent binding to trip over when you move a script between
machines.

**What it costs you:** one carve-out. A panel key cannot change how the account
authenticates. These answer `403 PANEL_KEY_CANNOT_MANAGE_IDENTITY`:

| Refused route | What it would change |
| --- | --- |
| `PUT /user/password` | The login credential. |
| `POST /user/account-bindings/email` and `/send-code` | An identity that can receive a password reset. |
| `DELETE /user/account-bindings/:provider` | An existing identity. |
| `POST /user/auth-identities/bind/start` | A new OAuth identity. |
| `POST /user/totp/{setup,enable,disable,send-code}` | The second factor. |
| `POST /user/totp/step-up` | Authorises sensitive session actions; meaningless without a session. |
| `POST /user/passkeys/register/{begin,finish}`, `DELETE /user/passkeys/:id`, `PATCH /user/passkeys/:id` | WebAuthn authenticators. |

Everything else works: usage, keys, groups, balance, payments, redeem,
subscriptions, seller console, marketplace orders, and every `GET`.

The reason is short and worth remembering: a session token is bound to your IP
and browser and is revoked when those change; an API key is not. A credential
with no binding should not be able to repoint the account's own authentication.
So a leaked panel key can **spend, read, and manage inference keys — it cannot
take over the account.** For identity changes, use a session token.

Two more things that differ from an inference key:

- **No balance or quota check.** A panel key works at a zero balance, and
  `status: quota_exhausted` does not block it. Otherwise you could not rotate a
  key when broke.
- **Expiry, enable/disable and IP allow/deny lists still apply.** Set an
  `expires_in_days` and an `ip_whitelist` when you create it; those are the
  controls that bound a panel key.

Audit rows for panel-key writes carry `auth_method = "panel_key"`, so they are
filterable alongside `jwt` in the admin audit log.

## Credential 2 — the session token (the only way to change identity)

The panel accepts a signed session token, a panel key, or an inference key that
it then rejects by name. An inference key sent to `/api/v1/*` returns
`401 INFERENCE_KEY_ON_PANEL`.

A session token is still required for the identity routes listed under
[Credential 3](#credential-3--the-panel-key-skip-api-v1-everything-but-identity).
Everything below about obtaining one applies; it is just no longer needed for
ordinary panel work.

Obtain it one of three ways:

1. **From a signed-in browser** (the reliable path):

   ```
   devtools → Application → Local Storage → https://subtoken.net → auth_token
   ```

   The neighbouring `refresh_token` is what keeps it alive.

2. **Headless login**, when the operator has Turnstile disabled:

   ```bash
   curl -sS -X POST "$SUBTOKEN_API/auth/login" \
     -H 'Content-Type: application/json' \
     -d '{"email":"you@example.com","password":"…"}' | jq '.data.access_token'
   ```

   `subtoken.net` runs Turnstile, so this returns
   `400 TURNSTILE_VERIFICATION_FAILED` unless you supply a `turnstile_token`
   (or the Tencent/Aliyun proof fields `tencent_captcha_ticket` /
   `tencent_captcha_randstr`). Treat headless login as unavailable by default.

3. **Two-factor completion**, when the account has TOTP enabled. The first call
   answers `{"requires_2fa": true, "temp_token": "…"}`; finish with:

   ```bash
   curl -sS -X POST "$SUBTOKEN_API/auth/login/2fa" \
     -H 'Content-Type: application/json' \
     -d '{"temp_token":"…","totp_code":"123456"}' | jq '.data.access_token'
   ```

```bash
export SUBTOKEN_JWT='eyJhbGciOi…'
```

### Refreshing an expired session

```bash
curl -sS -X POST "$SUBTOKEN_API/auth/refresh" \
  -H 'Content-Type: application/json' \
  -d "{\"refresh_token\":\"$SUBTOKEN_REFRESH\"}" | jq '.data'
```

Returns a new `access_token` **and a new `refresh_token`**. Refresh tokens
rotate and are single-use: a replayed refresh token revokes the whole session.
Always overwrite both values together, and never run two refreshes concurrently
for the same session.

## Response envelope

Panel endpoints (`/api/v1/*`) always wrap the payload:

```json
{ "code": 0, "message": "success", "data": { "…": "…" } }
```

`code == 0` is success. Anything else is an error, and `data` may be absent.
Unwrap with `jq '.data'`.

Errors add two optional fields:

```json
{
  "code": 401,
  "message": "Token has expired",
  "reason": "TOKEN_EXPIRED",
  "metadata": { }
}
```

`reason` is the stable, machine-readable string; `message` is prose that may be
reworded or translated (`Accept-Language` changes it).

Gateway endpoints (`/v1/*`) are **not** wrapped and use OpenAI-style errors:

```json
{ "error": { "message": "Invalid API key", "type": "authentication_error" } }
```

Do not pipe a `/v1/*` response through `jq '.data'` expecting the envelope —
on the gateway `.data` is the model list or the usage payload itself.

## Status codes worth special-casing

| Status | Meaning | What to do |
| --- | --- | --- |
| `400 TURNSTILE_VERIFICATION_FAILED` | Captcha gate on login/register | Use a browser session token instead. |
| `400 api_key_in_query_deprecated` | Key passed in the query string | Move it to a header. |
| `401 UNAUTHORIZED` / `INVALID_AUTH_HEADER` | Missing or malformed `Authorization` | Check the header, not the key. |
| `401 INVALID_TOKEN` | Token is not a valid panel JWT | An API key was sent to the panel. |
| `401 TOKEN_EXPIRED` | Session token past its lifetime | Refresh, or re-read from the browser. |
| `401 INVALID_API_KEY` | Gateway rejected the key | Verify the key and that it is enabled. |
| `401 API_KEY_REQUIRED` | No credential on a `/v1/*` route | Add a header. |
| `403` | Authenticated but not permitted (e.g. seller endpoints without consent) | Check `seller_status`. |
| `404` | **Often a disabled feature, not a missing route** | Check `/api/v1/settings/public` flags. |
| `423 ADMIN_COMPLIANCE_ACK_REQUIRED` | Operator compliance acknowledgement pending | Admin-only; out of scope here. |
| `429` | Rate limited | Back off; see below. |

## Pagination

List endpoints return the same envelope:

```json
{
  "items": [],
  "total": 137,
  "page": 1,
  "page_size": 20,
  "pages": 7
}
```

Iterate with `page` and cap `page_size` (100 is the ceiling on the paginated
list routes). Do not ask for one huge page.

In an interactive session, request a page, inspect `page` and `pages`, then
request the next page as needed. The loop above is an optional read-only helper;
prefer page-by-page inspection during an agent task, especially before acting
on the returned records.

Two shapes break the pattern: `/redeem/history` and `/announcements` return
bare arrays, and seller collections wrap in `{ "items": [...] }` without totals.

## Rate limits

Panel routes are limited per user, with a stricter tier on heavy reads: the
whole `/usage` group, `/channel-monitor-v2`, `/user/api-keys/:id/usage/daily`,
`/user/accounts/:id/claude/reset-credits`, and `/seller/earnings`.

Public endpoints are limited per client IP.

Auth endpoints carry their own server-side budget and **fail closed** when the
store is unreachable, so an infrastructure blip looks like a rate limit:

| Endpoint | Limit |
| --- | --- |
| `POST /auth/register` | 5 / min |
| `POST /auth/login`, `/auth/login/2fa` | 20 / min |
| `POST /auth/refresh` | 30 / min |
| `POST /auth/send-verify-code` | 5 / min |
| `POST /auth/validate-promo-code`, `/auth/validate-invitation-code` | 10 / min |
| `POST /auth/forgot-password` | 5 / min |
| `POST /auth/reset-password` | 10 / min |

On `429`, wait and retry with exponential backoff. Do not retry a `4xx` that is
not `429` — a rejected login or code will not succeed on a second attempt and
repeated attempts deepen the limit.

## Conventions worth carrying into every request

- `Accept-Language: en` pins backend messages to English, which keeps parsing
  stable across locales.
- `timezone: <IANA name>` on GETs sets the day boundary for all usage and
  dashboard figures. Without it the server picks a default, so "today" can
  disagree with the dashboard. Send your own zone explicitly.
- `Accept: application/json` on every call.
