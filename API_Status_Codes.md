# API Status Codes & Error Reference

> This document describes all possible error cases when calling the 6789TRX API.
> Every error response follows a unified structure with an `errorCode` for client-side handling.

---

## Error Response Structure

```json
{
  "success": false,
  "data": null,
  "errorCode": "ERROR_CODE",
  "message": "Detailed error description"
}
```

---

## HTTP Status Code Summary

| HTTP Code | Meaning | When it occurs |
|-----------|---------|----------------|
| `200` | Success | Request was processed successfully |
| `400` | Bad Request | Request is missing fields or has an invalid format |
| `401` | Unauthorized | API Key is missing, invalid, or expired |
| `404` | Not Found | Reclaim: the wallet has no delegation from your API Key |
| `409` | Conflict | Reclaim: the wallet is shared with another contract, or a reclaim is already in progress |
| `422` | Unprocessable Entity | Request is valid but cannot be executed (quota exhausted, contract expired...) |
| `429` | Too Many Requests | More than 15 requests/second from your IP, or IP blocked |
| `500` | Internal Server Error | Internal system error, or the TRON delegate backend failed |

---

## Error Code Details

### 🔴 400 — Bad Request

---

#### `WALLET_REQUIRED`
```json
{
  "success": false,
  "data": null,
  "errorCode": "WALLET_REQUIRED",
  "message": "WalletAddress is required."
}
```
**Cause:** The `walletAddress` field is empty or missing from the request body.  
**Resolution:** Check the request body and ensure `walletAddress` is not empty.

---

#### `INVALID_ENERGY_AMOUNT`
```json
{
  "success": false,
  "data": null,
  "errorCode": "INVALID_ENERGY_AMOUNT",
  "message": "EnergyAmount must be between 65000 and 1690000."
}
```
**Endpoint:** `POST /api/v1/resources/custom-energy` only.  
**Cause:** `energyAmount` is missing, ≤ 0, or outside the allowed range of **65,000 – 1,690,000** Energy per request.  
**Resolution:** Send an `energyAmount` within the limits. No command is deducted.

---

#### `BAD_REQUEST`
```json
{
  "success": false,
  "data": null,
  "errorCode": "BAD_REQUEST",
  "message": "..."
}
```
**Cause:** The request body is not valid JSON, or a field value is invalid.  
**Resolution:** Verify `Content-Type: application/json` and the format of the request body.

---

### 🔴 401 — Unauthorized

---

#### `MISSING_AUTH`
```json
{
  "success": false,
  "data": null,
  "errorCode": "MISSING_AUTH",
  "message": "Authorization header with Bearer token is required"
}
```
**Cause:** The request does not include an `Authorization` or `X-API-Key` header.  
**Resolution:** Add the header `Authorization: Bearer {api_key}` to every request.

---

#### `INVALID_API_KEY`
```json
{
  "success": false,
  "data": null,
  "errorCode": "INVALID_API_KEY",
  "message": "Invalid or expired API key"
}
```
**Cause:** The API Key does not exist in the system, has been deactivated, or has expired.  
**Resolution:**
- Verify the API Key is correct (copy/paste again from the original source)
- The API Key may have been deactivated by an admin — contact support
- The key may have exceeded its `ExpiryDays`

---

### 🔴 404 — Not Found

---

#### `WALLET_NOT_FOUND`
```json
{
  "success": false,
  "data": null,
  "errorCode": "WALLET_NOT_FOUND",
  "message": "This wallet has no delegation from your API key."
}
```
**Endpoint:** `POST /api/v1/resources/reclaim` only.  
**Cause:** The wallet never received Energy from your API Key, so you cannot reclaim it.  
**Resolution:** Check the wallet address. Do not retry.

---

### 🔴 409 — Conflict

---

#### `WALLET_SHARED`
```json
{
  "success": false,
  "data": null,
  "errorCode": "WALLET_SHARED",
  "message": "This wallet has active delegations from another contract and cannot be reclaimed via your API key."
}
```
**Endpoint:** `POST /api/v1/resources/reclaim` only.  
**Cause:** The wallet also has active delegations from another contract. A reclaim takes back all of our Energy on the wallet, so it is refused to protect the other contract.  
**Resolution:** Wait until those delegations finish (at most 5 minutes), or contact support.

---

#### `RECLAIM_IN_PROGRESS`
```json
{
  "success": false,
  "data": null,
  "errorCode": "RECLAIM_IN_PROGRESS",
  "message": "A reclaim for this wallet is already in progress. Please retry in a few seconds."
}
```
**Endpoint:** `POST /api/v1/resources/reclaim` only.  
**Cause:** Another reclaim of this wallet (for example our automatic reclaim) is running right now.  
**Resolution:** Retry after 2–3 seconds. The next call usually returns `200` with `reclaimedRecords: 0` (already reclaimed).

---

### 🔴 422 — Unprocessable Entity

> These errors occur when the API Key is valid but the delegate command cannot be executed due to business logic constraints.

---

#### `CONTRACT_NOT_FOUND`
```json
{
  "success": false,
  "data": null,
  "errorCode": "CONTRACT_NOT_FOUND",
  "message": "No active contract found for this API key."
}
```
**Cause:** The API Key has not been assigned to any Contract in the system.  
**Resolution:** Contact the admin to configure a Contract for this API Key.

---

#### `CONTRACT_EXPIRED`
```json
{
  "success": false,
  "data": null,
  "errorCode": "CONTRACT_EXPIRED",
  "message": "Your contract has expired. Please contact support."
}
```
**Cause:** The Contract has passed its expiration date.  
**Resolution:** Contact the admin to renew or create a new Contract.

---

#### `INSUFFICIENT_COMMANDS`
```json
{
  "success": false,
  "data": null,
  "errorCode": "INSUFFICIENT_COMMANDS",
  "message": "Insufficient commands remaining in your contract."
}
```
**Cause:** The Contract has used up all its allocated commands.
- Single: costs 1 command
- Double: costs 2 commands
- Custom: costs `ceil(energyAmount / 65 000)` commands (e.g. 200,000 Energy = 4 commands)  

**Resolution:** Contact the admin to top up commands for the Contract.

---

### 🔴 429 — Too Many Requests

---

> There is **no hourly or daily request limit** per API Key — only the per-IP rate below.

---

#### `IP_BLOCKED`
```json
{
  "success": false,
  "data": null,
  "errorCode": "IP_BLOCKED",
  "message": "Your IP has been blocked. Contact support if this is a mistake."
}
```
**Cause:** Your IP was automatically blocked by the system due to abnormal behavior (repeatedly exceeding 15 requests/second, too many failed auth attempts).  
**Response headers included:** `Retry-After: 3600`  
**Resolution:** Contact support with your IP address to get it unblocked.

---

#### `TOO_MANY_REQUESTS`
```json
{
  "success": false,
  "data": null,
  "errorCode": "TOO_MANY_REQUESTS",
  "message": "Too many requests. Your IP will be temporarily blocked."
}
```
**Cause:** The IP exceeded 15 requests/second — the DDoS detection threshold.  
**Response headers included:** `Retry-After: 60`  
**Resolution:** Reduce your API call frequency. The IP is then blocked: **1 minute** for the first violation, **1 hour** for the second, **permanently** from the third (violations are remembered for 30 days).

---

### 🔴 500 — Internal Server Error

---

#### `EXTERNAL_ERROR`
```json
{
  "success": false,
  "data": null,
  "errorCode": "EXTERNAL_ERROR",
  "message": "External delegate failed: ..."
}
```
**Cause:** The backend delegate system (TRON node) rejected or did not respond.  
**Resolution:**
- Retry after a few seconds
- If the error persists (> 5 minutes), contact support with the `requestId`
- **Note:** Commands are **NOT deducted** when `EXTERNAL_ERROR` occurs

---

#### `RECLAIM_NOT_CONFIRMED`
```json
{
  "success": false,
  "data": null,
  "errorCode": "RECLAIM_NOT_CONFIRMED",
  "message": "Reclaim was sent but not yet confirmed on-chain (65000 Energy still delegated). Please retry."
}
```
**Endpoint:** `POST /api/v1/resources/reclaim` only.  
**Cause:** The reclaim was sent, but after ~10 seconds the TRON network still shows our Energy delegated to the wallet.  
**Resolution:**
- Retry the same call after a few seconds — it is safe to repeat
- **Do not** delegate from another provider until you receive `200`

---

#### `INTERNAL_ERROR`
```json
{
  "success": false,
  "data": null,
  "errorCode": "INTERNAL_ERROR",
  "message": "An unexpected error occurred"
}
```
**Cause:** An unexpected internal error (database timeout, unhandled exception...).  
**Resolution:**
- Retry after 5–10 seconds
- If the error persists, contact support with the `requestId`
- **Commands are NOT deducted** when `INTERNAL_ERROR` occurs

---

## Client-Side Error Handling Flowchart

```
Call API
  │
  ├─ 200 ──→ Execute the TRON transaction within 5 minutes ✅
  │
  ├─ 400 ──→ Fix the request, do not retry (incl. INVALID_ENERGY_AMOUNT)
  │
  ├─ 401 ──→ Check API Key
  │            ├─ MISSING_AUTH     → Add Authorization header
  │            └─ INVALID_API_KEY  → Contact admin
  │
  ├─ 404 ──→ Reclaim: WALLET_NOT_FOUND → Check the wallet address
  │
  ├─ 409 ──→ Reclaim
  │            ├─ WALLET_SHARED       → Wait or contact support
  │            └─ RECLAIM_IN_PROGRESS → Retry after 2–3s
  │
  ├─ 422 ──→ Cannot be processed
  │            ├─ CONTRACT_NOT_FOUND    → Contact admin
  │            ├─ CONTRACT_EXPIRED     → Contact admin to renew
  │            └─ INSUFFICIENT_COMMANDS → Contact admin to top up
  │
  ├─ 429 ──→ Too many requests
  │            ├─ TOO_MANY_REQUESTS → Slow down, wait per Retry-After header
  │            └─ IP_BLOCKED        → Contact support
  │
  └─ 500 ──→ Retry, then contact support
               ├─ RECLAIM_NOT_CONFIRMED → Retry the reclaim after a few seconds
               ├─ EXTERNAL_ERROR → Retry after 5s (max 3 attempts)
               └─ INTERNAL_ERROR → Retry after 10s (max 3 attempts)
```

---

## Retry Notes

| Error Code | Should Retry? | Wait Time |
|------------|--------------|-----------|
| `WALLET_REQUIRED` | ❌ No | — |
| `INVALID_ENERGY_AMOUNT` | ❌ No | — |
| `BAD_REQUEST` | ❌ No | — |
| `MISSING_AUTH` | ❌ No | — |
| `INVALID_API_KEY` | ❌ No | — |
| `CONTRACT_NOT_FOUND` | ❌ No | — |
| `CONTRACT_EXPIRED` | ❌ No | — |
| `INSUFFICIENT_COMMANDS` | ❌ No | — |
| `WALLET_NOT_FOUND` | ❌ No | — |
| `WALLET_SHARED` | ❌ No (wait or contact support) | — |
| `RECLAIM_IN_PROGRESS` | ✅ Yes | 2–3s |
| `RECLAIM_NOT_CONFIRMED` | ✅ Yes | 3–5s, until `200` |
| `EXTERNAL_ERROR` | ✅ Yes | 5s, max 3 attempts |
| `TOO_MANY_REQUESTS` | ✅ Yes | 60s |
| `IP_BLOCKED` | ❌ No (contact support) | — |
| `INTERNAL_ERROR` | ✅ Yes | 10s, max 3 attempts |

---

## Important Notes on Billing

| Scenario | Command deducted? |
|----------|-------------------|
| Successful request (200) | ✅ `data.commandsCharged` commands deducted (Single 1, Double 2, Custom `ceil(energyAmount / 65 000)`) |
| `EXTERNAL_ERROR` | ❌ Not deducted |
| `INTERNAL_ERROR` | ❌ Not deducted |
| `CONTRACT_*` | ❌ Not deducted |
| `TOO_MANY_REQUESTS` / `IP_BLOCKED` | ❌ Not deducted |
| `401` / `400` (incl. `INVALID_ENERGY_AMOUNT`) | ❌ Not deducted |
| `POST /reclaim` (any result) | ❌ Not deducted — and commands already charged for the reclaimed delegations are not refunded |
