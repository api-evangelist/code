---
name: Request and verify a Code payment
description: >-
  Accept a micropayment on the web with Code — render a "Pay with Code" button,
  create an idempotent payment intent, and verify the payment server-side (or via
  a JWT-signed webhook) before granting the paid item.
api: grpc/code-transaction-v2.proto
operations:
  - paymentIntents.create
  - paymentIntents.getStatus
  - SubmitIntent
  - GetIntentMetadata
---

# Request and verify a Code payment

Use this skill to charge a user as little as 5 cents on the web with Code. Code
is self-custodial (no sign-up, no API key); the browser signs the transaction
and your server verifies it.

## Steps

1. **Create a payment intent (server-side).** Call
   `code.paymentIntents.create({ amount, currency, destination, idempotencyKey, webhook })`
   using `@code-wallet/client`. `destination` is your Solana wallet address.
   Always pass a deterministic `idempotencyKey` (e.g. `${order.id}`) so retries
   never double-charge — see `conventions/code-conventions.yml`. Optionally pass
   `webhook: { url }` to receive server-side events. You get back
   `{ clientSecret, id }`.

2. **Render the button (client-side).** Load the Code elements bundle and call
   `code.elements.create('button', { currency, destination, amount })` then
   `.mount('#button-container')`. The button renders a scannable payment request
   and streams `success` / `cancel` events over a WebSocket to the Code
   Sequencer (`button.on('success', ...)`).

3. **Verify the payment (server-side).** Do not trust the browser. Either:
   - call `code.paymentIntents.getStatus({ intent })` with the intent id and
     confirm the state is complete, or
   - verify the JWT signature on the webhook notification and read its
     `intent`, `amount`, `destination`, and `state` fields
     (`asyncapi/code-webhooks.yml`).
   Under the hood this maps to the gRPC `GetIntentMetadata` /
   `SubmitIntent` operations in `grpc/code-transaction-v2.proto`.

4. **Grant the item** only after the server confirms the intent is paid to your
   `destination`.

## Error handling

`SubmitIntent` returns a message-level result — handle `DENIED`,
`INVALID_INTENT`, `SIGNATURE_ERROR`, and `STALE_STATE`
(`errors/code-problem-types.yml`). On `STALE_STATE`, refetch account state and
retry with the **same** `idempotencyKey`.
