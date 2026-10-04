---
name: click-integration
description: >
  Expert-level Click payment integration skill for Uzbekistan's Click SuperApp.
  Use whenever user mentions Click, click.uz, SHOP API, Merchant API, Click to'lov,
  Click integratsiya, Click callback, prepare/complete, click_trans_id, merchant_trans_id,
  Click fiscalization, Click button, Click invoice, card token, Click Pass, checkout.js,
  createPaymentRequest, Click Telegram, mobile SDK, merchant.click.uz, my.click.uz,
  api.click.uz, or Click error codes (-1 to -9). Covers standard SHOP (Prepare/Complete),
  Advanced/Split JSON SHOP, Merchant API (invoices, payments, tokens, reversal), payment button, inline checkout,
  CLICK Pass (QR POS), fiscalization (OFD/IKPU), Telegram bot payments, mobile SDK,
  CMS plugins (WooCommerce, OpenCart, 1C-Bitrix), testing, and deployment.
  Trigger for partial mentions like "click", "shop api", "click payment", "click pass",
  "click telegram", "click plugin" in Uzbek, Russian, or English.
---

# Click Payment Integration — Expert Guide

Use this guide to select and implement Click payment integrations in Uzbekistan. The references are practical, selective guides to official documentation, not a complete verbatim mirror. Source routes and protocol findings were last verified on 2026-10-05; unresolved upstream ambiguities are called out explicitly.

## Quick Reference

| Item | Value |
|------|-------|
| Payment page URL | `https://my.click.uz/services/pay/` |
| Merchant API endpoint | `https://api.click.uz/v2/merchant/` |
| Merchant cabinet | `https://merchant.click.uz` |
| Documentation | `https://docs.click.uz` |
| Standard SHOP API Protocol | HTTP/HTTPS POST, `application/x-www-form-urlencoded` |
| Advanced / Split Shop Protocol | Separate JSON callbacks, `application/json; charset=utf-8` |
| Merchant API Protocol | HTTPS, `application/json` (also supports `application/xml`) |
| Currency | UZS: SHOP/Merchant/checkout amounts in **so'm**; fiscalization and Telegram use **tiyin** |
| Amount format | float with 2 decimal places (e.g., `1000.00`) |
| SHOP auth | MD5 `sign_string`; standard and JSON variants have different formulas; lookup/reconciliation omissions need clarification |
| Merchant API auth | SHA1 digest in `Auth` header |
| Checkout.js CDN | `https://my.click.uz/pay/checkout.js` |
| Android SDK | `https://github.com/click-llc/android-msdk` |

## Choosing the Right Integration Method

Click offers **multiple** ways to accept payments. Read the appropriate reference file for full details.

### 1. Standard SHOP API — "Click calls YOUR server" (most common)
- User pays via Click → Click sends form-urlencoded Prepare (`action=0`) / Complete (`action=1`)
- Implement the configured callback URLs; one endpoint can handle both standard actions
- **Read**: `references/02-shop-api-requests.md`

### JSON SHOP variants — Choose only the enabled service contract
- **Advanced Shop**: optional Getinfo 0, Prepare 1, Complete 2, Check 3, Compare 4; account/order fields are in `params`
- **Split Shop**: optional Getinfo 0, Prepare 1, Confirm 2; Prepare returns `split: [{cntrg_id, amount}]`, with allocations summing to the payment total
- These are not standard SHOP with extra fields: encoding, action numbers, payment identity, and signatures differ
- **Read**: `references/16-advanced-shop.md` or `references/17-split-shop.md`

### 2. Payment Button/Link — "Redirect user to Click"
- Simple link/form redirects user to my.click.uz payment page
- Works with SHOP API callback on your server
- **Read**: `references/05-payment-button.md`

### 3. Inline Checkout — "Pay on YOUR site without redirect"
- Embed `checkout.js` widget — payment form opens as overlay
- No redirect to my.click.uz needed
- **Read**: `references/06-inline-checkout.md`

### 4. Merchant API — "YOU call Click's server"
- Create invoices, check payment status, refund, card tokens
- Supplements SHOP API, not a replacement
- **Read**: `references/08-merchant-api-requests.md`

### 5. CLICK Pass — "QR-code POS payment"
- Merchant scans QR from user's Click app
- For physical retail, kiosks
- **Read**: `references/10-click-pass.md`

### 6. Telegram Bot Payments
- Accept payments inside Telegram via Click provider
- Uses Telegram Bot API with Click provider_token
- **Read**: `references/12-telegram-payments.md`

### 7. Mobile SDK / Deep Links
- Android SDK library or deep link integration (Android + iOS)
- **Read**: `references/13-mobile-sdk.md`

### Decision Matrix

| Method | Who initiates | Where user pays | Best for |
|--------|--------------|-----------------|----------|
| SHOP API + Payment Button | User clicks link | Click web/app | E-commerce, web |
| SHOP API + Inline Checkout | User on your site | Overlay on your site | SPA, custom UX |
| Advanced Shop JSON | Click calls your billing | Configured Click payment flow | Prepayment account lookup, outcome check, reconciliation |
| Split Shop JSON | Click calls your billing | Configured Click payment flow | Allocation to registered counterparties in Prepare |
| Merchant API Invoice | Merchant sends invoice | User confirms in Click app | Subscriptions, push billing |
| Merchant API Card Token | Merchant charges token | No user interaction | Recurring, card-on-file |
| CLICK Pass | Merchant scans QR | Already in Click app | Physical retail, POS |
| Telegram Payments | User in Telegram | Telegram payment UI | Telegram bots |
| Mobile SDK / Deep Link | User in your app | Click app or browser | Mobile apps |

## Sign String Formulas (Standard SHOP API Only)

| Request | Formula |
|---------|---------|
| Prepare (action=0) | `MD5(click_trans_id + service_id + SECRET_KEY + merchant_trans_id + amount + action + sign_time)` |
| Complete (action=1) | `MD5(click_trans_id + service_id + SECRET_KEY + merchant_trans_id + merchant_prepare_id + amount + action + sign_time)` |

**CRITICAL**: Parameters concatenated WITHOUT separators. Use constant-time comparison (e.g., `crypto.timingSafeEqual`).

Advanced/Split Shop instead sign `MD5(click_paydoc_id + attempt_trans_id + service_id + SECRET_KEY + params-values-in-original-order + action + sign_time)`. `params` means concatenated values in transmitted order, **not** JSON serialization. Their docs omit signing fields on Getinfo/Compare; confirm the access/authentication contract instead of inventing one.

## Merchant API Authentication

```
Auth: {merchant_user_id}:{digest}:{timestamp}
```
- `digest` = `SHA1(timestamp + secret_key)`
- `timestamp` = UNIX timestamp (10-digit seconds)

The official `card_token/request` sample omits `Auth`, unlike verify/payment/delete. This does not prove the header is forbidden or causes 401/CORS errors. Keep secrets server-side and confirm the endpoint-specific requirement with Click; the client example retains the general authenticated pattern.

## Error Codes Summary (SHOP API)

| error | error_note | Description |
|-------|------------|-------------|
| 0 | Success | OK |
| -1 | SIGN CHECK FAILED! | Signature verification failed |
| -2 | Incorrect parameter amount | Wrong amount |
| -3 | Action not found | Unknown action |
| -4 | Already paid | Duplicate payment |
| -5 | User does not exist | Order/user not found |
| -6 | Transaction does not exist | Payment record not found |
| -7 | Failed to update user | DB/balance update error |
| -8 | Error in request from click | Malformed request |
| -9 | Transaction cancelled | Previously cancelled |

## Merchant API HTTP Error Codes

| Code | Description |
|------|-------------|
| 200, 201 | OK |
| 400 | Bad request (malformed data or URI) |
| 401 | Not authorized (auth error) |
| 403 | Forbidden (method not allowed) |
| 404 | Not found (method not found) |
| 406 | Not acceptable (invalid data type) |
| 410 | Gone (deprecated method) |
| 500 | Internal server error |
| 502 | Service is down or being upgraded |

## Before You Start — Setup Checklist

1. Register with Click and sign contract with connected bank
2. Receive credentials: `merchant_id`, `service_id`, `SECRET_KEY`, `merchant_user_id`
3. Get access to merchant cabinet at `merchant.click.uz`
4. Set **Prepare URL** and **Complete URL** in merchant cabinet → Сервисы → pencil icon
5. Request service activation from Click support (disabled by default!)
6. If NOT on TAS-IX: provide domain + IP + port for firewall whitelisting
7. Static IP required — notify Click before changing

## Critical Implementation Rules

1. **Select the protocol first** — standard SHOP uses Prepare 0 / Complete 1; JSON Advanced/Split uses lookup 0 / Prepare 1 / completion 2
2. **Use the matching request encoding** — form-urlencoded for standard SHOP, JSON for Advanced/Split
3. **Amounts in SO'M for SHOP/Merchant/checkout** — fiscalization and Telegram instead use tiyin
4. **Verify each documented signature** with constant-time comparison; clarify authentication for JSON Getinfo/Compare, whose samples omit signing fields
5. **Standard SHOP: check request `error`** — a negative CLICK error expects -9, subject to the paid-cancellation ambiguity below
6. **Prevent duplicate billing/fulfillment** — standard SHOP uses `click_trans_id`; Split tracks `click_paydoc_id` + `attempt_trans_id` and phase/state
7. **Correlate the prepared billing record** in standard Complete and JSON Complete/Confirm/Check
8. **Standard Complete error=0 → paid outcome; error<0 → cancellation** — preserve scenario 8 and the upstream ambiguity described below
9. **Fiscalization mandatory** for more than 1 IKPU code
10. **Service must be activated** by Click support before real payments
11. **IP must be static** — notify Click before any change
12. **Log click_paydoc_id** — shown in user's SMS, needed for support queries
13. **Preserve ID precision** — standard/Split CLICK IDs are documented as 64-bit; do not use unchecked JavaScript `parseInt` or ordinary JSON parsing for full-width IDs
14. **Test standard SHOP without real charges** — current browser Playground and 15-scenario Postman generator are in `references/04-shop-api-testing.md`; neither verifies Advanced/Split

## Common Gotchas

- **Callback URLs must be publicly accessible** — `localhost` won't work in production
- **Prepare URL validation** — merchant cabinet validates format; must be valid HTTPS URL
- **sign_string concatenation** — NO separators between params
- **merchant_prepare_id overflow** — use proper integer, `Date.now() % 2147483647` causes collisions
- **Response must always be JSON** with all required fields, even on error
- **Standard Complete cancellation is contradictory upstream** — Postman scenario 7 repeats a confirmed payment with `error=0` and expects `-4`; scenario 8 cancels it with `error=-5017` and expects `-9`. The errors page agrees with negative-error cancellation; requests prose leaves tension after successful debit. Do not reorder the example's cancellation-before-paid check speculatively; obtain Click's production clarification.
- **Merchant fulfillment failure after successful debit is separate** — requests prose requires success acknowledgement, then real Merchant API `payment/reversal`, not a fabricated negative CLICK error.

## Reference Files — Practical Official-Documentation Guides

These guides preserve documented distinctions and flag source inconsistencies. Follow their canonical source links for the current enabled-service contract.

### SHOP API
- `references/01-shop-api-overview.md` — General provisions, terms, flow diagram
- `references/02-shop-api-requests.md` — Prepare & Complete full spec with code examples
- `references/03-shop-api-errors.md` — All error codes (Click-side and merchant-side)
- `references/04-shop-api-testing.md` — Browser Playground, Postman generator, exact 15-scenario matrix

### Payment Integration
- `references/05-payment-button.md` — Payment link URL and HTML form (with redirect)
- `references/06-inline-checkout.md` — checkout.js widget, createPaymentRequest() JS API

### Merchant API
- `references/07-merchant-api-overview.md` — General provisions, terms, flow diagram, contract info
- `references/08-merchant-api-requests.md` — All endpoints: invoice, payment status, reversal, card token
- `references/09-merchant-api-errors.md` — HTTP status codes

### Additional
- `references/10-click-pass.md` — QR-code POS payments, confirm mode
- `references/11-fiscalization.md` — OFD submit_items, submit_qrcode, get fiscal data
- `references/12-telegram-payments.md` — Bot setup, sendInvoice, pre_checkout_query, live mode
- `references/13-mobile-sdk.md` — Android SDK, iOS deep links, return_url handling
- `references/14-server-examples.md` — Official PHP, Django repos + community Node.js/TypeScript
- `references/15-cms-plugins.md` — WooCommerce, OpenCart, Drupal, 1C-Bitrix, Joomla, CS-Cart
- `references/16-advanced-shop.md` — JSON Getinfo/Prepare/Complete/Check/Compare, original-order signatures, auth/type caveats
- `references/17-split-shop.md` — JSON Getinfo/Prepare/Confirm, allocation sums, retry identity, auth/cancellation caveats
