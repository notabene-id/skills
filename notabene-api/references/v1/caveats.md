# Caveats (v1)

The v1 behaviours that break integrations, and the facts these guides could not confirm. The other v1 guides link here
rather than repeat them: [Getting started](getting-started.md), [Send](send.md), [Receive](receive.md),
[Webhooks](webhooks.md).

## Quick start

Check the five that bite most: amounts are integer base units; accept needs `ACK`; `/tx/info` and `/tx/update`
succeed silently on transactions you can't see; omit keys rather than send `""`; build idempotency on
`transactionRef`.

## Gotchas

1. **Amounts are integer base units.** `transactionAmount` is a string in the smallest unit (wei, satoshi, 10^-6 for
   USDC): `"1000000000000000000"` is 1 ETH. Notabene's own v1↔current bridge multiplies by 10^decimals into v1 and
   divides out. `amountInLocalCurrency.amountInLocalCurrency` is a plain JSON fiat number: parse as decimal, not
   float. Absurd exponents like `1e2000000000` are rejected.
2. **Assets are symbols** (`ETH`, `USDC`), not CAIP-19, unless you use the `{ "caip19": … }` form. Send
   `customAssetPrice` only for assets Notabene can't price: its `{ price, currency }` form is feature-flagged, accepts
   only `USD` or `EUR`, and can fail the whole create.
3. **`did:ethr` institution DIDs** are directory ids with no DIDComm endpoint or resolvable document; look up with
   `GET /tf/simple/vasps/{vaspDID}` (search: `GET /tf/simple/vasps`). Not interchangeable with the `did:web` the same
   institution has on the current platform.
4. **Customer DIDs aren't always DIDs.** `originatorDid` / `beneficiaryDid` may be `did:email:…`; store as opaque.
5. **Confirm ≠ accept.** Confirm asserts you control the destination; accept is the compliance decision. Never confirm
   an unchecked address (confirming on a counterparty's behalf let transfers complete with nobody acting).
6. **Accept needs `ACK`.** Accept at `SENT` is refused although the published map lists `SENT`. Confirm (or let
   autoConfirm run), then accept, retrying briefly for read lag.
7. **Rules are asynchronous.** Auto-approve (`NEW → SENT`) and autoConfirm (`SENT → ACK`) lag the triggering call by
   up to several seconds. A beneficiary accepting a `NEW` transaction can land it in `ACCEPTED` without `SENT` ever
   being published: wait for `SENT` before acting.
8. **Silent success.** Out-of-scope `/tx/update` → 200 no-op; `/tx/info` → empty 200. Re-read to verify writes.
9. **Omit, don't blank.** An empty-string DID on `/tx/update` clears the stored value; an empty `txHash` on
   `/tx/notify` fails validation.
10. **PII escrow.** Originator PII is encrypted to the originator institution's escrow key and released to the
    beneficiary only after it confirms (until then the block is `{}` for it). Cleartext you post is stored encrypted
    (IPFS-backed) and even `/tx/list?decrypt=true` doesn't resolve every field: keep your own copy of what you sent.
11. **Idempotency is yours.** `/tx/create` has no idempotency key. Use a unique `transactionRef` and, before retrying a
    timed-out create, check `GET /tx/list?vaspDID=…&transactionRef=…`. Webhooks are at-least-once.
12. **Above-threshold data** rules (persons, account number, id/birth/address): see [Send § Create](send.md#2-create);
    validate with `/tx/validate/full`.
13. **Deposit-notify ordering.** A beneficiary `/tx/update` with full data closes the originator's window
    ([Receive § Deposit first](receive.md#deposit-first-txnotify)).
14. **Casing and pagination differ.** `/tx/list` uses `vaspDID` and `paginateInfo { totalItems, … }`; `/v1/addresses*`
    uses `vasp_did`, `per_page` and `pagination { page, per_page, total }`; `/tx/cancel` uses `vasp_did`.
15. **`pending-` hashes.** A `txHash` starting with `pending-` is a pre-settlement placeholder, not an on-chain hash.

## Unverified facts

- **Token URL.** Kept in a secret; `https://auth.notabene.id/oauth/token` is inferred from the production Auth0 issuer
  (`https://auth.notabene.id/`; QA `https://auth.notabene.dev/`).
- **Integrator audience.** Only Notabene's privileged node-tier audience (`https://node.api.notabene.id`,
  `https://node.api-qa.eu.notabene.dev`; not for integrators) and a test harness using the API base URL appear.
  Presume the API base URL (e.g. `https://api.notabene.id`); confirm in onboarding material.
- **Webhook signing secret.** How an institution gets its Svix secret, and the envelope's `version` value, aren't shown.
- **Unexercised endpoints.** `/tx/receive`, `/tx/updateTR`, `NOT_READY`, `WAITING_FOR_INFORMATION`, and the
  `processBlockchainTransaction` / `haltBlockchainTransaction` events are described from types and comments only.
- **Address query webhook.** The request Notabene sends to your URL isn't modelled.
- **US region.** No separate US v1 base URL appears in the config.
