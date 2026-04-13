[← Previous](05-create-withdraw.md) · [Index](README.md) · **06 — Callback (Webhook)** · [Next →](07-query-transaction-status.md)

---

# 06 — Callback (Webhook)

> **Direction**: Server → Merchant (the platform calls **your** endpoint)

When a transaction status changes, the system sends an **HMAC-signed POST request** to the `callback_url` you provided when creating the deposit or withdrawal.

---

## Table of Contents

- [How It Works](#how-it-works)
- [Callback Headers](#callback-headers)
- [Callback Payload](#callback-payload)
- [Signature Verification](#signature-verification)
  - [Steps](#steps)
  - [Example — Node.js](#example--nodejs)
  - [Example — Python](#example--python)
  - [Example — Go](#example--go)
- [Expected Response](#expected-response)
- [Retry Behavior](#retry-behavior)
- [Callback Log](#callback-log)
- [Best Practices](#best-practices)

---

## How It Works

```
┌──────────────┐   status change   ┌──────────────┐
│   Platform   │ ── POST ────────► │  Your Server │
│   (P2P)      │                   │ callback_url │
└──────────────┘ ◄── HTTP 200 ──── └──────────────┘
```

1. Transaction status changes (e.g., `initial` → `assigned` → `success`).
2. Platform builds a JSON payload with the updated transaction info.
3. Platform signs the payload with **HMAC-SHA256** using your `client_secret`.
4. Platform sends a **POST** request to your `callback_url`.
5. Your server verifies the signature and processes the update.

---

## Callback Headers

| Header         | Description                                                                    |
| -------------- | ------------------------------------------------------------------------------ |
| `Content-Type` | `application/json`                                                             |
| `x-signature`  | HMAC-SHA256 hex digest (see [Signature Verification](#signature-verification)) |
| `x-timestamp`  | Unix epoch in seconds when the callback was sent                               |

---

## Callback Payload

The request body is a JSON object with the following fields:

| Field                      | Type   | Description                                             |
| -------------------------- | ------ | ------------------------------------------------------- |
| `uuid`                     | string | Transaction UUID                                        |
| `amount`                   | number | Original requested amount                               |
| `customer_transfer_amount` | number | Exact amount the customer transferred                   |
| `customer_request_amount`  | number | Amount requested by customer                            |
| `fee`                      | number | Transaction fee                                         |
| `currency`                 | string | Currency code (e.g., `"THB"`)                           |
| `type`                     | string | `"deposit"` or `"withdraw"`                             |
| `transaction_status`       | string | Current status (see status tables below)                |
| `payment_method`           | string | Payment method: `"auto"`, `"qr"`, or `"direct"`         |
| `merchant_order_id`        | string | Your internal order reference (empty string if not set) |
| `bank_code`                | string | Bank code (e.g., `"KBANK"`)                             |
| `bank_account_number`      | string | Bank account number                                     |
| `bank_account_name`        | string | Bank account holder name                                |

### Sample Payload

```json
{
	"uuid": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
	"amount": 1000,
	"customer_transfer_amount": 1000.25,
	"customer_request_amount": 1000,
	"fee": 10,
	"currency": "THB",
	"type": "deposit",
	"transaction_status": "success",
	"payment_method": "auto",
	"merchant_order_id": "ORDER-001",
	"bank_code": "KBANK",
	"bank_account_number": "9876543210",
	"bank_account_name": "บัญชีรับเงิน"
}
```

---

## Signature Verification

You **should** verify the `x-signature` header to ensure the callback is genuinely from the platform and has not been tampered with.

The signature is computed as:

```
HMAC-SHA256(client_secret, "{your_client_id}|{x-timestamp}|{raw_request_body}|")
```

> **Note**: The combined string ends with a trailing `|` (empty query string part).

### Steps

1. Read the raw request body (do not parse/re-serialize — use the raw bytes).
2. Read the `x-timestamp` header.
3. Build the combined string: `{your_client_id}|{x-timestamp}|{raw_body}|`
4. Compute HMAC-SHA256 using your `client_secret` as the key.
5. Hex-encode the result and compare with the `x-signature` header.

### Example — Node.js

```javascript
const crypto = require("crypto");

const CLIENT_ID = "your_client_id";
const CLIENT_SECRET = "your_client_secret";

function verifyCallback(req) {
	const signature = req.headers["x-signature"];
	const timestamp = req.headers["x-timestamp"];
	const rawBody = req.rawBody; // raw request body as string

	const combined = `${CLIENT_ID}|${timestamp}|${rawBody}|`;
	const expected = crypto
		.createHmac("sha256", CLIENT_SECRET)
		.update(combined)
		.digest("hex");

	return crypto.timingSafeEqual(
		Buffer.from(signature, "hex"),
		Buffer.from(expected, "hex"),
	);
}

// Express.js example
const express = require("express");
const app = express();

// Important: capture raw body for signature verification
app.use(
	"/webhook",
	express.json({
		verify: (req, res, buf) => {
			req.rawBody = buf.toString();
		},
	}),
);

app.post("/webhook/deposit", (req, res) => {
	if (!verifyCallback(req)) {
		return res.status(401).json({ error: "Invalid signature" });
	}

	const { uuid, transaction_status, amount, type } = req.body;
	console.log(
		`Transaction ${uuid} (${type}): ${transaction_status}, amount: ${amount}`,
	);

	// Process the callback (update your database, notify user, etc.)

	res.status(200).json({ received: true });
});
```

### Example — Python

```python
import hmac
import hashlib

CLIENT_ID = "your_client_id"
CLIENT_SECRET = "your_client_secret"

def verify_callback(signature: str, timestamp: str, raw_body: str) -> bool:
    combined = f"{CLIENT_ID}|{timestamp}|{raw_body}|"
    expected = hmac.new(
        CLIENT_SECRET.encode(),
        combined.encode(),
        hashlib.sha256
    ).hexdigest()
    return hmac.compare_digest(signature, expected)

# Flask example
from flask import Flask, request, jsonify

app = Flask(__name__)

@app.route("/webhook/deposit", methods=["POST"])
def handle_callback():
    signature = request.headers.get("x-signature", "")
    timestamp = request.headers.get("x-timestamp", "")
    raw_body = request.get_data(as_text=True)

    if not verify_callback(signature, timestamp, raw_body):
        return jsonify({"error": "Invalid signature"}), 401

    data = request.get_json()
    print(f"Transaction {data['uuid']}: {data['transaction_status']}")

    return jsonify({"received": True}), 200
```

### Example — Go

```go
package main

import (
	"crypto/hmac"
	"crypto/sha256"
	"encoding/hex"
	"fmt"
	"io"
	"net/http"
)

const (
	clientID     = "your_client_id"
	clientSecret = "your_client_secret"
)

func verifyCallback(signature, timestamp, rawBody string) bool {
	combined := fmt.Sprintf("%s|%s|%s|", clientID, timestamp, rawBody)
	mac := hmac.New(sha256.New, []byte(clientSecret))
	mac.Write([]byte(combined))
	expected := hex.EncodeToString(mac.Sum(nil))
	return hmac.Equal([]byte(signature), []byte(expected))
}

func webhookHandler(w http.ResponseWriter, r *http.Request) {
	body, _ := io.ReadAll(r.Body)
	signature := r.Header.Get("x-signature")
	timestamp := r.Header.Get("x-timestamp")

	if !verifyCallback(signature, timestamp, string(body)) {
		http.Error(w, `{"error":"Invalid signature"}`, http.StatusUnauthorized)
		return
	}

	fmt.Printf("Callback received: %s\n", string(body))
	w.WriteHeader(http.StatusOK)
	w.Write([]byte(`{"received":true}`))
}
```

---

## Expected Response

Your server should return an **HTTP 200** response to acknowledge receipt. The response body is not strictly required, but returning a simple JSON object is recommended:

```json
{
	"received": true
}
```

| Response Code | Meaning                                           |
| ------------- | ------------------------------------------------- |
| `200`         | Callback acknowledged — platform marks as success |
| Any other     | Platform marks the callback attempt as failed     |

---

## Retry Behavior

- The platform sends the callback **once** per status change.
- If the callback fails (non-200 response or network error), the attempt is logged but **not automatically retried**.
- You can check the callback history via [Query Transaction Status](07-query-transaction-status.md).

---

## Callback Log

Every callback attempt is recorded by the platform. Each log entry contains:

| Field                | Type     | Description                           |
| -------------------- | -------- | ------------------------------------- |
| `transaction_uuid`   | string   | Transaction UUID                      |
| `transaction_status` | string   | Status at the time of callback        |
| `callback_url`       | string   | The URL that was called               |
| `request_body`       | string   | JSON payload that was sent            |
| `response_body`      | string   | Your server's response body           |
| `response_status`    | string   | HTTP status code from your server     |
| `error_message`      | string   | Error message (if the request failed) |
| `is_success`         | boolean  | Whether the callback was successful   |
| `created_at`         | datetime | Timestamp of the callback attempt     |

---

## Transaction Statuses That Trigger Callbacks

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

> **Important**: Always use `transaction_status` from the callback payload to update your records. Do not rely on the order of callbacks — network latency may cause them to arrive out of order. Use the status value itself as the source of truth.

---

## Best Practices

1. **Verify the signature** — Always validate `x-signature` before processing. Reject requests with invalid signatures.
2. **Use the raw body** — Parse JSON only after signature verification. Do not re-serialize the body for signature computation.
3. **Respond quickly** — Return HTTP 200 as fast as possible. Process the business logic asynchronously if needed.
4. **Be idempotent** — Your callback handler should handle receiving the same status update more than once without side effects.
5. **Use `merchant_order_id`** — Map callbacks to your internal orders using this field.
6. **Check `transaction_status`** — Only take action on final statuses (`success`, `terminated`, `canceled`) for settling orders.
7. **Log everything** — Keep your own log of received callbacks for debugging and reconciliation.
8. **Use HTTPS** — Always use an HTTPS `callback_url` in production to protect the payload in transit.

---

[← Previous](05-create-withdraw.md) · [Index](README.md) · [Next →](07-query-transaction-status.md)
