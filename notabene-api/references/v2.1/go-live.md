# Go live

Everything between a first sandbox call and production traffic, in order. In the dashboard, each step shows whether
your institution has done it, read from what you have actually set up and sent.

## Your progress

1. **Wallets.** Connect a wallet provider under **Integration → Wallet Providers** if one holds your addresses;
   otherwise you are self-managed. [Choose your integration](choose-your-integration.md)
2. **API credential.** Create one under **Integration → API Credentials**. [Getting started](getting-started.md)
3. **Webhook endpoint.** Add one under **Integration → Webhooks**. [Webhooks](webhooks.md)
4. **Customers.** Register the customers you transfer for. [Customers](customers.md)
5. **Deposit addresses.** Register each one as you hand it out. [Wallet addresses](wallets.md)
6. **A first transfer** in sandbox. [Send a transfer](transact-send.md)
7. **A first settlement**: broadcast after `ReadyToSettle`, then report the transaction hash.

When every step is done in sandbox, repeat the setup in production (its own credential, endpoint and addresses),
then check the list below.

## Before production traffic

These cannot be read from your account, so check them in your code.

### Credentials and environment

- [ ] A production API credential, stored in your secrets manager; sandbox and production never share one.
- [ ] The base URL and token `audience` both point at your production environment.
- [ ] Tokens are cached and refreshed before `expires_in`, not minted per request.

### Correctness

- [ ] Every transfer create sends a `ref` you persisted first, and retries replay the identical body.
- [ ] A retried customer registration treats `409` as already done.
- [ ] Amounts are whole-unit decimal strings, built from your ledger's decimal type, never floats.
- [ ] A `409` on authorize re-reads and retries once.
- [ ] Outgoing withdrawals broadcast only after `ReadyToSettle`, then call settle with the transaction hash.
- [ ] Deposits credit on `Settled`, and `SettledIncompleteAuthorization` is treated as unfinished.

### Webhooks

- [ ] The production endpoint has its own signing secret.
- [ ] Signatures are verified on the raw body.
- [ ] Handlers de-duplicate on `svix-id` and answer `2xx` within seconds.
- [ ] A daily job lists transfers with `requires_action: true`, so nothing waits on a lost event.

### Customers and addresses

- [ ] Every deposit address is registered with `POST /api/v1/wallets` as it is issued, in CAIP-10 form.
- [ ] Your withdrawal form collects the fields assess reports as `required`, and no more.
