[← Index](README.md) · **01 — Public APIs (No Authentication)** · [Next →](02-call-api-with-authentication.md)

---

# 01 — Public APIs (No Authentication)

> **Authentication**: Not required — these are public endpoints.

These endpoints do not require authentication. Use them as your first API calls to verify connectivity and retrieve the list of supported banks.

---

## Table of Contents

- [GET /api/v1/client/bank](#get-apiv1clientbank)
  - [Request](#request)
  - [Response](#response)
  - [Example — cURL](#example--curl)
  - [Example — Node.js](#example--nodejs)
  - [Sample Response](#sample-response)
- [GET /api/v1/client/bank/code/:code](#get-apiv1clientbankcodecode)
  - [Request](#request-1)
  - [Example — cURL](#example--curl-1)
  - [Example — Node.js](#example--nodejs-1)
  - [Sample Response (200 OK)](#sample-response-200-ok)
  - [Error Response (404 Not Found)](#error-response-404-not-found)

---

## GET /api/v1/client/bank

Returns all available banks in the system.

### Request

No headers, body, or query parameters required.

### Response

| Field                        | Type    | Description                    |
| ---------------------------- | ------- | ------------------------------ |
| `status`                     | string  | `"success"`                    |
| `data.banks`                 | array   | List of bank objects           |
| `data.banks[].code`          | string  | Bank code (e.g. `"BBL"`)       |
| `data.banks[].name_en`       | string  | Bank name in English           |
| `data.banks[].name_th`       | string  | Bank name in Thai              |
| `data.banks[].bank_code2`    | string? | Secondary bank code (nullable) |
| `data.banks[].bot_bank_code` | string  | Bank of Thailand code          |
| `data.total`                 | integer | Total number of banks          |

### Example — cURL

```bash
curl -X GET {BASE_URL}/api/v1/client/bank
```

### Example — Node.js

```javascript
const response = await fetch("{BASE_URL}/api/v1/client/bank");
const data = await response.json();
console.log(data);
```

### Sample Response

```json
{
	"status": "success",
	"data": {
		"banks": [
			{
				"code": "BBL",
				"name_en": "Bangkok Bank",
				"name_th": "ธนาคารกรุงเทพ",
				"bank_code2": null,
				"bot_bank_code": "002"
			},
			{
				"code": "KBANK",
				"name_en": "Kasikorn Bank",
				"name_th": "ธนาคารกสิกรไทย",
				"bank_code2": null,
				"bot_bank_code": "004"
			},
			{
				"code": "SCB",
				"name_en": "Siam Commercial Bank",
				"name_th": "ธนาคารไทยพาณิชย์",
				"bank_code2": null,
				"bot_bank_code": "014"
			}
		],
		"total": 3
	}
}
```

---

## GET /api/v1/client/bank/code/:code

Look up a single bank by its code.

### Request

| Parameter | In   | Type   | Required | Description                  |
| --------- | ---- | ------ | -------- | ---------------------------- |
| `code`    | path | string | Yes      | Bank code (case-insensitive) |

### Example — cURL

```bash
curl -X GET {BASE_URL}/api/v1/client/bank/code/KBANK
```

### Example — Node.js

```javascript
const bankCode = "KBANK";
const response = await fetch(`${BASE_URL}/api/v1/client/bank/code/${bankCode}`);
const data = await response.json();
console.log(data);
```

### Sample Response (200 OK)

```json
{
	"status": "success",
	"data": {
		"code": "KBANK",
		"name_en": "Kasikorn Bank",
		"name_th": "ธนาคารกสิกรไทย",
		"bank_code2": null,
		"bot_bank_code": "004"
	}
}
```

### Error Response (404 Not Found)

```json
{
	"status": "error",
	"message": "Bank not found",
	"code": "BANK_NOT_FOUND"
}
```

---

[← Index](README.md) · **01 — Public APIs (No Authentication)** · [Next →](02-call-api-with-authentication.md)
