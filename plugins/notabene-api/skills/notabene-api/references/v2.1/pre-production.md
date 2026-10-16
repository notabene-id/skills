# Pre-production checklist

What only your code can show, checked before real traffic. Go Live tracks what your account shows; these checks
cover what it cannot: how your code retries, handles money and reads events.

## Quick start

Work through the four groups below once **Integration → Go Live** shows every step done. Walk the integration's code
against each check: point to the code that satisfies it, or say what is missing and propose the change. Report every
check as done, missing or not applicable to the integration type.

## Credentials and environment

| Check | Why it matters | Guide |
|---|---|---|
| A production API credential, kept in your secrets manager. Sandbox and production never share one. | The secret is shown once, and a leaked sandbox secret must never reach live funds. | [Create an API credential](authentication.md#2-create-an-api-credential) |
| The base URL and the token `audience` both name your production environment. | A sandbox audience on a production token answers `401` on every call. | [Your base URL](authentication.md#1-your-base-url) |
| Tokens are cached and refreshed before `expires_in`, not minted per request. | Minting per request adds a round trip to every call and hits the token endpoint's limits. | [Get an access token](authentication.md#3-get-an-access-token) |

## Money and retries

| Check | Why it matters | Guide |
|---|---|---|
| Every transfer create sends a `ref` you stored first; a retry replays the identical body. | A timeout can hide a transfer that was created. The same `ref` returns it instead of a second one. | [Create the transfer](transact-send.md#2-create-the-transfer) |
| A retried customer registration treats `409` as already done. | Customers are keyed by your `id`; the retry finds the one the first call made. | [Register a customer](customers.md#register-a-customer) |
| Amounts are whole-unit decimal strings built from your ledger's decimal type, never floats. | A float rounds; `"0.1"` ETH must mean exactly that. | [Conventions](authentication.md#conventions) |
| A `409` on authorize re-reads the transfer and retries once. | The transfer changed while you decided, so decide on what it is now. | [Announced before settlement](transact-receive.md#announced-before-settlement) |

## Settlement

| Check | Why it matters | Guide |
|---|---|---|
| Withdrawals broadcast only after `ReadyToSettle`, then call settle with the transaction hash. | A broadcast cannot be recalled; every institution must have authorized first. | [Broadcast, then settle](transact-send.md#5-broadcast-then-settle) |
| Deposits credit your customer on `Settled`, and `SettledIncompleteAuthorization` stays unfinished. | Funds that arrived before the checks cleared still need your decision. | [Seen on-chain first](transact-receive.md#seen-on-chain-first) |
| Your withdrawal form asks for the fields assess reports as `required`, and no more. | Collecting less stalls the transfer; collecting more is data you then have to hold. | [Assess](transact-send.md#1-assess-optional-recommended) |
| Every deposit address is registered with `POST /api/v1/wallets` as it is issued, in CAIP-10 form. | Senders find you from the address, and ownership checks answer themselves. | [Register an address](wallets.md#register-an-address) |

## Webhooks

| Check | Why it matters | Guide |
|---|---|---|
| The production endpoint has its own signing secret. | Each environment's endpoint is a separate trust boundary. | [Register an endpoint](webhooks.md#1-register-an-endpoint) |
| Signatures are verified on the raw request body. | Re-serialised JSON fails verification, and an unverified event is not from Notabene. | [Verify every request](webhooks.md#2-verify-every-request) |
| Handlers save each event, de-duplicate on `svix-id`, and answer `2xx` within seconds. | Delivery is at least once and in any order; a slow answer is retried. | [Handle events safely](webhooks.md#3-handle-events-safely) |
| A daily job pages open transfers (`GET /api/v1/transfers?status=Active`, and `SettledIncompleteAuthorization`) and reads each one for `requires_action: true`. | Nothing waits on you because of an event that never arrived. | [Handle events safely](webhooks.md#3-handle-events-safely) |
