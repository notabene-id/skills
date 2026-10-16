# Receive a transfer (beneficiary, v1)

As the beneficiary institution you register your customers' deposit addresses, then answer each incoming transaction:
confirm you control the destination, read the originator PII it releases, and accept or decline. When funds land
on-chain before any Travel Rule data, you report the deposit with `/tx/notify` so the originator can supply it.
Statuses and the verb table are in [Getting started](getting-started.md); pitfalls in [Caveats](caveats.md).

## Quick start

1. Register deposit addresses with `POST /address/registerCustomer` (enables routing and autoConfirm).
2. On a webhook with `status: "SENT"` and your DID in `beneficiaryVASPdid`, check the destination is yours, then
   `POST /tx/confirm?id=<id>` → `ACK`.
3. Re-read `GET /tx/info?id=<id>` for the originator PII under `ivms101.originator` (or its `pii` mirror; escrowed
   until you confirm).
4. `POST /tx/accept?id=<id>` → `ACCEPTED`, or `POST /tx/decline` with `{ "id", "reason" }` → `DECLINED`.

## Answering a transaction

1. **Receive** `notification.transactionUpdated` at `SENT`. The webhook copy may omit PII; read `/tx/info`.
2. **Confirm** with `POST /tx/confirm?id=<id>` (no body): `SENT → ACK`, asserting your institution controls
   `transactionBlockchainInfo.destination`. Confirm is not accept; send it only after checking the address is your
   customer's. An autoConfirm rule does it for addresses in your address book.
3. **Re-read after `ACK`.** Before you confirm, `ivms101.originator` holds placeholder persons with empty names: treat
   the PII as present only when a person has a non-empty name identifier (`primaryIdentifier` / `legalPersonName`).
4. **Decide** (each verb returns the updated transaction):
   - `POST /tx/accept?id=<id>` (no body), from `ACK` only. The status read can lag the confirm by seconds; retry a
     refused accept briefly.
   - `POST /tx/decline` (also legal from `SENT`, `SAVED`, `INCOMPLETE`), or `POST /tx/notReady` (can't decide yet →
     `NOT_READY`), with `{ "id", "reason" }`.
5. **Alternatives at `SENT` only:**
   - `POST /tx/reject` → `REJECTED`, body `{ "id": "6f1c…-uuid", "reason": "unknown destination address" }`.
   - `POST /tx/redirect?id=<id>&to=<did>` hands it to another institution: status stays `SENT`, only
     `beneficiaryVASPdid` changes. Confirmed at `SENT`; one source also allows `NEW`, `MISSING_BENEFICIARY_DATA`,
     `WAITING_FOR_INFORMATION` (unconfirmed). The originator side has no redirect verb: its institution is swapped
     with `/tx/update` `originatorVASPdid` ([Send § 5](send.md#5-record-the-on-chain-hash)).

### Bulk actions

`POST /tx/bulk` applies one action to every transaction between two institutions, optionally narrowed by
`originAddress` / `destinationAddress`:

```json
{
  "action": "accept",
  "originatorVASPdid": "did:ethr:0xaaaa…",
  "beneficiaryVASPdid": "did:ethr:0xbbbb…",
  "originAddress": "0xOriginAddress",
  "destinationAddress": "0xBeneficiaryAddress",
  "reason": "…"
}
```

`action`: `accept`, `approve`, `cancel`, `confirm`, `decline`, `reject`, `notReady`, `redirect` (also needs
`beneficiaryVASPdidTo`). Response: `200` with transactions or `202` with ids.

### `/tx/receive`

`POST /tx/receive` takes `{ "transactionId", "address", "asset", "originatorVASPdid", "beneficiaryVASPdid" }`, all
required. Its effect is unverified; for deposits seen first, use `/tx/notify`.

## Deposit first: `/tx/notify`

A **deposit notification**, not a settlement notice: funds arrived on-chain with no Travel Rule transfer, and you tell
Notabene so the originator institution can supply its data.

```json
{
  "txHash": "0x9c1d…",
  "transactionAsset": "ETH",
  "transactionAmount": "1000000000000000000",
  "beneficiaryVASPdid": "did:ethr:0xbbbb…",
  "destination": "0xBeneficiaryAddress",
  "origin": "0xOriginAddress",
  "customerRef": "cust-123"
}
```

Required: `transactionAmount`, `transactionAsset`, `beneficiaryVASPdid`, `destination`. Optional: `txHash` (omit
rather than `""`; must match `^[^\s]+$`), `origin`, `originatorVASPdid` (not in the published schema but honoured),
`transactionAssetDecimals`, `customerRef`, `network`, `targetProtocol` (`GTR`), `encrypted`, `beneficiaryNameHash`,
`matchingParameters` (match on `transactionAmount`, `transactionAsset`, `origin`). `id` is for Notabene-internal
callers only; omit it.

- **Matches an existing transaction** (e.g. `txHash` + `destination` + `beneficiaryVASPdid`): v1 links to it and
  auto-confirms it to `SENT`/`ACK`; it belongs to whichever flow created it.
- **No match:** a new transaction at `INCOMPLETE`. Notabene resolves the originator from `origin` when it can; without
  that or `originatorVASPdid`, it may have no originator and stay invisible to the institution that owes the data.

Then, depending on who sends `/tx/update` first (the resulting status is keyed on the caller; party data is
accepted only at `INCOMPLETE` or `MISSING_BENEFICIARY_DATA`):

- **Originator first (the happy path):** the originator institution's update yields `NEW`; it `/tx/approve`s →
  `SENT`, autoConfirm → `ACK`, you accept → `ACCEPTED`. You read its persons in cleartext under `ivms101.originator`
  via `/tx/info`.
- **You first:** valid beneficiary data **completes the notify flow**: v1 sets `SENT`, autoConfirm moves it to `ACK`
  a second later, and the originator's later update gets a 400, so its persons never arrive. **Let the originator go
  first**, or send both sides (`originator` and `beneficiary`, top-level in the request) yourself in one update.

From `ACK`, accept as usual.

The `txHash` **is** the settlement; a notify transaction has no further settlement step.

## Address registration and ownership

Registered addresses let Notabene route incoming transfers to you and let autoConfirm fire.

`POST /address/registerCustomer`:

```json
{ "vaspDID": "did:ethr:0xbbbb…", "asset": "ETH", "address": "0xBeneficiaryAddress", "customerRef": "cust-123" }
```

`vaspDID`, `address`, `customerRef` are required; `asset` is a symbol (CAIP-19 / Coingecko forms allowed). Returns
`200` with no body.

| Call | Returns |
| --- | --- |
| `GET /v1/addresses/address-ownerships/{address}?vasp_did=&asset=` | `{ owner_vasp_did, address, address_type (HOSTED\|UNHOSTED), asset, source }`, or 404. `owner_vasp_name` deprecated. |
| `GET /v1/addresses?vasp_did=&source=ADDRESS_GRAPH[&source_type=INTERNAL\|EXTERNAL&page=&per_page=]` | `{ pagination: { page, per_page, total }, metadata: { source }, data: [{ address, owner_vasp_did, customer_did, customer_ref }] }` |
| `DELETE /v1/addresses?vasp_did=&addresses=<a1>,<a2>` | `202 { message }` |
| `GET /address/upload/{vaspDID}` | Presigned `{ uploadUrl, uploadFields[] }`: POST your CSV as multipart with every upload field verbatim. |
| `POST /integrations/{vaspDID}/address-query-webhook` | Body `{ "url", "params": "?k=v" }` (leading `?`) registers a URL Notabene calls to ask whether an address is yours; response has a `secret`. |
| `DELETE /integrations/{vaspDID}/address-query-webhook` | No body; returns `{ isDeleted, message }`. |
