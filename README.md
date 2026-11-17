# API Integration Guide — TRON Network Fee Reduction Service

> **Version:** 1.3 
> **Base URL:** `https://enterprise-api.watchlist.win`  
> **Last Updated:** November 2026

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

### 1.2 Order Types

| Type | Endpoint | Energy Delegated | Commands charged | Use when |
|------|----------|------------------|------------------|----------|
| **Single** | `POST /api/v1/resources/single` | Base level **~65,000 Energy** | 1 | Transferring USDT/TRC-20 tokens to a **personal wallet with an existing balance > 0** |
| **Double** | `POST /api/v1/resources/double` | Double amount **~130,000 Energy** | 2 | Transferring USDT/TRC-20 tokens to an **exchange wallet** or any **wallet with a balance of 0** (first-time activation requires more Energy) |
| **Custom** | `POST /api/v1/resources/custom-energy` | **Exactly the amount you request** (`energyAmount`) | `ceil(energyAmount / 65 000)` | A single transaction that needs a specific amount of Energy (e.g. a heavy smart contract call, swap) |

> ⚠️ **Important:** Energy only exists for **5 minutes** after delegation.  
> If not used within 5 minutes, the Energy is reclaimed and **the order is still charged**.

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
| `400` | Malformed request or missing field |
| `401` | Missing or invalid API Key |
| `422` | Valid request but cannot be processed (quota exhausted, contract expired...) |
| `429` | Too many requests from your IP (> 15 requests/second) or IP blocked |
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
    "data": {
      "requestId": "0HNQ1790C3TET:00000001",
      "externalRequestId": "56bdd271-b41e-46a7-b3d5-896d9f0f1733",
      "timestamp": "2026-11-13T10:57:39.4835705"
    },
    "errorMessage": null,
    "requestId": "0HNQ1790C3TET:00000001",
    "processingTimeMs": 312,
    "resourceType": 1,
    "commandsCharged": 1,
    "remainingCommands": 39,
    "unitPrice": {
      "amount": 1.7,
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
| `data.data.externalRequestId` | ID of the delegation on our TRON backend — include it when contacting support |
| `data.data.timestamp` | Time the Energy was delegated (UTC+7) |
| `data.processingTimeMs` | Processing time (ms) |
| `data.resourceType` | `1` = Single, `2` = Double, `4` = Custom |
| `data.commandsCharged` | Commands deducted by **this** request (Single = 1, Double = 2, Custom = see [4.3](#43-custom-energy)) |
| `data.remainingCommands` | Commands remaining in your contract **after** this request |
| `data.unitPrice.amount` | Unit price per command as stated in your contract (rounded to 2 decimals) |
| `data.unitPrice.currency` | Currency of the unit price, as stated in your contract (`TRX` or `USDT`) |

> 💡 `unitPrice` is the price **per command**. A request costs `commandsCharged × unitPrice.amount` — e.g. a Double request consumes 2 commands, so it costs `2 × unitPrice.amount`.  
> The price is returned in your contract's own currency — it is **not** converted.

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
    "data": {
      "requestId": "0HNQ1790C3TET:00000002",
      "externalRequestId": "8f0c2a91-3d4e-4b7a-9c1e-2f6a7b8c9d0e",
      "timestamp": "2026-11-13T10:58:12.1020345"
    },
    "errorMessage": null,
    "requestId": "0HNQ1790C3TET:00000002",
    "processingTimeMs": 287,
    "resourceType": 2,
    "commandsCharged": 2,
    "remainingCommands": 37,
    "unitPrice": {
      "amount": 1.7,
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

**Commands charged:** `ceil(energyAmount / 65 000)` — the amount is rounded **up** to whole commands:

| `energyAmount` | Commands charged |
|----------------|------------------|
| 65,000 | 1 |
| 100,000 | 2 |
| 200,000 | 4 |
| 1,000,000 | 16 |

**Limits:** `energyAmount` must be between **65,000** and **1,000,000** Energy per request. Requests outside this range are rejected with 400 `INVALID_ENERGY_AMOUNT` and **no command is deducted**.

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
| `energyAmount` | integer | ✅ | Amount of Energy to delegate, from **65,000** to **1,000,000** |

**Response (200 — Success):**
```json
{
  "success": true,
  "data": {
    "success": true,
    "data": {
      "requestId": "0HNQ1790C3TET:00000003",
      "externalRequestId": "c3d9e1f2-7a4b-4c8d-9e0f-1a2b3c4d5e6f",
      "timestamp": "2026-11-17T15:30:12.3450000"
    },
    "errorMessage": null,
    "requestId": "0HNQ1790C3TET:00000003",
    "processingTimeMs": 341,
    "resourceType": 4,
    "commandsCharged": 4,
    "energyAmount": 200000,
    "remainingCommands": 33,
    "unitPrice": {
      "amount": 1.7,
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
| `data.commandsCharged` | Commands deducted — in the example above `ceil(200 000 / 65 000) = 4`, costing `4 × unitPrice.amount` |

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
| `INVALID_ENERGY_AMOUNT` | 400 | Custom: `energyAmount` missing or outside 65,000 – 1,000,000 | Send an amount within the limits |
| `CONTRACT_NOT_FOUND` | 422 | Key has no associated contract | Contact admin |
| `CONTRACT_EXPIRED` | 422 | Contract has expired | Contact to renew |
| `INSUFFICIENT_COMMANDS` | 422 | Quota commands exhausted | Contact to top up |
| `TOO_MANY_REQUESTS` | 429 | More than 15 requests/second from your IP | Slow down, wait per `Retry-After` |
| `IP_BLOCKED` | 429 | IP blocked after repeated violations | Contact support |
| `EXTERNAL_ERROR` | 500 | Temporary TRON backend error | Retry after 5s |
| `INTERNAL_ERROR` | 500 | System error | Retry after 10s |

> 📖 Full details for each error code: [API_Status_Codes.md](./API_Status_Codes.md)

---

## 6. Rate Limit & Quota

There is **no hourly or daily request limit** per API Key. The system has **2 independent limits**:

| Type | Limit | When exceeded |
|------|-------|---------------|
| **IP request rate** | **15 requests/second** per IP | 429 `TOO_MANY_REQUESTS`, then the IP is temporarily blocked |
| **Contract Quota** | Commands in your contract (see `remainingCommands` in the response body) | 422 `INSUFFICIENT_COMMANDS` |

- **IP request rate** — exceeding 15 requests/second blocks your IP: **1 minute** for the first violation, **1 hour** for the second, and **permanently** from the third (violation history is kept for 30 days). While blocked, every request returns 429 `IP_BLOCKED`. Both 429 responses include a `Retry-After` header (seconds). If you need a high request rate, queue your calls on your side.
- **Contract Quota** — total commands in your contract (counted from `EffectiveDate`). Every successful Single/Double/Custom response returns `data.commandsCharged` and `data.remainingCommands` — the commands left **after** that request. When exhausted, the API returns 422 `INSUFFICIENT_COMMANDS`. Contact [@askemily](https://t.me/askemily) to top up.

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
  ├─ 429 TOO_MANY_REQUESTS → Slow down, wait per Retry-After header
  ├─ 429 IP_BLOCKED → Contact support
  ├─ 500 EXTERNAL_ERROR → Retry after 5s (max 3 attempts)
  ├─ 500 INTERNAL_ERROR → Retry after 10s (max 3 attempts)
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
    # → Execute the TRON transaction within 5 minutes
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
        CommandsCharged   int64  `json:"commandsCharged"`
        EnergyAmount      *int64 `json:"energyAmount"` // only present for Custom
        RemainingCommands int64  `json:"remainingCommands"`
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
    // → Execute the TRON transaction within 5 minutes
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

### 9.4 Postman

Import the collection by creating an environment with:

| Variable | Value |
|----------|-------|
| `base_url` | `https://enterprise-api.watchlist.win` |
| `api_key` | `pas_sk_your_key_here` |

Then use `{{base_url}}` and the header `Authorization: Bearer {{api_key}}` in all requests.

---

## 10. FAQ

**Q: What happens if I call the API but don't use the Energy within 5 minutes?**  
A: The Energy is reclaimed to the pool and the command is still deducted. Make sure your TRON transaction is ready before calling the API.

**Q: Should I use Single or Double?**  
A: Use Single for standard TRC-20 transfers (USDT, USDC...). Use Double if the transaction involves complex smart contracts or requires 2 consecutive transactions.

**Q: When should I use Custom instead of Double?**  
A: Use Custom when one transaction needs a specific amount of Energy that Single/Double do not match — e.g. 200,000 Energy for a heavy smart contract call. You are charged `ceil(energyAmount / 65 000)` commands, shown in `data.commandsCharged`. For ordinary TRC-20 transfers, Single/Double remain the cheapest choice.

**Q: Is a command deducted for `EXTERNAL_ERROR`?**  
A: No. Only successful requests (HTTP 200) result in a command being deducted.

**Q: How do I know how many contract quota commands remain?**  
A: Read `data.remainingCommands` from any successful Single/Double/Custom response — it is the number of commands left **after** that request. When the quota is exhausted, you will receive a 422 `INSUFFICIENT_COMMANDS`. To top up, contact [@askemily](https://t.me/askemily).

**Q: How much does each command cost?**  
A: Read `data.unitPrice` from any successful Single/Double/Custom response. It is the per-command price in your contract's currency (`TRX` or `USDT`), exactly as stated in your contract. Each request costs `data.commandsCharged × unitPrice.amount` (Single = 1 command, Double = 2, Custom = `ceil(energyAmount / 65 000)`).

**Q: My IP is blocked — what should I do?**  
A: Contact support via Telegram with your IP address to get it unblocked.

---

## 11. Support

For any questions about integration, errors, or to obtain an API Key:

**Telegram:** [@askemily](https://t.me/askemily)

When contacting us, please provide:
- `requestId` from the response (if there is an error)
- The error code encountered
- The time the error occurred (UTC+7)

---

