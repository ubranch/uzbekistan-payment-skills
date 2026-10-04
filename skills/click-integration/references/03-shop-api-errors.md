# SHOP API — Error Codes (Ошибки)

> Source: https://docs.click.uz/shop-api/errors — last verified 2026-10-05.

## Response Format

The service provider returns a response in **JSON** format.

### Successful Operation Response

| # | Parameter | Type | Description |
|---|-----------|------|-------------|
| 1 | error | int | Operation status code. Must be "0" for success |
| 2 | error_note | varchar | Description of the status |
| 3 | click_trans_id | bigint (64-bit) | Payment ID in CLICK system; see the upstream type discrepancy below |
| 4 | merchant_trans_id | varchar | Payment ID in online store systems |
| 5 | merchant_prepare_id or merchant_confirm_id | int | Payment ID in the billing system of the supplier for confirmation |

The current [request specification](https://docs.click.uz/shop-api/requests) uses `bigint` for `click_trans_id` and `click_paydoc_id`, while the errors page still labels `click_trans_id` as `int`. Preserve the wider ID; do not truncate to 32 bits or round through an unsafe JavaScript number. See [ID precision and the bounded Node.js example](02-shop-api-requests.md#id-precision).

### Error Operation Response

| # | Parameter | Type | Description |
|---|-----------|------|-------------|
| 1 | error | int | Operation status code |
| 2 | error_note | varchar | Description of the status |

## Errors Returned BY CLICK System (in requests to your server)

| # | error | error_note | Description |
|---|-------|------------|-------------|
| 1 | 0 | Success | Successful operation |
| 2 | < 0 | Description of the error | When receiving a negative error code, the supplier MUST cancel the payment in the billing system and return the error code **-9** |

## Errors Returned BY Supplier's System (your server's responses)

| # | error | error_note | Description |
|---|-------|------------|-------------|
| 1 | 0 | Success | Successful request |
| 2 | -1 | SIGN CHECK FAILED! | Signature verification error |
| 3 | -2 | Incorrect parameter amount | Invalid payment amount |
| 4 | -3 | Action not found | The requested action is not found |
| 5 | -4 | Already paid | Transaction was previously confirmed; duplicate successful confirmation expects -4. Cancellation of a confirmed payment is contradictory upstream; see below |
| 6 | -5 | User does not exist | User/order not found (check parameter merchant_trans_id) |
| 7 | -6 | Transaction does not exist | Transaction not found (check parameter merchant_prepare_id) |
| 8 | -7 | Failed to update user | Error occurred while changing user data (changing account balance, etc.) |
| 9 | -8 | Error in request from click | Error in the request from CLICK (not all transmitted parameters, etc.) |
| 10 | -9 | Transaction cancelled | Transaction was previously canceled (when attempting to confirm or cancel previously canceled transaction) |

## Error Handling Rules

1. **When Click sends error < 0 in Prepare**: Your server should return error -9 (Transaction cancelled)
2. **When Click sends error < 0 in Complete**: Your server MUST cancel/unreserve the order and return error -9
3. **When Click sends error = 0 in Complete**: This means funds were deducted — you MUST fulfill the order. You CANNOT return an error (except -4 if already paid, -9 if already cancelled)
4. **If fulfillment fails after successful Complete**: Respond "success" and then initiate cancellation via Merchant API reversal endpoint

## Upstream Cancellation Contradiction

The [errors page](https://docs.click.uz/shop-api/errors) says any negative CLICK error requires billing cancellation and response `-9`. The [requests prose](https://docs.click.uz/shop-api/requests) limits error responses after successful debit to duplicate confirmation (`-4`) or confirmation of a previously cancelled payment (`-9`). The live [Postman suite](https://docs.click.uz/testing/postman), however, explicitly expects `-9` in scenario 8: `error=-5017` cancels a payment already confirmed in scenario 6. Scenario 7 sends successful confirmation again and expects `-4`.

These statements are not a resolved universal rule for cancelling paid transactions. The example handler preserves the explicit scenario 8 cancellation-before-paid-check behavior; do not reorder it speculatively. Obtain Click's production clarification. Merchant fulfillment failure after a successful debit is a **different case**: acknowledge success and initiate the real Merchant API `payment/reversal`, not a fabricated negative CLICK error.
