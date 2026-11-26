# Changelog

All notable changes to the 6789TRX API and its documentation. The full API reference is in [README.md](./README.md); error codes are in [API_Status_Codes.md](./API_Status_Codes.md).

**Legend:** ⚠️ = may require a change in your integration.

---

## 1.6.2 — November 2026

- ⚠️ `data.data.timestamp` in [Single](./README.md#41-single-delegate)/[Double](./README.md#42-double-delegate)/[Custom](./README.md#43-custom-energy) responses and `data.reclaimedAt` in [Reclaim](./README.md#44-reclaim-on-demand) responses are now in **UTC** with a `Z` suffix (e.g. `2026-11-13T03:57:39.4835705Z`) instead of UTC+7 without an offset. If you parse or display these times as UTC+7, update your code. All times returned by the API are now UTC.

## 1.6.1 — November 2026

- New endpoint [`GET /api/v1/resources/balance`](./README.md#45-balance--history): commands left in your current contract, total commands purchased and used by your API Key, and the history of your most recent successful requests (newest first, up to 1,000) with **cursor pagination** (`?cursor=` + `nextCursor`). Times in this endpoint are UTC. No command is deducted. New error `400` `INVALID_CURSOR`.

## 1.6 — November 2026

- When the service is temporarily unavailable, the API now returns a **JSON error** instead of an HTML page: `502`/`503` `SERVICE_UNAVAILABLE` (with `Retry-After: 5`) and `504` `GATEWAY_TIMEOUT`. On `504` the request may or may not have been processed — check the wallet's Energy before retrying. See [API_Status_Codes.md](./API_Status_Codes.md#-502--503--504--service-unavailable--gateway-timeout).

## 1.5 — November 2026

- ⚠️ **Custom Energy is billed in fractional commands**, rounded up to 0.01: 200,000 Energy now costs **3.08** commands instead of 4. See [4.3 Custom Energy](./README.md#43-custom-energy).
- ⚠️ `commandsCharged` and `remainingCommands` can be **decimal numbers** (up to 2 decimals, e.g. `3.08`, `33.92`). If your code parses them as integers, switch to a decimal/float type — see the [Go sample](./README.md#84-go).
- ⚠️ **One active Custom delegation per wallet.** While a wallet still holds Custom Energy that has not been reclaimed, every new Single/Double/Custom request for that wallet is rejected with **409 `WALLET_HAS_ACTIVE_CUSTOM`** (no command deducted). Call [`/reclaim`](./README.md#44-reclaim-on-demand) or wait until it is reclaimed, then retry.
- Delegated Energy is reclaimed **3 minutes** after delegation at the latest (previously 5). A Custom delegation is reclaimed as soon as **90%** of its Energy has been used.
- Double delegates **~135,000 Energy**, for one transaction.
- New field `data.txHashes`: hashes of the delegate transactions on TRON (look them up on Tronscan). [Reclaim](./README.md#44-reclaim-on-demand) also returns it, with the reclaim transaction hashes. `data.data.externalRequestId` is now the delegate transaction hash too.
- When all commands are used (less than 1 command left), the contract ends automatically and further calls return **401 `INVALID_API_KEY`** instead of 422.

## 1.4.1 — November 2026

- Clarified Custom Energy billing: you receive exactly the `energyAmount` requested, and commands are rounded up.

## 1.4 — November 2026

- New endpoint [`POST /api/v1/resources/reclaim`](./README.md#44-reclaim-on-demand): reclaim the Energy delegated to a wallet on demand.
- Custom Energy maximum raised to **1,690,000** Energy per request.

## 1.3 — November 2026

- Added [Custom Energy](./README.md#43-custom-energy) details, clarified error codes and the response structure.

## 1.2 — November 2026

- More detail on API responses and the contract quota.
