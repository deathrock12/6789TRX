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
| `422` | Unprocessable Entity | Request is valid but cannot be executed (quota exhausted, contract expired...) |
| `429` | Too Many Requests | Rate limit exceeded |
| `500` | Internal Server Error | Internal system error |

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

**Resolution:** Contact the admin to top up commands for the Contract.

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

### 🔴 429 — Too Many Requests

---

#### `RATE_LIMIT_EXCEEDED`
```json
{
  "success": false,
  "data": null,
  "errorCode": "RATE_LIMIT_EXCEEDED",
  "message": "Rate limit exceeded. Please try again later."
}
```
**Cause:** The API Key has exceeded the allowed number of commands per hour.  
**Response headers included:**
```http
X-RateLimit-Limit: 1000
X-RateLimit-Reset: 2026-04-12T02:00:00.000Z
Retry-After: 1847
```
**Resolution:** Wait until `X-RateLimit-Reset` or after `Retry-After` seconds.

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
**Cause:** Your IP was automatically blocked by the system due to abnormal behavior (too many requests in a short time, too many failed auth attempts).  
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
**Resolution:** Reduce your API call frequency. The IP will be automatically unblocked after 1–60 minutes depending on the number of violations.

---

### 🔴 500 — Internal Server Error

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
  ├─ 400 ──→ Fix the request, do not retry
  │
  ├─ 401 ──→ Check API Key
  │            ├─ MISSING_AUTH     → Add Authorization header
  │            └─ INVALID_API_KEY  → Contact admin
  │
  ├─ 422 ──→ Cannot be processed
  │            ├─ CONTRACT_NOT_FOUND    → Contact admin
  │            ├─ CONTRACT_EXPIRED     → Contact admin to renew
  │            ├─ INSUFFICIENT_COMMANDS → Contact admin to top up
  │            └─ EXTERNAL_ERROR       → Retry after 5s (max 3 attempts)
  │
  ├─ 429 ──→ Wait per Retry-After header then retry
  │
  └─ 500 ──→ Retry after 10s (max 3 attempts) → Contact support
```

---

## Retry Notes

| Error Code | Should Retry? | Wait Time |
|------------|--------------|-----------|
| `WALLET_REQUIRED` | ❌ No | — |
| `BAD_REQUEST` | ❌ No | — |
| `MISSING_AUTH` | ❌ No | — |
| `INVALID_API_KEY` | ❌ No | — |
| `CONTRACT_NOT_FOUND` | ❌ No | — |
| `CONTRACT_EXPIRED` | ❌ No | — |
| `INSUFFICIENT_COMMANDS` | ❌ No | — |
| `EXTERNAL_ERROR` | ✅ Yes | 5s, max 3 attempts |
| `RATE_LIMIT_EXCEEDED` | ✅ Yes | Per `Retry-After` |
| `TOO_MANY_REQUESTS` | ✅ Yes | 60s |
| `IP_BLOCKED` | ❌ No (contact support) | — |
| `INTERNAL_ERROR` | ✅ Yes | 10s, max 3 attempts |

---

## Important Notes on Billing

| Scenario | Command deducted? |
|----------|-------------------|
| Successful request (200) | ✅ 1 or 2 commands deducted |
| `EXTERNAL_ERROR` | ❌ Not deducted |
| `INTERNAL_ERROR` | ❌ Not deducted |
| `CONTRACT_*` | ❌ Not deducted |
| `RATE_LIMIT_EXCEEDED` | ❌ Not deducted |
| `401` / `400` | ❌ Not deducted |
