# Account security

Panel endpoints under `/api/v1/user`. **Session token required.** Every change
here is audited server-side and may need step-up verification.

A **panel key** (`skp-…`) covers the rest of the panel API but is refused on every
route in this file with `403 PANEL_KEY_CANNOT_MANAGE_IDENTITY`. That is
deliberate: these routes change how the account authenticates, and an API key has
no IP or User-Agent binding to protect that. Use a session token here even though
a panel key would be easier. See
[references/auth.md](auth.md#credential-3--the-panel-key-apiv1-everything-but-identity).

```bash
H=(-H "Authorization: Bearer $SUBTOKEN_JWT" -H 'Accept: application/json')
```

## Password

```bash
curl -sS "${H[@]}" -H 'Content-Type: application/json' \
  -X PUT "$SUBTOKEN_API/user/password" \
  -d '{"old_password":"…","new_password":"…"}' | jq '.data.message'
```

Minimum length is 6 characters. There is no password-reset-by-API here — the
flow is `POST /api/v1/auth/forgot-password` then
`POST /api/v1/auth/reset-password`, both email-gated and captcha-aware, so use
the browser.

## TOTP (authenticator 2FA)

```bash
curl -sS "${H[@]}" "$SUBTOKEN_API/user/totp/status" | jq '.data'
# { "enabled": true, "enabled_at": 1759000000, "feature_enabled": true }
```

`feature_enabled: false` means the operator switched TOTP off; `enabled` is
whether **you** have it on. `enabled_at` is a Unix timestamp in **seconds**.

Setup and teardown are confirmed by an **email code or your password** —
whichever the operator configured:

```bash
curl -sS "${H[@]}" "$SUBTOKEN_API/user/totp/verification-method" | jq '.data.method'
# "email" or "password"
```

### Enabling

```bash
# Step 1 — prove yourself, receive the secret and a setup token
setup=$(curl -sS "${H[@]}" -H 'Content-Type: application/json' \
  -X POST "$SUBTOKEN_API/user/totp/setup" -d '{"password":"…"}')
echo "$setup" | jq '.data | {secret, qr_code_url, countdown}'
```

For `method: "email"` the body is `{"email_code":"123456"}` instead, and you
need the code first:

```bash
curl -sS "${H[@]}" -X POST "$SUBTOKEN_API/user/totp/send-code" | jq '.data.success'
```

`secret` is the base32 TOTP secret and `qr_code_url` is an otpauth:// URI for
the authenticator app. `setup_token` authorises the enable step and expires —
note `countdown` (seconds). **Capture `secret` and `setup_token` now**; a new
setup call invalidates the old pair.

```bash
# Step 2 — confirm a live code from the authenticator
curl -sS "${H[@]}" -H 'Content-Type: application/json' \
  -X POST "$SUBTOKEN_API/user/totp/enable" -d "{
    \"totp_code\": \"$(oathtool --totp -b SECRET)\",
    \"setup_token\": $(echo "$setup" | jq '.data.setup_token')
  }" | jq '.data.success'
```

```bash
# Step 3 — verify login now demands 2FA
curl -sS -H 'Content-Type: application/json' -X POST "$SUBTOKEN_API/auth/login" \
  -d '{"email":"you@example.com","password":"…"}' | jq '.data'
# → { "requires_2fa": true, "temp_token": "…", "user_email_masked": "y***u@example.com" }

curl -sS -H 'Content-Type: application/json' -X POST "$SUBTOKEN_API/auth/login/2fa" \
  -d '{"temp_token":"…","totp_code":"123456"}' | jq '.data.access_token'
```

**Store the secret somewhere safe before leaving step 1.** Without it, or
without a backup, losing the authenticator means losing the account.

### Disabling

```bash
curl -sS "${H[@]}" -H 'Content-Type: application/json' \
  -X POST "$SUBTOKEN_API/user/totp/disable" -d '{"password":"…"}' | jq '.data.success'
```

Weakens the account. Confirm the user asked for it.

### Step-up verification

Grants the current session a short sudo window for sensitive operations. On the
user side it is what stands in front of high-risk actions:

```bash
curl -sS "${H[@]}" -H 'Content-Type: application/json' \
  -X POST "$SUBTOKEN_API/user/totp/step-up" -d '{"code":"123456"}' |
  jq '.data | {verified, expires_in}'
```

`expires_in` is seconds. The grant is **per session** — it dies with the session
and does not outlive the token, so re-run it per session rather than caching the
result. A `403` on a sensitive action usually means "run step-up first".

## Passkeys

WebAuthn ceremonies need a browser with `PublicKeyCredential`; there is no
pure-curl path. The two halves are visible over HTTP so a scripted client can
drive them.

```bash
curl -sS "${H[@]}" "$SUBTOKEN_API/user/passkeys" |
  jq '.data[] | {id, name, created_at, last_used_at, backup}'
```

### Registration

```bash
# begin → hand options.publicKey to the browser, get the attestation back
curl -sS "${H[@]}" -H 'Content-Type: application/json' \
  -X POST "$SUBTOKEN_API/user/passkeys/register/begin" -d '{"name":"laptop"}' |
  jq '.data | {session_token, options}'

# finish → session_token + the browser's credential
curl -sS "${H[@]}" -H 'Content-Type: application/json' \
  -X POST "$SUBTOKEN_API/user/passkeys/register/finish" -d '{
    "session_token": "…",
    "credential": { "id": "…", "rawId": "…", "type": "public-key",
                    "response": { "clientDataJSON": "…", "attestationObject": "…" } }
  }' | jq '.data'
```

Both the id and the raw bytes are **base64url** — the frontend converts with
`-`→`+` and `_`→`/` before decoding. Binary-in-JSON always needs that step.

```bash
# Rename / delete (delete re-confirms with the password)
curl -sS "${H[@]}" -H 'Content-Type: application/json' \
  -X PATCH "$SUBTOKEN_API/user/passkeys/4" -d '{"name":"work laptop"}'

curl -sS "${H[@]}" -H 'Content-Type: application/json' \
  -X DELETE "$SUBTOKEN_API/user/passkeys/4" -d '{"password":"…"}'
```

Passkey **login** is public (no session yet):
`POST /api/v1/auth/passkey/login/begin` then
`POST /api/v1/auth/passkey/login/finish`, both captcha-gated on this
deployment. See [auth.md](auth.md).

## Identity bindings

Bindable identities: `linuxdo`, `oidc`, `wechat`, `dingtalk`. `email` is the
primary identity, managed from the profile form rather than bound here.

`github` and `google` are **login** providers only — they work on the sign-in
path but are not bindable, so they never appear in the binding map and are
rejected by the bind endpoint.

```bash
curl -sS "${H[@]}" "$SUBTOKEN_API/auth/me" |
  jq '.data.identities | to_entries[] | {provider: .key, bound: .value.bound,
       can_bind: .value.can_bind, can_unbind: .value.can_unbind,
       start: .value.bind_start_path}'
```

`bound_count` above `1` means several accounts share that provider identity;
`can_unbind: false` in that case — unbinding would lock out the others.

### Binding an OAuth provider

OAuth binding is a **browser redirect**, not a POST. The API can return the
authorization URL for inspection, then the REPL/browser session opens it.

```bash
# Ask the server for the URL to open in the browser
curl -sS "${H[@]}" -H 'Content-Type: application/json' \
  -X POST "$SUBTOKEN_API/user/auth-identities/bind/start" \
  -d '{"provider":"oidc","redirect_to":"/settings/profile"}' |
  jq '.data | {provider, authorize_url, method, use_browser_redirect}'
```

`method` is `GET` and `use_browser_redirect` is `true` — so treat it as a URL to
open, not a request to make. `provider` must be one of `linuxdo`, `oidc`,
`wechat`, `dingtalk`; anything else (including `github` and `google`, which are
login-only) is `400`.

The dashboard instead reads `bind_start_path` off `/auth/me` and navigates
directly. That path already carries the correct WeChat `mode` (`open` vs `mp`,
chosen from the user agent), so **prefer `bind_start_path` when you have it** —
building the URL by hand gets WeChat wrong.

Either way, mint the bind cookie first:

```bash
curl -sS "${H[@]}" -X POST "$SUBTOKEN_API/auth/oauth/bind-token"
```

Skipping that makes the provider callback fail to associate the new identity
with the session.

```bash
# Unbind
curl -sS "${H[@]}" -X DELETE "$SUBTOKEN_API/user/account-bindings/oidc" | jq '.data'
```

Never leave yourself with **zero** usable identity. The email identity is
managed from the profile form and `can_unbind: false` — you cannot remove your
own recovery path here.

### Binding a different email

```bash
curl -sS "${H[@]}" -H 'Content-Type: application/json' \
  -X POST "$SUBTOKEN_API/user/account-bindings/email/send-code" -d '{"email":"new@example.com"}'

curl -sS "${H[@]}" -H 'Content-Type: application/json' \
  -X POST "$SUBTOKEN_API/user/account-bindings/email" -d '{
    "email": "new@example.com",
    "verify_code": "123456",
    "password": "…"
  }' | jq '.data | {email, email_bound}'
```

All three fields required, including your password. This changes the account's
primary address — it is where password resets and balance alerts go.

## Notification email

Low-balance alerts, separate from the login address.

```bash
curl -sS "${H[@]}" -H 'Content-Type: application/json' \
  -X POST "$SUBTOKEN_API/user/notify-email/send-code" -d '{"email":"ops@example.com"}'

curl -sS "${H[@]}" -H 'Content-Type: application/json' \
  -X POST "$SUBTOKEN_API/user/notify-email/verify" -d '{"email":"ops@example.com","code":"123456"}'

# Mute one address without removing it ("" targets the primary address)
curl -sS "${H[@]}" -H 'Content-Type: application/json' \
  -X PUT "$SUBTOKEN_API/user/notify-email/toggle" -d '{"email":"ops@example.com","disabled":true}'

# Remove entirely
curl -sS "${H[@]}" -X DELETE "$SUBTOKEN_API/user/notify-email" -d '{"email":"ops@example.com"}'
```

`DELETE` takes its target in the **body**, not the path. Each entry is
`{email, disabled, verified}` — `disabled` mutes, `verified` says whether the
address was confirmed. Toggle before deleting when you only want to silence it.

## Sessions

```bash
# Log out — revokes the refresh token server-side
curl -sS -H 'Content-Type: application/json' -X POST "$SUBTOKEN_API/auth/logout" \
  -d "{\"refresh_token\":\"$SUBTOKEN_REFRESH\"}"

# Revoke every session everywhere
curl -sS "${H[@]}" -X POST "$SUBTOKEN_API/auth/revoke-all-sessions" | jq '.data.message'
```

`revoke-all-sessions` is the panic button: it kills every refresh token for the
account, including the one you are using. Your current access token keeps
working until it expires, and you will need to sign in again from scratch. Use
it after a suspected compromise, and expect to re-authenticate afterwards.

`POST /api/v1/auth/logout` is **public** — it accepts an unauthenticated request
precisely so a stale client can still revoke its refresh token.

## Checklist after a suspected compromise

Run these steps interactively. The bulk-disable loop below is an optional
emergency helper only when the user has explicitly asked to disable every key;
otherwise inspect the key list and disable selected keys one at a time, checking
each response.

```bash
# 1. Cut every key off (reversible — prefer this to deletion)
for id in $(curl -sS "${H[@]}" "$SUBTOKEN_API/keys?page_size=100" | jq -r '.data.items[].id'); do
  curl -sS "${H[@]}" -H 'Content-Type: application/json' \
    -X PUT "$SUBTOKEN_API/keys/$id" -d '{"status":"inactive"}' > /dev/null
done

# 2. Kill all sessions (logs you out too)
curl -sS "${H[@]}" -X POST "$SUBTOKEN_API/auth/revoke-all-sessions"

# 3. Audit recent use
curl -sS "${H[@]}" "$SUBTOKEN_API/keys?page_size=100" |
  jq '.data.items[] | {id, name, last_used_at, last_used_ip, status}'

curl -sS "${H[@]}" "$SUBTOKEN_API/usage?page=1&page_size=50&timezone=UTC" |
  jq '.data.items[] | {created_at, model, ip_address, user_agent, actual_cost}'

# 4. Confirm 2FA is on
curl -sS "${H[@]}" "$SUBTOKEN_API/user/totp/status" | jq '.data'

# 5. Rotate: new key, move tools, verify, then delete the old one
```

Rotate rather than reuse: create the replacement key, update every tool that
held the old value, confirm traffic on the new key, and only then delete the
old one.
