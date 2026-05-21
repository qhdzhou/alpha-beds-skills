---
name: alpha-beds-hotel-fluxa
description: >-
  Alpha Beds Hotel Booking via FluxA — AI agents can search hotels worldwide with
  natural language, browse rooms, verify real-time rates, create orders, and pay with
  USDC via FluxA Agent Wallet. Use this skill when the user asks to find, compare,
  or book hotels and wants to pay with crypto (USDC on Base).
---

# Alpha Beds Hotel Booking (FluxA Payment)

**Skill version: 1.1.0**

Alpha Beds gives AI agents a complete hotel booking API — search 700,000+ hotels worldwide with natural language, browse rooms, verify rates, and book with USDC payment via FluxA, all through a single authenticated REST API.

## Prerequisites

- **FluxA Agent Wallet** — MUST have the `fluxa-agent-wallet` skill installed and initialized (wallet linked). Install with:
  ```bash
  npx skills add -s fluxa-agent-wallet -y -g FluxA-Agent-Payment/FluxA-AI-Wallet-MCP
  ```
- **Alpha Beds API Key** — Contact the Alpha Beds team to obtain a Bearer token for API access.

## Base URL

```
https://www.alpha-beds.com/api/fluxa/v1
```

## Authentication

All requests require a Bearer token:

```
Authorization: Bearer <your-api-key>
```

## Operation Types — Read vs Write

Classify every endpoint before calling it. This is critical: in this flow the **agent's own FluxA wallet spends real USDC**, so a write that runs without explicit user confirmation can cost the user money that cannot be recovered.

### Read operations — call freely as the task requires

| Endpoint | What it does |
|----------|--------------|
| `POST /intent` | Natural-language hotel search |
| `POST /rooms` | Browse rooms and rates |
| `POST /verify` | Verify real-time price and lock the rate (no commitment, no charge) |

### Write / money-moving operations — require explicit user confirmation **every time**

| Endpoint | What it does | Why it needs confirmation |
|----------|--------------|---------------------------|
| `POST /create` | Creates the booking order and generates a USDC payment link | Commits a booking; starts the 15-minute payment clock |
| `POST /create/confirm` | Captures the pre-authorized USDC payment and confirms the booking | **Irreversible server-side capture** — USDC leaves the wallet on this call |

**Where the money actually moves:** the FluxA wallet authorization happens at the `payment.paymentUrl` returned by `/create` (Step 4), handled by the FluxA Agent Wallet skill — **not** inside `/create/confirm`. `/create/confirm` (Step 5) only captures a payment that was already authorized at `paymentUrl`. Never call `/create/confirm` before that authorization completes, or it returns `paymentStatus: "pending"`.

**Never** treat a vague intent like "book it" / "just pay" / "go ahead" as sufficient confirmation for a write. See **Confirmation Rules** below.

## Booking Flow — 5 Steps

### Step 1 — Search Hotels (Natural Language)

```http
POST /intent
Content-Type: application/json
Authorization: Bearer <key>

{
  "query": "Hotel in Tokyo for 2 nights from April 20, 4-star+, breakfast included"
}
```

The AI engine extracts destination, dates, guest count, star rating, breakfast, and cancellation preferences from natural language (English and Chinese supported).

**Response (status: "complete"):**

```json
{
  "code": 10000,
  "data": {
    "status": "complete",
    "intent": {
      "destination": "Tokyo",
      "checkIn": "2026-04-20",
      "checkOut": "2026-04-22",
      "starMin": 4,
      "breakfast": true
    },
    "hotels": [
      {
        "hotelId": "Tmnbkq9BCgsa+vPjmNAnFA==",
        "hotelName": "Conrad Tokyo",
        "headline": "In Tokyo (Minato)",
        "starRating": 5,
        "ratingScore": "4.7/5",
        "ratingsDescribe": "Exceptional",
        "guestRatingCount": 948,
        "address": "1-9-1 Higashi-Shinbashi, Minato-ku",
        "price": 1148.43,
        "currency": "USD",
        "image": "https://...",
        "pricePolicyInfo": ["Free Breakfast", "Free Cancellation"]
      }
    ],
    "totalMatched": 245,
    "searchParams": { "regionId": "3593" }
  }
}
```

If `status` is `"incomplete"`, the response includes `missingConstraints` — ask the user for the missing information and re-query.

**Save:** `hotelId`, `searchParams.regionId`, `price`, `currency`.

### Step 2 — Browse Rooms

```http
POST /rooms
Content-Type: application/json
Authorization: Bearer <key>

{
  "hotelId": "<hotelId from step 1>",
  "city": "Tokyo",
  "regionId": "<regionId from step 1>",
  "checkIn": "2026-04-20",
  "checkOut": "2026-04-22",
  "adults": 2,
  "roomCount": 1,
  "checkInInfoStr": "[{\"childrenAgeList\":[],\"adults\":2,\"children\":0}]"
}
```

Returns full room list with rates, bed types, cancellation policies, and images.

**Save:** `data.roomResponeStr` (opaque token, required for step 4), plus the chosen room's `roomId` and its `roomRates[].rateId`.

The room objects live at `data.hotelInfos.hotelSearchResps[0].hotelRoomsMapResp.all[]` — each entry has `roomId` and a `roomRates[]` array (each rate has `rateId`, `offerPrice`, policy tags). Treat this path as the documented source for `roomId`/`rateId`; if the response shape differs, re-inspect rather than assuming the nesting.

### Step 3 — Verify Price

Checks real-time availability and locks the rate. Internally fetches all required booking tokens in one call.

```http
POST /verify
Content-Type: application/json
Authorization: Bearer <key>

{
  "offerId": "<hotelId>",
  "quoteId": "<roomId>",
  "ratePlanId": "<rateId>",
  "checkIn": "2026-04-20",
  "checkOut": "2026-04-22",
  "regionId": "<regionId from step 1>",
  "roomCount": 1,
  "adults": 2,
  "city": "Tokyo"
}
```

**Response fields:**

| Field | Description |
|-------|-------------|
| `data.hotelResponeStr` | Booking token (opaque, pass to step 4) |
| `data.rateVerifyStr` | Rate verification token (opaque, pass to step 4) |
| `data.serial` | Transaction serial |
| `data.roomRate.totalPrice` | Verified total price |
| `data.roomRate.currency` | Currency code |
| `data.roomRate.pricePolicyInfo` | Cancellation/breakfast policy tags |

**Present the verified price to the user and get confirmation before proceeding.**

### Step 4 — Create Order + Payment Link

Creates the booking order and generates a FluxA USDC payment link in one call.

```http
POST /create
Content-Type: application/json
Authorization: Bearer <key>

{
  "ratePlanId": "<rateId>",
  "offerId": "<hotelId>",
  "quoteId": "<roomId>",
  "serial": "<serial from step 3>",
  "hotelResponeStr": "<from step 3>",
  "roomResponeStr": "<from step 2>",
  "rateVerifyStr": "<from step 3>",
  "totalPrice": 1148.43,
  "contactName": "LAST/FIRST",
  "contactEmail": "guest@example.com",
  "contactPhone": "17710150618",
  "countryCode": "86",
  "guests": [
    { "index": 1, "text": "LAST", "value": "FIRST" }
  ],
  "checkInInfo": [
    { "adults": 2, "children": 0, "childrenAgeList": [] }
  ]
}
```

**Response:**

```json
{
  "success": true,
  "data": {
    "otaOrderId": "1204310735579074560",
    "orderState": "UNPAID",
    "remainingTime": 900,
    "payment": {
      "paymentUrl": "https://walletapi.fluxapay.xyz/authorize/...",
      "mandateId": "mandate-uuid",
      "amount": 1148.43,
      "currency": "USD",
      "expiresAt": "2026-04-20T12:15:00Z"
    }
  }
}
```

**`payment.paymentUrl`** is a FluxA authorization link. Use the FluxA Agent Wallet skill to authorize the payment:

```
The hotel booking order has been created. Total: $1148.43 USD.
Please authorize the payment: <paymentUrl>
```

Payment must be completed within **15 minutes**.

### Step 5 — Confirm Payment

After the payment is authorized via FluxA (at `paymentUrl`), capture the authorized payment to confirm the booking:

```http
POST /create/confirm
Content-Type: application/json
Authorization: Bearer <key>

{
  "otaOrderId": "<from step 4>",
  "mandateId": "<from step 4 payment.mandateId>",
  "totalPrice": 1148.43,
  "currency": "USD"
}
```

`otaOrderId` and `mandateId` identify the order and its authorized payment. `totalPrice` and `currency` must match the `payment.amount` / `payment.currency` returned by `/create` — they let the server reject capture if the amount drifted from what was authorized. Pass the `/create` values verbatim; do not recompute them.

**Success response:**

```json
{
  "success": true,
  "data": {
    "confirmed": true,
    "paymentStatus": "paid",
    "orderStatus": "ORDER_PAYED_ING",
    "transactionId": "fluxa-tx-id",
    "otaOrderId": "1204310735579074560"
  }
}
```

If the payment hasn't been authorized yet, the response returns `paymentStatus: "pending"`. Wait for the user to authorize and retry.

## Confirmation Rules

Each write operation is a **separate** confirmation. Confirming order creation does **not** authorize payment — re-confirm before paying.

### Before `POST /create` (create order)

Restate, in natural consumer language:

- Hotel name, room type, check-in / check-out dates, number of nights
- Guests and contact name/email
- Verified total price and currency (use the `/verify` total, not the search-stage price)
- Cancellation / breakfast policy returned by `/verify`

Then ask:

> 确认后我会创建订单并生成支付链接，订单需在 15 分钟内支付，超时需重新查询。是否确认创建？
> （I'll create the order and generate a payment link; the order must be paid within 15 minutes or it expires. Confirm?）

### Before `POST /create/confirm` (capture USDC payment)

**Precondition:** the FluxA wallet authorization at `paymentUrl` (from `/create`) must already be complete — otherwise this call returns `paymentStatus: "pending"`. This call **captures** the authorized payment and is irreversible. Restate:

- Order number (`otaOrderId`)
- Exact amount and currency to be paid (e.g. `$1148.43 USD`)
- That this captures the USDC payment from the connected FluxA wallet
- Payment deadline — show `payment.expiresAt` (from the `/create` response) as a local time. Do **not** display the raw `remainingTime` value: it is a static snapshot from order creation, not a live countdown.

Then ask:

> 这一步将从 FluxA 钱包以 USDC 实际支付 $xxx，支付后不可撤销。是否确认支付？
> （This pays $xxx in USDC from your FluxA wallet and cannot be undone. Confirm payment?）

### General principles

- A vague "帮我订 / 付了吧 / go ahead" is **never** enough for a write. Get a clear yes against the restated facts.
- One confirmation covers exactly one write call. Don't reuse an earlier confirmation for a later step.
- If the verified price differs from the search-stage price, surface the difference and re-confirm before creating the order.
- Confirmation prompts use natural language only — never mention internal tokens, mandate IDs, or raw API fields.

## Output & Hidden Fields

### Never show these to the user

| Field | Why |
|-------|-----|
| `hotelResponeStr`, `roomResponeStr`, `rateVerifyStr` | Opaque booking tokens |
| `serial` | Internal transaction serial |
| `mandateId` | FluxA payment mandate ID |
| Encrypted `hotelId`, `roomId`, `rateId` | Opaque internal IDs |
| `searchParams`, `regionId`, `checkInInfoStr` | Internal search/booking parameters |
| API key / Bearer token | Credential |
| Raw request/response JSON | Internal payload |

### Safe to show

Hotel name / star / rating / address; room name, bed type, cancellation & breakfast tags; prices and currency (USD); order number (`otaOrderId`); order state and payment status; payment deadline; the payment transaction reference (`transactionId` from `/create/confirm` — show it as the payment receipt the user can use for on-chain lookup or disputes); the FluxA `paymentUrl` (the user needs it to authorize payment — treat it as a **sensitive one-time authorization link**: show it only to the requesting user, and do not log it, repeat it, or include it in summaries).

### Output rules

- Default to the user's language (Simplified Chinese or English); keep code identifiers in English only when necessary.
- Summarize tool results — never paste raw JSON or internal IDs.
- Prices are in **USD**; show them as returned. Do not invent prices, policies, or deadlines.
- If a field (policy, deadline, fee, status) is not returned, say it was not returned — do not fabricate.
- Show the order number on its own line so it is cleanly copyable.

## Pre-Action Checklist

Before calling any endpoint, verify:

1. **Operation type** — is this a read or a write? (See Operation Types.)
2. **Confirmation** — for a write, has the user explicitly confirmed against restated facts?
3. **Hidden fields** — will my reply leak any opaque token, mandate ID, or credential?
4. **No invention** — am I about to state a price/policy/deadline the tool did not return?
5. **Token chaining** — do I have the exact tokens from the prior step (`roomResponeStr` from `/rooms`; `serial` + `hotelResponeStr` + `rateVerifyStr` from `/verify`; `otaOrderId` + `mandateId` from `/create`)?

## Error Handling

| Code | Meaning | Action |
|------|---------|--------|
| 10000 | Success | — |
| 10002 | Invalid parameters | Check required fields |
| 10020 | AI intent extraction failed | Rephrase the query |
| 10021 | Hotel search failed | Retry or change destination |
| 402 | Payment not authorized | Wait for user to authorize via FluxA wallet, then retry `/create/confirm` |
| 401 / 403 | Auth / API key invalid (if returned at HTTP level; auth failures may also surface as app code `10002`) | Treat as an invalid key — tell the user to obtain or renew one from the Alpha Beds team. **Never ask them to paste a key into the chat.** |

**Error output rules:**

- Never expose stack traces, raw error JSON, tokens, signatures, or the API key in user-facing text.
- On a write failure, do not blindly retry — re-check order/payment status first when relevant (an order may already exist or already be paid).
- **Idempotency:** if `/create` times out or errors, do **not** call it again before checking whether an order was already created — a blind retry can produce a duplicate order/hold. If you do need to re-create, re-confirm with the user first.
- **Never retry `/create/confirm` blindly** either — re-check `paymentStatus` first; the payment may already be captured.
- On a service/config/auth failure, do not ask the user to re-send guest data they already provided.

## Quick Reference (Python)

```python
import requests

BASE = "https://www.alpha-beds.com/api/fluxa/v1"
HEADERS = {"Authorization": "Bearer <key>", "Content-Type": "application/json"}

def post(path, body):
    return requests.post(f"{BASE}{path}", json=body, headers=HEADERS).json()

# 1. Search
r = post("/intent", {"query": "Tokyo hotel 2 nights from Apr 20, 4-star, breakfast"})
hotel = r["data"]["hotels"][0]
region = r["data"]["searchParams"]["regionId"]

# 2. Rooms
rooms = post("/rooms", {
    "hotelId": hotel["hotelId"], "checkIn": "2026-04-20", "checkOut": "2026-04-22",
    "regionId": region, "adults": 2, "roomCount": 1,
    "checkInInfoStr": '[{"adults":2,"children":0,"childrenAgeList":[]}]'
})
room_token = rooms["data"]["roomResponeStr"]
room = rooms["data"]["hotelInfos"]["hotelSearchResps"][0]["hotelRoomsMapResp"]["all"][0]
rate = room["roomRates"][0]

# 3. Verify
v = post("/verify", {
    "offerId": hotel["hotelId"], "quoteId": room["roomId"], "ratePlanId": rate["rateId"],
    "checkIn": "2026-04-20", "checkOut": "2026-04-22",
    "regionId": region, "roomCount": 1, "adults": 2, "city": "Tokyo"
})

# 4. Create order + payment link
order = post("/create", {
    "ratePlanId": rate["rateId"], "offerId": hotel["hotelId"], "quoteId": room["roomId"],
    "serial": v["data"]["serial"],
    "hotelResponeStr": v["data"]["hotelResponeStr"],
    "roomResponeStr": room_token,
    "rateVerifyStr": v["data"]["rateVerifyStr"],
    "totalPrice": v["data"]["roomRate"]["totalPrice"],   # /verify-locked total, not the stale search/room price
    "contactName": "ZHOU/MARK", "contactEmail": "guest@example.com",
    "guests": [{"index": 1, "text": "ZHOU", "value": "MARK"}],
    "checkInInfo": [{"adults": 2, "children": 0, "childrenAgeList": []}]
})
payment_url = order["data"]["payment"]["paymentUrl"]
# → User authorizes payment at payment_url via FluxA wallet (one-time auth link — show only to the user, don't log/forward)

# 5. Confirm after payment — pass the /create-authorized amount verbatim, never recompute
confirm = post("/create/confirm", {
    "otaOrderId": order["data"]["otaOrderId"],
    "mandateId": order["data"]["payment"]["mandateId"],
    "totalPrice": order["data"]["payment"]["amount"],
    "currency": order["data"]["payment"]["currency"]
})
```

## Important Notes

- All prices are in **USD**
- Hotel IDs, room IDs are **encrypted** — treat as opaque strings
- `hotelResponeStr`, `roomResponeStr`, `rateVerifyStr` are **opaque booking tokens** — do not parse or modify
- Payment window is **15 minutes** from order creation
- Guest names must be in **LAST/FIRST** format (Latin characters)
- The `/intent` endpoint supports natural language in **English and Chinese**
- `/create` and `/create/confirm` are **write operations** — confirm explicitly with the user before each (see Confirmation Rules)
- `/create/confirm` **captures** the pre-authorized USDC from the FluxA wallet — the capture is **irreversible** and requires explicit user confirmation (see Confirmation Rules)
- Never reveal `mandateId`, `serial`, `*ResponeStr` tokens, encrypted IDs, or the API key to the user (see Output & Hidden Fields)
