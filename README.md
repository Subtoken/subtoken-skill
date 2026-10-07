<div align="center">

# Subtoken

### Your account. Your keys. Your agent.

An agent skill for the **Subtoken user API**.<br>
Manage keys, inspect spend, track payments, and work with the marketplace.

[![Subtoken](https://img.shields.io/badge/subtoken.net-Visit-166534?style=flat-square)](https://subtoken.net)
[![Agent Skill](https://img.shields.io/badge/Agent_Skill-subtoken-18181b?style=flat-square)](SKILL.md)
[![Panel API](https://img.shields.io/badge/Panel_API-%2Fapi%2Fv1-0369a1?style=flat-square)](references/auth.md)
[![Gateway API](https://img.shields.io/badge/Gateway_API-%2Fv1-7c2d12?style=flat-square)](references/key-introspection.md)

[Install](#install) &nbsp; / &nbsp; [Quick start](#quick-start) &nbsp; / &nbsp; [API references](#api-references)

</div>

---

## What you can do

| Account and keys | Usage and billing | Marketplace and supply |
| --- | --- | --- |
| Create and rotate API keys | Inspect spend and usage logs | Browse listings and purchase accounts |
| Manage groups and entitlements | Check quotas and effective rates | Register seller accounts and proxies |
| Review account security | Track recharge orders and refunds | Manage listings, earnings, and payouts |

The skill provides endpoint references, request examples, response shapes, and
credential rules for the APIs behind [subtoken.net](https://subtoken.net).
It contains documentation and agent instructions; it does not run a service.

## Install

Install with the [Skills CLI](https://github.com/vercel-labs/skills):

```bash
npx skills add Subtoken/subtoken-skill
```

Select your agent and installation scope when prompted. You can also clone the
repository and use [SKILL.md](SKILL.md) directly:

```bash
git clone https://github.com/Subtoken/subtoken-skill.git
```

### Ask your agent

> Use the Subtoken skill to check my balance and summarize this week's usage.

> List my inference keys and their daily spend, without printing any secrets.

> Check the status of my latest recharge order.

For agents that support explicit skill invocation, use `$subtoken`.
The skill favors an interactive session: inspect each response before making
the next request, and confirm the target before a consequential write.

## Quick start

Read [SKILL.md](SKILL.md) for the operating guide. It explains credential
selection, safe request practices, and how to navigate the API references.

For scripted panel work, configure the base URLs and the credentials you need:

```bash
export SUBTOKEN_BASE_URL='https://subtoken.net'
export SUBTOKEN_API="${SUBTOKEN_BASE_URL}/api/v1"
export SUBTOKEN_GW="${SUBTOKEN_BASE_URL}/v1"

# Gateway calls: inference key, sk-...
export SUBTOKEN_API_KEY='sk-REPLACE_WITH_YOUR_INFERENCE_KEY'

# Panel calls: panel key, skp-...; preferred for routine automation
export SUBTOKEN_PANEL_KEY='skp-REPLACE_WITH_YOUR_PANEL_KEY'
export SUBTOKEN_PANEL_AUTH="$SUBTOKEN_PANEL_KEY"
```

Create keys in the dashboard under **API Keys**, choosing **Inference** or
**Panel** as the key type. The plaintext is shown once at creation; subsequent
reads return a masked value. Keep the original in your credential store.

Use a session token only when needed for identity and authentication changes.
Panel keys cannot change password, email, TOTP, passkeys, or identity bindings.
See [Authentication](references/auth.md) and [Security](references/security.md)
for credential setup and route boundaries.

Start with read-only checks:

```bash
# Confirm the panel identity and current balance
curl -sS -H "Authorization: Bearer $SUBTOKEN_PANEL_AUTH" \
  "$SUBTOKEN_API/auth/me" | jq '.data | {email, balance, seller_status}'

# Inspect models available to an inference key
curl -sS -H "Authorization: Bearer $SUBTOKEN_API_KEY" \
  "$SUBTOKEN_GW/models" | jq '.data[].id'
```

## Credential map

| Credential | Use it for | Boundary |
| --- | --- | --- |
| Inference key (`sk-...`) | `/v1/*`: model requests and key introspection | Cannot authenticate to panel routes |
| Panel key (`skp-...`) | `/api/v1/*`: routine account and dashboard automation | Cannot infer or change identity/authentication settings |
| Session token | Panel routes, including identity and security changes | Expires; refresh tokens rotate |
| No credential | Public read-only routes | Private account data or mutations |

The database key type determines authorization; a prefix alone does not. Sending
an inference key to a panel route returns `INFERENCE_KEY_ON_PANEL`, while a
panel key sent to the gateway returns `PANEL_KEY_CANNOT_INFER`.

## Safe operating pattern

1. Identify the credential and API surface before making a request.
2. Read the target resource and verify its current state.
3. For writes, confirm the target and consequences; perform one mutation at a time.
4. Inspect the response, then read the resource back to verify the result.
5. Keep credentials and returned secrets out of logs, transcripts, and output.

Money writes need special care. Payment-order creation, redeem-code redemption,
and payout requests have no documented `Idempotency-Key` contract. If a request
times out, inspect the resulting order, redemption history, or payout state
before retrying. Marketplace purchases use a required `nonce`; a retry must use
the same nonce to retrieve the original order. See the operation-specific notes
in [Payments](references/payments.md), [Account](references/account.md),
[Seller](references/seller.md), and [Marketplace](references/marketplace.md).

## API references

| Guide | What it covers |
| --- | --- |
| [Authentication](references/auth.md) | Response envelope, credentials, errors, pagination, rate limits |
| [Account and keys](references/account.md) | Profile, API keys, groups, redeem codes, subscriptions, affiliate |
| [Usage](references/usage.md) | Logs, summaries, dashboard rollups, per-key usage |
| [Payments](references/payments.md) | Checkout, plans, order creation, verification, refunds |
| [Marketplace](references/marketplace.md) | Listings, purchases, rentals, order status |
| [Seller](references/seller.md) | Proxies, accounts, OAuth onboarding, listings, payouts |
| [Security](references/security.md) | Password, TOTP, passkeys, step-up, identity bindings, sessions |
| [Key introspection](references/key-introspection.md) | Key validation, models, effective billing |
| [Public endpoints](references/public.md) | Anonymous settings, stats, model plaza, health |

## Scope

This skill documents Subtoken's **user-facing** API at `subtoken.net`. It does
not cover `/api/v1/admin/*` or other Sub2API administration APIs. For endpoint
availability, feature flags, and current limits, prefer live API responses over
examples in the references.

---

<div align="center">

**[Open Subtoken](https://subtoken.net) &nbsp; / &nbsp; [Read the skill](SKILL.md) &nbsp; / &nbsp; [Browse authentication](references/auth.md)**

</div>
