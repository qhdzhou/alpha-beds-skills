---
name: alpha-beds-hotel-fluxa
description: >-
  Alpha Beds Hotel Booking via FluxA — AI agents can search hotels worldwide with
  natural language, browse rooms, verify real-time rates, create orders, and pay with
  USDC via FluxA Agent Wallet. Use this skill when the user asks to find, compare,
  or book hotels and wants to pay with crypto (USDC on Base).
---

# Alpha Beds Hotel Booking (FluxA Payment)

**Skill version: 1.0.0**

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

**Save:** `data.roomResponeStr` (opaque token, required for step 4), chosen room's `roomId` and `roomRates[].rateId`.

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

After the payment is authorized via FluxA, execute the payment and confirm the booking:

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

## Error Handling

| Code | Meaning | Action |
|------|---------|--------|
| 10000 | Success | — |
| 10002 | Invalid parameters | Check required fields |
| 10020 | AI intent extraction failed | Rephrase the query |
| 10021 | Hotel search failed | Retry or change destination |
| 402 | Payment not authorized | Wait for user to authorize via FluxA |

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
    "totalPrice": rate["offerPrice"],
    "contactName": "ZHOU/MARK", "contactEmail": "guest@example.com",
    "guests": [{"index": 1, "text": "ZHOU", "value": "MARK"}],
    "checkInInfo": [{"adults": 2, "children": 0, "childrenAgeList": []}]
})
payment_url = order["data"]["payment"]["paymentUrl"]
# → User authorizes payment at payment_url via FluxA wallet

# 5. Confirm after payment
confirm = post("/create/confirm", {
    "otaOrderId": order["data"]["otaOrderId"],
    "mandateId": order["data"]["payment"]["mandateId"],
    "totalPrice": rate["offerPrice"], "currency": "USD"
})
```

## Important Notes

- All prices are in **USD**
- Hotel IDs, room IDs are **encrypted** — treat as opaque strings
- `hotelResponeStr`, `roomResponeStr`, `rateVerifyStr` are **opaque booking tokens** — do not parse or modify
- Payment window is **15 minutes** from order creation
- Guest names must be in **LAST/FIRST** format (Latin characters)
- The `/intent` endpoint supports natural language in **English and Chinese**
