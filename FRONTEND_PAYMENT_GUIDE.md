# Qwickcare Payment System — Frontend Integration Guide

This document walks through every step a needed to implement the full payment flow: funding a wallet, paying for a service, confirming completion, and handling refunds.

---

## Overview of the Flow

```
Patient funds wallet  →  Pays for service  →  Provider marks complete
→  Patient confirms  →  Provider gets paid  →  Provider requests payout
```

All requests require a `Bearer` token in the `Authorization` header.

---

## Idempotency — Required for All Mutation Requests

Several endpoints require an `Idempotency-Key` header to prevent accidental duplicate operations (double charges, double payouts, double settlements). Requests without the header on protected routes will be rejected with `400`.

### How it works

- Generate a UUID v4 on the client before making the request
- Send it as a header: `Idempotency-Key: <uuid-v4>`
- If the request succeeds and you send the **same key again** (e.g. after a network timeout), the server returns the **original cached response** without processing it again
- If the request failed, you can safely retry with the **same key** — the server will process it fresh
- Keys expire after 24 hours

### Which endpoints require it

| Endpoint                                   | Why                                 |
| ------------------------------------------ | ----------------------------------- |
| `POST /payments`                           | Prevents double charge              |
| `POST /wallets/patient/fund/card`          | Prevents duplicate funding          |
| `POST /wallets/patient/fund/bank-transfer` | Prevents duplicate transfer record  |
| `POST /wallets/provider/payout`            | Prevents double payout              |
| `POST /settlements/:paymentId/complete`    | Prevents double settlement creation |
| `POST /settlements/:settlementId/confirm`  | Prevents double wallet credit       |
| `POST /refunds`                            | Prevents duplicate refund request   |
| `POST /refunds/:id/approve`                | Prevents double refund credit       |

### Implementation pattern

```javascript
import { v4 as uuidv4 } from "uuid";

async function createPayment(payload) {
  // Generate once per user action — NOT per retry
  const idempotencyKey = uuidv4();

  const response = await fetch("/api/v1/payments", {
    method: "POST",
    headers: {
      Authorization: `Bearer ${token}`,
      "Content-Type": "application/json",
      "Idempotency-Key": idempotencyKey, // ← required
    },
    body: JSON.stringify(payload),
  });

  return response.json();
}
```

> **Important:** Generate the key once when the user taps the button, then reuse that same key for retries of the same action. Generating a new key on each retry defeats the purpose.The key should be deleted once the payment is successfull.

---

## Step 1 — Fund Patient Wallet

The patient must have funds in their wallet before they can pay for any service.

### Option A — Card Payment (recommended)

**`POST /wallets/patient/fund/card`** · requires `Idempotency-Key`

```json
// Request
{
  "amount": 5000,
  "currency": "NGN"
}

// Response
{
  "paymentLink": "https://checkout.flutterwave.com/v3/hosted/pay/...",
  "txRef": "WFT-abc123",
  "amount": 5000
}
```

Redirect the patient to `paymentLink`. Flutterwave will call your webhook when payment is confirmed and the wallet will be credited automatically.

After Flutterwave redirects back, call `GET /payments/verify?tx_ref=WFT-abc123&transaction_id=...&status=successful` to confirm the wallet was credited — this handles cases where the webhook is delayed.

---

### Option B — Virtual Bank Account

**`GET /wallets/patient/virtual-account`**

```json
// Response
{
  "accountNumber": "0123456789",
  "bankName": "Wema Bank",
  "accountName": "John Doe - Healthcare Wallet",
  "instructions": "Transfer any amount to this account to fund your wallet. Funds will be credited automatically."
}
```

---

### Option C — Manual Bank Transfer (admin-approved)

**`POST /wallets/patient/fund/bank-transfer`** · requires `Idempotency-Key`

```json
// Request
{
  "amount": 10000,
  "senderAccountNumber": "0987654321",
  "senderBankName": "Access Bank",
  "senderAccountName": "Jane Doe",
  "transferDate": "2024-06-24",
  "referenceNumber": "TRF20240624001",
  "proofOfPayment": "https://cdn.example.com/receipt.jpg"
}

// Response
{
  "id": "uuid",
  "status": "pending",
  "amount": 10000,
  "message": "Transfer recorded. Pending admin verification."
}
```

---

### Check Wallet Balance

**`GET /wallets/balance/:type`** — `:type` is `patient` or `provider`

```json
// Response
{
  "balance": 15000.0,
  "heldBalance": 5000.0,
  "currency": "NGN"
}
```

> `heldBalance` is money locked in escrow for in-progress services. The spendable amount is `balance - heldBalance`.

---

## Step 2 — Pay for a Service

**`POST /payments`** · requires `Idempotency-Key`

You must link the payment to a service using **one of two approaches**. Do not send both in the same request.

### Option A — Generic service

```json
// Request
{
  "patientId": "patient-uuid",
  "providerId": "provider-uuid",
  "serviceId": "service-uuid", // UUID of the specific service record
  "serviceType": "consultation", // required when using serviceId
  "paymentType": "consultation",
  "amount": 5000,
  "currency": "NGN",
  "description": "Online consultation with Dr. Smith"
}
```

### Option B — appointment Flow

```json
// Request
{
  "patientId": "patient-uuid",
  "providerId": "provider-uuid",
  "appointmentId": "appointment-uuid", // use only if not using serviceId
  "paymentType": "appointment",
  "amount": 5000,
  "currency": "NGN"
}
```

### Response (same for both)

```json
{
  "id": "payment-uuid",
  "transactionReference": "Qwickcare-abc123",
  "status": "completed",
  "amount": 5000,
  "currency": "NGN",
  "patientId": "patient-uuid",
  "providerId": "provider-uuid",
  "serviceId": "service-uuid", // null for appointmentId payments
  "serviceType": "consultation", // null for appointmentId payments
  "appointmentId": null, // null for serviceId payments
  "paidAt": "2024-06-24T10:00:00Z",
  "platformCommission": 500,
  "transactionFee": 0,
  "netProviderEarnings": 4500
}
```

**Payment status after this call is `completed`.** Money is debited from the patient wallet and held in escrow — the provider has not received it yet.

### `serviceType` values

| Value          | Use for                      |
| -------------- | ---------------------------- |
| `consultation` | Online/offline consultations |
| `appointment`  | General appointments         |
| `treatment`    | Treatments or procedures     |
| `diagnostic`   | Diagnostic services          |
| `medication`   | Medication orders            |
| `home_service` | Home visit services          |
| `lab_test`     | Lab tests                    |
| `prescription` | Prescriptions                |
| `surgery`      | Surgical procedures          |
| `therapy`      | Therapy sessions             |
| `vaccination`  | Vaccinations                 |
| `follow_up`    | Follow-up visits             |

---

## Step 3 — Provider Marks Service Complete

Called by the **provider's** frontend after the service has been delivered.

**`POST /settlements/:paymentId/complete`** · requires `Idempotency-Key`

No request body — provider ID is read from the JWT.

```json
// Response
{
  "id": "settlement-uuid",
  "paymentId": "payment-uuid",
  "status": "awaiting_confirmation",
  "grossAmount": 5000,
  "platformCommission": 500,
  "transactionFee": 0,
  "netAmount": 4500
}
```

After this, the payment status moves to `awaiting_patient_confirmation`. Show the patient a prompt to confirm.

---

## Step 4 — Patient Confirms Service Completion

**`POST /settlements/:settlementId/confirm`** · requires `Idempotency-Key`

```json
// Request (body is optional)
{
  "notes": "Great consultation, very helpful."
}

// Response
{
  "id": "settlement-uuid",
  "status": "settled",
  "paymentId": "payment-uuid",
  "netAmount": 4500,
  "settledAt": "2024-06-24T11:30:00Z"
}
```

After this, the provider wallet is credited with `netAmount` and the payment status becomes `settled`.

---

## Step 5 — Provider Requests Payout

### First: Add a bank account (one-time setup)

**`GET /banks`** — Get supported banks

**`GET /banks/verify?accountNumber=0123456789&bankCode=044`** — Verify account before saving

**`POST /wallets/provider/bank-details`** — Save bank account

```json
// Request
{
  "bankCode": "044",
  "bankName": "Access Bank",
  "accountNumber": "0123456789"
}
```

### Then: Request payout

**`POST /wallets/provider/payout`** · requires `Idempotency-Key`

```json
// Request
{
  "amount": 4500,
  "bankDetailId": "bank-detail-uuid"
}

// Response
{
  "id": "payout-uuid",
  "status": "processing",
  "amount": 4500,
  "bankName": "Access Bank",
  "accountNumber": "0123456789",
  "transferReference": "PO-xyz789",
  "createdAt": "2024-06-24T12:00:00Z"
}
```

Flutterwave processes the transfer. Status updates to `completed` via webhook.

---

## Refunds

Patients can request a refund. Refunds are only possible before the provider has initiated a payout — once money has left the platform, it requires manual support resolution.

**`POST /refunds`** · requires `Idempotency-Key`

```json
// Request
{
  "paymentId": "payment-uuid",
  "amount": 5000,
  "reason": "Service was not delivered as described."
}

// Response
{
  "id": "refund-uuid",
  "status": "pending",
  "amount": 5000,
  "paymentId": "payment-uuid",
  "reason": "Service was not delivered as described.",
  "createdAt": "2024-06-24T13:00:00Z"
}
```

Admin approves or rejects. On approval the patient wallet is credited. Track status via `GET /refunds/patient/:patientId`.

---

## Fetching History

| What                        | Endpoint                                        |
| --------------------------- | ----------------------------------------------- |
| Wallet balance              | `GET /wallets/balance/:type`                    |
| Wallet transactions         | `GET /wallets/transactions/:walletId`           |
| Patient payments            | `GET /payments/patient/:patientId`              |
| Provider payments           | `GET /payments/provider/:providerId`            |
| Payments for a service      | `GET /payments/service/:serviceType/:serviceId` |
| Payments for an appointment | `GET /payments/appointment/:appointmentId`      |
| Payment by ID               | `GET /payments/:id`                             |
| Patient settlements         | `GET /settlements/patient/:patientId`           |
| Provider settlements        | `GET /settlements/provider/:providerId`         |
| Patient refunds             | `GET /refunds/patient/:patientId`               |
| Provider payouts            | `GET /wallets/provider/payouts`                 |
| Funding history             | `GET /wallets/patient/funding-history`          |

All list endpoints accept `?page=1&limit=20` query params.

---

## Payment Status Reference

| Status                          | Meaning                                           |
| ------------------------------- | ------------------------------------------------- |
| `pending`                       | Payment initiated, not yet processed              |
| `processing`                    | Being processed                                   |
| `completed`                     | Paid — funds in escrow                            |
| `awaiting_patient_confirmation` | Provider marked service done, waiting for patient |
| `settled`                       | Patient confirmed — provider credited             |
| `refunded`                      | Fully refunded                                    |
| `partially_refunded`            | Partial refund issued                             |
| `failed`                        | Payment failed                                    |
| `cancelled`                     | Cancelled                                         |

---

## Error Responses

```json
{
  "statusCode": 400,
  "message": "Insufficient wallet balance",
  "error": "Bad Request"
}
```

Common codes: `400` bad request · `401` unauthenticated · `403` forbidden · `404` not found · `409` conflict (duplicate idempotency key in flight)
