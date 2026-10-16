# Embedded checks with customer tokens

Use this when customers enter withdrawal details in your own web or mobile app and you want Travel Rule checks to run
as they type: whether the rule applies, which institution owns the destination, which fields to ask for. Your
credential never leaves your backend.

## Quick start

Your front end gets a short-lived, read-only key for one customer, so it can check a withdrawal as they type.

1. Backend: `POST /api/v1/entities/{your DID}/delegate-token` with the customer's id.
2. Front end: call `POST /api/v1/customers/{id}/transfers/assess` with that token as the customer types.
3. Backend: create the transfer as usual.

## 1. Mint a customer token (backend)

The customer must already exist: register them once ([Customers](customers.md)), using as `id` the identifier you
will pass as `customer_ref`. Minting for an unknown customer returns `400`.

```bash
curl -s https://sandbox.notabene.id/api/v1/entities/$YOUR_DID/delegate-token \
  -H "Authorization: Bearer $TOKEN" -H 'content-type: application/json' \
  -d '{"customer_ref":"urn:uuid:0c8f6a52-6e83-4f2b-9d0a-7b1f0e7c2a41"}'
```

`$YOUR_DID` is your institution's DID (from `GET /api/v1/identity`). `customer_ref` is the customer's `id`. The response
carries `access_token` and `expires_in` (about an hour). Mint one per session and hand it to your front end.

## 2. Call the customer routes (front end)

The token works only on routes under `/api/v1/customers/{customer_ref}/…`, and only for the customer it was minted
for:

| Route | Use it to |
|---|---|
| `POST …/transfers/assess` | Show whether the Travel Rule applies and which fields to collect |
| `POST …/wallet-ownerships/discover` | Name the institution that owns the destination address |
| `GET …/network/entities`, `GET …/network/entities/{did}` | Let the customer pick the receiving institution |
| `GET …/assets`, `GET …/jurisdictions` | Populate asset and country pickers |
| `GET …/wallets/{address}/proofs` | Show the ownership proofs already submitted for a self-hosted wallet |
| `GET …/transfers/{id}` | Show the status of the customer's transfer |

## 3. Create the transfer (backend)

The front end posts what it collected to your backend, which creates the transfer exactly as in
[Send a transfer](transact-send.md). Customer tokens are read-only: they cannot create transfers or submit proofs.

To prove a customer owns a self-hosted wallet, the front end collects the signature (for example `eip-191` from
their wallet) and passes it to your backend, which submits it with your API credential:
`POST /api/v1/customers/{customer}/wallets/{address}/proofs`, or inline as `beneficiary_proof_of_ownership` when it
creates the transfer.

## Revoking

`POST /api/v1/entities/{your DID}/delegate-token/revoke-all` with the `customer_ref` invalidates every outstanding
token for that customer, for example when they sign out everywhere or their account is closed.
