# Merchant Integration Guide

API documentation for merchants integrating with the P2P platform.

**Swagger UI**: `{BASE_URL}/client-docs/index.html`

**Base URL**: `{BASE_URL}`

---

## Table of Contents

- [Prerequisites](#prerequisites)
- [Pages](#pages)
- [Quick Overview](#quick-overview)
  - [Integration Flow](#integration-flow)
  - [Authentication](#authentication)
  - [Response Format](#response-format)

---

## Prerequisites

Before you begin, make sure you have received the following from the platform administrator:

| Credential         | Description                                       |
| ------------------ | ------------------------------------------------- |
| `client_id`        | Your unique merchant identifier                   |
| `client_secret`    | Secret key used for HMAC-SHA256 signature         |
| `enable_test_mode` | If enabled, you can bypass signature verification |

---

## Pages

| #   | Page                                                       | Description                                            |
| --- | ---------------------------------------------------------- | ------------------------------------------------------ |
| 01  | [Public APIs](01-call-public-api.md)                       | APIs that require no authentication                    |
| 02  | [Authenticated APIs](02-call-api-with-authentication.md)   | How to call APIs with `x-client-id` & HMAC signature   |
| 03  | [Create Customer](03-create-customer.md)                   | Register a customer bank account                       |
| 04  | [Create Deposit](04-create-deposit.md)                     | Submit a deposit transaction, get payment link, cancel |
| 05  | [Create Withdraw](05-create-withdraw.md)                   | Submit a withdrawal transaction, cancel                |
| 06  | [Callback (Webhook)](06-callback-webhook.md)               | How the platform notifies you of status changes        |
| 07  | [Query Transaction Status](07-query-transaction-status.md) | Check a transaction's current status                   |

---

## Quick Overview

### Integration Flow

```
1. Call GET /api/v1/client/bank          → Get list of supported banks (public, no auth)
2. Call GET /api/v1/client/merchant-info  → Verify your credentials work (authenticated)
3. Call POST /api/v1/client/customer      → Create a customer bank account
4. Call POST /api/v1/client/tx/deposit    → Submit a deposit transaction
5. Call GET  /api/v1/client/tx/payment-link/:uuid → Get payment link info
6. Customer uploads slip via payment-link page → POST /api/v1/payment-link/upload
7. Call POST /api/v1/client/tx/cancel-deposit  → Cancel a deposit
8. Call POST /api/v1/client/tx/withdraw   → Submit a withdrawal transaction
9. Call POST /api/v1/client/tx/cancel-withdraw → Cancel a withdrawal
```

### Authentication

Most endpoints require three HTTP headers:

| Header        | Required | Description                                                               |
| ------------- | -------- | ------------------------------------------------------------------------- |
| `x-client-id` | Yes      | Your merchant `client_id`                                                 |
| `x-signature` | Yes      | HMAC-SHA256 signature (see [page 02](02-call-api-with-authentication.md)) |
| `x-timestamp` | Yes      | Current Unix epoch in seconds                                             |

> **Test Mode**: If test mode is enabled for your merchant, you can set `x-signature` equal to your `x-client-id` value to bypass signature verification during development.

### Response Format

**Success**

```json
{
  "status": "success",
  "data": { ... }
}
```

**Error**

```json
{
	"status": "error",
	"message": "Description of the error",
	"code": "ERROR_CODE"
}
```

---

**Next →** [01 — Public APIs](01-call-public-api.md)
