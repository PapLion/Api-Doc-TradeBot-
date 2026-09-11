# BloxTop Delivery API - Bot Developer Handoff

This document is for the developer implementing the Roblox delivery bot.

Use this environment only for Stealbox sandbox testing. Do not send these credentials to logs, Discord, GitHub, screenshots, or production.

## Sandbox configuration

```env
DELIVERY_BASE_URL=https://bloxtop-delivery-sandbox.vercel.app
DELIVERY_API_KEY=1a7fbd75b08a9a7e491597d51cf40781da5e9045f3c5fa77f5b9833aec1236c9
```

Use environment variables. Do not hardcode either value in the bot source.

Only these public routes are part of the contract:

```text
POST /claim
POST /deliveries
POST /deliveries/reserve
POST /deliveries/next
POST /deliveries/reconcile
POST /deliveries/{delivery_id}/result
POST /deliveries/{delivery_id}/release
POST /deliveries/{delivery_id}/retry-required
```

Always use `POST`. Do not call `/api/...` aliases or rely on `GET` behavior.

## What the bot must do

The delivery worker repeatedly requests the next eligible delivery, delivers every returned item in Roblox, and reports exactly one final outcome.

```text
POST /deliveries/next
  -> 204: no work; wait and poll again
  -> 200: persist the DTO and deliver every item
       -> all items delivered: POST /result with completed:true
       -> temporary technical issue: persist and retry locally; do not call the API
       -> player absent/cancelled but may retry: POST /release
       -> player must start Claim again: POST /retry-required
       -> definite delivery failure: POST /result with completed:false
       -> uncertain HTTP response: retry the exact same /result
```

Important rules:

- Persist the complete delivery DTO before entering Roblox.
- Treat `delivery_id` as an opaque string.
- Never modify, shorten, decode, or reconstruct `delivery_id`.
- Never use `order_number` instead of `delivery_id`.
- Deliver every item and its exact `quantity`.
- Select game automation using `items[].game.id`.
- Select the item using `items[].item_code`.
- Never identify an item using its visible title alone.
- Send `completed:true` only after all items were actually delivered.
- Do not call `/next` again while a reserved delivery has an unresolved result.
- Clear locally persisted delivery state only after `/result` returns `200`.

## Delivery outcomes and retries

The bot, not the website, decides whether a Roblox trade is temporarily
retryable or terminal.

| Situation | Bot action |
|---|---|
| Roblox/network error or bot restart | Keep the same persisted `delivery_id` and retry internally. Do not call BloxTop yet. |
| Player does not join or cancels but can try again later | Call `/release`. The order returns to the pending list immediately. |
| Player selected the wrong Roblox account or must restart the claim flow | Call `/retry-required`. The order disappears from the bot list until the buyer completes Claim again. |
| Fraud, impossible delivery, or a final business failure | Call `/result` with `completed:false`. This is terminal and goes to support. |
| Every item delivered | Call `/result` with `completed:true`. Shopify creates the fulfillment. |

### Release a delivery for another attempt

```http
POST /deliveries/{delivery_id}/release
Authorization: Bearer DELIVERY_API_KEY
Content-Type: application/json
```

```json
{"reason":"Player did not join"}
```

`reason` is optional. A successful call returns `200 {"ok":true}` and puts
the same order back in the pending list immediately. The old `delivery_id` is
no longer valid for `/result`; when the bot reserves the order again, it gets a
new `delivery_id`.

### Require the buyer to Claim again

```http
POST /deliveries/{delivery_id}/retry-required
Authorization: Bearer DELIVERY_API_KEY
Content-Type: application/json
```

```json
{"reason":"Confirm another Roblox account"}
```

`reason` is optional. A successful call returns `200 {"ok":true}`. This does
not mark the order as terminally failed. It removes it from the bot queue until
the buyer completes `/claim` again with their order number, email, and a
confirmed Roblox username. The buyer may choose a different Roblox username.

Both routes are idempotent: if their successful HTTP response is lost, retry
the exact same request. They return `401` for a bad key, `404` for an unknown
or replaced delivery ID, `409` for a delivery that cannot transition, and
`503` when Shopify is temporarily unavailable.

## Authentication

`/deliveries/next` and `/deliveries/{delivery_id}/result` require:

```http
Authorization: Bearer DELIVERY_API_KEY
```

The scheme, one space, and exact key are required. Missing or incorrect authentication returns:

```http
HTTP/1.1 401 Unauthorized
```

The body is empty. A `401` is not retryable: stop the worker and fix its configuration.

`POST /claim` is public and does not use the delivery API key.

All API responses use:

```http
Cache-Control: no-store
```

## Optional: inspect and choose a pending delivery

The normal worker flow remains `/deliveries/next` followed by `/result`.
These two optional endpoints are for a bot that needs to choose its own order
based on its local trade or inventory logic.

Only customer-confirmed MM2 claims appear here. A Shopify purchase where the
customer never completed `/claim` is never returned and cannot block another
delivery.

### Choose one worker mode

**Automatic mode (existing):** call `POST /deliveries/next`. It atomically
selects and reserves one available delivery for the bot.

**Manual-selection mode (optional):** call `POST /deliveries`, choose an
`order_number` from the read-only list using the bot's own logic, then call
`POST /deliveries/reserve` for that exact order.

Both modes return the same reserved-delivery payload and both must finish with
`POST /deliveries/{delivery_id}/result`. The list endpoint never returns a
`delivery_id`; that ID is created only after `/next` or `/reserve` reserves the
claim. Do not try to reserve an already in-flight delivery again.

### List pending claims

```http
POST /deliveries
Authorization: Bearer DELIVERY_API_KEY
```

This is read-only: it does not reserve, reorder, or change any delivery.

```bash
curl --request POST "$DELIVERY_BASE_URL/deliveries" \
  --header "Authorization: Bearer $DELIVERY_API_KEY"
```

```json
{
  "deliveries": [
    {
      "order_number": "#1009",
      "roblox_username": "ExamplePlayer",
      "items": [
        {
          "game": {"id": "murdermystery2", "name": "Murder Mystery 2"},
          "display_name": "Seer",
          "item_code": "Seer",
          "quantity": 1
        }
      ]
    }
  ]
}
```

### Reserve a selected claim

```http
POST /deliveries/reserve
Authorization: Bearer DELIVERY_API_KEY
Content-Type: application/json
```

```json
{"order_number":"#1009"}
```

```bash
curl --request POST "$DELIVERY_BASE_URL/deliveries/reserve" \
  --header "Authorization: Bearer $DELIVERY_API_KEY" \
  --header "Content-Type: application/json" \
  --data '{"order_number":"#1009"}'
```

If it is still pending and eligible, the API atomically reserves it and
returns the same delivery DTO as `/deliveries/next`, including `delivery_id`.
Use that identifier with `/deliveries/{delivery_id}/result` exactly as usual.

- `200`: reservation succeeded; persist and process the returned delivery.
- `409`: it is no longer pending or another worker already reserved it; refresh
  the list and choose again.
- `400`: invalid request body; fix it instead of retrying unchanged.
- `401`: missing or incorrect API key.
- `503`: Shopify is temporarily unavailable; retry with bounded backoff.

### Recover a reservation after a bot restart

If the bot crashes after `/reserve` or `/next` returns `200`, before it can
persist the delivery DTO, it must recover the existing reservation by order
number instead of reserving the order again. Re-reserving can return `409` and
must never create a second delivery.

```http
POST /deliveries/reconcile
Authorization: Bearer DELIVERY_API_KEY
Content-Type: application/json
```

```bash
curl --request POST "$DELIVERY_BASE_URL/deliveries/reconcile" \
  --header "Authorization: Bearer $DELIVERY_API_KEY" \
  --header "Content-Type: application/json" \
  --header "Accept: application/json" \
  --data '{"order_number":"#1009"}'
```

Successful response:

```json
{
  "delivery_id": "opaque-delivery-id",
  "order_number": "#1009",
  "roblox_username": "ExamplePlayer",
  "delivery_status": "processing",
  "items": [
    {
      "game": {"id": "murdermystery2", "name": "Murder Mystery 2"},
      "display_name": "Seer",
      "item_code": "Seer",
      "quantity": 1
    }
  ]
}
```

This endpoint is read-only: it does not reserve, create a fulfillment, or
change any Shopify metafield. It returns the persisted `delivery_id` and the
current status so the bot can resume the same delivery and later call the
existing `/result`, `/release`, or `/retry-required` endpoint.

- `200`: an existing reservation was recovered.
- `400`: invalid order number or request body; fix the request.
- `401`: missing or invalid delivery API key.
- `404`: no persisted reservation exists for that order number.
- `405`: method other than `POST`.
- `503`: Shopify is temporarily unavailable; retry with bounded backoff.

The bot should call this only during recovery when it has an order number but
no usable persisted `delivery_id`. Repeating the same reconciliation request
is safe and does not mutate the delivery.

## 1. Claim an order

This normally comes from the customer-facing claim form, not the delivery loop. It associates a paid Shopify order with a Roblox username.

### Request

```http
POST /claim
Content-Type: application/json
```

```json
{
  "order_number": "1009",
  "email": "customer@example.com",
  "roblox_username": "ExamplePlayer"
}
```

`order_number` may include or omit `#`. The email must exactly match the Shopify order contact email.

### Response

```http
HTTP/1.1 202 Accepted
Content-Type: application/json
```

```json
{"accepted":true}
```

The response is intentionally generic. `202` means the request was evaluated; it does not reveal whether the order/email combination existed or was eligible.

Do not repeat a successful structural claim merely because `/next` initially returns `204`.

Possible responses:

| HTTP | Bot/client action |
|---:|---|
| `202` | Claim evaluated. Begin or continue normal `/next` polling. |
| `400` | Fix invalid JSON or fields. Do not retry unchanged input. |
| `503` | Retry with bounded backoff. |

## 2. Request the next delivery

### Request

```http
POST /deliveries/next
Authorization: Bearer DELIVERY_API_KEY
```

No body is required.

### No delivery available

```http
HTTP/1.1 204 No Content
```

This is normal idle behavior. Wait 3 seconds and poll again. Never run a tight loop.

Shopify indexes order metafields asynchronously, so several `204` responses may occur immediately after a valid claim.

### Delivery available

```http
HTTP/1.1 200 OK
Content-Type: application/json
```

```json
{
  "delivery_id": "opaque-delivery-id",
  "order_number": "#1009",
  "roblox_username": "ExamplePlayer",
  "items": [
    {
      "game": {
        "id": "murdermystery2",
        "name": "Murder Mystery 2"
      },
      "display_name": "Seer",
      "item_code": "Seer",
      "quantity": 1
    }
  ]
}
```

The DTO contains no customer email, address, price, Shopify GID, credentials, or Service Fee item.

Validate before starting delivery:

- `delivery_id` is a non-empty string.
- `order_number` is a non-empty support reference.
- `roblox_username` is a non-empty string.
- `items` is a non-empty array.
- Every item has `game.id === "murdermystery2"`.
- Every item has a non-empty `item_code`.
- Every `quantity` is a positive integer.

If the DTO is malformed or includes another game, do not deliver it and alert the API operator. Do not invent corrected values.

Possible responses:

| HTTP | Worker action |
|---:|---|
| `200` | Persist and process this delivery. |
| `204` | Wait 3 seconds and poll again. |
| `401` | Stop; API key is missing or invalid. |
| `503` | Retry `/next` with bounded backoff. |

## MM2-only behavior

The server, not the bot, filters the Shopify queue:

- Service Fee lines are ignored.
- At least one deliverable item must remain.
- Every deliverable item must have canonical game ID `murdermystery2`.
- MM2 plus Service Fee is eligible.
- Another game by itself is skipped.
- MM2 mixed with another game is skipped.
- Skipped orders are not reserved, returned, or marked failed.

The bot must still validate `game.id` defensively before automation. Never filter by `display_name` or Shopify product title.

## 3. Report the delivery result

Use the exact `delivery_id` returned by `/next` as the URL path value.

```http
POST /deliveries/{delivery_id}/result
Authorization: Bearer DELIVERY_API_KEY
Content-Type: application/json
```

### Successful Roblox delivery

Only after every item was delivered:

```json
{"completed":true}
```

Successful response:

```http
HTTP/1.1 200 OK
Content-Type: application/json
```

```json
{"ok":true}
```

`completed:true` immediately authorizes the API to create the Shopify fulfillment. Shopify fulfillment is the completed state; there is no separate `delivery_status=completed` value.

The operation is idempotent. Repeating the identical URL and body after a lost or uncertain response returns `200` without creating a duplicate fulfillment.

### Failed Roblox delivery

When the delivery definitely failed:

```json
{
  "completed": false,
  "reason": "Player did not join"
}
```

`reason` is optional. When provided, it must be non-empty plain text with at most 240 characters.

Do not send `failure_reason`; the public field is `reason`.

`completed:false` records the order as failed and creates no Shopify fulfillment. Retrying the same failure is safe; the first reason is preserved.

### Result retry behavior

Shopify may take several seconds to index a `delivery_id` immediately after `/next` returns `200`. During this propagation window, `/result` can temporarily return `404`.

For a timeout, connection reset, `503`, or an initial `404`:

1. Keep the exact same `delivery_id`.
2. Keep the same `completed` value.
3. Keep the same `reason`, if present.
4. Retry the identical request.
5. Do not call `/next` for replacement work.

Recommended bounded schedule:

```text
1 second -> 2 seconds -> 4 seconds -> 8 seconds -> 12 seconds
Maximum: 8 attempts or 45 seconds total
```

An initial `404` is retryable only for the identifier just returned by `/next`. If it remains `404` after the bounded window, stop that delivery and report it to the operator.

Possible responses:

| HTTP | Worker action |
|---:|---|
| `200` | Result confirmed. Clear persisted delivery state. |
| `400` | Request is invalid. Stop and report the implementation error. |
| `401` | Stop the worker and fix its API key. |
| Initial `404` | Retry the exact result with bounded backoff. |
| Persistent `404` | Stop and report; do not request replacement work. |
| `409` | Stop and report a delivery-state conflict. |
| `503` | Retry the exact result with bounded backoff. |
| Timeout/reset | Retry the exact result; its outcome may already have been recorded. |

## Reference worker algorithm

```text
load persisted unresolved delivery, if one exists

if no delivery_id is persisted but the bot knows the order number from the
interrupted job:
  POST /deliveries/reconcile with that order number
  if 200: persist the recovered DTO and resume it
  if 404: report that no reservation can be recovered; do not reserve blindly

while running:
  if an unresolved delivery exists:
    resume it or resend its persisted final result
    do not call /next
    continue

  response = POST /deliveries/next with Bearer key and 25-second timeout

  if response == 204:
    wait 3 seconds
    continue

  if response == 401:
    stop worker and alert operator

  if response == 503 or transport failed:
    bounded backoff
    continue

  if response != 200:
    alert operator
    bounded backoff
    continue

  validate delivery DTO
  persist complete DTO atomically

  outcome = deliver every item using game.id + item_code + quantity

  persist final result before sending it

  if outcome succeeded:
    result body = {completed: true}
  else:
    result body = {completed: false, reason: safe short reason}

  retry identical POST /deliveries/{delivery_id}/result until 200
  apply the bounded result rules above

  after 200 only:
    clear persisted DTO and result
```

Use a 25-second HTTP timeout for every individual request. A client timeout does not prove the server failed; retry idempotently according to the tables above.

## curl examples

Claim:

```bash
curl --request POST "$DELIVERY_BASE_URL/claim" \
  --connect-timeout 5 \
  --max-time 25 \
  --header "Content-Type: application/json" \
  --data '{"order_number":"1009","email":"customer@example.com","roblox_username":"ExamplePlayer"}'
```

Next delivery:

```bash
curl --request POST "$DELIVERY_BASE_URL/deliveries/next" \
  --connect-timeout 5 \
  --max-time 25 \
  --header "Authorization: Bearer $DELIVERY_API_KEY"
```

Successful result:

```bash
curl --request POST "$DELIVERY_BASE_URL/deliveries/<delivery_id>/result" \
  --connect-timeout 5 \
  --max-time 25 \
  --header "Authorization: Bearer $DELIVERY_API_KEY" \
  --header "Content-Type: application/json" \
  --data '{"completed":true}'
```

Failed result:

```bash
curl --request POST "$DELIVERY_BASE_URL/deliveries/<delivery_id>/result" \
  --connect-timeout 5 \
  --max-time 25 \
  --header "Authorization: Bearer $DELIVERY_API_KEY" \
  --header "Content-Type: application/json" \
  --data '{"completed":false,"reason":"Player did not join"}'
```

## Logging requirements

Safe fields to log:

- Endpoint name.
- HTTP status.
- Attempt number.
- Request duration.
- Local correlation ID.
- `order_number` when support needs it.
- `game.id`, `item_code`, and `quantity`.

Never log:

- `DELIVERY_API_KEY` or the Authorization header.
- Full `delivery_id`.
- Customer email, name, address, or payment data.
- Shopify credentials or GIDs.
- Raw request/response dumps that may contain sensitive values.

## Verified sandbox behavior

The stable endpoint was verified end-to-end on July 24, 2026:

```text
invalid API key -> 401
claim -> 202
next -> temporary 204 responses while Shopify indexed the claim
next -> 200 with MM2 Seer x1
result completed:true -> temporary 404 responses while delivery_id indexed
same result retry -> 200
identical completed:true retry -> 200
Shopify order -> FULFILLED
active unfulfilled quantity -> 0
duplicate fulfillment -> none
```

Verified deployment:

```text
dpl_3KN5WZosprD6vpC2TxgxB4QzY6Nh
```

This hostname is a dedicated sandbox connected only to the Stealbox development store. It is not the BloxTop production API.

---

## Update: MM2 inventory snapshot endpoint

This is a new endpoint for the bot's real Roblox inventory. It does not change
the existing claim or delivery endpoints.

### What the bot must do

The bot remains the source of truth for the items it physically has. Whenever
its MM2 inventory changes, it should send a complete current snapshot to
BloxTop, rather than trying to calculate a separate Shopify adjustment for each
trade.

```text
Bot inventory changes
  -> bot sends its current MM2 stock list
  -> BloxTop validates each item as MM2
  -> BloxTop updates Shopify inventory
  -> Shopify controls whether the product can still be purchased
```

The bot does **not** need to query Shopify or decrement Shopify stock after a
normal delivery. Shopify already handles stock reserved or consumed by customer
purchases. The bot only reports what it currently has in Roblox.

### Request

```text
PUT /inventory/snapshot
Authorization: Bearer <DELIVERY_API_KEY>
Content-Type: application/json
```

It uses the same `DELIVERY_API_KEY` as `/deliveries/next` and
`/deliveries/{delivery_id}/result`.

```json
{
  "snapshot_id": "550e8400-e29b-41d4-a716-446655440000",
  "items": [
    {"item_code": "Lightbringer", "quantity": 3},
    {"item_code": "Gemstone", "quantity": 1}
  ]
}
```

Requirements:

- `snapshot_id` must be a new UUID for each newly generated snapshot.
- Reuse the **same** `snapshot_id` and exactly the same body when retrying a
  request after a timeout or connection failure.
- `items` is the bot's full current MM2 inventory list, not a `+1` / `-1`
  adjustment list.
- `item_code` must match the BloxTop variant item code exactly.
- `quantity` is a non-negative integer. Use `0` when the bot has no units.
- Do not include Service Fee or items from another game.

### Successful response

```json
{
  "ok": true,
  "received_items": 2,
  "updated_items": 2
}
```

After `200`, Shopify has accepted the stock snapshot. Repeating the same
snapshot is safe and does not create an extra stock adjustment.

### Errors and retries

- `401`: the delivery API key is missing or invalid. Do not retry until the
  configuration is fixed.
- `422`: an item is not a valid automatic MM2 item or the body is invalid. Fix
  the inventory data; do not retry unchanged.
- `503`, timeout, or network failure: retain the snapshot locally and retry the
  identical request with bounded backoff.

For local testing, a JSON file is sufficient to persist the last unsent
snapshot. For a long-running production bot, SQLite is safer.

### curl example

```bash
curl --request PUT "$DELIVERY_BASE_URL/inventory/snapshot" \
  --connect-timeout 5 \
  --max-time 25 \
  --header "Authorization: Bearer $DELIVERY_API_KEY" \
  --header "Content-Type: application/json" \
  --data '{
    "snapshot_id":"550e8400-e29b-41d4-a716-446655440000",
    "items":[
      {"item_code":"Lightbringer","quantity":3},
      {"item_code":"Gemstone","quantity":1}
    ]
  }'
```

### Safety guarantees

- BloxTop accepts only variants whose real Shopify configuration says they are
  Murder Mystery 2 items.
- Service Fee and other games are rejected and cannot be changed through this
  endpoint.
- The endpoint updates stock in the dedicated MM2 bot inventory location.
- This endpoint does not alter `/claim`, `/deliveries/next`, or
  `/deliveries/{delivery_id}/result`.
