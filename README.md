# API Integration Guide — 6789TRX Energy Service

> **Version:** 1.0  
> **Base URL:** `https://api.6789trx.com`  
> **Last Updated:** April 2026

---

## 1. Introduction

### 1.1 System Overview

**6789TRX** là dịch vụ cho thuê **Energy trên mạng TRON** — giúp giảm chi phí giao dịch TRC-20 một cách đáng kể.

#### Tại sao cần Energy?

Mạng TRON tính phí giao dịch theo 2 loại tài nguyên:

| Tài nguyên | Dùng cho | Nếu thiếu |
|------------|----------|-----------|
| **Bandwidth** | Mọi giao dịch | Tốn ~268 TRX |
| **Energy** | Smart contract (TRC-20 transfer, swap...) | Tốn 13–65 TRX/giao dịch |

Để có Energy, người dùng thường phải **freeze TRX** (khóa vốn) hoặc **đốt TRX** trực tiếp. Cả 2 cách đều tốn kém.

#### Giải pháp của 6789TRX

Thay vì tự có Energy, bạn **thuê tạm từ pool của chúng tôi**:

```
Bạn gọi API
    ↓
Hệ thống delegate Energy vào ví của bạn (~0.3s)
    ↓
Bạn thực hiện giao dịch TRON với chi phí gần 0
    ↓
Sau 5 phút → Energy tự động reclaim về pool
```

**Tiết kiệm:** Thay vì đốt 13–65 TRX/giao dịch, bạn chỉ trả phí dịch vụ thấp hơn nhiều lần.

---

### 1.2 Hai loại lệnh

| Loại | Endpoint | Energy delegate | Dùng khi |
|------|----------|-----------------|----------|
| **Single** | `POST /api/v1/resources/single` | Mức cơ bản | Chuyển TRC-20 thông thường (USDT, USDC...) |
| **Double** | `POST /api/v1/resources/double` | Gấp đôi | Tương tác contract phức tạp, hoặc 2 giao dịch liên tiếp |

> ⚠️ **Quan trọng:** Energy chỉ tồn tại **5 phút** sau khi delegate.  
> Nếu không kịp sử dụng trong 5 phút, Energy bị reclaim và **lệnh vẫn bị tính phí**.

---

### 1.3 Base URL

```
https://api.6789trx.com
```

---

## 2. Authentication

### 2.1 Loại xác thực

API sử dụng **API Key** — truyền qua HTTP header.

### 2.2 Lấy API Key

Liên hệ qua Telegram: [@trx_savings](https://t.me/trx_savings) để được cấp API Key.

API Key có dạng:
```
pas_sk_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

### 2.3 Cách truyền API Key

**Cách 1 — Bearer Token (khuyến nghị):**
```http
Authorization: Bearer pas_sk_your_key_here
```

### 2.4 Lưu ý bảo mật

- ❌ **Không** nhúng API Key vào code client-side (browser, mobile app)
- ❌ **Không** commit API Key lên Git
- ✅ Chỉ gọi API từ **server** (backend)
- ✅ Lưu API Key trong **environment variable** hoặc secret manager

---

## 3. Common Conventions

### 3.1 Request

```http
Content-Type: application/json
Accept: application/json
Authorization: Bearer {api_key}
```

### 3.2 Response Format

Mọi response đều trả về JSON theo cấu trúc thống nhất:

```json
{
  "success": true | false,
  "data": { ... } | null,
  "errorCode": null | "ERROR_CODE",
  "message": null | "Mô tả lỗi"
}
```

### 3.3 HTTP Status Codes

| Code | Ý nghĩa |
|------|---------|
| `200` | Thành công |
| `400` | Request sai định dạng hoặc thiếu field |
| `401` | API Key thiếu hoặc không hợp lệ |
| `422` | Request hợp lệ nhưng không thể thực hiện (hết quota, hết hạn...) |
| `429` | Vượt quá rate limit |
| `500` | Lỗi hệ thống nội bộ |

---

## 4. Endpoints

### 4.1 Single Delegate

```
POST /api/v1/resources/single
```

**Mô tả:** Delegate Energy cho 1 giao dịch TRC-20 thông thường. Tốn **1 lệnh** trong quota.

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

| Field | Type | Required | Mô tả |
|-------|------|----------|-------|
| `walletAddress` | string | ✅ | Địa chỉ ví TRON nhận Energy (bắt đầu bằng `T`) |

**Response (200 — Thành công):**
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

| Field | Mô tả |
|-------|-------|
| `data.requestId` | ID để trace/debug request |
| `data.processingTimeMs` | Thời gian xử lý (ms) |
| `data.resourceType` | `1` = Single |

---

### 4.2 Double Delegate

```
POST /api/v1/resources/double
```

**Mô tả:** Delegate Energy gấp đôi — cho 2 giao dịch liên tiếp hoặc 1 contract interaction phức tạp. Tốn **2 lệnh** trong quota.

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

**Response (200 — Thành công):**
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

**Cấu trúc lỗi:**
```json
{
  "success": false,
  "data": null,
  "errorCode": "CONTRACT_EXPIRED",
  "message": "Your contract has expired. Please contact support."
}
```

**Các error code thường gặp:**

| errorCode | HTTP | Nguyên nhân | Xử lý |
|-----------|------|-------------|-------|
| `MISSING_AUTH` | 401 | Không có header Authorization | Thêm header |
| `INVALID_API_KEY` | 401 | API Key sai hoặc bị tắt | Liên hệ support |
| `WALLET_REQUIRED` | 400 | `walletAddress` trống | Kiểm tra request |
| `CONTRACT_NOT_FOUND` | 422 | Key chưa có hợp đồng | Liên hệ admin |
| `CONTRACT_EXPIRED` | 422 | Hợp đồng hết hạn | Liên hệ gia hạn |
| `INSUFFICIENT_COMMANDS` | 422 | Hết lệnh trong quota | Liên hệ nạp thêm |
| `EXTERNAL_ERROR` | 422 | Backend TRON lỗi tạm thời | Retry sau 5s |
| `RATE_LIMIT_EXCEEDED` | 429 | Vượt giới hạn/giờ | Chờ theo `Retry-After` |
| `INTERNAL_ERROR` | 500 | Lỗi hệ thống | Retry sau 10s |

> 📖 Chi tiết đầy đủ từng error code: [API_Status_Codes.md](./API_Status_Codes.md)

---

## 6. Rate Limit

Sau mỗi request đã xác thực, response trả về các headers sau:

```http
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 847
X-RateLimit-Reset: 2026-04-12T11:00:00.000Z
Retry-After: 1847          (chỉ có khi bị 429)
```

| Header | Mô tả |
|--------|-------|
| `X-RateLimit-Limit` | Giới hạn số lần gọi tối đa trong 1 giờ (theo giờ UTC+7) |
| `X-RateLimit-Remaining` | Số lần gọi còn lại trong **giờ hiện tại** |
| `X-RateLimit-Reset` | Thời điểm reset về đầu giờ tiếp theo (ISO 8601, UTC) |
| `Retry-After` | Số giây cần chờ — chỉ xuất hiện khi bị 429 |

> Double request tốn **2 slot** trong `X-RateLimit-Remaining`.

### Lưu ý quan trọng về 2 loại quota

Hệ thống có **2 giới hạn độc lập**:

| Loại | Header | Reset khi nào | Khi vượt |
|------|--------|---------------|----------|
| **Hourly Rate Limit** | `X-RateLimit-Remaining` | Đầu mỗi giờ UTC+7 | 429 `RATE_LIMIT_EXCEEDED` |
| **Contract Quota** | *(không expose)* | Khi admin nạp thêm | 422 `INSUFFICIENT_COMMANDS` |

- **Hourly Rate Limit** — đếm số request trong giờ hiện tại, reset tự động mỗi giờ. Thể hiện qua `X-RateLimit-Remaining`.
- **Contract Quota** — tổng số lệnh trong hợp đồng của bạn (tính từ `EffectiveDate`). **Không có header nào expose số này.** Khi hết, API trả 422 `INSUFFICIENT_COMMANDS`. Liên hệ [@trx_savings](https://t.me/trx_savings) để kiểm tra hoặc nạp thêm.

---

## 7. Flow Examples

### 7.1 Luồng cơ bản — Chuyển USDT

```
1. Chuẩn bị giao dịch USDT trên ví TRxxxxxxxx
        ↓
2. Gọi POST /api/v1/resources/single
   Body: { "walletAddress": "TRxxxxxxxx" }
        ↓
3. Nhận response { success: true, requestId: "..." }
        ↓
4. Thực hiện giao dịch USDT ngay (trong vòng 5 phút)
        ↓
5. Giao dịch thành công với phí gần 0 TRX ✅
```

### 7.2 Luồng xử lý lỗi — Retry

```
Gọi API
  ├─ 200 → Thực hiện giao dịch ngay ✅
  ├─ 422 EXTERNAL_ERROR → Retry sau 5s (tối đa 3 lần)
  ├─ 429 RATE_LIMIT_EXCEEDED → Chờ theo header Retry-After
  ├─ 500 INTERNAL_ERROR → Retry sau 10s (tối đa 3 lần)
  └─ 401/400/422 (khác) → Không retry, kiểm tra lại config
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

  // Energy đã delegate — thực hiện giao dịch TRON trong 5 phút
  return result.data.requestId;
}

// Với retry tự động
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

# Sử dụng
try:
    data = delegate_with_retry('TRxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx')
    print(f'Energy delegated! RequestId: {data["requestId"]}')
    # → Thực hiện giao dịch TRON trong 5 phút
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

// Sử dụng
try {
    $data = delegateWithRetry('TRxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx');
    echo "Energy delegated! RequestId: " . $data['requestId'];
    // → Thực hiện giao dịch TRON trong 5 phút
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
        RequestID       string `json:"requestId"`
        ProcessingTimeMs int   `json:"processingTimeMs"`
        ResourceType    int    `json:"resourceType"`
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
    // → Thực hiện giao dịch TRON trong 5 phút
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

Import collection bằng cách tạo environment với:

| Variable | Value |
|----------|-------|
| `base_url` | `https://api.6789trx.com` |
| `api_key` | `pas_sk_your_key_here` |

Sau đó dùng `{{base_url}}` và header `Authorization: Bearer {{api_key}}` trong mọi request.

---

## 10. FAQ

**Q: Nếu tôi gọi API nhưng không kịp dùng trong 5 phút thì sao?**  
A: Energy bị reclaim về pool, lệnh vẫn bị trừ. Hãy đảm bảo giao dịch TRON đã sẵn sàng trước khi gọi API.

**Q: Tôi nên dùng Single hay Double?**  
A: Dùng Single cho chuyển TRC-20 thông thường (USDT, USDC...). Dùng Double nếu giao dịch liên quan đến smart contract phức tạp hoặc cần thực hiện 2 giao dịch liên tiếp.

**Q: `EXTERNAL_ERROR` có bị trừ lệnh không?**  
A: Không. Chỉ request thành công (HTTP 200) mới bị trừ lệnh.

**Q: Làm sao biết còn bao nhiêu lệnh trong giờ này?**  
A: Xem header `X-RateLimit-Remaining` trong response của bất kỳ request nào — đây là số lần gọi còn lại trong giờ hiện tại (reset mỗi giờ UTC+7).

**Q: Làm sao biết contract quota còn bao nhiêu lệnh?**  
A: Hiện tại API **không expose** contract quota qua header. Khi hết quota, bạn sẽ nhận 422 `INSUFFICIENT_COMMANDS`. Để kiểm tra số lệnh còn lại trong hợp đồng, liên hệ [@trx_savings](https://t.me/trx_savings).

**Q: Tôi bị block IP, phải làm gì?**  
A: Liên hệ support qua Telegram kèm địa chỉ IP để được mở khóa.

---

## 11. Support

Mọi thắc mắc về tích hợp, lỗi, hoặc cần cấp API Key:

**Telegram:** [@trx_savings](https://t.me/trx_savings)

Khi liên hệ, vui lòng cung cấp:
- `requestId` từ response (nếu có lỗi)
- Error code gặp phải
- Thời điểm xảy ra lỗi (UTC+7)

---

## 12. Về 6789TRX

**6789TRX** là nền tảng cung cấp dịch vụ năng lượng TRON với mạng lưới cộng đồng toàn cầu.

### Facebook

| Khu vực | Trang |
|---------|-------|
| 🇻🇳 Việt Nam | [6789trx.vietnam](https://web.facebook.com/6789trx.vietnam) |
| 🇨🇳 Trung Quốc | [6789trx.chinese](https://web.facebook.com/6789trx.chinese) |
| 🇹🇭 Thái Lan | [6789trx.thailand](https://web.facebook.com/6789trx.thailand) |
| 🇮🇩 Indonesia | [6789trx.indonesia](https://web.facebook.com/6789trx.indonesia) |
| 🇬🇧 English | [6789trx.english](https://web.facebook.com/6789trx.english) |
| 🇪🇸 Español | [6789trx.spanish](https://web.facebook.com/6789trx.spanish) |
| 🇧🇷 Português | [6789trx.portuguese](https://web.facebook.com/6789trx.portuguese) |
| 🇸🇦 Arabic | [6789trx.arabic](https://web.facebook.com/6789trx.arabic) |
| 🇮🇳 Hindi | [6789trx.hindi](https://web.facebook.com/6789trx.hindi) |
| 🇫🇷 Français | [6789trx.french](https://web.facebook.com/6789trx.french) |
| 🇷🇺 Русский | [6789trx.russia](https://web.facebook.com/6789trx.russia) |

### YouTube

| Khu vực | Kênh |
|---------|------|
| 🇻🇳 Việt Nam | [@6789trx.vietnam](https://www.youtube.com/@6789trx.vietnam) |
| 🇨🇳 Trung Quốc | [@6789trx.chinese](https://www.youtube.com/@6789trx.chinese) |
| 🇹🇭 Thái Lan | [@6789trx.thailand](https://www.youtube.com/@6789trx.thailand) |
| 🇮🇩 Indonesia | [@6789trx.indonesia](https://www.youtube.com/@6789trx.indonesia) |
| 🇬🇧 English | [@6789trx.english](https://www.youtube.com/@6789trx.english) |
| 🇪🇸 Español | [@6789trx.spanish](https://www.youtube.com/@6789trx.spanish) |
| 🇧🇷 Português | [@6789trx.portuguese](https://www.youtube.com/@6789trx.portuguese) |
| 🇸🇦 Arabic | [@6789trx.arabic](https://www.youtube.com/@6789trx.arabic) |
| 🇮🇳 Hindi | [@hindi.6789trx](https://www.youtube.com/@hindi.6789trx) |
| 🇫🇷 Français | [@6789trx.french](https://www.youtube.com/@6789trx.french) |
| 🇷🇺 Русский | [@6789trx.russia](https://www.youtube.com/@6789trx.russia) |
