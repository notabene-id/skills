# Customers

Every transfer names two customers: the originator who sends and the beneficiary who receives. Registering yours
once gives Notabene the details the Travel Rule asks for, so each transfer only has to say who, not everything about
them.

## Contents

- [Quick start](#quick-start)
- [How it works](#how-it-works)
- [Why it matters](#why-it-matters)
- [Decisions](#decisions)
- [Register a customer](#register-a-customer)
- [Name them on transfers](#name-them-on-transfers)
- [Where the personal data is held](#where-the-personal-data-is-held)
- [Update and delete](#update-and-delete)
- [Customers on the other side](#customers-on-the-other-side)

## Quick start

Give each customer an id of your own, send their details once, then name them by that id.

1. When a customer onboards, `POST /api/v1/customers` with your id for them and their name.
2. On every transfer, pass that id as `originator_id` or `beneficiary_id`.
3. When their details change, `PUT /api/v1/customers/{id}` with what changed.

## How it works

Register a customer once; every transfer after that names them by id, and their details travel with it.

## Why it matters

- **The Travel Rule is about people.** What travels with a transfer is who sent it and who receives it, as IVMS-101,
  the same originator and beneficiary data a wire carries. Registering stores it once; every transfer reuses it.
- **Other features hang off a customer.** Deposit addresses are linked to one ([Wallet addresses](wallets.md)),
  customer tokens for the embedded components are minted for one ([Embedded components](transact-embedded.md)), and
  ownership proofs for self-hosted wallets are filed under one.
- **Incoming transfers are checked against your record.** When a counterparty names your customer as the
  beneficiary, the name it sends is matched against what you registered.

## Decisions

| Decision | Options | Recommended |
|---|---|---|
| The customer's id | Any IRI you generate: `urn:uuid:…`, `did:…`, `mailto:…`, `https:…` | A `urn:uuid:` that never changes, derived from your own user id. Use a DID only if you will attach self-hosted wallet proofs inline on transfers, which require one. |
| When to register | At onboarding, or let the first transfer create them | At onboarding. A transfer that names an unknown id creates the customer, but only with what that transfer carried. |
| How much to send | From the name alone up to address, date of birth and national id | The name. The embedded components ask the customer for whatever else a transfer requires ([Embedded components](transact-embedded.md)); a server-only form collects what [assess](transact-send.md#1-collect-transfer-details-from-your-customer) reports as `required`. Collect no more. |
| Where the personal data is held | Notabene hosts it encrypted, or you encrypt it yourself end to end | Let Notabene host it, unless policy requires that nobody but you and the counterparty can ever read it ([below](#where-the-personal-data-is-held)). |
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

Name each customer on a transfer by id: `originator_id` when your customer sends, `beneficiary_id` when they
receive. A profile inline on the transfer (`originator`, `beneficiary`) is optional and goes alongside the id, not
instead of it. The API reference describes the `*_id` fields as an alternative to the inline profile, but an inline
profile sent without its id is rejected, and a profile sent as `ivms_encrypted` always needs its `*_id`, since no id
can be read from an encrypted blob. For a registered customer, an inline profile replaces the stored one, so
send it only when the details really changed.

The counterparty's customer gets an id too: yours to choose, for example a `urn:uuid:` per transfer, with the
profile you know inline. Name them on every withdrawal: the Travel Rule asks for the beneficiary's
name, and a transfer that gives a `settlement_address` but no `beneficiary_entity_did` is rejected without one.

## Where the personal data is held

Two ways to handle your customers' personal data. The first is the default and what every example on these pages
does; the second keeps Notabene from ever reading it.

### Hosted by Notabene, encrypted

You send the profile as `ivms`, in the clear over TLS. Notabene encrypts it at rest with a key held for your
institution, exchanges it with the counterparty institution over the Notabene network, sealed to that
institution's key, and uses it for what the transfer needs: name matching, screening, the dashboard's view of a
transfer. The counterparty's data reaches you the same way, and `GET /api/v1/transfers/{id}?include_ivms=true`
shows it as `ivms`.

### End to end, with your own keys

You encrypt the profile yourself, to the counterparty institution's published key, and Notabene passes it through
without decrypting it. Nobody but you and that institution can read it, and in exchange Notabene cannot match
names or screen it for you: those checks are yours. It is per transfer, never on a registered customer, because
each payload is sealed to one recipient.

- **Encrypt.** Resolve the counterparty's DID (`beneficiary_entity_did` or an agent on the transfer) to its DID
  document and take the `keyAgreement` entry named `#pii`: a P-256 public key. Encrypt the IVMS-101 JSON as a JWE
  compact string, `ECDH-ES` with `A256GCM`, with that entry's full id (`did:web:counterparty.example#pii`) as the
  `kid` in the protected header: that is how Notabene knows who it is for.
- **Send it.** On create, pass `ivms_encrypted` instead of `ivms` in `originator` or `beneficiary`, with the
  party's id alongside, and optionally `ivms_encrypted_fields`, the IVMS-101 JSON paths the payload contains, so the
  counterparty knows what it holds before opening it. When the counterparty was not known at create, or more than one
  institution needs the data, send one payload per recipient afterwards with
  `POST /api/v1/transfers/{id}/parties/{role}/encrypted-pii` (`role` is `originator` or `beneficiary`).
- **Receive it.** Publish your own `#pii` key, a P-256 key agreement entry in the DID document you host for your
  institution's `did:web`, and keep its private half. Counterparties, and Notabene's own exchange, then seal your
  customers' counterparty data to it; `include_ivms=true` returns it as `ivms_encrypted`, a JWE for you to decrypt.
  Without a published key, what you receive is sealed to the key Notabene holds for you and read back as `ivms`.

Both modes can mix on one integration: send end to end to the counterparties that require it, hosted to the rest.

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
