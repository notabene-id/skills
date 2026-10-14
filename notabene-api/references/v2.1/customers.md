# Customers

Every transfer names two customers: the originator who sends and the beneficiary who receives. Registering yours
once gives Notabene the details the Travel Rule asks for, so each transfer only has to say who, not everything about
them.

## Quick start

Give each customer an id of your own, send their details once, then name them by that id.

1. When a customer onboards, `POST /api/v1/customers` with your id for them and their name.
2. On every transfer, pass that id as `originator_id` or `beneficiary_id`.
3. When their details change, `PUT /api/v1/customers/{id}` with what changed.

## Why it matters

- **The Travel Rule is about people.** What travels with a transfer is who sent it and who receives it, as IVMS-101,
  the same originator and beneficiary data a wire carries. Registering stores it once; every transfer reuses it.
- **Other features hang off a customer.** Deposit addresses are linked to one ([Wallet addresses](wallets.md)),
  customer tokens for your front end are minted for one ([Embedded checks](transact-embedded.md)), and ownership
  proofs for self-hosted wallets are filed under one.
- **Incoming transfers are checked against your record.** When a counterparty names your customer as the
  beneficiary, the name it sends is matched against what you registered.

## Decisions

| Decision | Options | Recommended |
|---|---|---|
| The customer's id | Any IRI you generate: `urn:uuid:…`, `did:…`, `mailto:…`, `https:…` | A `urn:uuid:` that never changes, derived from your own user id. Use a DID only if you will attach self-hosted wallet proofs inline on transfers, which require one. |
| When to register | At onboarding, or let the first transfer create them | At onboarding. A transfer that names an unknown id creates the customer, but only with what that transfer carried. |
| How much to send | From the name alone up to address, date of birth and national id | The name, plus what [assess](transact-send.md) reports as `required` for the jurisdictions you send to. Collect no more. |
| Person or business | `naturalPerson` or `legalPerson` | Whatever the customer is: the rules for each differ. |

## Register a customer

```bash
curl -s https://sandbox.notabene.id/api/v1/customers \
  -H "Authorization: Bearer $TOKEN" -H 'content-type: application/json' \
  -d '{
    "id": "urn:uuid:0c8f6a52-6e83-4f2b-9d0a-7b1f0e7c2a41",
    "type": "Individual",
    "jurisdiction": "DE",
    "profile": {"ivms": {"naturalPerson": {"name": {"nameIdentifier": [
      {"primaryIdentifier": "Smith", "secondaryIdentifier": "Alice", "naturalPersonNameIdentifierType": "LEGL"}]}}}}
  }'
```

- **The id is yours.** Notabene never generates one. Store it on your user record.
- **Registering twice** answers `409`: treat it as already done.
- **A business** uses `legalPerson`, with `legalPersonName` and `legalPersonNameIdentifierType` (`LEGL`, `SHRT` or
  `TRAD`) in each name identifier.
- **Field names inside `profile.ivms`** are IVMS-101's own camelCase; the rest of the body is snake_case.

## Name them on transfers

Pass the id on every transfer: `originator_id` when your customer sends, `beneficiary_id` when they receive. The id is
always required. A profile inline on the transfer (`originator`, `beneficiary`) is optional; when you send one, it
replaces the stored profile, so send it only when the details really changed.

The counterparty's customer gets an id too: yours to choose, for example a `urn:uuid:` per transfer, with the
profile you know inline.

## Update and delete

- **`PUT /api/v1/customers/{id}`** changes only what you send: profile fields you omit keep their values.
- **`DELETE /api/v1/customers/{id}`** removes the customer, and answers `409` once any transfer names them.
- **Reading a profile back:** customer responses carry the id, type, jurisdiction and status, not the personal data.
  Read a transfer with `GET /api/v1/transfers/{id}?include_ivms=true` to see what it carried.

## Customers on the other side

`GET /api/v1/customers` lists your own customers by default. Its `type` filter also lists:

- **`CounterpartyCustomer`** — the people on the other side of your transfers.
- **`OurCustomerReceived`** — someone a counterparty says is yours, but you never registered. Transfers to them wait
  for human review; registering them with `POST /api/v1/customers` makes them yours.
