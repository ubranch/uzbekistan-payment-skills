# Subscribe API — Complete Method Reference

## Method Index

Subscribe API methods are independent and work separately. They are divided by where they run:

**Client-side methods (Frontend)** — use `X-Auth: {merchant_id}` (no key):
| Method | Purpose |
|--------|---------|
| cards.create | Create card token from card number + expiry |
| cards.get_verify_code | Request SMS OTP code for card verification |
| cards.verify | Verify card with OTP code |

**Server-side methods (Backend)** — use `X-Auth: {merchant_id}:{key}`:

*Card methods:*
| Method | Purpose |
|--------|---------|
| cards.check | Check card token status |
| cards.remove | Delete card token |

*Receipt methods:*
| Method | Purpose |
|--------|---------|
| receipts.create | Create payment receipt |
| receipts.pay | Pay a receipt with card token |
| receipts.send | Send invoice via SMS |
| receipts.cancel | Cancel/refund a paid receipt |
| receipts.check | Check receipt status |
| receipts.get | Get full receipt info |
| receipts.get_all | Get receipts for a time period |
| receipts.set_fiscal_data | Send fiscal data to Payme |
| receipts.confirm_hold | Confirm (capture) held funds |

## Protocol Overview

Unlike Merchant API, in Subscribe API **your app sends requests to Payme**.
Protocol: JSON-RPC 2.0 over HTTPS.

| Environment | Endpoint |
|-------------|----------|
| Test | `https://checkout.test.paycom.uz/api` |
| Production | `https://checkout.paycom.uz/api` |

### Authentication
Header `X-Auth`:
- **Frontend** (client-side): `X-Auth: {merchant_id}`
- **Backend** (server-side): `X-Auth: {merchant_id}:{key}`

### Important Rules
- Merchant must display "Powered by Payme" label
- NEVER store raw card data on your server — only tokens
- Card input form must NOT have `name` attributes on inputs
- Card input form `<form>` must NOT have `action` attribute
- Form must contain: Payme logo, link to Payme offer, note that data is stored on Payme servers

---

## CARD METHODS

### cards.create — Create Card Token

**Side:** Frontend (X-Auth without key)

#### Request
```json
{
  "id": 123,
  "method": "cards.create",
  "params": {
    "card": {
      "number": "8600069195406311",
      "expire": "0399"
    },
    "save": true
  }
}
```

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| card.number | String | Yes | Full card number |
| card.expire | String | Yes | Expiry in MMYY format |
| save | Boolean | No | `true` = reusable token; `false` = one-time token |
| account | Object | No | Account object |
| customer | String | No | User identifier (phone, uid, email) |

#### Response
```json
{
  "jsonrpc": "2.0",
  "id": 123,
  "result": {
    "card": {
      "number": "860006******6311",
      "expire": "03/99",
      "token": "NTg0YTg0ZDYy...",
      "recurrent": true,
      "verify": false
    }
  }
}
```

---

### cards.get_verify_code — Request SMS Verification Code

**Side:** Frontend

#### Request
```json
{
  "id": 123,
  "method": "cards.get_verify_code",
  "params": {
    "token": "NTg0YTg0ZDYy..."
  }
}
```

#### Response
```json
{
  "jsonrpc": "2.0",
  "id": 123,
  "result": {
    "sent": true,
    "phone": "99890*****31",
    "wait": 60000
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| sent | Boolean | Whether SMS was sent |
| phone | String | Masked phone number |
| wait | Number | OTP validity period in ms |

---

### cards.verify — Verify Card with SMS Code

**Side:** Frontend

#### Request
```json
{
  "id": 123,
  "method": "cards.verify",
  "params": {
    "token": "NTg0YTg0ZDYy...",
    "code": "666666"
  }
}
```

#### Response
```json
{
  "jsonrpc": "2.0",
  "id": 123,
  "result": {
    "card": {
      "number": "860006******6311",
      "expire": "03/99",
      "token": "NTg0YTgxZWYy...",
      "recurrent": true,
      "verify": true
    }
  }
}
```

Note: Token may change after verification. Always use the latest token.

---

### cards.check — Check Card Token Status

**Side:** Backend (X-Auth with key)

#### Request
```json
{
  "id": 123,
  "method": "cards.check",
  "params": {
    "token": "NTg1Yjc4OWMy..."
  }
}
```

#### Response
Returns card object with `number`, `expire`, `token`, `recurrent`, `verify`.

---

### cards.remove — Delete Card Token

**Side:** Backend (X-Auth with key)

#### Request
```json
{
  "id": 123,
  "method": "cards.remove",
  "params": {
    "token": "NTg1Yjc4OWMy..."
  }
}
```

#### Response
```json
{
  "jsonrpc": "2.0",
  "id": 123,
  "result": {
    "success": true
  }
}
```

---

### cards.remove — Delete Card Token

**Side:** Backend (X-Auth with key)

#### Request
```json
{
  "id": 123,
  "method": "cards.remove",
  "params": {
    "token": "NTg1Yjc4OWMy..."
  }
}
```

#### Response
```json
{
  "jsonrpc": "2.0",
  "id": 123,
  "result": {
    "success": true
  }
}
```

---

## RECEIPT METHODS

All receipt methods use **Backend** auth (X-Auth with key).

### receipts.create — Create Payment Receipt

#### Request
```json
{
  "id": 4,
  "method": "receipts.create",
  "params": {
    "amount": 500000,
    "account": {
      "order_id": "test"
    },
    "description": "Payment for order #test",
    "detail": {
      "receipt_type": 0,
      "shipping": {
        "title": "Delivery",
        "price": 500000
      },
      "items": [
        {
          "title": "Product",
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

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| amount | Number | Yes | Amount in tiyin |
| account | Object | Yes | Account fields matching kassa settings |
| description | String | No | Payment description |
| detail | Object | No | Fiscal detail object |

#### Response
Returns full `receipt` object with `_id`, `state: 0`, `create_time`, etc.

---

### receipts.pay — Pay a Receipt

#### Request
```json
{
  "id": 123,
  "method": "receipts.pay",
  "params": {
    "id": "62da73b0803aced907a52b46",
    "token": "NTg1Yjc4OWMy...",
    "payer": {
      "phone": "998901304527"
    }
  }
}
```

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| id | String | Yes | Receipt ID from receipts.create |
| token | String | Yes | Verified card token |
| payer | Object | No | Anti-fraud info (phone, email, name, ip) |

#### Payer Object (Anti-fraud)
```json
{
  "payer": {
    "id": "user_123",
    "phone": "998901234567",
    "email": "user@example.com",
    "name": "John Doe",
    "ip": "192.168.1.1"
  }
}
```

#### Response
Returns receipt object with `state: 4` (paid), `pay_time` populated.

---

### receipts.send — Send Invoice via SMS

#### Request
```json
{
  "id": 123,
  "method": "receipts.send",
  "params": {
    "id": "62da73b0803aced907a52b46",
    "phone": "998901234567"
  }
}
```

Sends payment link via SMS to the specified phone number.

---

### receipts.send — Send Invoice via SMS

Sends payment link via SMS to the specified phone number.

#### Request
```json
{
  "id": 123,
  "method": "receipts.send",
  "params": {
    "id": "62da73b0803aced907a52b46",
    "phone": "998901304527"
  }
}
```

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| id | String | Yes | Receipt ID |
| phone | String | Yes | Recipient phone number |

#### Response
```json
{
  "jsonrpc": "2.0",
  "result": {
    "success": true
  }
}
```

---

### receipts.cancel — Cancel/Refund Receipt

#### Request
```json
{
  "id": 123,
  "method": "receipts.cancel",
  "params": {
    "id": "62da73b0803aced907a52b46"
  }
}
```

#### Response
Returns receipt with `state: 21` (queued for cancellation).

---

### receipts.check — Check Receipt Status

#### Request
```json
{
  "id": 123,
  "method": "receipts.check",
  "params": {
    "id": "62da73b0803aced907a52b46"
  }
}
```

---

### receipts.check — Check Receipt Status

#### Request
```json
{
  "id": 123,
  "method": "receipts.check",
  "params": {
    "id": "62da73b0803aced907a52b46"
  }
}
```

#### Response
```json
{
  "jsonrpc": "2.0",
  "id": 123,
  "result": {
    "state": 4
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| state | Number | Receipt state (see Receipt States below) |

---

### receipts.get — Get Full Receipt Info

#### Request
```json
{
  "id": 123,
  "method": "receipts.get",
  "params": {
    "id": "62da73b0803aced907a52b46"
  }
}
```

Returns complete receipt object with all fields.

---

### receipts.get_all — Get Receipts for Period

#### Request
```json
{
  "id": 123,
  "method": "receipts.get_all",
  "params": {
    "count": 10,
    "from": 1636398000000,
    "to": 1636484400000,
    "offset": 0
  }
}
```

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| count | Integer | Yes | Number of receipts (max 50) |
| from | Timestamp | Yes | Start date |
| to | Timestamp | Yes | End date |
| offset | Integer | No | Number of receipts to skip from start |

#### Response
Returns array of receipt objects.

---

### receipts.set_fiscal_data — Send Fiscal Data to Payme

Transmits fiscal receipt data from your OFD (fiscal data operator) to Payme.

#### Request
```json
{
  "id": 123,
  "method": "receipts.set_fiscal_data",
  "params": {
    "id": "2e0b1bc1f1eb50d487ba268d",
    "fiscal_data": {
      "status_code": 0,
      "message": "accepted",
      "terminal_id": "EP000000000025",
      "receipt_id": 121,
      "date": "20220706221021",
      "fiscal_sign": "800031554082",
      "qr_code_url": "https://fiscal-receipt-url"
    }
  }
}
```

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| id | String | Yes | Receipt ID in Payme DB |
| fiscal_data.receipt_id | Integer | Yes | Unique payment number for VFM |
| fiscal_data.qr_code_url | String | Yes | Fiscal receipt URL |
| fiscal_data.status_code | Integer | No | Status code |
| fiscal_data.message | String | No | Error detail if OFD registration failed |
| fiscal_data.terminal_id | String | No | Virtual Fiscal Module number |
| fiscal_data.date | String | No | OFD registration date |
| fiscal_data.fiscal_sign | String | No | Fiscal signature |

#### Response
```json
{
  "jsonrpc": "2.0",
  "id": 123,
  "result": {
    "success": true
  }
}
```

---

## HOLD (Authorization/Capture)

Hold allows you to freeze (authorize) funds on a card without immediately capturing them. Useful for booking scenarios where you confirm the charge later.

**Prerequisites:** Must implement Subscribe API card tokenization. Notify Payme tech support to enable hold on your kassa.

**Important:** Hold can only be tested in production mode, not sandbox.

### Step 1: Create Receipt with Hold
Add `"hold": true` to `receipts.create`:
```json
{
  "id": 123,
  "method": "receipts.create",
  "params": {
    "account": { "order_id": 106 },
    "amount": 2500,
    "hold": true
  }
}
```

### Step 2: Pay with Hold
Add `"hold": true` to `receipts.pay`:
```json
{
  "id": 123,
  "method": "receipts.pay",
  "params": {
    "id": "{{receipt_id}}",
    "token": "{{card_token}}",
    "hold": true
  }
}
```
Response will have `"state": 5` (held/authorized).

### Step 3a: Confirm Hold (Capture funds)
```json
{
  "id": 123,
  "method": "receipts.confirm_hold",
  "params": {
    "id": "624c0c2b0ac8b463e47422c7"
  }
}
```
Response will have `"state": 4` (paid/captured).

### Step 3b: Cancel Hold (Release funds)
Use standard `receipts.cancel` method.

### Hold Timeouts
- **UZCARD**: Auto-cancels hold after 30 days
- **HUMO**: Can only confirm (not cancel) after 30 days; confirm/cancel within 30 days

---

## Receipt States (Subscribe API)

| State | Description |
|-------|-------------|
| 0 | Created, awaiting payment confirmation |
| 1 | First stage checks, creating transaction in merchant billing |
| 2 | Card debit in progress |
| 3 | Closing transaction in merchant billing |
| 4 | Paid successfully |
| 5 | Held (authorized, awaiting capture) |
| 6 | Hold command received, transitioning to state 5. If stuck — contact Payme support |
| 20 | Paused for manual intervention |
| 21 | Queued for cancellation |
| 30 | Queued for closing transaction in merchant billing |
| 50 | Cancelled |

---

## Complete Subscribe API Payment Flow

### Step-by-step:

1. **Create token**: `cards.create` → get `token` (frontend)
2. **Request SMS**: `cards.get_verify_code` → SMS sent to cardholder (frontend)
3. **Verify card**: `cards.verify` → card verified, get final `token` (frontend)
4. **Send token to backend**: securely transmit token to your server
5. **Create receipt**: `receipts.create` → get receipt `_id` (backend)
6. **Pay receipt**: `receipts.pay` with receipt `_id` + card `token` (backend)
7. **Check status**: `receipts.check` or `receipts.get` to verify (backend)

### For Saved Cards (recurring):
- Skip steps 1-3 if you already have a verified, saved token
- Go directly to step 5: create receipt and pay

### For Invoice via SMS:
- Create receipt (step 5)
- Instead of step 6, use `receipts.send` to send payment link via SMS

---

## Security Recommendations

1. Never store raw card numbers — only tokens
2. Offer PIN-code setup when saving a card token
3. Require PIN before paying with saved token
4. Delete token after 3 incorrect PIN attempts
5. Use `payer` object in receipts.pay for anti-fraud
6. Validate card token status with `cards.check` before payment

## Test Credentials

- Test cabinet login: your phone number
- Test cabinet password: `qwerty`
- Test SMS code: `666666`
- See main SKILL.md for test card numbers
