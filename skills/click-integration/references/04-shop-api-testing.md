# Standard SHOP API — Playground and Postman Testing

> Sources: [browser Playground](https://docs.click.uz/testing/playground), [Postman generator](https://docs.click.uz/testing/postman) — last verified 2026-10-05.
> Exact official SPA source: [Playground module](https://docs.click.uz/assets/js/c9a82400.4e12ebbc.js), [Postman page and generator](https://docs.click.uz/assets/js/a1331985.5159e565.js). Direct deep-route HTML fetches may return the homepage; these modules identify the actual testing routes.

These tools simulate **standard SHOP** form-urlencoded Prepare (`action=0`) and Complete (`action=1`) against **your endpoint**. They do not establish that a real card was debited and do not test the JSON Advanced/Split protocols. Use isolated test billing/orders: a simulated Complete can still trigger your own fulfillment code. No real small-value payment is needed for these checks.

## Browser Playground

1. Open the official Playground. Set the test service ID, test secret, Prepare URL, and Complete URL. Never use production credentials for an exploratory test.
2. Set an amount in so'm and test `click_trans_id`/`click_paydoc_id`. The current UI generates a read-only `merchant_trans_id` with a regenerate control; arrange matching isolated billing data before running a success case.
3. Press **Prepare** and inspect the displayed form body, HTTP status, and JSON response. The UI saves the returned `merchant_prepare_id` for Complete.
4. Press **Complete**, or use **Prepare → Complete** for the two-step flow. The flow control checks for a returned prepare ID; it is not proof of actual settlement or business correctness.
5. Exercise duplicate/cancellation states deliberately and compare both the response and the test billing ledger. Test IDs must stay within JavaScript's safe-integer range when using the UI's ordinary JSON response parser.

### CORS and request direction

The source executes browser `fetch(merchantUrl, {method: 'POST', headers: {'Content-Type': 'application/x-www-form-urlencoded'}, body})`. Requests go **directly from the browser to your server**, not through a Click sandbox proxy. The secret signs the request in the browser; do not share it in screenshots, logs, exported collections, or frontend application code.

For a dedicated test endpoint, narrowly allow the official `https://docs.click.uz` origin and needed methods/headers, handling OPTIONS if your deployment requires it. Do **not** disable CORS or add wildcard permissions in production. The official page mentions an Allow CORS extension; that is not production hardening. Use Postman when browser origin/private-network restrictions prevent a local endpoint test. Real Click server-to-server callbacks are not authenticated by CORS; signature verification remains necessary.

## Postman Collection: Setup and State

1. Open the generator and fill `service_id`, test `secret_key`, `click_trans_id`, `click_paydoc_id`, `amount`, `merchant_trans_id`, `sign_time`, Prepare URL, and Complete URL. Choose an existing unpaid test order and an amount **other than 300**; scenario 5 forces 300 to test mismatch. The form initializes the date/time; verify it against the signed value you send.
2. Generate/download the collection JSON and import it through **File → Import** in Postman. Inspect **Edit → Variables**. The collection contains secrets; keep it out of version control and public workspaces.
3. Run scenarios 1–3 in order with the intended fixtures. For scenario 2, set a nonexistent test account/order; restore an existing unpaid one before scenario 3. After scenario 3, manually copy the returned `merchant_prepare_id` into collection variables. Complete signatures are recomputed by per-request CryptoJS.MD5 pre-request scripts.
4. Run 4–8 against that prepared payment. Before scenario 9, create/select a fresh unpaid test payment and use fresh transaction IDs as needed by your billing model. Copy scenario 9's prepare ID; scenario 10 overrides it with `999999999`; restore the valid ID for scenario 11. Before scenario 12, select another fresh unpaid fixture and copy its prepare ID for 13–15.
5. Compare response bodies and billing effects with the matrix below. Do not blindly **Run Collection** before populating the state-dependent IDs. The generator reuses collection variables; it does not create your merchant's orders or automatically reset paid/cancelled billing state between groups.

The official table/prose calls its expectation `error_code`, but standard SHOP response bodies use **`error`**, as shown by the official JSON example. Do not switch your callback to the Merchant API `error_code` envelope. Return the standard Prepare/Complete identifiers and response fields from [the request guide](02-shop-api-requests.md).

## Exact 15-Scenario Matrix

| # | Request / action | Expected response `error` | Input/state under test |
|---|---|---|---|
| 1 | Prepare / 0 | -1 | Incorrect signature |
| 2 | Prepare / 0 | -5 | User/order not found |
| 3 | Prepare / 0 | 0 | Valid unpaid order; save `merchant_prepare_id` |
| 4 | Complete / 1 | -1 | Incorrect signature |
| 5 | Complete / 1 | -2 | Incorrect amount, forced `amount=300` |
| 6 | Complete / 1 | 0 | Successful confirmation of scenario 3 payment |
| 7 | Complete / 1 | -4 | Repeat successful confirmation of scenario 6 payment (`error=0`) |
| 8 | Complete / 1 | -9 | Cancel the already confirmed payment, forced CLICK `error=-5017` |
| 9 | Prepare / 0 | 0 | New preparation for prepare-ID validation; save its prepare ID |
| 10 | Complete / 1 | -6 | Nonexistent `merchant_prepare_id=999999999` |
| 11 | Complete / 1 | 0 | Valid prepare ID from scenario 9 |
| 12 | Prepare / 0 | 0 | New preparation for cancellation; save its prepare ID |
| 13 | Complete / 1 | -9 | Cancel payment, forced CLICK `error=-5017` |
| 14 | Complete / 1 | -9 | Repeat cancellation, forced CLICK `error=-5017` |
| 15 | Complete / 1 | -9 | Successful-confirmation request (`error=0`) for the previously cancelled payment |

## Cancellation Ambiguity: Preserve Scenario 8

Both the rendered official table and its generator specify scenario 8's `-9`, distinct from scenario 7's duplicate-confirmation `-4`. The [errors page](https://docs.click.uz/shop-api/errors) also requires cancellation and `-9` for a negative CLICK error. The [requests prose](https://docs.click.uz/shop-api/requests), however, says successful Prepare/debit should not receive an error except duplicate confirmation (`-4`) or confirmation of a previously cancelled payment (`-9`). This leaves cancellation after success contradictory upstream.

The [example handler](02-shop-api-requests.md) deliberately checks a negative CLICK error before its paid-state check to match scenario 8. Do not reorder it on an assumption that cancellation of a paid transaction must always return `-4`. Obtain Click's production clarification for this case. A **merchant fulfillment failure after debit** is different: the prose requires a success acknowledgement followed by real Merchant API `payment/reversal`, not an invented provider-failure callback.

Passing these request simulations does not activate a service, register a legacy software report, or prove real settlement. Coordinate production activation with Click separately.
