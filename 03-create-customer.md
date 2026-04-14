[← Previous](02-call-api-with-authentication.md) · [Index](README.md) · **03 — Customer Account Management** · [Next →](04-create-deposit.md)

---

# 03 — Customer Account Management

> **Authentication**: Required — `x-client-id`, `x-signature`, `x-timestamp`

Manage customer bank accounts before submitting deposit or withdrawal transactions. Each customer account is linked to a specific bank and identified by a unique UUID.

---

## Table of Contents

- [POST /api/v1/client/customer](#post-apiv1clientcustomer) — Create customer account
  - [Request Headers](#request-headers)
  - [Request Body](#request-body)
  - [Response](#response)
  - [Example — cURL](#example--curl)
  - [Example — Node.js](#example--nodejs)
- [GET /api/v1/client/customer](#get-apiv1clientcustomer) — List all customer accounts
- [GET /api/v1/client/customer/:uuid](#get-apiv1clientcustomeruuid) — Get customer account by UUID
- [PATCH /api/v1/client/customer/:uuid](#patch-apiv1clientcustomeruuid) — Update customer account
- [DELETE /api/v1/client/customer/:uuid](#delete-apiv1clientcustomeruuid) — Delete customer account
- [Error Responses](#error-responses)

---

## POST /api/v1/client/customer

Creates a new customer bank account under your merchant.

### Request Headers

```
x-client-id:  your_client_id
x-signature:  computed_hmac_signature
x-timestamp:  1742385600
```

See [02 — Authenticated APIs](02-call-api-with-authentication.md#signature-calculation) for how to compute the signature.

### Request Body

| Field                  | Type   | Required | Description                                                                      |
| ---------------------- | ------ | -------- | -------------------------------------------------------------------------------- |
| `bank_code`            | string | Yes      | Bank code from [GET /api/v1/client/bank](01-call-public-api.md) (e.g. `"KBANK"`) |
| `bank_account_number`  | string | Yes      | Customer's bank account number                                                   |
| `bank_account_name`    | string | Yes      | Account holder name (Thai)                                                       |
| `bank_account_name_en` | string | No       | Account holder name (English)                                                    |
| `status`               | string | No       | `"active"` (default) or `"inactive"`                                             |
| `ref_code`             | string | No       | Your reference code for this customer                                            |
| `tag`                  | string | No       | Custom tag for categorization                                                    |
| `note`                 | string | No       | Internal note                                                                    |

### Response

**201 Created**

| Field                  | Type     | Description                        |
| ---------------------- | -------- | ---------------------------------- |
| `uuid`                 | string   | Unique customer account identifier |
| `merchant_uuid`        | string   | Your merchant UUID                 |
| `bank_code`            | string   | Bank code                          |
| `bank_account_number`  | string   | Bank account number                |
| `bank_account_name`    | string   | Account name (Thai)                |
| `bank_account_name_en` | string   | Account name (English)             |
| `status`               | string   | `"active"` or `"inactive"`         |
| `ref_code`             | string?  | Reference code (nullable)          |
| `tag`                  | string?  | Tag (nullable)                     |
| `note`                 | string?  | Note (nullable)                    |
| `created_at`           | datetime | ISO 8601 creation timestamp        |

### Example — cURL

```bash
# Variables
CLIENT_ID="your_client_id"
CLIENT_SECRET="your_client_secret"
TIMESTAMP=$(date +%s)

# Request body
REQUEST_BODY='{"bank_code":"KBANK","bank_account_number":"1234567890","bank_account_name":"ทดสอบ ชื่อ","bank_account_name_en":"Test Name","ref_code":"CUST-001"}'

# Build combined string
QUERY_STRING=""
COMBINED="${CLIENT_ID}|${TIMESTAMP}|${REQUEST_BODY}|${QUERY_STRING}"

# Compute HMAC-SHA256 signature
SIGNATURE=$(echo -n "$COMBINED" | openssl dgst -sha256 -hmac "$CLIENT_SECRET" | awk '{print $2}')

# Call the API
curl -X POST {BASE_URL}/api/v1/client/customer \
  -H "Content-Type: application/json" \
  -H "x-client-id: ${CLIENT_ID}" \
  -H "x-signature: ${SIGNATURE}" \
  -H "x-timestamp: ${TIMESTAMP}" \
  -d "${REQUEST_BODY}"
```

### Example — Node.js

```javascript
const crypto = require("crypto");

const CLIENT_ID = "your_client_id";
const CLIENT_SECRET = "your_client_secret";
const BASE_URL = "{BASE_URL}";

function generateSignature(
	clientId,
	clientSecret,
	timestamp,
	body = "{}",
	queryString = "",
) {
	const combined = `${clientId}|${timestamp}|${body}|${queryString}`;
	return crypto
		.createHmac("sha256", clientSecret)
		.update(combined)
		.digest("hex");
}

async function createCustomer() {
	const timestamp = Math.floor(Date.now() / 1000).toString();

	const body = {
		bank_code: "KBANK",
		bank_account_number: "1234567890",
		bank_account_name: "ทดสอบ ชื่อ",
		bank_account_name_en: "Test Name",
		ref_code: "CUST-001",
	};

	const bodyString = JSON.stringify(body);
	const signature = generateSignature(
		CLIENT_ID,
		CLIENT_SECRET,
		timestamp,
		bodyString,
	);

	const response = await fetch(`${BASE_URL}/api/v1/client/customer`, {
		method: "POST",
		headers: {
			"Content-Type": "application/json",
			"x-client-id": CLIENT_ID,
			"x-signature": signature,
			"x-timestamp": timestamp,
		},
		body: bodyString,
	});

	const data = await response.json();
	console.log("Response:", JSON.stringify(data, null, 2));
}

createCustomer();
```

### Sample Response (201 Created)

```json
{
	"status": "success",
	"data": {
		"uuid": "550e8400-e29b-41d4-a716-446655440000",
		"merchant_uuid": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
		"bank_code": "KBANK",
		"bank_account_number": "1234567890",
		"bank_account_name": "ทดสอบ ชื่อ",
		"bank_account_name_en": "Test Name",
		"status": "active",
		"ref_code": "CUST-001",
		"tag": null,
		"note": null,
		"created_at": "2026-03-19T10:30:00Z"
	}
}
```

---

## GET /api/v1/client/customer

Retrieves all customer accounts for your merchant with pagination.

### Request Headers

```
x-client-id:  your_client_id
x-signature:  computed_hmac_signature
x-timestamp:  1742385600
```

### Query Parameters

| Parameter | Type | Required | Description                                |
| --------- | ---- | -------- | ------------------------------------------ |
| `page`    | int  | No       | Page number (default: `1`)                 |
| `limit`   | int  | No       | Items per page (default: `10`, max: `100`) |

### Example — cURL

```bash
CLIENT_ID="your_client_id"
CLIENT_SECRET="your_client_secret"
TIMESTAMP=$(date +%s)

REQUEST_BODY="{}"
QUERY_STRING="page=1&limit=10"
COMBINED="${CLIENT_ID}|${TIMESTAMP}|${REQUEST_BODY}|${QUERY_STRING}"

SIGNATURE=$(echo -n "$COMBINED" | openssl dgst -sha256 -hmac "$CLIENT_SECRET" | awk '{print $2}')

curl -X GET "{BASE_URL}/api/v1/client/customer?page=1&limit=10" \
  -H "x-client-id: ${CLIENT_ID}" \
  -H "x-signature: ${SIGNATURE}" \
  -H "x-timestamp: ${TIMESTAMP}"
```

### Sample Response (200 OK)

```json
{
	"status": "success",
	"data": {
		"customer_accounts": [
			{
				"uuid": "550e8400-e29b-41d4-a716-446655440000",
				"merchant_uuid": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
				"bank_code": "KBANK",
				"bank_account_number": "1234567890",
				"bank_account_name": "ทดสอบ ชื่อ",
				"bank_account_name_en": "Test Name",
				"status": "active",
				"ref_code": "CUST-001",
				"tag": null,
				"note": null,
				"created_at": "2026-03-19T10:30:00Z"
			}
		],
		"metadata": {
			"page": 1,
			"limit": 10,
			"total": 1
		}
	}
}
```

---

## GET /api/v1/client/customer/:uuid

Retrieves a specific customer account by its UUID.

### Request Headers

```
x-client-id:  your_client_id
x-signature:  computed_hmac_signature
x-timestamp:  1742385600
```

### Path Parameters

| Parameter | Type   | Required | Description                  |
| --------- | ------ | -------- | ---------------------------- |
| `uuid`    | string | Yes      | UUID of the customer account |

### Example — cURL

```bash
CLIENT_ID="your_client_id"
CLIENT_SECRET="your_client_secret"
TIMESTAMP=$(date +%s)
CUSTOMER_UUID="550e8400-e29b-41d4-a716-446655440000"

REQUEST_BODY="{}"
QUERY_STRING=""
COMBINED="${CLIENT_ID}|${TIMESTAMP}|${REQUEST_BODY}|${QUERY_STRING}"

SIGNATURE=$(echo -n "$COMBINED" | openssl dgst -sha256 -hmac "$CLIENT_SECRET" | awk '{print $2}')

curl -X GET "{BASE_URL}/api/v1/client/customer/${CUSTOMER_UUID}" \
  -H "x-client-id: ${CLIENT_ID}" \
  -H "x-signature: ${SIGNATURE}" \
  -H "x-timestamp: ${TIMESTAMP}"
```

### Sample Response (200 OK)

```json
{
	"status": "success",
	"data": {
		"uuid": "550e8400-e29b-41d4-a716-446655440000",
		"merchant_uuid": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
		"bank_code": "KBANK",
		"bank_account_number": "1234567890",
		"bank_account_name": "ทดสอบ ชื่อ",
		"bank_account_name_en": "Test Name",
		"status": "active",
		"ref_code": "CUST-001",
		"tag": null,
		"note": null,
		"created_at": "2026-03-19T10:30:00Z"
	}
}
```

---

## PATCH /api/v1/client/customer/:uuid

Updates an existing customer account. All fields are optional — only the fields you include in the request body will be updated.

### Request Headers

```
Content-Type: application/json
x-client-id:  your_client_id
x-signature:  computed_hmac_signature
x-timestamp:  1742385600
```

### Path Parameters

| Parameter | Type   | Required | Description                  |
| --------- | ------ | -------- | ---------------------------- |
| `uuid`    | string | Yes      | UUID of the customer account |

### Request Body

All fields are optional. Only include the fields you want to update.

| Field                  | Type   | Required | Description                                                     |
| ---------------------- | ------ | -------- | --------------------------------------------------------------- |
| `bank_code`            | string | No       | Bank code from [GET /api/v1/client/bank](01-call-public-api.md) |
| `bank_account_number`  | string | No       | Customer's bank account number                                  |
| `bank_account_name`    | string | No       | Account holder name (Thai)                                      |
| `bank_account_name_en` | string | No       | Account holder name (English)                                   |
| `status`               | string | No       | `"active"` or `"inactive"`                                      |
| `ref_code`             | string | No       | Your reference code for this customer                           |
| `tag`                  | string | No       | Custom tag for categorization                                   |
| `note`                 | string | No       | Internal note                                                   |

### Example — cURL

```bash
CLIENT_ID="your_client_id"
CLIENT_SECRET="your_client_secret"
TIMESTAMP=$(date +%s)
CUSTOMER_UUID="550e8400-e29b-41d4-a716-446655440000"

REQUEST_BODY='{"bank_account_name":"ชื่อใหม่","bank_account_name_en":"New Name","status":"active"}'

QUERY_STRING=""
COMBINED="${CLIENT_ID}|${TIMESTAMP}|${REQUEST_BODY}|${QUERY_STRING}"

SIGNATURE=$(echo -n "$COMBINED" | openssl dgst -sha256 -hmac "$CLIENT_SECRET" | awk '{print $2}')

curl -X PATCH "{BASE_URL}/api/v1/client/customer/${CUSTOMER_UUID}" \
  -H "Content-Type: application/json" \
  -H "x-client-id: ${CLIENT_ID}" \
  -H "x-signature: ${SIGNATURE}" \
  -H "x-timestamp: ${TIMESTAMP}" \
  -d "${REQUEST_BODY}"
```

### Example — Node.js

```javascript
const crypto = require("crypto");

const CLIENT_ID = "your_client_id";
const CLIENT_SECRET = "your_client_secret";
const BASE_URL = "{BASE_URL}";

function generateSignature(
	clientId,
	clientSecret,
	timestamp,
	body = "{}",
	queryString = "",
) {
	const combined = `${clientId}|${timestamp}|${body}|${queryString}`;
	return crypto
		.createHmac("sha256", clientSecret)
		.update(combined)
		.digest("hex");
}

async function updateCustomer(customerUUID) {
	const timestamp = Math.floor(Date.now() / 1000).toString();

	const body = {
		bank_account_name: "ชื่อใหม่",
		bank_account_name_en: "New Name",
		status: "active",
	};

	const bodyString = JSON.stringify(body);
	const signature = generateSignature(
		CLIENT_ID,
		CLIENT_SECRET,
		timestamp,
		bodyString,
	);

	const response = await fetch(
		`${BASE_URL}/api/v1/client/customer/${customerUUID}`,
		{
			method: "PATCH",
			headers: {
				"Content-Type": "application/json",
				"x-client-id": CLIENT_ID,
				"x-signature": signature,
				"x-timestamp": timestamp,
			},
			body: bodyString,
		},
	);

	const data = await response.json();
	console.log("Response:", JSON.stringify(data, null, 2));
}

updateCustomer("550e8400-e29b-41d4-a716-446655440000");
```

### Sample Response (200 OK)

```json
{
	"status": "success",
	"data": {
		"uuid": "550e8400-e29b-41d4-a716-446655440000",
		"merchant_uuid": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
		"bank_code": "KBANK",
		"bank_account_number": "1234567890",
		"bank_account_name": "ชื่อใหม่",
		"bank_account_name_en": "New Name",
		"status": "active",
		"ref_code": "CUST-001",
		"tag": null,
		"note": null,
		"created_at": "2026-03-19T10:30:00Z"
	}
}
```

---

## DELETE /api/v1/client/customer/:uuid

Deletes a customer account by UUID.

### Request Headers

```
x-client-id:  your_client_id
x-signature:  computed_hmac_signature
x-timestamp:  1742385600
```

### Path Parameters

| Parameter | Type   | Required | Description                  |
| --------- | ------ | -------- | ---------------------------- |
| `uuid`    | string | Yes      | UUID of the customer account |

### Example — cURL

```bash
CLIENT_ID="your_client_id"
CLIENT_SECRET="your_client_secret"
TIMESTAMP=$(date +%s)
CUSTOMER_UUID="550e8400-e29b-41d4-a716-446655440000"

REQUEST_BODY="{}"
QUERY_STRING=""
COMBINED="${CLIENT_ID}|${TIMESTAMP}|${REQUEST_BODY}|${QUERY_STRING}"

SIGNATURE=$(echo -n "$COMBINED" | openssl dgst -sha256 -hmac "$CLIENT_SECRET" | awk '{print $2}')

curl -X DELETE "{BASE_URL}/api/v1/client/customer/${CUSTOMER_UUID}" \
  -H "x-client-id: ${CLIENT_ID}" \
  -H "x-signature: ${SIGNATURE}" \
  -H "x-timestamp: ${TIMESTAMP}"
```

### Response

**204 No Content** — Customer account deleted successfully (empty response body).

---

## Error Responses

### Invalid Request Body (400)

```json
{
	"status": "error",
	"message": "Field validation error or JSON parsing error",
	"code": "INVALID_REQUEST"
}
```

### Duplicate Account (400)

Returned when a customer account with the same bank code and account number already exists under your merchant.

```json
{
	"status": "error",
	"message": "Customer account already exists",
	"code": "CUSTOMER_ACCOUNT_ALREADY_EXISTS"
}
```

### Invalid Bank Code (400)

The `bank_code` must be one of the codes returned by [GET /api/v1/client/bank](01-call-public-api.md).

```json
{
	"status": "error",
	"message": "Bank not found or invalid",
	"code": "BANK_NOT_FOUND"
}
```

### Invalid UUID Format (400)

```json
{
	"status": "error",
	"message": "Invalid UUID format",
	"code": "INVALID_UUID_FORMAT"
}
```

### Unauthorized (401)

```json
{
	"status": "error",
	"message": "unauthorized",
	"code": "UNAUTHORIZED"
}
```

### Customer Account Not Found (404)

```json
{
	"status": "error",
	"message": "customer account not found",
	"code": "CUSTOMER_ACCOUNT_NOT_FOUND"
}
```

---

[← Previous](02-call-api-with-authentication.md) · [Index](README.md) · **03 — Customer Account Management** · [Next →](04-create-deposit.md)
