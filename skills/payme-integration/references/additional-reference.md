# Additional Integration References

Official sources checked **2026-10-05**. Legacy SDK/module requirements below describe provider documentation, not current platform support or tested live payments.

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
1. Send `/mybots` to BotFather.
2. Select the bot → "Bot Settings" → "Payments".
3. Select the provider listed as **Paycom.Uz** in Payme's documentation.
4. Choose **Connect Paycom Test** for intentional testing or **Connect Paycom Live** for live onboarding. For live onboarding, sign in to Payme Business, select the active business, and select/create its bot kassa.
5. Use the resulting payment provider token with Telegram Bot API's `sendInvoice`; it is distinct from the bot's API token.

### Adding Kassa for Bot

The business must be active. A Telegram bot works **only with a dedicated bot kassa**, not an existing web/on-site kassa. Enter the business through the bot's provider connection flow to create/select the bot kassa.

| Setting | Requirement |
|---------|-------------|
| Kassa name | At most 256 characters |
| Payment limits | Set minimum and maximum amounts in **UZS (sums)** in the cabinet |
| Rounding | Enable "Round to sums" only if required |

Sources: [Bot connection and restrictions](https://developer.help.paycom.uz/telegram-bot/podklyuchenie-bota/), [bot kassa settings](https://developer.help.paycom.uz/telegram-bot/dobavlenie-kassy-dlya-bota/).

### Payment Flow
1. The bot calls `sendInvoice` for the selected order.
2. After confirmation, Telegram sends a `pre_checkout_query` / `PreCheckoutQuery`.
3. Validate the order, currency, amount and availability; call `answerPreCheckoutQuery` **within 10 seconds of receiving the query** (`ok: true` to approve, otherwise `ok: false` with an error message).
4. Telegram/Payme processes the approved payment.
5. Fulfill the order only after the bot receives `successful_payment` (`SuccessfulPayment`); an approved pre-checkout query is not proof of payment.

Sources: [Payme payment flow and 10-second deadline](https://developer.help.paycom.uz/telegram-bot/diagramma-protsessa-oplaty/), [Telegram answerPreCheckoutQuery](https://core.telegram.org/bots/api#answerprecheckoutquery).

### Telegram `sendInvoice` Parameters
Key params: `provider_token`, `title`, `description`, `payload`, `currency` (`"UZS"`), `prices` (array of `LabeledPrice`, amounts in the currency's smallest units — tiyin for UZS). Cabinet limits above are in sums, not tiyin.

---

## Mobile SDK Integration (Android)

### Setup and Source Caveat

Official links: [Android SDK repository](https://github.com/PaycomUZ/AndroidSDK/), [library connection](https://developer.help.paycom.uz/integratsiya-s-mobilnym-prilozheniem/podklyuchenie-biblioteki/), [result handling](https://developer.help.paycom.uz/integratsiya-s-mobilnym-prilozheniem/obrabotka-rezultata/).

The provider page still shows legacy Gradle `compile` with an unspecified version (`uz.paycom:payment:$last version`), and the [mobile overview](https://developer.help.paycom.uz/integratsiya-s-mobilnym-prilozheniem/) still links a historical Bintray publication. Confirm an available SDK version/resolver and supported Android configuration with Payme; this guide does not invent a current version or dependency repository.

### Open Card Tokenization UI

Use `PaymentActivity`, not an undocumented checkout facade. The sample below is placed in `YourActivity`, with an order sum of 5 UZS and an explicit sandbox flag set by the app's environment.

```java
import android.content.Intent;
import uz.paycom.payment.PaymentActivity;
import uz.paycom.payment.model.Result;
import uz.paycom.payment.utils.PaycomSandBox;
import static uz.paycom.payment.PaymentActivity.EXTRA_ID;
import static uz.paycom.payment.PaymentActivity.EXTRA_AMOUNT;
import static uz.paycom.payment.PaymentActivity.EXTRA_SAVE;
import static uz.paycom.payment.PaymentActivity.EXTRA_LANG;
import static uz.paycom.payment.PaymentActivity.EXTRA_RESULT;

// Inside YourActivity's click handler:
Intent intent = new Intent(YourActivity.this, PaymentActivity.class);
intent.putExtra(EXTRA_ID, merchantId); // Public kassa ID, never the secret key
final Double sum = 5.00; // UZS in this SDK; backend receipt amount is 500 tiyin
intent.putExtra(EXTRA_AMOUNT, sum);
intent.putExtra(EXTRA_SAVE, true); // Offer card saving for repeated payments
intent.putExtra(EXTRA_LANG, "RU"); // Documented values: "RU" or "UZ"
PaycomSandBox.setEnabled(useSandbox); // true only for intentional sandbox use
startActivityForResult(intent, 0);
```

**Amount boundary:** unlike checkout/Subscribe API's tiyin amounts, this SDK takes a UZS `Double` sum. The official [PaymentActivity source](https://raw.githubusercontent.com/PaycomUZ/AndroidSDK/HEAD/payment/src/main/java/uz/paycom/payment/PaymentActivity.java) reads a double, and [VerifyCardTask](https://raw.githubusercontent.com/PaycomUZ/AndroidSDK/HEAD/payment/src/main/java/uz/paycom/payment/api/task/VerifyCardTask.java) multiplies it by 100 before the API call. Confirm the behavior of the SDK version you install; keep backend amounts as integer tiyin.

### Handle Result
```java
@Override
protected void onActivityResult(int requestCode, int resultCode, Intent data) {
    super.onActivityResult(requestCode, resultCode, data);
    if (requestCode != 0) return;
    if (resultCode == RESULT_OK && data != null) {
        Result result = data.getParcelableExtra(EXTRA_RESULT);
        if (result != null) {
            String token = result.getToken();
            // Transmit token securely to your backend; do not log it.
            // RESULT_OK here means card-token UI success, NOT payment success.
        }
    } else if (resultCode == RESULT_CANCELED) {
        // Card-token UI cancelled; no payment confirmation.
    }
}
```

The documented `Result` fields are:

| Field | Meaning |
|-------|---------|
| number | Masked card number |
| expire | Card expiry |
| token | Card token to send to the backend for receipt payment |
| recurrent | Reusable token if true; if false, one payment with exactly the same amount |
| verify | Whether SMS cardholder verification passed |

The official [Result class](https://raw.githubusercontent.com/PaycomUZ/AndroidSDK/HEAD/payment/src/main/java/uz/paycom/payment/model/Result.java) exposes `getToken()`; its historical recurring getter is spelled `isRecurent()`, while the documentation calls the field `recurrent`. Do not assume differently named getters exist.

On the backend, use the token with Subscribe API `receipts.create` + `receipts.pay`, and verify the receipt's paid state (`4`) before fulfillment. The returned token and Android `RESULT_OK` are **not proof of payment**. No kassa secret key belongs in the app.

### Mobile SDK Test Cards
Use current [Subscribe API test cards](../SKILL.md#test-cards-subscribe-api) for the configured sandbox; SMS code: `666666`. The historical SDK README lists older expiry values, so do not treat its card table as current sandbox evidence.

---

## CMS Plugins

The [official CMS index](https://developer.help.paycom.uz/plaginy-dlya-cms/) lists these **nine** repositories:

| CMS | Historical requirements listed by Payme | GitHub |
|-----|-----------------------------------------|--------|
| WooCommerce | PHP 5.4+, WordPress 4.x+, WooCommerce 3.x+ | https://github.com/PaycomUZ/woocommerce-payment-gateway |
| OpenCart 2.x (listed as "OperCart") | PHP 5.3+, web server, MySQLi | https://github.com/PaycomUZ/PayMe-Gateway-for-OperCart-ver-2.x |
| OpenCart 3.x | OpenCart 3.x | https://github.com/PaycomUZ/opencart-payment-gateway |
| Magento | PHP 7.1 for Magento 2.2; PHP 7.0 for Magento 2.1.11 | https://github.com/PaycomUZ/Magento-Payment-Module |
| JoomShopping | Joomla 3.x+, JoomShopping 4.15+ | https://github.com/PaycomUZ/Joomshopping-Payment-Payme-gateway |
| VirtueMart | Joomla 3.7.0+, VirtueMart 3.0.10+ | https://github.com/PaycomUZ/virtuemart-gateway-payme |
| 1C-Bitrix (listed as "1C-Bitriks") | PHP 5.3+, Bitrix 18.5.150+, web server, MySQLi | https://github.com/PaycomUZ/Payme-modul-for-1C-Bitriks |
| Webasyst Shop-Script | PHP 5.3+, Webasyst 1.10.7+, Shop-Script 8.1.1+, web server, MySQLi | https://github.com/PaycomUZ/PayMe-Gateway-Webasyst-Shop-Script |
| CS-Cart | Provider requirements are inconsistent; check repository compatibility | https://github.com/PaycomUZ/Payme-modul-for-CS-Cart |

These old requirements are not recommendations to deploy obsolete PHP/CMS versions or evidence of current maintenance. The CS-Cart entry incorrectly names PrestaShop 1.4.10+ in its requirements; that is not evidence of an official PrestaShop module. Confirm compatibility against the actual repository and Payme before installing.

---

## Server Implementation Examples

The [official server examples page](https://developer.help.paycom.uz/primer-realizatsii-servera/) links:

| Language | GitHub URL |
|----------|-----------|
| PHP | https://github.com/PaycomUZ/paycom-integration-php-template |
| Java / Kotlin | https://github.com/PaycomUZ/paycom-integration-java-template |

It does not list an official Node.js template. Node.js/Go routing patterns in [SKILL.md](../SKILL.md#common-implementation-patterns) are illustrative merchant-side patterns, not official templates.

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

Source: [Payme resources](https://developer.help.paycom.uz/resursy/) (checked 2026-10-05).

| Asset | URL |
|-------|-----|
| Color logo (PNG) | https://cdn.payme.uz/logo/payme_color.png |
| Mono logo (PNG) | https://cdn.payme.uz/logo/payme_white.png |
| Color logo (SVG) | https://cdn.payme.uz/logo/payme_color.svg |
| Mono logo (SVG) | https://cdn.payme.uz/logo/payme_white.svg |
| Checkout JS SDK | https://cdn.paycom.uz/integration/js/checkout.min.js |
