# Getting started

Everything a Notabene Transact integration needs before its first transfer: an environment, an API credential, a
token, a webhook endpoint, and the handful of conventions the whole API shares. Read this once, then pick your path in
[Choose your integration](choose-your-integration.md).

## Quick start

Notabene sits between you and the institution you are sending to, and tells you when a transfer is safe to send. To
see that work in sandbox:

1. Create an API credential under **Integration → API Credentials**, and swap it for a token (step 3 below).
2. Add a webhook endpoint under **Integration → Webhooks**.
3. Create a transfer, wait for the `ReadyToSettle` webhook, then report the transaction hash with settle. [Send a
   transfer](transact-send.md) has the calls.

**Integration → Go Live** tracks each step as you complete it. The rest of this page explains them, and the
conventions every call shares.

## How it works

When your customer withdraws crypto, the Travel Rule requires you to send who they are, and who they are paying, to
the institution receiving the funds: the same originator and beneficiary information a wire transfer carries.
Notabene Transact finds that institution, exchanges the data with it, and tells you when it is safe to send.

```text
Your customer (originator) → your institution → counterparty institution → their customer (beneficiary)
```

1. You create a transfer: the amount, the destination address, and both customers.
2. Notabene identifies the institution behind the address and exchanges the Travel Rule data with it.
3. Each institution authorizes. Webhooks tell you what is still needed, then that the transfer is `ReadyToSettle`.
4. You broadcast on-chain, as you do today, and tell Notabene the transaction hash.

Deposits run the same chain in the other direction. [How Notabene Transact works](overview.md) explains the model in
more depth.

## 1. Your base URL

| Environment | Base URL | Token issuer |
|---|---|---|
| Sandbox | `https://sandbox.notabene.id` | `https://auth.notabene.id/` |
| Production (EU) | `https://eu-a.notabene.id` | `https://auth.notabene.id/` |
| Production (US) | `https://us-a.notabene.id` | `https://auth.notabene.id/` |

Build against sandbox first. Your institution lives in one region; use the base URL your onboarding names. The
examples in these guides use sandbox.

## 2. Create an API credential

In the dashboard, open **Integration → API Credentials** and create one. You get a `client_id` and a `client_secret`; the
secret is shown once, so store it in your secrets manager straight away. You can hold up to three credentials, which
is enough to rotate without downtime.

## 3. Get an access token

Exchange the credential for a token with the OAuth2 client-credentials grant, with the `audience` shown below.

```bash
curl -s https://auth.notabene.id/oauth/token \
  -H 'content-type: application/json' \
  -d '{"grant_type":"client_credentials","client_id":"YOUR_CLIENT_ID",
       "client_secret":"YOUR_CLIENT_SECRET","audience":"https://sandbox.notabene.id"}'
```

Send it on every request as `Authorization: Bearer <access_token>`, and reuse it until shortly before `expires_in`
runs out. The token is bound to your institution: no header or parameter selects which institution you act as.

Check it works:

```bash
curl -s https://sandbox.notabene.id/api/v1/identity -H "Authorization: Bearer $TOKEN"
```

The response names your institution's DID. You are ready to call the API.

## 4. Receive webhooks

Notabene tells your systems when a transfer needs them by webhook. In the dashboard, open **Integration → Webhooks**,
add the HTTPS URL that receives events, and store its signing secret (`whsec_…`) beside your API credential. Send a
test event from the same page. [Webhooks](webhooks.md) covers verifying and handling deliveries.

## Conventions

- **Reference.** The API reference (dashboard: **Integration → Getting Started → API Reference**) is authoritative
  for every path, field and enum, and names the permission each operation needs. These guides cover order and
  judgement; take field names from the reference.
- **Identifiers.** Institutions and customers are DIDs or IRIs (`did:web:…`, `urn:uuid:…`). Assets are CAIP-19
  (`eip155:1/erc20:0xa0b8…eb48`) or a plain symbol (`USDC`). Addresses accept a bare address, CAIP-10
  (`eip155:1:0xab…`) or a `did:pkh`. Settlements are a transaction hash. Personal data is IVMS-101.
- **Amounts** are decimal strings in the asset's whole units: `"1.5"` ETH, never wei.
- **Idempotency.** Creating a transfer takes a `ref`, a UUID you generate and store first. Replaying the same body
  with the same `ref` returns the original transfer; a different body under the same `ref` is a `409`. Customers are
  keyed by their `id` instead: a retry answers `409` when the customer already exists, so treat that as "already
  done". Re-registering a wallet with the same content succeeds; different content for the same `address` is a `409`.
- **Casing.** Bodies are snake_case, with a few exceptions the reference shows (the assess body and most webhook
  payloads are camelCase). Copy names from the reference rather than guessing.
- **Errors** are RFC 9457 problem documents with `status`, `title`, `detail` (what to fix) and per-field `errors[]`.
- **Pagination.** Lists return `items` and a `next_cursor`; pass the cursor back until it is absent.

| Status | Meaning | What to do |
|---|---|---|
| `401` | No valid token, or no institution matches it | Re-mint the token; check its `audience` against step 3 |
| `403` | The token lacks the operation's permission | Check the permission named in the reference |
| `409` | Conflicting `ref`, or a transfer's inputs changed while you acted | Re-read the record, then retry once |
| `422` | Validation failed | Read `errors[]` and fix the named fields |
| `5xx` | Our fault | Retry with backoff, replaying the same body (and `ref`) |

## Next

[Choose your integration](choose-your-integration.md) — the shortest path depends on what you are building.
