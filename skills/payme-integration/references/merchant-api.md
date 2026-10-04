# Merchant API — Complete Method Reference

## Method Index

Merchant API has 6 mandatory methods + 1 optional:

| # | Method | Purpose |
|---|--------|---------|
| 1 | CheckPerformTransaction | Check if payment is possible |
| 2 | CreateTransaction | Create a financial transaction |
| 3 | PerformTransaction | Complete the transaction (credit funds) |
| 4 | CancelTransaction | Cancel created or completed transaction |
| 5 | CheckTransaction | Check transaction state |
| 6 | GetStatement | List transactions for reconciliation |
| 7 | SetFiscalData *(optional)* | Receive fiscal data from Payme |

## Protocol Overview

Payme Business sends JSON-RPC 2.0 requests to your Endpoint URL via HTTP POST over TLS.
Your server must always respond with HTTP status 200.

### Request Format
```json
{
  "method": "MethodName",
  "params": { ... },
  "id": 2032
}
```

Headers:
```
POST https://your-endpoint.com/api/payment/payme HTTP/1.1
Content-Type: text/json; charset=UTF-8
Authorization: Basic base64(login:password)
```

### Response Format — Success
```json
{
  "result": { ... },
  "id": 2032
}
```

### Response Format — Error
```json
{
  "error": {
    "code": -31050,
    "message": {
      "ru": "Номер телефона не найден",
      "uz": "Raqam ro'yhatda yo'q",
      "en": "Phone number not found"
    },
    "data": "phone"
  },
  "id": 2032
}
```

---

## Method 1: CheckPerformTransaction

Checks if payment is possible before creating a transaction.

**When called:** Before any transaction — validates account and amount.

### Request
```json
{
  "method": "CheckPerformTransaction",
  "params": {
    "amount": 500000,
    "account": {
      "order_id": "197"
    }
  }
}
```

| Param | Type | Description |
|-------|------|-------------|
| amount | Number | Payment amount in tiyin |
| account | Object | Consumer account fields (defined by your business logic) |

### Response — Success
```json
{
  "result": {
    "allow": true
  }
}
```

### Response — With Fiscalization Detail
```json
{
  "result": {
    "allow": true,
    "additional": {
      "field_name": "field_value"
    },
    "detail": {
      "receipt_type": 0,
      "shipping": {
        "title": "Delivery to address",
        "price": 500000
      },
      "items": [
        {
          "discount": 10000,
          "title": "Product Name",
          "price": 505000,
          "count": 2,
          "code": "00702001001000001",
          "units": 241092,
          "vat_percent": 15,
          "package_code": "123456"
        }
      ]
    }
  }
}
```

### Errors
| Code | When |
|------|------|
| -31001 | Amount doesn't match order |
| -31050 to -31099 | Invalid account field. `message` must be localized. `data` must contain the account field name |

### Implementation Checklist
- [ ] Validate amount > 0
- [ ] Check account exists (order_id is valid)
- [ ] Check order amount matches `amount` param
- [ ] Check order is in payable state
- [ ] Return `detail` object for fiscalization
- [ ] Check all systems (DB, etc.) are operational — if not, return -32400

---

## Method 2: CreateTransaction

Creates a financial transaction in your billing system.

**When called:** After successful CheckPerformTransaction.

### Request
```json
{
  "method": "CreateTransaction",
  "params": {
    "id": "5305e3bab097f420a62ced0b",
    "time": 1399114284039,
    "amount": 500000,
    "account": {
      "order_id": "197"
    }
  }
}
```

| Param | Type | Description |
|-------|------|-------------|
| id | String (24 chars) | Payme transaction ID |
| time | Timestamp (13 digits) | Payme transaction creation time |
| amount | Number | Amount in tiyin |
| account | Object | Consumer account |

### Response
```json
{
  "result": {
    "create_time": 1399114284039,
    "transaction": "5123",
    "state": 1
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| create_time | Timestamp | Creation time in your system |
| transaction | String | Your internal transaction ID |
| state | Number | Transaction state (must be `1`) |
| receivers | Array/null | Optional list of payment receivers |

### Chain Payment (Multiple Receivers)
```json
{
  "result": {
    "create_time": 1399114284039,
    "transaction": "5123",
    "state": 1,
    "receivers": [
      { "id": "5305e3bab097f420a62ced0b", "amount": 200000 },
      { "id": "4215e6bab097f420a62ced01", "amount": 300000 }
    ]
  }
}
```

### Errors
| Code | When |
|------|------|
| -31001 | Invalid amount |
| -31008 | Cannot perform operation (order already being paid, etc.) |
| -31050 to -31099 | Invalid account |

### Implementation Checklist
- [ ] Store transaction persistently (DB, not in-memory)
- [ ] Validate account exists
- [ ] Validate amount matches order
- [ ] **Lock/reserve the order** — prevent modifications
- [ ] Set order status to "awaiting payment"
- [ ] If transaction already exists with same `id` — return existing data (idempotent)
- [ ] If transaction exists with same account but **different** `id` and state=1 — return error -31008
- [ ] Auto-cancel after timeout: 12 hours (43,200,000 ms) → state=-1, reason=4

---

## Method 3: PerformTransaction

Completes the transaction — credits funds and marks order as paid.

**When called:** After successful debit from customer's card.

### Request
```json
{
  "method": "PerformTransaction",
  "params": {
    "id": "5305e3bab097f420a62ced0b"
  }
}
```

### Response
```json
{
  "result": {
    "transaction": "5123",
    "perform_time": 1399114284039,
    "state": 2
  }
}
```

### Errors
| Code | When |
|------|------|
| -31003 | Transaction not found |
| -31008 | Cannot perform (wrong state — not state 1) |

### Implementation Checklist
- [ ] Find transaction by Payme `id`
- [ ] Verify state is `1` (created)
- [ ] Credit merchant account
- [ ] Set order status to "paid"
- [ ] Set state to `2`, record `perform_time`
- [ ] If already performed (state=2) — return same result (idempotent)
- [ ] If cancelled (state=-1) — return error -31008

---

## Method 4: CancelTransaction

Cancels both created (state=1) and completed (state=2) transactions.

### Request
```json
{
  "method": "CancelTransaction",
  "params": {
    "id": "5305e3bab097f420a62ced0b",
    "reason": 1
  }
}
```

### Response
```json
{
  "result": {
    "transaction": "5123",
    "cancel_time": 1399114284039,
    "state": -1
  }
}
```

State logic:
- If was state 1 (created) → becomes **-1**
- If was state 2 (performed) → becomes **-2**

### Errors
| Code | When |
|------|------|
| -31003 | Transaction not found |
| -31007 | Order fulfilled, cannot cancel (goods delivered) |

### Implementation Checklist
- [ ] Find transaction by Payme `id`
- [ ] If state=1: cancel, set state=-1, restore order to original state
- [ ] If state=2: cancel, set state=-2 (refund scenario)
- [ ] If already cancelled: return existing cancel data (idempotent)
- [ ] Record `cancel_time` and `reason`

---

## Method 5: CheckTransaction

Returns current state of a transaction.

### Request
```json
{
  "method": "CheckTransaction",
  "params": {
    "id": "5305e3bab097f420a62ced0b"
  }
}
```

### Response
```json
{
  "result": {
    "create_time": 1399114284039,
    "perform_time": 1399114285002,
    "cancel_time": 0,
    "transaction": "5123",
    "state": 2,
    "reason": null
  }
}
```

### Errors
| Code | When |
|------|------|
| -31003 | Transaction not found |

---

## Method 6: GetStatement

Returns transaction list for reconciliation. **Implementation is MANDATORY.**

### Request
```json
{
  "method": "GetStatement",
  "params": {
    "from": 1399114284039,
    "to": 1399120284000
  }
}
```

### Response
```json
{
  "result": {
    "transactions": [
      {
        "id": "5305e3bab097f420a62ced0b",
        "time": 1399114284039,
        "amount": 500000,
        "account": { "order_id": "197" },
        "create_time": 1399114284039,
        "perform_time": 1399114285002,
        "cancel_time": 0,
        "transaction": "5123",
        "state": 2,
        "reason": null,
        "receivers": null
      }
    ]
  }
}
```

### Implementation Rules
- Search by **Payme creation time** (the `time` field from CreateTransaction)
- Include ALL transactions where: `from <= time <= to`
- Sort by creation time **ascending**
- Only include transactions that were successfully created (CreateTransaction returned without error)
- Return empty array if no transactions in period

---

## Method 7: SetFiscalData (Optional)

Receives fiscal data from Payme after successful payment. Not mandatory to implement.

### Request — Payment Fiscal
```json
{
  "method": "SetFiscalData",
  "params": {
    "id": "61396aaed8b87a4c215ae556",
    "type": "PERFORM",
    "fiscal_data": {
      "receipt_id": 121,
      "status_code": 0,
      "message": "accepted",
      "terminal_id": "EP000000000025",
      "fiscal_sign": "800031554082",
      "qr_code_url": "https://...",
      "date": "20220706221021"
    }
  }
}
```

### Request — Cancel Fiscal
Same format but `type: "CANCEL"`. Store separately as a different fiscal receipt.

### Response — Success
```json
{ "result": { "success": true } }
```

### Errors
| Code | When |
|------|------|
| -32001 | Receipt with given id not found |
| -32700 | Invalid JSON |
| -32602 | Invalid parameters |

---

## Data Types Reference

| Type | Format | Example |
|------|--------|---------|
| ID | 24-char hex string | `"5305e3bab097f420a62ced0b"` |
| Timestamp | 13-digit Unix ms | `1399114284039` |
| Amount | Positive integer (tiyin) | `500000` (= 5000 UZS) |
| Account | JSON object | `{"order_id": "197"}` |
| State | Integer | `1`, `2`, `-1`, `-2` |
| Reason | Integer or null | `1`-`5`, `10`, or `null` |
