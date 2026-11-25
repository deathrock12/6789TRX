# API Integration Guide — TRON Network Fee Reduction Service

> **Version:** 1.6.1 
> **Base URL:** `https://enterprise-api.watchlist.win`  
> **Last Updated:** November 2026

> 🆕 **What's new in 1.6.1:** New endpoint [`GET /api/v1/resources/balance`](#45-balance--history): commands left and your recent request history, with cursor pagination.  
> 📋 Full version history: [CHANGELOG.md](./CHANGELOG.md)

---

## 1. Introduction

### 1.1 System Overview

We offer an **Energy rental service on the TRON network** — significantly reducing TRC-20 transaction costs.

**Our** acts as a liquidity pool that holds a large amount of frozen TRX, generating Energy that is delegated to your wallet on demand. You only pay a small service fee per transaction instead of burning TRX yourself.

#### Why do you need Energy?

The TRON network charges transaction fees based on 2 types of resources:

| Resource | Used for | If insufficient |
|----------|----------|-----------------|
| **Bandwidth** | All transactions | Costs 0.1 - 2 TRX |
| **Energy** | Smart contracts (TRC-20 transfer, swap...) | Costs 13–65 TRX/transaction |

To obtain Energy, users typically need to **freeze TRX** (lock capital) or **burn TRX** directly. Both approaches are costly.

#### Our solution

Instead of acquiring Energy yourself, you **rent it temporarily from our pool**:

```
You call the API
    ↓
The system delegates Energy to your wallet (~0.3s)
    ↓
You execute the TRON transaction at near-zero cost
    ↓
Once used, or at the latest after 3 minutes → Energy is automatically reclaimed to the pool
```

**Savings:** Instead of burning 13–65 TRX per transaction, you pay a much lower service fee.

---

### 1.2 Order Types

| Type | Endpoint | Energy Delegated | Commands charged | Use when |
|------|----------|------------------|------------------|----------|
| **Single** | `POST /api/v1/resources/single` | Base level **~65,000 Energy** | 1 | Transferring USDT/TRC-20 tokens to a **personal wallet with an existing balance > 0** |
| **Double** | `POST /api/v1/resources/double` | Double amount **~135,000 Energy** | 2 | Transferring USDT/TRC-20 tokens to an **exchange wallet** or any **wallet with a balance of 0** (first-time activation requires more Energy) |
| **Custom** | `POST /api/v1/resources/custom-energy` | **Exactly the amount you request** (`energyAmount`) | `energyAmount / 65,000`, rounded **up** to 0.01 (e.g. 200,000 → 3.08) | A single transaction that needs a specific amount of Energy (e.g. a heavy smart contract call, swap) |

> ⚠️ **Important:** Energy only exists for **3 minutes** after delegation.  
> If not used within 3 minutes, the Energy is reclaimed and **the order is still charged**.

---

### 1.3 Base URL

```
https://enterprise-api.watchlist.win
```

---

## 2. Authentication

### Authentication Type
The API uses an API Key — passed via HTTP header.

### Obtaining an API Key
To obtain an API Key, you need to purchase an API integration package. If you are unsure where to purchase it, please contact via Telegram: [@askemily](https://t.me/askemily) for support.

API Key format:
```
pas_sk_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

### How to Pass the API Key
Method 1 — Bearer Token (recommended):
```http
Authorization: Bearer pas_sk_your_key_here
```

### Security Notes

- ❌ **Do not** embed the API Key in client-side code (browser, mobile app)
- ❌ **Do not** commit the API Key to Git
- ✅ Only call the API from a **server** (backend)
- ✅ Store the API Key in an **environment variable** or secret manager

---

## 3. Common Conventions

### 3.1 Request

```http
Content-Type: application/json
Accept: application/json
Authorization: Bearer {api_key}
```

### 3.2 Response Format

All responses return JSON in a unified structure:

```json
{
  "success": true | false,
  "data": { ... } | null,
  "errorCode": null | "ERROR_CODE",
  "message": null | "Error description"
}
```

### 3.3 HTTP Status Codes

| Code | Meaning |
|------|---------|
| `200` | Success |
| `400` | Malformed request or missing field (Balance: invalid `cursor`) |
| `401` | Missing or invalid API Key |
| `404` | Reclaim: the wallet has no delegation from your API Key |
| `409` | Delegate: the wallet still holds Custom Energy that has not been reclaimed (`WALLET_HAS_ACTIVE_CUSTOM`). Reclaim: the wallet cannot be reclaimed right now (shared with another contract, or a reclaim is already in progress) |
| `422` | Valid request but cannot be processed (not enough commands left for this request, contract expired...) |
| `429` | Too many requests from your IP (> 15 requests/second) or IP blocked |
| `500` | Internal server error |
| `502` / `503` | Service temporarily unavailable (`SERVICE_UNAVAILABLE`) — retry after `Retry-After` seconds |
| `504` | Gateway timeout (`GATEWAY_TIMEOUT`) — the request **may or may not** have been processed |

---

## 4. Endpoints

### 4.1 Single Delegate

```
POST /api/v1/resources/single
```

**Description:** Delegates Energy for 1 standard TRC-20 transaction. Consumes **1 command** from the quota — requires at least **1** command remaining.

**Request Headers:**
```http
Authorization: Bearer {api_key}
Content-Type: application/json
```

**Request Body:**
```json
{
  "walletAddress": "TRxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `walletAddress` | string | ✅ | TRON wallet address to receive Energy (must start with `T`) |

**Response (200 — Success):**
```jsonc
{
  "success": true,
  "data": {
    "success": true,
    "data": {
      "requestId": "0HNQ1790C3TET:00000001",
      "externalRequestId": "4f6c3b0e9a1d7c2b8e5f0a3d6c9b2e1f4a7d0c3b6e9f2a5d8c1b4e7f0a3d6c9b",
      "timestamp": "2026-11-13T10:57:39.4835705"
    },
    "errorMessage": null,
    "requestId": "0HNQ1790C3TET:00000001",
    "processingTimeMs": 312,
    "resourceType": 1,
    "commandsCharged": 1,
    "remainingCommands": 39,
    "txHashes": [
      "4f6c3b0e9a1d7c2b8e5f0a3d6c9b2e1f4a7d0c3b6e9f2a5d8c1b4e7f0a3d6c9b"
    ],
    "unitPrice": {
      "amount": 1.7, // Example
      "currency": "TRX"
    },
    "usageCount": 0
  },
  "message": null,
  "errorCode": null,
  "traceId": "",
  "timestamp": "2026-11-13T03:57:39.7986526Z"
}
```

| Field | Description |
|-------|-------------|
| `data.requestId` | ID for tracing/debugging the request |
| `data.data.externalRequestId` | Hash of the (first) delegate transaction on TRON — include it when contacting support. In rare cases where the hash is unavailable it is an internal ID (UUID) |
| `data.data.timestamp` | Time the Energy was delegated (UTC+7) |
| `data.processingTimeMs` | Processing time (ms) |
| `data.resourceType` | `1` = Single, `2` = Double, `4` = Custom |
| `data.commandsCharged` | Commands deducted by **this** request (Single = 1, Double = 2, Custom = see [4.3](#43-custom-energy)). Number, up to 2 decimals |
| `data.remainingCommands` | Commands remaining in your contract **after** this request. Number, up to 2 decimals (e.g. `39`, `33.92`) |
| `data.txHashes` | Hashes of the Energy delegate transactions on TRON — look them up on [Tronscan](https://tronscan.org). Usually **1** hash; a large request may be split into **several** delegate transactions, one hash each. Empty `[]` if the hashes could not be retrieved (the Energy may still have been delegated — check the wallet's resources) |
| `data.unitPrice.amount` | Unit price per command as stated in your contract (rounded to 2 decimals) |
| `data.unitPrice.currency` | Currency of the unit price, as stated in your contract (`TRX` or `USDT`) |

> 💡 `unitPrice` is the price **per command**. A request costs `commandsCharged × unitPrice.amount` — e.g. a Double request consumes 2 commands, so it costs `2 × unitPrice.amount`.  
> The price is returned in your contract's own currency — it is **not** converted.

> ⚠️ `commandsCharged` and `remainingCommands` are **decimal numbers** — whole for Single/Double (`1`, `2`), fractional once you use Custom (`3.08`, `33.92`). Parse them as decimal/float, **not** integer.

---

### 4.2 Double Delegate

```
POST /api/v1/resources/double
```

**Description:** Delegates double the Energy for **one** transaction that needs more Energy than a standard transfer — e.g. sending USDT to an exchange wallet or to a wallet with a zero balance. Consumes **2 commands** from the quota — requires at least **2** commands remaining.

> ⚠️ Double is **not** meant for 2 separate transactions: once the Energy left on the wallet drops below the Double level (after your first transaction), we reclaim the rest automatically, so a second transaction would burn TRX. For 2 transactions, call Single twice — one call per transaction.

**Request Headers:**
```http
Authorization: Bearer {api_key}
Content-Type: application/json
```

**Request Body:**
```json
{
  "walletAddress": "TRxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
}
```

**Response (200 — Success):**
```jsonc
{
  "success": true,
  "data": {
    "success": true,
    "data": {
      "requestId": "0HNQ1790C3TET:00000002",
      "externalRequestId": "a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90",
      "timestamp": "2026-11-13T10:58:12.1020345"
    },
    "errorMessage": null,
    "requestId": "0HNQ1790C3TET:00000002",
    "processingTimeMs": 287,
    "resourceType": 2,
    "commandsCharged": 2,
    "remainingCommands": 37,
    "txHashes": [
      "a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90"
    ],
    "unitPrice": {
      "amount": 1.7, // Example
      "currency": "TRX"
    },
    "usageCount": 0
  },
  "message": null,
  "errorCode": null,
  "traceId": "",
  "timestamp": "2026-11-13T03:58:12.4150218Z"
}
```

The response fields are the same as [Single Delegate](#41-single-delegate).

---

### 4.3 Custom Energy

```
POST /api/v1/resources/custom-energy
```

**Description:** Delegates **exactly the amount of Energy you specify** — for one transaction that needs more (or a specific amount of) Energy than Single/Double, such as a heavy smart contract interaction.

**Commands charged:** `ceil(energyAmount × 100 / 65,000) / 100` — i.e. `energyAmount / 65,000` rounded **up** to 2 decimals. You pay only for the Energy you request, and you receive **exactly** that `energyAmount`.

| `energyAmount` requested | Energy delegated | Commands charged |
|---|---|---|
| 65,000 (min) | 65,000 | 1 |
| 100,000 | 100,000 | 1.54 |
| 130,000 | 130,000 | 2 |
| 200,000 | 200,000 | 3.08 |
| 1,000,000 | 1,000,000 | 15.39 |
| 1,690,000 (max) | 1,690,000 | 26 |

**Remaining commands:** a Custom request is accepted as long as `remainingCommands` covers its charge — e.g. with 1.50 commands left you can still request up to 97,500 Energy (1.50), while Double is rejected with 422 `INSUFFICIENT_COMMANDS`.

**Reclaim:** a Custom delegation is reclaimed as soon as **90%** of its Energy has been used, or at the latest **3 minutes** after delegation.

**One active Custom per wallet:** Custom Energy does not stack. While a wallet still holds Custom Energy that has not been reclaimed, **any** new request for that wallet — Single, Double or Custom, from any API key — is rejected with 409 `WALLET_HAS_ACTIVE_CUSTOM` and **no command is deducted**. To send more Energy to the same wallet, either use up the current Energy (it is then reclaimed automatically), call [`POST /api/v1/resources/reclaim`](#44-reclaim-on-demand) for the wallet (unused Energy is not refunded), or wait up to 3 minutes. Requesting the full amount you need in **one** Custom call (up to 1,690,000) is the cheapest option. Single/Double before a Custom request are not affected.

**Limits:** `energyAmount` must be between **65,000** and **1,690,000** (65,000 × 26) Energy per request. Requests outside this range are rejected with 400 `INVALID_ENERGY_AMOUNT` and **no command is deducted**.

**Bandwidth:** if your contract includes Bandwidth coverage, a Custom request receives the same Bandwidth as a Single request — enough for **1 USDT (TRC-20) transfer** (~350 Bandwidth). Use the delegated Energy in one transaction. Additional transactions on the same wallet, or a single transaction larger than a standard USDT transfer (batch/multi-send, contract deployment, multi-signature...), will burn TRX for Bandwidth at your own cost.

**Request Headers:**
```http
Authorization: Bearer {api_key}
Content-Type: application/json
```

**Request Body:**
```json
{
  "walletAddress": "TRxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
  "energyAmount": 200000
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `walletAddress` | string | ✅ | TRON wallet address to receive Energy (must start with `T`) |
| `energyAmount` | integer | ✅ | Amount of Energy to delegate, from **65,000** to **1,690,000** |

**Response (200 — Success):**
```jsonc
{
  "success": true,
  "data": {
    "success": true,
    "data": {
      "requestId": "0HNQ1790C3TET:00000003",
      "externalRequestId": "9e8d7c6b5a4f3e2d1c0b9a8f7e6d5c4b3a2f1e0d9c8b7a6f5e4d3c2b1a0f9e8d",
      "timestamp": "2026-11-17T15:30:12.3450000"
    },
    "errorMessage": null,
    "requestId": "0HNQ1790C3TET:00000003",
    "processingTimeMs": 341,
    "resourceType": 4,
    "commandsCharged": 3.08,
    "energyAmount": 200000,
    "remainingCommands": 33.92,
    "txHashes": [
      "9e8d7c6b5a4f3e2d1c0b9a8f7e6d5c4b3a2f1e0d9c8b7a6f5e4d3c2b1a0f9e8d",
      "0f1e2d3c4b5a69788796a5b4c3d2e1f00f1e2d3c4b5a69788796a5b4c3d2e1f0"
    ],
    "unitPrice": {
      "amount": 1.7, // Example
      "currency": "TRX"
    },
    "usageCount": 0
  },
  "message": null,
  "errorCode": null,
  "traceId": "",
  "timestamp": "2026-11-17T08:30:12.6780000Z"
}
```

The response fields are the same as [Single Delegate](#41-single-delegate), plus:

| Field | Description |
|-------|-------------|
| `data.energyAmount` | Energy delegated by this request (only present for Custom) |
| `data.commandsCharged` | Commands deducted — in the example above `ceil(200 000 × 100 / 65 000) / 100 = 3.08`, costing `3.08 × unitPrice.amount` |
| `data.txHashes` | In the example above the 200,000 Energy was delivered in **2** delegate transactions. The total delegated is always `energyAmount`, however many transactions it takes |

### 4.4 Reclaim (on demand)

```
POST /api/v1/resources/reclaim
```

**Description:** Takes back **right now** the Energy (and Bandwidth, if your contract includes it) that we delegated to a wallet, and **returns only after the reclaim is confirmed**. Normally we reclaim automatically — as soon as the delegated Energy has been used, or at the latest 3 minutes after delegation. Call this endpoint when you need to control the exact moment, e.g. **before delegating Energy to the same wallet from another provider**, so our reclaim can never collide with your other delegation.

**How it works:**
1. We reclaim all Energy (and Bandwidth) we delegated to the wallet for your contract.
2. We re-check the TRON network until **none of our Energy remains delegated** to the wallet (usually 1–5 seconds, up to ~10 seconds).
3. Only then do we respond `200` — when you receive it, it is safe to delegate from another provider.

**Billing:**
- This endpoint **does not deduct any command**.
- Commands already charged for the reclaimed delegations are **not refunded**, and any unused Energy is lost. Call it only after your transaction has been **confirmed on-chain** — if the transaction is still pending, it may burn TRX for Energy/Bandwidth.

**Request:**
```http
POST /api/v1/resources/reclaim
Authorization: Bearer pas_sk_your_key_here
Content-Type: application/json

{
  "walletAddress": "TRxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `walletAddress` | string | ✅ | Wallet that received Energy from your API Key |

**Response (200 — Reclaimed and confirmed):**
```json
{
  "success": true,
  "data": {
    "walletAddress": "TRxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
    "reclaimedRecords": 2,
    "delegatedEnergyRemaining": 0,
    "bandwidthReclaimed": true,
    "reclaimedAt": "2026-11-18T10:15:42.1234567",
    "txHashes": [
      "3c2b1a0f9e8d7c6b5a4f3e2d1c0b9a8f7e6d5c4b3a2f1e0d9c8b7a6f5e4d3c2b",
      "7a6f5e4d3c2b1a0f9e8d7c6b5a4f3e2d1c0b9a8f7e6d5c4b3a2f1e0d9c8b7a6f"
    ]
  },
  "message": null,
  "errorCode": null,
  "traceId": "",
  "timestamp": "2026-11-18T03:15:42.2345678Z"
}
```

| Field | Description |
|-------|-------------|
| `data.walletAddress` | The wallet that was reclaimed |
| `data.reclaimedRecords` | Number of delegations reclaimed by this call. `0` = nothing left to reclaim (already reclaimed earlier) — still a success, so the call is safe to repeat |
| `data.delegatedEnergyRemaining` | Energy from us still delegated to the wallet — always `0` on success |
| `data.bandwidthReclaimed` | `true` if Bandwidth was reclaimed in this call |
| `data.reclaimedAt` | Time the reclaim was confirmed (UTC+7); `null` when there was nothing to reclaim |
| `data.txHashes` | Hashes of the reclaim (UnDelegateResource) transactions on TRON — the Energy reclaim first (one per source wallet that delegated to you), then the Bandwidth reclaim if any. Look them up on [Tronscan](https://tronscan.org). Empty array `[]` when there was nothing to reclaim |

**Errors specific to this endpoint:**

| errorCode | HTTP | Meaning | What to do |
|-----------|------|---------|------------|
| `WALLET_NOT_FOUND` | 404 | This wallet never received Energy from your API Key | Check the wallet address |
| `WALLET_SHARED` | 409 | The wallet also has active delegations from another contract, so it cannot be reclaimed with your key | Wait for them to finish, or contact support |
| `RECLAIM_IN_PROGRESS` | 409 | A reclaim of this wallet is already running (e.g. our automatic reclaim) | Retry after 2–3 seconds |
| `RECLAIM_NOT_CONFIRMED` | 500 | The reclaim was sent but not yet confirmed on-chain | Retry the same call after a few seconds — **do not** delegate from another provider until you get `200` |
| `EXTERNAL_ERROR` | 500 | The reclaim service did not respond | Retry after 5 seconds |

---

### 4.5 Balance & History

```
GET /api/v1/resources/balance
```

**Description:** Returns how many commands your API Key has left, together with the history of your **most recent** successful requests (newest first, up to `data.history.maxItems`, currently 1,000). The history is paginated with a **cursor**. This endpoint is read-only: it **does not deduct any command**.

**Request — first page:**
```http
GET /api/v1/resources/balance
Authorization: Bearer pas_sk_your_key_here
```

**Request — next pages:** pass the `data.history.nextCursor` value from the previous response:
```http
GET /api/v1/resources/balance?cursor=NjM5MzExNjE5NjYwMDAwMDAwOmRiYTZhMzUx
Authorization: Bearer pas_sk_your_key_here
```

| Query parameter | Type | Required | Description |
|-----------------|------|----------|-------------|
| `cursor` | string | ❌ | Omit for the first page. For the next page, send `data.history.nextCursor` from the previous response exactly as received. This is the only query parameter |

**Response (200):**
```json
{
  "success": true,
  "data": {
    "contract": {
      "status": "active",
      "effectiveDate": "2026-11-24T13:06:43Z",
      "purchasedCommands": 120,
      "usedCommands": 76,
      "remainingCommands": 44
    },
    "history": {
      "items": [
        {
          "time": "2026-11-25T02:41:08Z",
          "walletAddress": "TRxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
          "type": "custom",
          "energyAmount": 200000,
          "commands": 3.08,
          "txHash": "9f8e7d6c5b4a39281706f5e4d3c2b1a09f8e7d6c5b4a39281706f5e4d3c2b1a0"
        },
        {
          "time": "2026-11-24T13:07:12Z",
          "walletAddress": "TRxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
          "type": "double",
          "energyAmount": null,
          "commands": 2,
          "txHash": "a1f09485d6dd9fa589841518aa53da87ea908eab4ce8de6bd39c4c4d169ce463"
        }
      ],
      "limit": 50,
      "maxItems": 1000,
      "nextCursor": "NjM5MzExNjE5NjYwMDAwMDAwOmRiYTZhMzUx",
      "hasMore": true
    }
  },
  "message": null,
  "errorCode": null,
  "traceId": "",
  "timestamp": "2026-11-25T02:45:10.1234567Z"
}
```

| Field | Description |
|-------|-------------|
| `data.contract.status` | `active`, or `expired` once your contract's end date has passed |
| `data.contract.effectiveDate` | Start date of your current contract (UTC) |
| `data.contract.purchasedCommands` | Total commands purchased with this API Key, **all contracts** (current and previous) |
| `data.contract.usedCommands` | Total commands used by successful requests with this API Key, **all contracts**. Can be fractional (e.g. `15.39`) |
| `data.contract.remainingCommands` | Commands left in your **current** contract — the same quota used to accept or reject your requests (see [6](#6-rate-limit--quota)). Commands left over from a previous contract are not carried over, so this can be less than `purchasedCommands − usedCommands` |
| `data.history.items` | Successful Single/Double/Custom requests of this API Key (all contracts), **newest first**, limited to the most recent `maxItems`. Failed requests are not listed (they are not charged) |
| `items[].time` | Time of the request (UTC) |
| `items[].walletAddress` | Wallet that received the Energy |
| `items[].type` | `single`, `double` or `custom` |
| `items[].energyAmount` | Energy requested — Custom only, `null` for Single/Double |
| `items[].commands` | Commands charged for the request (Custom can be fractional, e.g. `3.08`) |
| `items[].txHash` | Hash of the delegate transaction on TRON (look it up on [Tronscan](https://tronscan.org)); `null` if not available |
| `data.history.limit` | Maximum number of items per page (set by us; may change) |
| `data.history.maxItems` | Maximum number of requests available in the history (set by us; may change). Older requests are not returned, but they are still counted in `usedCommands` |
| `data.history.hasMore` | `true` if there are older items within the most recent `maxItems` |
| `data.history.nextCursor` | Pass it as `?cursor=` to get the next (older) page; `null` on the last page |

> ⏱️ **Times:** all times in this endpoint are **UTC** (`Z` suffix).

**Paging through the whole history:**

```text
cursor = null
do {
  res    = GET /api/v1/resources/balance            (first page: no cursor)
           GET /api/v1/resources/balance?cursor={cursor}   (next pages)
  save(res.data.history.items)
  cursor = res.data.history.nextCursor
} while (res.data.history.hasMore)
```

When the loop ends you have your most recent `maxItems` requests. If you need older ones, contact support.

- **Send the cursor exactly as received.** It is already URL-safe, so no URL-encoding is needed. Do not build or change it yourself; it is an opaque value and its format may change.
- **There is no `limit` query parameter.** The page size (`data.history.limit`) is set by us and may change. Rely on `hasMore` and `nextCursor`, not on the number of items per page.
- **`data.contract` is in every page** and always shows your figures at the moment of that call.
- **New requests while you are paging:** requests you make after fetching the first page do not show up in the pages you are walking through, so no item is ever repeated. They appear at the top of a new first page. They also count toward `maxItems`, so a few of your oldest requests may drop off the end of the history while you page.
- **No history yet:** `items` is `[]`, `hasMore` is `false` and `nextCursor` is `null`.

**Errors specific to this endpoint:**

| errorCode | HTTP | Meaning | What to do |
|-----------|------|---------|------------|
| `INVALID_CURSOR` | 400 | `cursor` is not a value returned by this endpoint | Omit it to start again from the first page |
| `CONTRACT_NOT_FOUND` | 422 | Your API Key has no active contract | Contact support |

---

## 5. Error Handling

**Error structure:**
```json
{
  "success": false,
  "data": null,
  "errorCode": "CONTRACT_EXPIRED",
  "message": "Your contract has expired. Please contact support."
}
```

**Common error codes:**

| errorCode | HTTP | Cause | Resolution |
|-----------|------|-------|------------|
| `MISSING_AUTH` | 401 | No Authorization header | Add the header |
| `INVALID_API_KEY` | 401 | Wrong or disabled API Key — also returned once your contract has **ended** (all commands used) | Contact support to top up / renew |
| `WALLET_REQUIRED` | 400 | `walletAddress` is empty | Check the request |
| `INVALID_CURSOR` | 400 | Balance: `cursor` is not a value returned by `/balance` | Omit it to start from the first page |
| `INVALID_ENERGY_AMOUNT` | 400 | Custom: `energyAmount` missing or outside 65,000 – 1,690,000 | Send an amount within the limits |
| `CONTRACT_NOT_FOUND` | 422 | Key has no associated contract | Contact admin |
| `CONTRACT_EXPIRED` | 422 | Contract has expired | Contact to renew |
| `INSUFFICIENT_COMMANDS` | 422 | Not enough commands left for this request (Single needs ≥ 1, Double ≥ 2, Custom ≥ its charge) | Request a smaller Custom amount, or top up |
| `WALLET_NOT_FOUND` | 404 | Reclaim: wallet has no delegation from your API Key | Check the wallet address |
| `WALLET_SHARED` | 409 | Reclaim: wallet has active delegations from another contract | Contact support |
| `RECLAIM_IN_PROGRESS` | 409 | Reclaim: a reclaim of this wallet is already running | Retry after 2–3s |
| `WALLET_HAS_ACTIVE_CUSTOM` | 409 | Single/Double/Custom: the wallet still holds Custom Energy that has not been reclaimed | Call `/reclaim` for the wallet, or wait until it is reclaimed (≤ 3 minutes), then retry |
| `TOO_MANY_REQUESTS` | 429 | More than 15 requests/second from your IP | Slow down, wait per `Retry-After` |
| `IP_BLOCKED` | 429 | IP blocked after repeated violations | Contact support |
| `EXTERNAL_ERROR` | 500 | Temporary TRON backend error | Retry after 5s |
| `RECLAIM_NOT_CONFIRMED` | 500 | Reclaim: sent but not yet confirmed on-chain | Retry after a few seconds |
| `INTERNAL_ERROR` | 500 | System error | Retry after 10s |
| `SERVICE_UNAVAILABLE` | 502 / 503 | The service is temporarily unavailable (maintenance or failover); usually the request did not reach the API | Retry after `Retry-After` seconds (5s); for Single/Double, check the wallet's Energy first |
| `GATEWAY_TIMEOUT` | 504 | The service did not answer in time; the request **may or may not** have been processed | Check the wallet's Energy on Tronscan before retrying — a repeated Single/Double request is charged again |

> 📖 Full details for each error code: [API_Status_Codes.md](./API_Status_Codes.md)

---

## 6. Rate Limit & Quota

There is **no hourly or daily request limit** per API Key. The system has **2 independent limits**:

| Type | Limit | When exceeded |
|------|-------|---------------|
| **IP request rate** | **15 requests/second** per IP | 429 `TOO_MANY_REQUESTS`, then the IP is temporarily blocked |
| **Contract Quota** | Commands in your contract (see `remainingCommands` in the response body) | 422 `INSUFFICIENT_COMMANDS`; once fully used, the contract ends → 401 `INVALID_API_KEY` |

- **IP request rate** — exceeding 15 requests/second blocks your IP: **1 minute** for the first violation, **1 hour** for the second, and **permanently** from the third (violation history is kept for 30 days). While blocked, every request returns 429 `IP_BLOCKED`. Both 429 responses include a `Retry-After` header (seconds). If you need a high request rate, queue your calls on your side.
- **Contract Quota** — total commands in your contract (counted from `EffectiveDate`). Every successful Single/Double/Custom response returns `data.commandsCharged` and `data.remainingCommands` — the commands left **after** that request (may be fractional, e.g. `33.92`). You can also check them at any time with [`GET /api/v1/resources/balance`](#45-balance--history). When the commands left are fewer than a request needs, the API returns 422 `INSUFFICIENT_COMMANDS`. When the commands left after a request fall below the smallest possible request (**1** command = a Single request or a 65,000 Energy Custom request), the contract **ends automatically** and your API Key is deactivated: further calls return **401 `INVALID_API_KEY`** (`"API key has expired"`), not 422. Contact [@askemily](https://t.me/askemily) to top up.

---

## 7. Flow Examples

### 7.1 Basic Flow — USDT Transfer

```
1. Prepare the USDT transaction on wallet TRxxxxxxxx
        ↓
2. Call POST /api/v1/resources/single
   Body: { "walletAddress": "TRxxxxxxxx" }
        ↓
3. Receive response { success: true, requestId: "..." }
        ↓
4. Execute the USDT transaction immediately (within 3 minutes)
        ↓
5. Transaction succeeds at near-zero TRX fee ✅
```

### 7.2 Error Handling Flow — Retry

```
Call API
  ├─ 200 → Execute the transaction immediately ✅
  ├─ 429 TOO_MANY_REQUESTS → Slow down, wait per Retry-After header
  ├─ 429 IP_BLOCKED → Contact support
  ├─ 500 EXTERNAL_ERROR → Retry after 5s (max 3 attempts)
  ├─ 500 INTERNAL_ERROR → Retry after 10s (max 3 attempts)
  ├─ 502/503 SERVICE_UNAVAILABLE → Retry after Retry-After seconds (max 3 attempts); Single/Double: check the wallet's Energy first
  ├─ 504 GATEWAY_TIMEOUT → Check the wallet's Energy first, retry only if it was not delegated
  ├─ Non-JSON response (e.g. an HTML error page)
  │     ├─ returned immediately → Treat as service unavailable, retry after 5s
  │     └─ returned after a long wait (e.g. 524) → Treat as GATEWAY_TIMEOUT, check the wallet's Energy first
  └─ 401/400/422 (other) → Do not retry, review your config
```

---

## 8. Code Samples

### 8.1 Node.js / TypeScript

```typescript
const API_KEY = process.env.TRX_API_KEY!;
const BASE_URL = 'https://enterprise-api.watchlist.win';

async function delegateSingle(walletAddress: string): Promise<string> {
  const response = await fetch(`${BASE_URL}/api/v1/resources/single`, {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${API_KEY}`,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({ walletAddress }),
  });

  const result = await response.json();

  if (!result.success) {
    throw new Error(`[${result.errorCode}] ${result.message}`);
  }

  // Energy has been delegated — execute the TRON transaction within 3 minutes
  return result.data.requestId;
}

// With automatic retry
async function delegateWithRetry(walletAddress: string, maxRetries = 3): Promise<string> {
  const retryableCodes = ['EXTERNAL_ERROR', 'INTERNAL_ERROR'];

  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      return await delegateSingle(walletAddress);
    } catch (err: any) {
      const code = err.message.match(/\[(\w+)\]/)?.[1];
      if (!retryableCodes.includes(code) || attempt === maxRetries) throw err;
      const delay = code === 'EXTERNAL_ERROR' ? 5000 : 10000;
      console.log(`Retry ${attempt}/${maxRetries} after ${delay}ms...`);
      await new Promise(res => setTimeout(res, delay));
    }
  }
  throw new Error('Max retries exceeded');
}
```

---

### 8.2 Python

```python
import os
import time
import requests

API_KEY = os.environ['TRX_API_KEY']
BASE_URL = 'https://enterprise-api.watchlist.win'

HEADERS = {
    'Authorization': f'Bearer {API_KEY}',
    'Content-Type': 'application/json',
}

def delegate_single(wallet_address: str) -> dict:
    response = requests.post(
        f'{BASE_URL}/api/v1/resources/single',
        headers=HEADERS,
        json={'walletAddress': wallet_address},
        timeout=15,
    )
    result = response.json()
    if not result['success']:
        raise Exception(f"[{result['errorCode']}] {result['message']}")
    return result['data']

def delegate_double(wallet_address: str) -> dict:
    response = requests.post(
        f'{BASE_URL}/api/v1/resources/double',
        headers=HEADERS,
        json={'walletAddress': wallet_address},
        timeout=15,
    )
    result = response.json()
    if not result['success']:
        raise Exception(f"[{result['errorCode']}] {result['message']}")
    return result['data']

def delegate_with_retry(wallet_address: str, max_retries: int = 3) -> dict:
    retryable = {'EXTERNAL_ERROR', 'INTERNAL_ERROR'}
    for attempt in range(1, max_retries + 1):
        try:
            return delegate_single(wallet_address)
        except Exception as e:
            code = str(e).split(']')[0].lstrip('[')
            if code not in retryable or attempt == max_retries:
                raise
            delay = 5 if code == 'EXTERNAL_ERROR' else 10
            print(f'Retry {attempt}/{max_retries} after {delay}s...')
            time.sleep(delay)

# Usage
try:
    data = delegate_with_retry('TRxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx')
    print(f'Energy delegated! RequestId: {data["requestId"]}')
    # → Execute the TRON transaction within 3 minutes
except Exception as e:
    print(f'Failed: {e}')
```

---

### 8.3 PHP

```php
<?php
define('TRX_API_KEY', getenv('TRX_API_KEY'));
define('BASE_URL', 'https://enterprise-api.watchlist.win');

function delegateSingle(string $walletAddress): array {
    $ch = curl_init(BASE_URL . '/api/v1/resources/single');
    curl_setopt_array($ch, [
        CURLOPT_RETURNTRANSFER => true,
        CURLOPT_POST           => true,
        CURLOPT_HTTPHEADER     => [
            'Authorization: Bearer ' . TRX_API_KEY,
            'Content-Type: application/json',
        ],
        CURLOPT_POSTFIELDS     => json_encode(['walletAddress' => $walletAddress]),
        CURLOPT_TIMEOUT        => 15,
    ]);
    $body   = curl_exec($ch);
    $result = json_decode($body, true);
    curl_close($ch);

    if (!$result['success']) {
        throw new Exception("[{$result['errorCode']}] {$result['message']}");
    }
    return $result['data'];
}

function delegateWithRetry(string $walletAddress, int $maxRetries = 3): array {
    $retryable = ['EXTERNAL_ERROR', 'INTERNAL_ERROR'];
    for ($i = 1; $i <= $maxRetries; $i++) {
        try {
            return delegateSingle($walletAddress);
        } catch (Exception $e) {
            preg_match('/\[(\w+)\]/', $e->getMessage(), $m);
            $code = $m[1] ?? '';
            if (!in_array($code, $retryable) || $i === $maxRetries) throw $e;
            sleep($code === 'EXTERNAL_ERROR' ? 5 : 10);
        }
    }
}

// Usage
try {
    $data = delegateWithRetry('TRxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx');
    echo "Energy delegated! RequestId: " . $data['requestId'];
    // → Execute the TRON transaction within 3 minutes
} catch (Exception $e) {
    echo "Failed: " . $e->getMessage();
}
```

---

### 8.4 Go

```go
package main

import (
    "bytes"
    "encoding/json"
    "fmt"
    "net/http"
    "os"
    "time"
)

const baseURL = "https://enterprise-api.watchlist.win"

type DelegateRequest struct {
    WalletAddress string `json:"walletAddress"`
}

type DelegateResponse struct {
    Success   bool   `json:"success"`
    ErrorCode string `json:"errorCode"`
    Message   string `json:"message"`
    Data      *struct {
        RequestID         string `json:"requestId"`
        ProcessingTimeMs  int    `json:"processingTimeMs"`
        ResourceType      int    `json:"resourceType"`
        CommandsCharged   float64 `json:"commandsCharged"`   // decimal, e.g. 3.08
        EnergyAmount      *int64  `json:"energyAmount"`      // only present for Custom
        RemainingCommands float64 `json:"remainingCommands"` // decimal, e.g. 33.92
        TxHashes          []string `json:"txHashes"`         // delegate transaction hashes on TRON
        UnitPrice         *struct {
            Amount   float64 `json:"amount"`
            Currency string  `json:"currency"`
        } `json:"unitPrice"`
    } `json:"data"`
}

func delegateSingle(walletAddress string) (*DelegateResponse, error) {
    body, _ := json.Marshal(DelegateRequest{WalletAddress: walletAddress})

    req, _ := http.NewRequest("POST", baseURL+"/api/v1/resources/single", bytes.NewBuffer(body))
    req.Header.Set("Authorization", "Bearer "+os.Getenv("TRX_API_KEY"))
    req.Header.Set("Content-Type", "application/json")

    resp, err := (&http.Client{Timeout: 15 * time.Second}).Do(req)
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()

    var result DelegateResponse
    json.NewDecoder(resp.Body).Decode(&result)

    if !result.Success {
        return nil, fmt.Errorf("[%s] %s", result.ErrorCode, result.Message)
    }
    return &result, nil
}

func main() {
    result, err := delegateSingle("TRxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx")
    if err != nil {
        fmt.Println("Error:", err)
        return
    }
    fmt.Printf("Energy delegated! RequestId: %s\n", result.Data.RequestID)
    // → Execute the TRON transaction within 3 minutes
}
```

---

## 9. Testing

### 9.1 Health Check

```bash
curl https://enterprise-api.watchlist.win/api/v1/health
```

### 9.2 Test Single Delegate

```bash
curl -X POST https://enterprise-api.watchlist.win/api/v1/resources/single \
  -H "Authorization: Bearer pas_sk_your_key_here" \
  -H "Content-Type: application/json" \
  -d '{"walletAddress": "TRxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"}'
```

### 9.3 Test Custom Energy

```bash
curl -X POST https://enterprise-api.watchlist.win/api/v1/resources/custom-energy \
  -H "Authorization: Bearer pas_sk_your_key_here" \
  -H "Content-Type: application/json" \
  -d '{"walletAddress": "TRxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx", "energyAmount": 200000}'
```

### 9.4 Test Reclaim

```bash
curl -X POST https://enterprise-api.watchlist.win/api/v1/resources/reclaim \
  -H "Authorization: Bearer pas_sk_your_key_here" \
  -H "Content-Type: application/json" \
  -d '{"walletAddress": "TRxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"}'
```

### 9.5 Test Balance

```bash
curl https://enterprise-api.watchlist.win/api/v1/resources/balance \
  -H "Authorization: Bearer pas_sk_your_key_here"

# next page
curl "https://enterprise-api.watchlist.win/api/v1/resources/balance?cursor=NEXT_CURSOR_FROM_PREVIOUS_RESPONSE" \
  -H "Authorization: Bearer pas_sk_your_key_here"
```

### 9.6 Postman

Import the collection by creating an environment with:

| Variable | Value |
|----------|-------|
| `base_url` | `https://enterprise-api.watchlist.win` |
| `api_key` | `pas_sk_your_key_here` |

Then use `{{base_url}}` and the header `Authorization: Bearer {{api_key}}` in all requests.

---

## 10. FAQ

**Q: What happens if I call the API but don't use the Energy within 3 minutes?**  
A: The Energy is reclaimed to the pool and the command is still deducted. Make sure your TRON transaction is ready before calling the API.

**Q: I also use another Energy provider for the same wallet. How do I avoid conflicts?**  
A: Before delegating from the other provider, call [`POST /api/v1/resources/reclaim`](#44-reclaim-on-demand) for that wallet and wait for the `200` response — it is returned only after our Energy has been confirmed reclaimed on-chain. Our automatic reclaim will not touch the wallet while your reclaim call is running.

**Q: Should I use Single or Double?**  
A: Use Single for standard TRC-20 transfers (USDT, USDC...) to a wallet that already holds the token. Use Double for one transfer to an exchange wallet or to a wallet with a zero balance (it needs more Energy). Each call covers **one** transaction — for 2 transactions, call the API twice.

**Q: When should I use Custom instead of Double?**  
A: Use Custom when one transaction needs a specific amount of Energy that Single/Double do not match — e.g. 200,000 Energy for a heavy smart contract call. You are charged `energyAmount / 65,000` commands rounded up to 0.01 (200,000 Energy = 3.08 commands), shown in `data.commandsCharged`. For ordinary TRC-20 transfers, Single/Double remain the simplest choice.

**Q: Why do I get 409 `WALLET_HAS_ACTIVE_CUSTOM` when I call the API again for the same wallet?**  
A: The wallet still holds Custom Energy from an earlier request that has not been reclaimed yet. Custom Energy does not stack, so no new request (Single, Double or Custom) is accepted for that wallet until it is reclaimed — automatically once 90% is used or after at most 3 minutes, or immediately via [`POST /api/v1/resources/reclaim`](#44-reclaim-on-demand). No command is deducted for the rejected request. If one transaction needs more Energy, request the whole amount in a single Custom call.

**Q: How can I verify on-chain that the Energy was delegated?**  
A: Look up each hash in `data.txHashes` on [Tronscan](https://tronscan.org) — each is a delegate (DelegateResource) transaction to your wallet. A large request may be split into several transactions; together they add up to the Energy requested.

**Q: Is a command deducted for `EXTERNAL_ERROR`?**  
A: No. Only successful requests (HTTP 200) result in a command being deducted.

**Q: How do I know how many contract quota commands remain?**  
A: Call [`GET /api/v1/resources/balance`](#45-balance--history) at any time — `data.contract.remainingCommands` is the number of commands left, and `data.history` lists your past requests. You can also read `data.remainingCommands` from any successful Single/Double/Custom response — it is the number of commands left **after** that request. It can be fractional (e.g. `33.92`) once you use Custom. When the commands left are fewer than a request needs, you will receive a 422 `INSUFFICIENT_COMMANDS`. Once they drop below 1, the contract ends and your key returns 401 `INVALID_API_KEY`. To top up, contact [@askemily](https://t.me/askemily).

**Q: How much does each command cost?**  
A: Read `data.unitPrice` from any successful Single/Double/Custom response. It is the per-command price in your contract's currency (`TRX` or `USDT`), exactly as stated in your contract. Each request costs `data.commandsCharged × unitPrice.amount` (Single = 1 command, Double = 2, Custom = `energyAmount / 65,000` rounded up to 0.01).

**Q: I have a fractional number of commands left (e.g. 1.50). Can I still use all of it?**  
A: Yes, as long as at least 1 command is left — use Single (1 command) or a Custom request whose charge fits, e.g. 1.50 commands = up to 97,500 Energy. Double needs 2 commands. Every request needs at least 1 command, so once less than 1 command is left nothing can be requested any more and the contract ends automatically.

**Q: My IP is blocked — what should I do?**  
A: Contact support via Telegram with your IP address to get it unblocked.

---

## 11. Support

For any questions about integration, errors, or to obtain an API Key:

**Telegram:** [@askemily](https://t.me/askemily)

When contacting us, please provide:
- `requestId` from the response (if there is an error)
- `txHashes` / `externalRequestId` from the response (if Energy was not received)
- The error code encountered
- The time the error occurred (UTC+7)

---

