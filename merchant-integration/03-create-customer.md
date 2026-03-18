[← Previous](02-merchant-info.md) · [Index](README.md) · **03 — Create Customer** · [Next →](04-create-deposit.md)

---

# 03 — Create Customer Account

> **Authentication**: Required — `x-client-id`, `x-signature`, `x-timestamp`

Register a customer's bank account before submitting deposit or withdrawal transactions. Each customer account is linked to a specific bank and identified by a unique UUID.

---

## Table of Contents

- [POST /api/v1/client/customer](#post-apiv1clientcustomer)
  - [Request Headers](#request-headers)
  - [Request Body](#request-body)
  - [Response](#response)
  - [Example — cURL](#example--curl)
  - [Example — Node.js](#example--nodejs)
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

See [02 — Authenticated APIs](02-merchant-info.md#signature-calculation) for how to compute the signature.

### Request Body

| Field                  | Type   | Required | Description                                      |
| ---------------------- | ------ | -------- | ------------------------------------------------ |
| `bank_code`            | string | Yes      | Bank code from [GET /api/v1/client/bank](01-get-bank-list.md) (e.g. `"KBANK"`) |
| `bank_account_number`  | string | Yes      | Customer's bank account number                   |
| `bank_account_name`    | string | Yes      | Account holder name (Thai)                       |
| `bank_account_name_en` | string | No       | Account holder name (English)                    |
| `status`               | string | No       | `"active"` (default) or `"inactive"`             |
| `ref_code`             | string | No       | Your reference code for this customer            |
| `tag`                  | string | No       | Custom tag for categorization                    |
| `note`                 | string | No       | Internal note                                    |

### Response

**201 Created**

| Field                  | Type      | Description                        |
| ---------------------- | --------- | ---------------------------------- |
| `uuid`                 | string    | Unique customer account identifier |
| `merchant_uuid`        | string    | Your merchant UUID                 |
| `bank_code`            | string    | Bank code                          |
| `bank_account_number`  | string    | Bank account number                |
| `bank_account_name`    | string    | Account name (Thai)                |
| `bank_account_name_en` | string    | Account name (English)             |
| `status`               | string    | `"active"` or `"inactive"`        |
| `ref_code`             | string?   | Reference code (nullable)          |
| `tag`                  | string?   | Tag (nullable)                     |
| `note`                 | string?   | Note (nullable)                    |
| `created_at`           | datetime  | ISO 8601 creation timestamp        |

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

function generateSignature(clientId, clientSecret, timestamp, body = "{}", queryString = "") {
  const combined = `${clientId}|${timestamp}|${body}|${queryString}`;
  return crypto.createHmac("sha256", clientSecret).update(combined).digest("hex");
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
  const signature = generateSignature(CLIENT_ID, CLIENT_SECRET, timestamp, bodyString);

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

The `bank_code` must be one of the codes returned by [GET /api/v1/client/bank](01-get-bank-list.md).

```json
{
  "status": "error",
  "message": "Bank not found or invalid",
  "code": "BANK_NOT_FOUND"
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

---

[← Previous](02-merchant-info.md) · [Index](README.md) · **03 — Create Customer** · [Next →](04-create-deposit.md)
