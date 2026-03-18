[← Previous](01-call-public-api.md) · [Index](README.md) · **02 — Authenticated APIs** · [Next →](03-create-customer.md)

---

# 02 — Authenticated APIs

> **Authentication**: Required — `x-client-id`, `x-signature`, `x-timestamp`

This page explains how to call APIs that require authentication. Every authenticated request must include the three headers below. The **Merchant Info** endpoint is used here as a working example to verify your credentials.

---

## Table of Contents

- [Authentication Headers](#authentication-headers)
- [Signature Calculation](#signature-calculation)
  - [Combined String Format](#combined-string-format)
  - [Steps](#steps)
- [Test Mode (Development Shortcut)](#test-mode-development-shortcut)
- [GET /api/v1/client/merchant-info](#get-apiv1clientmerchant-info)
  - [Request Headers](#request-headers)
  - [Response](#response)
  - [Sample Response (200 OK)](#sample-response-200-ok)
- [Example — cURL](#example--curl)
- [Example — Node.js](#example--nodejs)
- [Error Responses](#error-responses)

---

## Authentication Headers

| Header         | Required | Description                                                                 |
| -------------- | -------- | --------------------------------------------------------------------------- |
| `x-client-id`  | Yes      | Your merchant `client_id`                                                   |
| `x-signature`  | Yes      | HMAC-SHA256 hex digest of the combined string (see below)                   |
| `x-timestamp`  | Yes      | Current Unix epoch in **seconds** (must be within 5 minutes of server time) |

---

## Signature Calculation

The signature is an **HMAC-SHA256** hash of a combined string, using your `client_secret` as the key.

### Combined String Format

```
{client_id}|{timestamp}|{request_body}|{query_string}
```

| Part            | Description                                                         |
| --------------- | ------------------------------------------------------------------- |
| `client_id`     | Value of `x-client-id` header                                       |
| `timestamp`     | Value of `x-timestamp` header (Unix epoch in seconds)               |
| `request_body`  | Raw JSON request body. Use `{}` if the body is empty (GET requests) |
| `query_string`  | URL query string **without** the leading `?`. Empty string if none  |

### Steps

1. Get the current Unix timestamp (seconds).
2. Build the combined string: `{client_id}|{timestamp}|{request_body}|{query_string}`
3. Compute HMAC-SHA256 using your `client_secret` as the key.
4. Hex-encode the result — this is your `x-signature`.

---

## Test Mode (Development Shortcut)

If **test mode** is enabled for your merchant account, you can bypass signature verification:

- Set `x-signature` to the **same value** as `x-client-id`.
- `x-timestamp` is not required in test mode.

```bash
# Test mode example — signature equals client_id
curl -X GET {BASE_URL}/api/v1/client/merchant-info \
  -H "x-client-id: your_client_id" \
  -H "x-signature: your_client_id"
```

> **Warning**: Test mode is only for development. Always use proper HMAC signatures in production.

---

## GET /api/v1/client/merchant-info

Returns the current server epoch timestamp. Use this to verify your credentials are valid.

### Request Headers

```
x-client-id:  your_client_id
x-signature:  computed_hmac_signature
x-timestamp:  1742385600
```

### Response

| Field        | Type    | Description              |
| ------------ | ------- | ------------------------ |
| `status`     | string  | `"success"`              |
| `data.epoch` | integer | Current Unix timestamp   |

### Sample Response (200 OK)

```json
{
  "status": "success",
  "data": {
    "epoch": 1742385600
  }
}
```

---

## Example — cURL

```bash
# Variables
CLIENT_ID="your_client_id"
CLIENT_SECRET="your_client_secret"
TIMESTAMP=$(date +%s)
REQUEST_BODY="{}"
QUERY_STRING=""

# Build combined string
COMBINED="${CLIENT_ID}|${TIMESTAMP}|${REQUEST_BODY}|${QUERY_STRING}"

# Compute HMAC-SHA256 signature
SIGNATURE=$(echo -n "$COMBINED" | openssl dgst -sha256 -hmac "$CLIENT_SECRET" | awk '{print $2}')

# Call the API
curl -X GET {BASE_URL}/api/v1/client/merchant-info \
  -H "x-client-id: ${CLIENT_ID}" \
  -H "x-signature: ${SIGNATURE}" \
  -H "x-timestamp: ${TIMESTAMP}"
```

---

## Example — Node.js

```javascript
const crypto = require("crypto");

const CLIENT_ID = "your_client_id";
const CLIENT_SECRET = "your_client_secret";
const BASE_URL = "{BASE_URL}";

/**
 * Generate HMAC-SHA256 signature for API authentication.
 *
 * @param {string} clientId     - Merchant client ID
 * @param {string} clientSecret - Merchant client secret
 * @param {string} timestamp    - Unix epoch in seconds
 * @param {string} body         - JSON request body (use "{}" for GET requests)
 * @param {string} queryString  - URL query string without leading "?"
 * @returns {string} Hex-encoded HMAC-SHA256 signature
 */
function generateSignature(clientId, clientSecret, timestamp, body = "{}", queryString = "") {
  const combined = `${clientId}|${timestamp}|${body}|${queryString}`;
  return crypto
    .createHmac("sha256", clientSecret)
    .update(combined)
    .digest("hex");
}

async function getMerchantInfo() {
  const timestamp = Math.floor(Date.now() / 1000).toString();
  const signature = generateSignature(CLIENT_ID, CLIENT_SECRET, timestamp);

  const response = await fetch(`${BASE_URL}/api/v1/client/merchant-info`, {
    method: "GET",
    headers: {
      "x-client-id": CLIENT_ID,
      "x-signature": signature,
      "x-timestamp": timestamp,
    },
  });

  const data = await response.json();
  console.log("Response:", JSON.stringify(data, null, 2));
}

getMerchantInfo();
```

### Output

```json
{
  "status": "success",
  "data": {
    "epoch": 1742385600
  }
}
```

---

## Error Responses

### Missing `x-client-id`

```json
{
  "status": "error",
  "message": "x-client-id header is required",
  "code": "CLIENT_ID_HEADER_REQUIRED"
}
```

### Missing `x-signature`

```json
{
  "status": "error",
  "message": "x-signature header is required",
  "code": "SIGNATURE_HEADER_REQUIRED"
}
```

### Missing `x-timestamp`

```json
{
  "status": "error",
  "message": "x-timestamp header is required",
  "code": "TIMESTAMP_HEADER_REQUIRED"
}
```

### Client Not Found

```json
{
  "status": "error",
  "message": "Client not found",
  "code": "CLIENT_NOT_FOUND"
}
```

### Invalid Signature

```json
{
  "status": "error",
  "message": "Invalid signature",
  "code": "INVALID_SIGNATURE"
}
```

### Request Expired (timestamp outside 5-minute window)

```json
{
  "status": "error",
  "message": "Request expired",
  "code": "REQUEST_EXPIRED"
}
```

---

[← Previous](01-call-public-api.md) · [Index](README.md) · **02 — Authenticated APIs** · [Next →](03-create-customer.md)
