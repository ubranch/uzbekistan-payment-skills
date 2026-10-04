# Additional Integration References

## Payment Button & QR Code Generation

Include the Payme JS SDK to auto-generate payment buttons and QR codes:

```html
<script src="https://cdn.paycom.uz/integration/js/checkout.min.js"></script>
```

### Generate Payment Button
```html
<body onload="Paycom.Button('#form-payme', '#button-container')">
  <form id="form-payme" method="POST" action="https://checkout.paycom.uz/">
    <input type="hidden" name="merchant" value="{MERCHANT_ID}">
    <input type="hidden" name="account[order_id]" value="197">
    <input type="hidden" name="amount" value="500">
    <input type="hidden" name="lang" value="ru">
    <input type="hidden" name="button" data-type="svg" value="colored">
    <div id="button-container"></div>
  </form>
</body>
```

Button styles: `value="colored"` or `value="white"`, format: `data-type="svg"` or `data-type="png"`, width: `data-width="200px"`

### Generate QR Code
```html
<body onload="Paycom.QR('#form-payme', '#qr-container')">
  <form id="form-payme" method="POST" action="https://checkout.paycom.uz/">
    <input type="hidden" name="merchant" value="{MERCHANT_ID}">
    <input type="hidden" name="account[order_id]" value="197">
    <input type="hidden" name="amount" value="500">
    <input type="hidden" name="lang" value="ru">
    <input type="hidden" name="qr" data-width="250">
    <div id="qr-container"></div>
  </form>
</body>
```

### JS API Methods
```javascript
Paycom.Button(form_selector, button_container_selector);
Paycom.QR(form_selector, qr_container_selector);
```
Selectors accept CSS selectors, jQuery objects, or HTML DOM objects.

---

## Checkout/Receipt Submission Errors

These errors occur when sending a receipt (check) to Payme via POST/GET:

| Code | Description |
|------|-------------|
| -31601 | Merchant not found or blocked |
| -31610 | Invalid field value |
| -31611 | Payment amount below minimum |
| -31612 | Payment amount above maximum |
| -31622 | Merchant service unavailable |
| -31623 | Merchant service working incorrectly |
| -31630 | Insufficient funds, invalid card number, expired card, blocked card, or corporate card |

---

## Telegram Bot Integration

### Creating a Bot
1. Open BotFather: https://telegram.me/BotFather
2. Send `/newbot` command
3. Choose name and username for the bot
4. Save the API token

### Connecting Bot to Payme
1. Send `/mybots` to BotFather
2. Select your bot → "Bot Settings" → "Payments"
3. Select "Payme" from the list
4. BotFather provides a payment provider token
5. Use this token with Telegram Bot API's `sendInvoice` method

### Adding Kassa for Bot
In Payme Business cabinet:
1. Go to your business
2. Click "Add kassa"
3. Select "Telegram bot" type
4. Configure the kassa settings

### Payment Flow Diagram
1. User selects product in bot
2. Bot calls `sendInvoice` Telegram API method
3. User sees payment form in Telegram
4. User pays via Payme
5. Telegram sends `successful_payment` update to bot
6. Bot confirms order

### Telegram `sendInvoice` Parameters
Key params: `provider_token`, `title`, `description`, `payload`, `currency` ("UZS"), `prices` (array of `LabeledPrice`)

---

## Mobile SDK Integration (Android)

### Setup
Add to `app/build.gradle`:
```groovy
dependencies {
    compile 'uz.paycom:payment:$last_version'
}
```

### Initialize Payment
```java
@Override
public void onClick(View v) {
    PaycomSdk.checkout(
        MainActivity.this,      // Activity
        paycomId,               // Merchant/Kassa ID
        amount,                 // Amount in tiyin
        account,                // Account object
        lang,                   // Language (ru/uz/en)
        description,            // Payment description
        detail                  // Detail object (fiscalization)
    );
}
```

### Handle Result
```java
@Override
protected void onActivityResult(int requestCode, int resultCode, Intent data) {
    super.onActivityResult(requestCode, resultCode, data);
    if (requestCode == PaycomSdk.REQUEST_CODE_CHECKOUT) {
        if (resultCode == RESULT_OK) {
            // Payment successful
            String token = data.getStringExtra(PaycomSdk.EXTRA_TOKEN);
        } else if (resultCode == RESULT_CANCELED) {
            // Payment cancelled
        }
    }
}
```

The SDK returns a **card token** on success. Use this token with Subscribe API `receipts.create` + `receipts.pay` on your backend to complete the payment.

### Mobile SDK Test Cards
Same test cards as Subscribe API (see main SKILL.md). SMS code: `666666`.

---

## CMS Plugins

Official Payme plugins for popular CMS platforms:

| CMS | Requirements | GitHub |
|-----|-------------|--------|
| WooCommerce | PHP 5.4+, WordPress 4.x+, WooCommerce 3.x+ | https://github.com/PaycomUZ/woocommerce-payment-gateway |
| OpenCart 2.x | PHP 5.3+, OpenCart 2.x | https://github.com/PaycomUZ/opencart2-payment-module |
| OpenCart 3.x | PHP 7.0+, OpenCart 3.x | https://github.com/PaycomUZ/opencart3-payment-module |
| PrestaShop | PHP 5.4+, PrestaShop 1.6+ | https://github.com/PaycomUZ/prestashop-payment-module |

---

## Server Implementation Examples

Official reference implementations:

| Language | GitHub URL |
|----------|-----------|
| PHP | https://github.com/PaycomUZ/paycom-integration-php-template |
| Node.js | https://github.com/PaycomUZ/paycom-integration-nodejs-template |

---

## Finding Kassa ID and KEY

### Finding Kassa ID
1. Log into Payme Business cabinet
2. Navigate to your business → Kassa list
3. Click on the kassa
4. ID is shown in the developer parameters section (24-char hex string)
5. Also visible in browser URL bar

### Finding KEY (Password)
1. Log into Payme Business cabinet
2. Navigate to your business → Kassa list
3. Click on the kassa → "Developer parameters"
4. KEY is displayed (36-char string)
5. TEST KEY for sandbox is shown separately

---

## Technical Support

Payme Business technical support:
- Phone: +998 78 150-22-26
- Phone: +998 78 150-22-24

---

## Resources & Logos

| Asset | URL |
|-------|-----|
| Color logo (PNG) | https://cdn.payme.uz/logo/payme_color.png |
| Mono logo (PNG) | https://cdn.payme.uz/logo/payme_white.png |
| Color logo (SVG) | https://cdn.payme.uz/logo/payme_color.svg |
| Mono logo (SVG) | https://cdn.payme.uz/logo/payme_white.svg |
| Checkout JS SDK | https://cdn.paycom.uz/integration/js/checkout.min.js |
