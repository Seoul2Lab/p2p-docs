[← Previous](06-callback-webhook.md) · [Index](README.md) · **07 — Query Transaction Status**

---

# 07 — Query Transaction Status

> **Authentication**: Required — `x-client-id`, `x-signature`, `x-timestamp`

Check the current status of a transaction by its UUID. Use this endpoint to verify a transaction's state — for example, after receiving a callback or when you need to poll for updates.

---

## Table of Contents

- [GET /api/v1/client/tx/check-status/:transaction_uuid](#get-apiv1clienttxcheck-statustransaction_uuid)
  - [Request Headers](#request-headers)
  - [Path Parameters](#path-parameters)
  - [Response](#response)
  - [Example — cURL](#example--curl)
  - [Example — Node.js](#example--nodejs)
  - [Sample Response (200 OK)](#sample-response-200-ok)
- [GET /api/v1/client/tx](#get-apiv1clienttx)
  - [Request Headers](#request-headers-list)
  - [Response](#response-list)
  - [Sample Response](#sample-response-list)
- [Transaction Statuses Reference](#transaction-statuses-reference)
- [Error Responses](#error-responses)

---

## GET /api/v1/client/tx/check-status/:transaction_uuid

Returns the status, amount, and redirect URL for a specific transaction.

### Request Headers

```
x-client-id:  your_client_id
x-signature:  computed_hmac_signature
x-timestamp:  1742385600
```

See [02 — Authenticated APIs](02-call-api-with-authentication.md#signature-calculation) for how to compute the signature.

### Path Parameters

| Parameter          | Type   | Required | Description                      |
| ------------------ | ------ | -------- | -------------------------------- |
| `transaction_uuid` | string | Yes      | UUID of the transaction to check |

### Response

**200 OK**

| Field          | Type    | Description                |
| -------------- | ------- | -------------------------- |
| `status`       | string  | Current transaction status |
| `amount`       | number  | Transaction amount         |
| `redirect_url` | string? | Redirect URL (nullable)    |

### Example — cURL

```bash
# Variables
CLIENT_ID="your_client_id"
CLIENT_SECRET="your_client_secret"
TIMESTAMP=$(date +%s)
TRANSACTION_UUID="a1b2c3d4-e5f6-7890-abcd-ef1234567890"

# For GET requests, use "{}" as the request body
REQUEST_BODY="{}"
QUERY_STRING=""

# Build combined string
COMBINED="${CLIENT_ID}|${TIMESTAMP}|${REQUEST_BODY}|${QUERY_STRING}"

# Compute HMAC-SHA256 signature
SIGNATURE=$(echo -n "$COMBINED" | openssl dgst -sha256 -hmac "$CLIENT_SECRET" | awk '{print $2}')

# Call the API
curl -X GET "{BASE_URL}/api/v1/client/tx/check-status/${TRANSACTION_UUID}" \
  -H "x-client-id: ${CLIENT_ID}" \
  -H "x-signature: ${SIGNATURE}" \
  -H "x-timestamp: ${TIMESTAMP}"
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

async function checkTransactionStatus(transactionUUID) {
	const timestamp = Math.floor(Date.now() / 1000).toString();
	const signature = generateSignature(CLIENT_ID, CLIENT_SECRET, timestamp);

	const response = await fetch(
		`${BASE_URL}/api/v1/client/tx/check-status/${transactionUUID}`,
		{
			method: "GET",
			headers: {
				"x-client-id": CLIENT_ID,
				"x-signature": signature,
				"x-timestamp": timestamp,
			},
		},
	);

	const data = await response.json();
	console.log("Response:", JSON.stringify(data, null, 2));
	return data;
}

checkTransactionStatus("a1b2c3d4-e5f6-7890-abcd-ef1234567890");
```

### Sample Response (200 OK)

```json
{
	"status": "success",
	"data": {
		"status": "success",
		"amount": 1000,
		"redirect_url": "https://your-domain.com/payment/complete"
	}
}
```

---

## GET /api/v1/client/tx {#get-apiv1clienttx}

Lists all transactions for your merchant account. Use this for a broader view or to search across multiple transactions.

> **Authentication**: Required — `x-client-id`, `x-signature`, `x-timestamp`

### Request Headers {#request-headers-list}

```
x-client-id:  your_client_id
x-signature:  computed_hmac_signature
x-timestamp:  1742385600
```

### Response {#response-list}

Returns an array of transaction objects with full details:

| Field                        | Type     | Description                     |
| ---------------------------- | -------- | ------------------------------- |
| `uuid`                       | string   | Unique transaction identifier   |
| `bank_account_number`        | string   | Bank account number             |
| `bank_account_name`          | string   | Bank account holder name        |
| `bank_code`                  | string   | Bank code                       |
| `bank_name_en`               | string   | Bank name in English            |
| `bank_name_th`               | string   | Bank name in Thai               |
| `customer_transfer_amount`   | number   | Exact amount transferred        |
| `amount`                     | number   | Original requested amount       |
| `customer_request_amount`    | number   | Amount requested by customer    |
| `fee`                        | number   | Transaction fee                 |
| `currency`                   | string   | Currency code                   |
| `type`                       | string   | `"deposit"` or `"withdraw"`     |
| `transaction_status`         | string   | Current status                  |
| `transaction_status_message` | string   | Human-readable status message   |
| `payment_method`             | string   | Payment method used             |
| `created_at`                 | datetime | Creation timestamp              |
| `updated_at`                 | datetime | Last update timestamp           |
| `redirect_url`               | string?  | Redirect URL (nullable)         |
| `callback_url`               | string?  | Callback URL (nullable)         |
| `merchant_order_id`          | string?  | Your order reference (nullable) |

### Sample Response {#sample-response-list}

```json
{
	"status": "success",
	"data": [
		{
			"uuid": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
			"bank_account_number": "9876543210",
			"bank_account_name": "บัญชีรับเงิน",
			"bank_code": "KBANK",
			"bank_name_en": "Kasikorn Bank",
			"bank_name_th": "ธนาคารกสิกรไทย",
			"customer_transfer_amount": 1000.25,
			"amount": 1000,
			"customer_request_amount": 1000,
			"fee": 10,
			"currency": "THB",
			"type": "deposit",
			"transaction_status": "success",
			"transaction_status_message": "Payment confirmed",
			"payment_method": "auto",
			"created_at": "2026-03-19T10:30:00Z",
			"updated_at": "2026-03-19T10:35:00Z",
			"redirect_url": "https://your-domain.com/payment/complete",
			"callback_url": "https://your-domain.com/webhook/deposit",
			"merchant_order_id": "ORDER-001"
		}
	]
}
```

---

## Transaction Statuses Reference

### Deposit Statuses

| Status       | Description                                            | Final? |
| ------------ | ------------------------------------------------------ | ------ |
| `initial`    | Transaction created, waiting to be matched             | No     |
| `matching`   | Being matched with a payee                             | No     |
| `assigned`   | Matched with a payee, waiting for payment confirmation | No     |
| `unassigned` | No payee available, waiting to be re-matched           | No     |
| `success`    | Payment confirmed                                      | Yes    |
| `expired`    | Customer did not pay or upload slip in time            | No     |
| `terminated` | Hard expired — no further action possible              | Yes    |
| `canceled`   | Canceled by merchant                                   | Yes    |

### Withdraw Statuses

| Status       | Description                                  | Final? |
| ------------ | -------------------------------------------- | ------ |
| `initial`    | Transaction created, waiting to be matched   | No     |
| `matching`   | Being matched with a payer                   | No     |
| `assigned`   | Matched with a payer, withdrawal in progress | No     |
| `unassigned` | No payer available, waiting to be re-matched | No     |
| `success`    | Funds transferred to customer                | Yes    |
| `terminated` | No payer found — no further action possible  | Yes    |
| `canceled`   | Canceled by merchant                         | Yes    |

> **Tip**: Only take final action (e.g., credit user, ship order) when the status is `success`. Poll or rely on callbacks to detect this state.

---

## Error Responses

### Transaction Not Found (422)

```json
{
	"status": "error",
	"message": "Transaction not found",
	"code": "TRANSACTION_NOT_FOUND"
}
```

### Missing Transaction UUID (400)

```json
{
	"status": "error",
	"message": "Transaction UUID is required",
	"code": "BAD_REQUEST"
}
```

### Unauthorized (401)

```json
{
	"status": "error",
	"message": "Merchant not authenticated",
	"code": "UNAUTHORIZED"
}
```

---

[← Previous](06-callback-webhook.md) · [Index](README.md)
