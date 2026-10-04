---
name: payme-integration
description: >
  Expert-level Payme payment system integration skill for Uzbekistan's Payme Business platform.
  Use this skill whenever the user mentions Payme, Paycom, payment integration in Uzbekistan,
  Merchant API, Subscribe API, to'lov tizimi, to'lov integratsiyasi, Payme checkout,
  Payme kassa, online to'lov, plastik karta to'lovi, chek yaratish, tranzaksiya,
  or any payment-related development for Uzbek market. Also trigger when user asks about
  JSON-RPC payment APIs, fiscalization (fiskalizatsiya), IKPU codes, or O'zbekiston to'lov
  tizimlari. This skill covers both Merchant API (server-to-server) and Subscribe API
  (card tokenization + receipts), sandbox testing, error handling, and production deployment.
  Trigger even for partial mentions like "payme", "paycom", "checkout.paycom.uz", or
  "merchant api" in any language (Uzbek, Russian, English).
---

# Payme Business Integration — Expert Guide

This skill makes you a **super-expert** on Payme Business payment integration. It covers the complete Payme ecosystem: Merchant API, Subscribe API, payment initialization, sandbox testing, fiscalization, error handling, and production deployment.

Official documentation checked **2026-10-05**. These are implementation guides, not evidence of tested live payments. Provider-specific source caveats are noted in the references.

## What is Payme Business?

Payme Business is a payment platform for businesses in Uzbekistan. It allows merchants to accept payments from Uzcard and HUMO bank cards online via the Payme mobile app, Telegram bots, email, or SMS. Funds are deposited to the business's bank account through Payme Business kassas (cash registers).

There are 3 types of kassa:
1. **Kassa with billing** — requires integration setup (Merchant API or Subscribe API)
2. **Kassa without billing** — requires integration setup
3. **Kassa for on-site payments** — works immediately after connection

Only legal entities can use kassas: IP, ChP, OOO, AO, GUP, SP, NOU.

## Quick Reference

| Item | Value |
|------|-------|
| Production checkout URL | `https://checkout.paycom.uz` |
| Sandbox checkout URL | `https://test.paycom.uz` |
| Subscribe API (test) | `https://checkout.test.paycom.uz/api` |
| Subscribe API (prod) | `https://checkout.paycom.uz/api` |
| Protocol | JSON-RPC 2.0 over HTTPS; use modern TLS (1.2+), not TLS 1.0/1.1 |
| Currency | UZS; Merchant/Subscribe API and checkout amounts use **tiyin** (1 UZS = 100 tiyin). The documented Android SDK takes a sum in UZS. |
| Documentation | `https://developer.help.paycom.uz/` |

## Choosing the Right API

Payme offers **two** independent APIs. Read `references/merchant-api.md` for Merchant API details and `references/subscribe-api.md` for Subscribe API details.

**Source caveat:** Payme's live `CheckTransaction` and `CheckPerformTransaction` documentation URL slugs are swapped. Identify the API by the page's method name and request payload, not the URL; see the source mapping in `references/merchant-api.md`.

### Merchant API — "Payme handles checkout UI"
- User clicks pay → redirected to Payme's checkout form → pays → returns to merchant site
- Payme sends JSON-RPC requests **to your server** (you implement the endpoint)
- Best for: web platforms, e-commerce sites, standard checkout flows
- You implement 6+1 methods: CheckPerformTransaction, CreateTransaction, PerformTransaction, CancelTransaction, CheckTransaction, GetStatement, (optional) SetFiscalData

### Subscribe API — "You handle checkout UI"
- You build the payment form in your own app/site
- Your app sends JSON-RPC requests **to Payme's server**
- Best for: mobile apps, custom checkout UX, recurring payments, card-on-file
- Card methods: cards.create, cards.get_verify_code, cards.verify, cards.check, cards.remove
- Receipt methods: receipts.create, receipts.pay, receipts.send, receipts.cancel, receipts.check, receipts.get, receipts.get_all

### Decision Matrix

| Criteria | Merchant API | Subscribe API |
|----------|-------------|---------------|
| Checkout UI | Payme's form | Your custom form |
| Who sends requests | Payme → Your server | Your server → Payme |
| Card data handling | Never touches your server | Tokenized on your server |
| Setup complexity | Medium | Higher |
| PCI compliance burden | Low | Medium (must follow rules) |
| Best for | Web sites, standard e-commerce | Apps, custom UX, subscriptions |

## Integration Workflow Overview

For detailed implementation instructions, read the appropriate reference file:
- **Merchant API**: `references/merchant-api.md`
- **Subscribe API**: `references/subscribe-api.md`

## Before You Start — Setup Checklist

1. Register business on Payme Business cabinet
2. Create a "web kassa" (web cash register) in the cabinet
3. Get your credentials:
   - **Merchant ID** (ID kassa) — 24-character hex string
   - **KEY** — production password (36 chars)
   - **TEST KEY** — sandbox password
4. Set your **Endpoint URL** — the URL where Payme will send requests (Merchant API)
5. Define **account parameters** — fields identifying the order (e.g., `order_id`)

## Payme IP Whitelist (Merchant API)

Payme sends requests ONLY from these IPs — whitelist them on your firewall:
```
185.234.113.1 through 185.234.113.15
```
(15 IP addresses total: 185.234.113.1 — 185.234.113.15)

## Authentication

### Merchant API (Payme → Your Server)
HTTP Basic Auth in `Authorization` header:
```
Authorization: Basic base64(login:password)
```
- `login`: provided by Payme tech specialist (usually "Paycom")
- `password`: KEY (production) or TEST KEY (sandbox)
- Validate credentials; return error `-32504` if invalid

### Subscribe API (Your Server → Payme)
Custom `X-Auth` header:
- Frontend: `X-Auth: {merchant_id}`
- Backend: `X-Auth: {merchant_id}:{key}`

## Error Codes — Complete Reference

### General Protocol Errors
| Code | Description |
|------|-------------|
| -32300 | Request method is not POST |
| -32700 | JSON parse error |
| -32600 | Missing required RPC fields or wrong field types |
| -32601 | Method not found |
| -32504 | Insufficient privileges (auth failure) |
| -32400 | System/internal error |

### Business Logic Errors
| Code | Description |
|------|-------------|
| -31001 | Invalid amount (doesn't match order) |
| -31003 | Transaction not found |
| -31007 | Cannot cancel — order already fulfilled |
| -31008 | Cannot perform operation (wrong state) |
| -31050 to -31099 | Invalid account fields (localized `message` required, `data` must contain account field name) |

## Transaction States

| State | Description | Initial State |
|-------|-------------|---------------|
| 1 | Created, awaiting confirmation | 0 |
| 2 | Successfully completed | 1 |
| -1 | Cancelled (before completion) | 1 |
| -2 | Cancelled after completion (refund) | 2 |

## Cancel Reasons

| Code | Description |
|------|-------------|
| 1 | Receiver(s) not found or inactive |
| 2 | Debit operation error in processing center |
| 3 | Transaction execution error |
| 4 | Cancelled by timeout |
| 5 | Refund |
| 10 | Unknown error |

## Receipt/Check States

| State | Description |
|-------|-------------|
| 0 | Created, awaiting payment confirmation |
| 1 | First stage checks, creating transaction |
| 2 | Card debit in progress |
| 3 | Closing transaction in merchant billing |
| 4 | Paid successfully |
| 5 | On hold |
| 6 | Hold command received; contact Payme support if this state persists |
| 20 | Paused for manual intervention |
| 21 | Queued for cancellation |
| 30 | Queued to close transaction in merchant billing |
| 50 | Cancelled |

Source: [Receipt states](https://developer.help.paycom.uz/metody-subscribe-api/sostoyaniya-cheka/) (checked 2026-10-05). Only state `4` means paid; queued/hold states are not payment completion.

## Fiscalization (Fiskalizatsiya)

For fiscalization, Payme documents a `detail` object in the CheckPerformTransaction response, including the required fiscal item fields:

```json
{
  "result": {
    "allow": true,
    "detail": {
      "receipt_type": 0,
      "items": [
        {
          "title": "Product name",
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

`vat_percent: 15`, IKPU and package codes above are **example data**, not a current tax-rate default. Use the actual applicable rate and product codes for the merchant/product; do not mechanically substitute a universal rate. Source: [CheckPerformTransaction page](https://developer.help.paycom.uz/metody-merchant-api/checktransaction/) (checked 2026-10-05; URL caveat above).

### Detail Items Fields
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| title | String | Yes | Product/service name |
| price | Number | Yes | Price per unit in tiyin |
| count | Number | Yes | Quantity |
| code | String | Yes | IKPU code (product identification code) |
| units | Number | No | Unit of measurement code |
| vat_percent | Number | Yes | VAT percentage |
| package_code | String | Yes | Package code (from IKPU details) |
| discount | Number | No | Discount in tiyin (total for quantity) |

IKPU codes can be verified at: `https://tasnif.soliq.uz/`

## Sandbox Testing

### Test Flow
1. Set Endpoint URL in Payme cabinet
2. Use TEST KEY for authentication
3. Go to `https://test.paycom.uz`
4. Enter Merchant ID and TEST KEY
5. Run two test scenarios:
   - **Scenario 1**: Create → Cancel (unconfirmed transaction)
   - **Scenario 2**: Create → Perform → Cancel (confirmed transaction)

### Test Cards (Subscribe API)
| Number | Expire | Note |
|--------|--------|------|
| 8600 0609 2109 0842 | 03/99 | SMS not connected |
| 8600 4954 7331 6478 | 03/99 | Normal test card |
| 8600 0691 9540 6311 | 03/99 | Normal test card |
| 3333 3364 1580 4657 | 03/99 | Expired card |
| 4444 4459 8745 9073 | 03/99 | Blocked card |
| 8600 1434 1777 0323 | 03/99 | System error simulation |
| 8600 1343 0184 9596 | 03/99 | 10-second delay, then error |

**SMS verification code for all test cards: `666666`**

### Checkout/Receipt Submission Errors
| Code | Description |
|------|-------------|
| -31601 | Merchant not found or blocked |
| -31610 | Invalid field value |
| -31611 | Amount below minimum |
| -31612 | Amount above maximum |
| -31622 | Merchant service unavailable |
| -31623 | Merchant service working incorrectly |
| -31630 | Insufficient funds / invalid card / expired / blocked / corporate card |

## Hold (Authorization/Capture) — Quick Reference

Hold freezes funds on card without capturing. Add `"hold": true` to `receipts.create` and `receipts.pay`. Then:
- **Capture**: `receipts.confirm_hold` → state becomes 4 (paid)
- **Release**: `receipts.cancel` → funds returned
- **UZCARD timeout**: auto-cancel after 30 days
- **HUMO timeout**: can only confirm after 30 days
- **Testing**: production only (not available in sandbox)
- **Prerequisite**: notify Payme tech support to enable hold on kassa

See `references/subscribe-api.md` for full Hold documentation.

## Payment Initialization (Redirecting to Payme)

Payment initialization (acceptance) is done via a payment form — a "check" (receipt). The check is formed on the merchant's side and sent to Payme using POST or GET methods. Additionally, you can generate payment buttons and QR codes using the Payme JS SDK.

Four methods available:
1. **POST form** — HTML form submission to Payme checkout
2. **GET URL** — Base64-encoded URL redirect
3. **Payment Button** — Auto-generated SVG/PNG button via JS SDK
4. **QR Code** — Auto-generated QR code via JS SDK

### Method 1: POST Form

Required: `merchant`, `amount` (tiyin), and the `account[...]` fields configured for the kassa. All fields below the optional-field comment may be omitted.

```html
<form method="POST" action="https://checkout.paycom.uz">
  <input type="hidden" name="merchant" value="{MERCHANT_ID}"/>
  <input type="hidden" name="amount" value="{AMOUNT_IN_TIYIN}"/>
  <input type="hidden" name="account[order_id]" value="{ORDER_ID}"/>
  <!-- Optional fields -->
  <input type="hidden" name="lang" value="uz"/>
  <input type="hidden" name="callback" value="https://merchant.example/payme/:transaction?order=:account.order_id"/>
  <input type="hidden" name="callback_timeout" value="15000"/>
  <input type="hidden" name="description[ru]" value="{DESCRIPTION_RU}"/>
  <input type="hidden" name="description[uz]" value="{DESCRIPTION_UZ}"/>
  <input type="hidden" name="description[en]" value="{DESCRIPTION_EN}"/>
  <input type="hidden" name="detail" value="{BASE64_JSON_DETAIL}"/>
  <button type="submit">Pay with Payme</button>
</form>
```

- `lang`: `ru`, `uz`, `en`; default `ru`.
- `callback`: return URL after payment or cancellation; defaults to the request's `Referer` if omitted. Payme substitutes `:transaction` (transaction ID, or `"null"` if creation failed) and `:account.{field}`. A redirect is **not** proof of payment.
- `callback_timeout`: delay after success, documented in milliseconds; `15000` above explicitly requests 15 seconds. The official page states a default of `15`; do not assume that means 15 seconds.
- `description`: use a single string field or localized `description[ru]`, `description[uz]`, `description[en]` fields.
- `detail`: **BASE64 of the UTF-8 JSON detail object itself**, not a JSON-RPC wrapper. It is optional for checkout submission; required fiscal item fields still apply when supplying fiscalization data.

Runnable Node.js encoding example (no network or credentials):
```javascript
const assert = require('node:assert/strict');
const detail = {
  receipt_type: 0,
  items: [{
    title: 'Mahsulot — 商品', price: 500, count: 1,
    code: '00702001001000001', vat_percent: 15, package_code: '123456'
  }]
};
// VAT and product codes are example data; replace with applicable merchant/product values.
const detailBase64 = Buffer.from(JSON.stringify(detail), 'utf8').toString('base64');
assert.deepEqual(JSON.parse(Buffer.from(detailBase64, 'base64').toString('utf8')), detail);
```

Place `detailBase64` in the form's `detail` value. In Merchant/Subscribe JSON-RPC responses/requests, `detail` remains a JSON object, not BASE64.

Source: [Checkout POST](https://developer.help.paycom.uz/initsializatsiya-platezhey/otpravka-cheka-po-metodu-post/) (checked 2026-10-05). The page calls `items` optional in one generic detail table but required for fiscalization in its example; use the fiscal required fields when fiscalizing, and confirm the kassa's setup with Payme.

### Method 2: GET URL

Encode `key=value` parameters separated by `;` as BASE64, then append the encoded value to `https://checkout.paycom.uz/`. `base64(params)` in the official documentation is notation, not a literal URL path.

| Param | Use | Description |
|-------|-----|-------------|
| m | Basic checkout field | Merchant ID or alias |
| ac.{field} | Basic checkout field | Account fields configured for the kassa |
| a | Basic checkout field | Amount in tiyin |
| l | Optional | Language (`ru`/`uz`/`en`) |
| c | Optional | Callback URL |
| ct | Optional | Callback timeout (ms) |
| cr | Optional | Currency code in ISO format |

The GET page does not label every field's requiredness or specify whether `cr` uses a numeric or alphabetic ISO code. Keep the basic fields above and confirm `cr`'s accepted representation for the kassa rather than guessing.

```javascript
const assert = require('node:assert/strict');
const params = 'm=587f72c72cac0d162c722ae2;ac.order_id=197;a=500';
const encoded = Buffer.from(params, 'utf8').toString('base64');
const checkoutUrl = `https://checkout.paycom.uz/${encoded}`;
assert.equal(Buffer.from(encoded, 'base64').toString('utf8'), params);
assert.equal(checkoutUrl, 'https://checkout.paycom.uz/bT01ODdmNzJjNzJjYWMwZDE2MmM3MjJhZTI7YWMub3JkZXJfaWQ9MTk3O2E9NTAw');
```

Source: [Checkout GET](https://developer.help.paycom.uz/initsializatsiya-platezhey/otpravka-cheka-po-metodu-get/) (checked 2026-10-05).

## Common Implementation Patterns

### Node.js/Express Merchant API Endpoint
```javascript
app.post('/api/payment/payme', (req, res) => {
  // 1. Validate Basic Auth (check KEY/TEST_KEY)
  const auth = req.headers.authorization;
  if (!validateAuth(auth)) {
    return res.json({
      error: { code: -32504, message: { ru: "Недостаточно привилегий", uz: "Ruxsat etilmagan", en: "Insufficient privileges" } },
      id: req.body.id
    });
  }

  const { method, params, id } = req.body;
  
  switch(method) {
    case 'CheckPerformTransaction':
      return handleCheckPerform(params, id, res);
    case 'CreateTransaction':
      return handleCreate(params, id, res);
    case 'PerformTransaction':
      return handlePerform(params, id, res);
    case 'CancelTransaction':
      return handleCancel(params, id, res);
    case 'CheckTransaction':
      return handleCheck(params, id, res);
    case 'GetStatement':
      return handleStatement(params, id, res);
    default:
      return res.json({
        error: { code: -32601, message: "Method not found", data: method },
        id
      });
  }
});
```

### Go Merchant API Handler Pattern
```go
func PaymeHandler(w http.ResponseWriter, r *http.Request) {
    // Must be POST
    if r.Method != http.MethodPost {
        respondError(w, -32300, "Method not POST", 0)
        return
    }
    // Validate Basic Auth
    if !validatePaymeAuth(r) {
        respondError(w, -32504, "Insufficient privileges", 0)
        return
    }
    // Parse JSON-RPC request
    var rpcReq RPCRequest
    if err := json.NewDecoder(r.Body).Decode(&rpcReq); err != nil {
        respondError(w, -32700, "Parse error", 0)
        return
    }
    // Route to handler
    switch rpcReq.Method {
    case "CheckPerformTransaction":
        handleCheckPerform(w, rpcReq)
    case "CreateTransaction":
        handleCreate(w, rpcReq)
    // ... etc
    }
}
```

## Critical Implementation Rules

1. **Always return HTTP 200** — even for errors. Non-200 = RPC error -32400
2. **API/checkout amounts are in TIYIN** — 1 UZS = 100 tiyin. 50,000 UZS = 5,000,000 tiyin. The documented Android SDK takes a UZS sum and converts it internally; see `references/additional-reference.md`.
3. **Timestamps are Unix milliseconds** — 13-digit numbers
4. **Transaction IDs from Payme** are 24-char hex strings
5. **CreateTransaction timeout** — transactions auto-cancel after 12 hours (43,200,000 ms)
6. **Idempotency** — CreateTransaction, PerformTransaction, CancelTransaction may be called multiple times with same params; return same result
7. **GetStatement is MANDATORY** — used for reconciliation
8. **Lock orders** — after CreateTransaction, prevent order modification
9. **Error messages must be localized** — provide `ru`, `uz`, `en` in message object for -31050 to -31099 errors
10. **Account field errors** — `data` field must contain the name of the invalid account sub-field

## Reference Files

For complete method specifications with all request/response examples:
- `references/merchant-api.md` — Full Merchant API (6+1 methods, all request/response, error codes, implementation checklists)
- `references/subscribe-api.md` — Full Subscribe API (cards + receipts + hold/authorize + fiscal data + receipt states)
- `references/additional-reference.md` — Telegram Bot, Mobile SDK (Android), QR/Button generation, CMS plugins, checkout errors, server examples, kassa ID/KEY lookup, tech support
