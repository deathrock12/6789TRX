# API Integration Guide — TRON Network Fee Reduction Service

> **Version:** 1.0  
> **Base URL:** `https://api.6789trx.com`  
> **Last Updated:** April 2026

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
After 5 minutes → Energy is automatically reclaimed to the pool
```

**Savings:** Instead of burning 13–65 TRX per transaction, you pay a much lower service fee.

---

### 1.2 Two Order Types

| Type | Endpoint | Energy Delegated | Use when |
|------|----------|------------------|----------|
| **Single** | `POST /api/v1/resources/single` | Base level **~65,000 Energy** | Transferring USDT/TRC-20 tokens to a **personal wallet with an existing balance > 0** |
| **Double** | `POST /api/v1/resources/double` | Double amount **~130,000 Energy** | Transferring USDT/TRC-20 tokens to an **exchange wallet** or any **wallet with a balance of 0** (first-time activation requires more Energy) |

> ⚠️ **Important:** Energy only exists for **5 minutes** after delegation.  
> If not used within 5 minutes, the Energy is reclaimed and **the order is still charged**.

---

### 1.3 Base URL

```
https://api.6789trx.com
```

---

## 2. Authentication

### 2.1 Authentication Type

The API uses **API Key** — passed via HTTP header.

### 2.2 Obtaining an API Key

Contact via Telegram: [@trx_savings](https://t.me/trx_savings) to receive an API Key.

API Key format:
```
pas_sk_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

### 2.3 How to Pass the API Key

**Method 1 — Bearer Token (recommended):**
```http
Authorization: Bearer pas_sk_your_key_here
```

### 2.4 Security Notes

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
| `400` | Malformed request or missing field |
| `401` | Missing or invalid API Key |
| `422` | Valid request but cannot be processed (quota exhausted, contract expired...) |
| `429` | Rate limit exceeded |
| `500` | Internal server error |

---

## 4. Endpoints

### 4.1 Single Delegate

```
POST /api/v1/resources/single
```

**Description:** Delegates Energy for 1 standard TRC-20 transaction. Consumes **1 command** from the quota.

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
```json
{
  "success": true,
  "data": {
    "success": true,
    "requestId": "0HN8K2QJ4P3VS:00000001",
    "processingTimeMs": 312,
    "resourceType": 1
  },
  "errorCode": null,
  "message": null
}
```

| Field | Description |
|-------|-------------|
| `data.requestId` | ID for tracing/debugging the request |
| `data.processingTimeMs` | Processing time (ms) |
| `data.resourceType` | `1` = Single |

---

### 4.2 Double Delegate

```
POST /api/v1/resources/double
```

**Description:** Delegates double the Energy — for 2 consecutive transactions or 1 complex contract interaction. Consumes **2 commands** from the quota.

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
```json
{
  "success": true,
  "data": {
    "success": true,
    "requestId": "0HN8K2QJ4P3VS:00000002",
    "processingTimeMs": 287,
    "resourceType": 2
  },
  "errorCode": null,
  "message": null
}
```

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
| `INVALID_API_KEY` | 401 | Wrong or disabled API Key | Contact support |
| `WALLET_REQUIRED` | 400 | `walletAddress` is empty | Check the request |
| `CONTRACT_NOT_FOUND` | 422 | Key has no associated contract | Contact admin |
| `CONTRACT_EXPIRED` | 422 | Contract has expired | Contact to renew |
| `INSUFFICIENT_COMMANDS` | 422 | Quota commands exhausted | Contact to top up |
| `EXTERNAL_ERROR` | 422 | Temporary TRON backend error | Retry after 5s |
| `RATE_LIMIT_EXCEEDED` | 429 | Hourly limit exceeded | Wait per `Retry-After` |
| `INTERNAL_ERROR` | 500 | System error | Retry after 10s |

> 📖 Full details for each error code: [API_Status_Codes.md](./API_Status_Codes.md)

---

## 6. Rate Limit

After each authenticated request, the response includes the following headers:

```http
X-RateLimit-Limit: 1000
X-RateLimit-Reset: 2026-04-12T11:00:00.000Z
Retry-After: 1847          (only present when receiving a 429)
```

| Header | Description |
|--------|-------------|
| `X-RateLimit-Limit` | Maximum number of calls allowed per hour (UTC+7) |
| `X-RateLimit-Reset` | Time when the counter resets to the next hour (ISO 8601, UTC) |
| `Retry-After` | Seconds to wait — only present when receiving a 429 |

### Important Notes on the 2 Quota Types

The system has **2 independent limits**:

| Type | Header | Resets when | When exceeded |
|------|--------|-------------|---------------|
| **Hourly Rate Limit** | *(not exposed)* | Start of each UTC+7 hour | 429 `RATE_LIMIT_EXCEEDED` |
| **Contract Quota** | *(not exposed)* | When admin tops up | 422 `INSUFFICIENT_COMMANDS` |

- **Hourly Rate Limit** — counts the number of requests in the current hour, resets automatically every hour.
- **Contract Quota** — total commands in your contract (counted from `EffectiveDate`). **No header exposes this value.** When exhausted, the API returns 422 `INSUFFICIENT_COMMANDS`. Contact [@trx_savings](https://t.me/trx_savings) to check or top up.

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
4. Execute the USDT transaction immediately (within 5 minutes)
        ↓
5. Transaction succeeds at near-zero TRX fee ✅
```

### 7.2 Error Handling Flow — Retry

```
Call API
  ├─ 200 → Execute the transaction immediately ✅
  ├─ 422 EXTERNAL_ERROR → Retry after 5s (max 3 attempts)
  ├─ 429 RATE_LIMIT_EXCEEDED → Wait per Retry-After header
  ├─ 500 INTERNAL_ERROR → Retry after 10s (max 3 attempts)
  └─ 401/400/422 (other) → Do not retry, review your config
```

---

## 8. Code Samples

### 8.1 Node.js / TypeScript

```typescript
const API_KEY = process.env.TRX_API_KEY!;
const BASE_URL = 'https://api.6789trx.com';

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

  // Energy has been delegated — execute the TRON transaction within 5 minutes
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
BASE_URL = 'https://api.6789trx.com'

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
    # → Execute the TRON transaction within 5 minutes
except Exception as e:
    print(f'Failed: {e}')
```

---

### 8.3 PHP

```php
<?php
define('TRX_API_KEY', getenv('TRX_API_KEY'));
define('BASE_URL', 'https://api.6789trx.com');

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
    // → Execute the TRON transaction within 5 minutes
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

const baseURL = "https://api.6789trx.com"

type DelegateRequest struct {
    WalletAddress string `json:"walletAddress"`
}

type DelegateResponse struct {
    Success   bool   `json:"success"`
    ErrorCode string `json:"errorCode"`
    Message   string `json:"message"`
    Data      *struct {
        RequestID        string `json:"requestId"`
        ProcessingTimeMs int    `json:"processingTimeMs"`
        ResourceType     int    `json:"resourceType"`
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
    // → Execute the TRON transaction within 5 minutes
}
```

---

## 9. Testing

### 9.1 Health Check

```bash
curl https://api.6789trx.com/api/v1/health
```

### 9.2 Test Single Delegate

```bash
curl -X POST https://api.6789trx.com/api/v1/resources/single \
  -H "Authorization: Bearer pas_sk_your_key_here" \
  -H "Content-Type: application/json" \
  -d '{"walletAddress": "TRxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"}'
```

### 9.3 Postman

Import the collection by creating an environment with:

| Variable | Value |
|----------|-------|
| `base_url` | `https://api.6789trx.com` |
| `api_key` | `pas_sk_your_key_here` |

Then use `{{base_url}}` and the header `Authorization: Bearer {{api_key}}` in all requests.

---

## 10. FAQ

**Q: What happens if I call the API but don't use the Energy within 5 minutes?**  
A: The Energy is reclaimed to the pool and the command is still deducted. Make sure your TRON transaction is ready before calling the API.

**Q: Should I use Single or Double?**  
A: Use Single for standard TRC-20 transfers (USDT, USDC...). Use Double if the transaction involves complex smart contracts or requires 2 consecutive transactions.

**Q: Is a command deducted for `EXTERNAL_ERROR`?**  
A: No. Only successful requests (HTTP 200) result in a command being deducted.

**Q: How do I know how many contract quota commands remain?**  
A: Currently, the API **does not expose** contract quota via any header. When the quota is exhausted, you will receive a 422 `INSUFFICIENT_COMMANDS`. To check your remaining contract commands, contact [@trx_savings](https://t.me/trx_savings).

**Q: My IP is blocked — what should I do?**  
A: Contact support via Telegram with your IP address to get it unblocked.

---

## 11. Support

For any questions about integration, errors, or to obtain an API Key:

**Telegram:** [@trx_savings](https://t.me/trx_savings)

When contacting us, please provide:
- `requestId` from the response (if there is an error)
- The error code encountered
- The time the error occurred (UTC+7)

---

## 12. About 6789TRX

**6789TRX** is a TRON energy service platform with a global community network.

### Facebook

| Region | Page |
|--------|------|
| 🇻🇳 Vietnam | [6789trx.vietnam](https://web.facebook.com/6789trx.vietnam) |
| 🇨🇳 China | [6789trx.chinese](https://web.facebook.com/6789trx.chinese) |
| 🇹🇭 Thailand | [6789trx.thailand](https://web.facebook.com/6789trx.thailand) |
| 🇮🇩 Indonesia | [6789trx.indonesia](https://web.facebook.com/6789trx.indonesia) |
| 🇬🇧 English | [6789trx.english](https://web.facebook.com/6789trx.english) |
| 🇪🇸 Español | [6789trx.spanish](https://web.facebook.com/6789trx.spanish) |
| 🇧🇷 Português | [6789trx.portuguese](https://web.facebook.com/6789trx.portuguese) |
| 🇸🇦 Arabic | [6789trx.arabic](https://web.facebook.com/6789trx.arabic) |
| 🇮🇳 Hindi | [6789trx.hindi](https://web.facebook.com/6789trx.hindi) |
| 🇫🇷 Français | [6789trx.french](https://web.facebook.com/6789trx.french) |
| 🇷🇺 Russian | [6789trx.russia](https://web.facebook.com/6789trx.russia) |

### YouTube

| Region | Channel |
|--------|---------|
| 🇻🇳 Vietnam | [@6789trx.vietnam](https://www.youtube.com/@6789trx.vietnam) |
| 🇨🇳 China | [@6789trx.chinese](https://www.youtube.com/@6789trx.chinese) |
| 🇹🇭 Thailand | [@6789trx.thailand](https://www.youtube.com/@6789trx.thailand) |
| 🇮🇩 Indonesia | [@6789trx.indonesia](https://www.youtube.com/@6789trx.indonesia) |
| 🇬🇧 English | [@6789trx.english](https://www.youtube.com/@6789trx.english) |
| 🇪🇸 Español | [@6789trx.spanish](https://www.youtube.com/@6789trx.spanish) |
| 🇧🇷 Português | [@6789trx.portuguese](https://www.youtube.com/@6789trx.portuguese) |
| 🇸🇦 Arabic | [@6789trx.arabic](https://www.youtube.com/@6789trx.arabic) |
| 🇮🇳 Hindi | [@hindi.6789trx](https://www.youtube.com/@hindi.6789trx) |
| 🇫🇷 Français | [@6789trx.french](https://www.youtube.com/@6789trx.french) |
| 🇷🇺 Russian | [@6789trx.russia](https://www.youtube.com/@6789trx.russia) |
