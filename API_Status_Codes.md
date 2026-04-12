# API Status Codes & Error Reference

> Tài liệu này mô tả tất cả các trường hợp lỗi có thể xảy ra khi gọi API 6789TRX.
> Mọi response lỗi đều theo cấu trúc thống nhất với `errorCode` để xử lý phía client.

---

## Cấu trúc Response lỗi

```json
{
  "success": false,
  "data": null,
  "errorCode": "ERROR_CODE",
  "message": "Mô tả lỗi chi tiết"
}
```

---

## Bảng tổng hợp HTTP Status Code

| HTTP Code | Ý nghĩa | Khi nào xảy ra |
|-----------|---------|---------------|
| `200` | Thành công | Request được xử lý thành công |
| `400` | Bad Request | Request thiếu hoặc sai định dạng |
| `401` | Unauthorized | API Key thiếu, không hợp lệ, hoặc hết hạn |
| `422` | Unprocessable Entity | Request hợp lệ nhưng không thể thực hiện được (hết quota, hết hạn hợp đồng...) |
| `429` | Too Many Requests | Vượt quá rate limit |
| `500` | Internal Server Error | Lỗi hệ thống nội bộ |

---

## Chi tiết từng Error Code

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
**Nguyên nhân:** Field `walletAddress` bị bỏ trống hoặc không có trong request body.  
**Xử lý:** Kiểm tra lại request body, đảm bảo `walletAddress` không rỗng.

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
**Nguyên nhân:** Request body không đúng định dạng JSON, hoặc giá trị không hợp lệ.  
**Xử lý:** Kiểm tra `Content-Type: application/json` và format của request body.

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
**Nguyên nhân:** Request không có header `Authorization` hoặc `X-API-Key`.  
**Xử lý:** Thêm header `Authorization: Bearer {api_key}` vào mọi request.

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
**Nguyên nhân:** API Key không tồn tại trong hệ thống, bị vô hiệu hóa, hoặc đã hết hạn.  
**Xử lý:**
- Kiểm tra lại API Key có đúng không (copy/paste lại từ nguồn gốc)
- API Key có thể đã bị admin deactivate — liên hệ support
- Key có thể đã hết `ExpiryDays`

---

### 🔴 422 — Unprocessable Entity

> Các lỗi này xảy ra khi API Key hợp lệ nhưng không thể thực hiện lệnh delegate do ràng buộc nghiệp vụ.

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
**Nguyên nhân:** API Key chưa được gán vào hợp đồng nào (Contract) trong hệ thống.  
**Xử lý:** Liên hệ admin để cấu hình Contract cho API Key.

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
**Nguyên nhân:** Hợp đồng (Contract) đã vượt quá ngày hết hạn.  
**Xử lý:** Liên hệ admin để gia hạn hoặc tạo mới Contract.

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
**Nguyên nhân:** Hợp đồng đã dùng hết số lệnh được phép.
- Single: tốn 1 lệnh
- Double: tốn 2 lệnh  

**Xử lý:** Liên hệ admin để nạp thêm lệnh vào Contract.

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
**Nguyên nhân:** Hệ thống backend delegate (TRON node) từ chối hoặc không phản hồi.  
**Xử lý:**
- Retry sau vài giây
- Nếu lỗi liên tục (> 5 phút), liên hệ support kèm `requestId`
- **Lưu ý:** Lệnh KHÔNG bị trừ khi `EXTERNAL_ERROR` xảy ra

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
**Nguyên nhân:** API Key đã vượt quá số lệnh cho phép trong 1 giờ.  
**Response headers đi kèm:**
```http
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 2026-04-12T02:00:00.000Z
Retry-After: 1847
```
**Xử lý:** Chờ đến thời điểm `X-RateLimit-Reset` hoặc sau `Retry-After` giây.

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
**Nguyên nhân:** IP của bạn bị hệ thống tự động block do phát hiện hành vi bất thường (quá nhiều request trong thời gian ngắn, quá nhiều lần auth thất bại).  
**Xử lý:** Liên hệ support kèm địa chỉ IP để được mở khóa.

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
**Nguyên nhân:** IP vượt quá 15 request/giây — ngưỡng phát hiện DDoS.  
**Xử lý:** Giảm tần suất gọi API. IP sẽ tự mở khóa sau 1–60 phút tùy số lần vi phạm.

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
**Nguyên nhân:** Lỗi nội bộ không mong muốn (database timeout, exception chưa xử lý...).  
**Xử lý:**
- Retry sau 5–10 giây
- Nếu lỗi liên tục, liên hệ support kèm `requestId`
- **Lệnh KHÔNG bị trừ** khi `INTERNAL_ERROR` xảy ra

---

## Sơ đồ xử lý lỗi phía client

```
Gọi API
  │
  ├─ 200 ──→ Thực hiện giao dịch TRON trong 5 phút ✅
  │
  ├─ 400 ──→ Sửa request, không retry
  │
  ├─ 401 ──→ Kiểm tra API Key
  │            ├─ MISSING_AUTH     → Thêm header Authorization
  │            └─ INVALID_API_KEY  → Liên hệ admin
  │
  ├─ 422 ──→ Không thể xử lý
  │            ├─ CONTRACT_NOT_FOUND    → Liên hệ admin
  │            ├─ CONTRACT_EXPIRED     → Liên hệ admin gia hạn
  │            ├─ INSUFFICIENT_COMMANDS → Liên hệ admin nạp thêm
  │            └─ EXTERNAL_ERROR       → Retry sau 5s (tối đa 3 lần)
  │
  ├─ 429 ──→ Chờ theo Retry-After header rồi retry
  │
  └─ 500 ──→ Retry sau 10s (tối đa 3 lần) → Liên hệ support
```

---

## Ghi chú về Retry

| Error Code | Có nên Retry? | Khoảng chờ |
|------------|--------------|------------|
| `WALLET_REQUIRED` | ❌ Không | — |
| `BAD_REQUEST` | ❌ Không | — |
| `MISSING_AUTH` | ❌ Không | — |
| `INVALID_API_KEY` | ❌ Không | — |
| `CONTRACT_NOT_FOUND` | ❌ Không | — |
| `CONTRACT_EXPIRED` | ❌ Không | — |
| `INSUFFICIENT_COMMANDS` | ❌ Không | — |
| `EXTERNAL_ERROR` | ✅ Có | 5s, tối đa 3 lần |
| `RATE_LIMIT_EXCEEDED` | ✅ Có | Theo `Retry-After` |
| `TOO_MANY_REQUESTS` | ✅ Có | 60s |
| `IP_BLOCKED` | ❌ Không (liên hệ support) | — |
| `INTERNAL_ERROR` | ✅ Có | 10s, tối đa 3 lần |

---

## Lưu ý quan trọng về tính phí

| Tình huống | Có bị trừ lệnh? |
|------------|----------------|
| Request thành công (200) | ✅ Trừ 1 hoặc 2 lệnh |
| `EXTERNAL_ERROR` | ❌ Không trừ |
| `INTERNAL_ERROR` | ❌ Không trừ |
| `CONTRACT_*` | ❌ Không trừ |
| `RATE_LIMIT_EXCEEDED` | ❌ Không trừ |
| `401` / `400` | ❌ Không trừ |
