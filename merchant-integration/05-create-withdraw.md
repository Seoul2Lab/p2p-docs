[← Previous](04-create-deposit.md) · [Index](README.md) · **05 — Create Withdraw**

---

# 05 — Create Withdrawal Transaction

> **Authentication**: Required — `x-client-id`, `x-signature`, `x-timestamp`

Submit a withdrawal request to send funds to a customer's bank account. The system will process the withdrawal through an available bank channel.

---

## Table of Contents

- [POST /api/v1/client/tx/withdraw](#post-apiv1clienttxwithdraw)
  - [Request Headers](#request-headers)
  - [Request Body](#request-body)
  - [Response](#response)
  - [Example — cURL](#example--curl)
  - [Example — Node.js](#example--nodejs)
  - [Sample Response (201 Created)](#sample-response-201-created)
- [Transaction Statuses (Withdraw)](#transaction-statuses-withdraw)
  - [State Transition Diagram](#state-transition-diagram)
  - [Flow Scenarios](#flow-scenarios)
- [Error Responses](#error-responses)

---

## POST /api/v1/client/tx/withdraw

Creates a new withdrawal transaction.

### Request Headers

```
Content-Type: application/json
x-client-id:  your_client_id
x-signature:  computed_hmac_signature
x-timestamp:  1742385600
```

See [02 — Authenticated APIs](02-merchant-info.md#signature-calculation) for how to compute the signature.

### Request Body

| Field                  | Type   | Required | Description                                                |
| ---------------------- | ------ | -------- | ---------------------------------------------------------- |
| `customer_account_uuid`| string | Yes      | UUID of the customer account (from [03 — Create Customer](03-create-customer.md)) |
| `amount`               | number | Yes      | Withdrawal amount (must be > 0)                            |
| `currency`             | string | Yes      | Currency code (must be enabled for your merchant)          |
| `callback_url`         | string | Yes      | Webhook URL — called when transaction status changes       |
| `merchant_order_id`    | string | No       | Your internal order reference (must be unique per merchant) |

> **Note**: Unlike deposit, withdrawal does not require `payment_method` or `redirect_url`.

### Response

**201 Created**

| Field                      | Type     | Description                              |
| -------------------------- | -------- | ---------------------------------------- |
| `uuid`                     | string   | Unique transaction identifier            |
| `bank_account_number`      | string   | Customer's bank account number           |
| `bank_account_name`        | string   | Customer's bank account name             |
| `bank_code`                | string   | Customer's bank code                     |
| `bank_name_en`             | string   | Bank name in English                     |
| `bank_name_th`             | string   | Bank name in Thai                        |
| `customer_transfer_amount` | number   | Amount to be transferred                 |
| `amount`                   | number   | Original requested amount                |
| `customer_request_amount`  | number   | Amount requested by customer             |
| `fee`                      | number   | Transaction fee                          |
| `currency`                 | string   | Currency code                            |
| `type`                     | string   | `"withdraw"`                             |
| `transaction_status`       | string   | Current status (see [statuses](#transaction-statuses-withdraw)) |
| `transaction_status_message`| string  | Human-readable status message            |
| `payment_method`           | string   | Payment method used                      |
| `created_at`               | datetime | Creation timestamp                       |
| `updated_at`               | datetime | Last update timestamp                    |
| `redirect_url`             | string?  | Redirect URL (nullable)                  |
| `callback_url`             | string?  | Callback URL (nullable)                  |
| `merchant_order_id`        | string?  | Your order reference (nullable)          |

### Example — cURL

```bash
# Variables
CLIENT_ID="your_client_id"
CLIENT_SECRET="your_client_secret"
TIMESTAMP=$(date +%s)

# Request body
REQUEST_BODY='{"customer_account_uuid":"550e8400-e29b-41d4-a716-446655440000","amount":500,"currency":"THB","callback_url":"https://your-domain.com/webhook/withdraw","merchant_order_id":"WD-001"}'

# Build combined string
QUERY_STRING=""
COMBINED="${CLIENT_ID}|${TIMESTAMP}|${REQUEST_BODY}|${QUERY_STRING}"

# Compute HMAC-SHA256 signature
SIGNATURE=$(echo -n "$COMBINED" | openssl dgst -sha256 -hmac "$CLIENT_SECRET" | awk '{print $2}')

# Call the API
curl -X POST {BASE_URL}/api/v1/client/tx/withdraw \
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

async function createWithdraw() {
  const timestamp = Math.floor(Date.now() / 1000).toString();

  const body = {
    customer_account_uuid: "550e8400-e29b-41d4-a716-446655440000",
    amount: 500,
    currency: "THB",
    callback_url: "https://your-domain.com/webhook/withdraw",
    merchant_order_id: "WD-001",
  };

  const bodyString = JSON.stringify(body);
  const signature = generateSignature(CLIENT_ID, CLIENT_SECRET, timestamp, bodyString);

  const response = await fetch(`${BASE_URL}/api/v1/client/tx/withdraw`, {
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

createWithdraw();
```

### Sample Response (201 Created)

```json
{
  "status": "success",
  "data": {
    "uuid": "b2c3d4e5-f6a7-8901-bcde-f12345678901",
    "bank_account_number": "1234567890",
    "bank_account_name": "ทดสอบ ชื่อ",
    "bank_code": "KBANK",
    "bank_name_en": "Kasikorn Bank",
    "bank_name_th": "ธนาคารกสิกรไทย",
    "customer_transfer_amount": 500,
    "amount": 500,
    "customer_request_amount": 500,
    "fee": 5,
    "currency": "THB",
    "type": "withdraw",
    "transaction_status": "initial",
    "transaction_status_message": "Transaction created",
    "payment_method": "direct",
    "created_at": "2026-03-19T10:30:00Z",
    "updated_at": "2026-03-19T10:30:00Z",
    "redirect_url": null,
    "callback_url": "https://your-domain.com/webhook/withdraw",
    "merchant_order_id": "WD-001"
  }
}
```

---

## Transaction Statuses (Withdraw)

| Status        | Description                                                              | Final? |
| ------------- | ------------------------------------------------------------------------ | ------ |
| `initial`     | Transaction created, waiting to be matched                               | No     |
| `matching`    | Being matched with a payer                                               | No     |
| `assigned`    | Matched with a payer, withdrawal in progress                             | No     |
| `unassigned`  | No payer available, waiting to be re-matched or terminated               | No     |
| `success`     | Funds transferred to customer                                            | Yes    |
| `terminated`  | No payer found — no further action possible                              | Yes    |
| `canceled`    | Canceled by merchant (only when not peered with a deposit transaction)   | Yes    |

### State Transition Diagram

```mermaid
stateDiagram-v2
    [*] --> initial: POST /tx/withdraw

    state "Withdrawal in Progress" as inprogress {
        initial --> matching: Matching service picks up
        matching --> assigned: Payer found
    }

    state "Normal Flow" as normal {
        assigned --> success: Funds transferred ✓
    }

    state "No Payer Available" as nopayer {
        matching --> unassigned: No payer available
        unassigned --> terminated: Timeout (termination period)
    }

    state "Canceled by Merchant" as cancel {
        initial --> canceled: Cancel
        matching --> canceled: Cancel
        assigned --> canceled: Cancel
    }

    success --> [*]
    terminated --> [*]
    canceled --> [*]
```

> **Note**: Cancellation is only available when the withdrawal is **not peered** with a deposit transaction.

#### Flow Scenarios

**1. Withdrawal in progress**
```
initial → matching → assigned
```

**2. Normal flow**
```
initial → matching → assigned → success ✓
```

**3. No payer available**
```
initial → matching → unassigned → terminated ✓
```

**4. Canceled by merchant** (only when not peered with a deposit)
```
initial → canceled ✓
initial → matching → canceled ✓
initial → matching → assigned → canceled ✓
```

> **Final statuses**: `success`, `terminated`, `canceled` — no further transitions possible.

---

## Error Responses

### Invalid Request Body (400)

```json
{
  "status": "error",
  "message": "Field validation error",
  "code": "BAD_REQUEST"
}
```

### Customer Not Found (404)

```json
{
  "status": "error",
  "message": "Customer not found",
  "code": "CUSTOMER_NOT_FOUND"
}
```

### Withdrawal Not Active (422)

```json
{
  "status": "error",
  "message": "Withdrawal is not active for this merchant",
  "code": "WITHDRAW_NOT_ACTIVE"
}
```

### Unsupported Currency (422)

```json
{
  "status": "error",
  "message": "Unsupported currency",
  "code": "UNSUPPORTED_CURRENCY"
}
```

### Amount Below Minimum (422)

```json
{
  "status": "error",
  "message": "Amount less than minimum",
  "code": "AMOUNT_LESS_THAN_MINIMUM"
}
```

### Amount Exceeds Maximum (422)

```json
{
  "status": "error",
  "message": "Amount exceeds the maximum",
  "code": "AMOUNT_EXCEEDS_MAXIMUM"
}
```

### Transfer Amount Less Than Fee (422)

```json
{
  "status": "error",
  "message": "The transfer amount is less than the total fee",
  "code": "AMOUNT_LESS_THAN_FEE"
}
```

### Pending Withdrawal Exists (422)

Customer already has an active withdrawal that hasn't completed yet.

```json
{
  "status": "error",
  "message": "This customer still has a pending withdraw transaction",
  "code": "PENDING_WITHDRAW"
}
```

### Duplicate Merchant Order ID (422)

```json
{
  "status": "error",
  "message": "Duplicate merchant_order_id for this merchant",
  "code": "DUPLICATE_MERCHANT_ORDER_ID"
}
```

### No Bank Channel Available (422)

```json
{
  "status": "error",
  "message": "No withdrawal bank channel is available",
  "code": "NO_WITHDRAW_CHANNEL"
}
```

### System Under Maintenance (422)

```json
{
  "status": "error",
  "message": "System is under maintenance",
  "code": "SYSTEM_MAINTENANCE"
}
```

---

[← Previous](04-create-deposit.md) · [Index](README.md) · **05 — Create Withdraw**
