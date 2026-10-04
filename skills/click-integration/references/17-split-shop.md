# Split Shop — JSON Prepare Splitting and Confirm

> Source: [official Split Shop](https://docs.click.uz/additional/shop-split) — last verified 2026-10-05.
> Exact source evidence: [official route module](https://docs.click.uz/assets/js/ab94b0fe.181b313c.js), metadata `additional/shop-split`, source `@site/docs/additional/shop-split.md`. The [route manifest](https://docs.click.uz/assets/js/main.f7b73d63.js) and [asset resolver](https://docs.click.uz/assets/js/runtime~main.a2df20c5.js) identify this module. Static deep-route HTML can be the SPA homepage, not the requested protocol page.

## Select Split Shop, Not a Standard SHOP Extension

SHOP SPLIT returns counterparties and their allocated amounts in the **Prepare response**. Confirm that Click has configured the service for this JSON contract and its counterparties; do not merely append a `split` array to a standard form-urlencoded SHOP callback.

| Action | Split Shop operation | Distinction |
|---|---|---|
| 0 | Getinfo (optional) | Customer/account information lookup before payment |
| 1 | Prepare | Verify payment and return details plus allocation list |
| 2 | Confirm | Complete the prepared billing payment |

Requests flow CLICK → supplier; responses flow supplier → CLICK. Use `Content-Type: application/json; charset=utf-8` for the documented JSON payloads. [Standard SHOP](02-shop-api-requests.md) instead sends form-urlencoded action 0 Prepare/action 1 Complete. [Advanced Shop](16-advanced-shop.md) adds action 3 Check/action 4 Compare; the Split source documents neither of those actions. Do not assume all JSON SHOP variants accept the same actions.

## Signed Fields, Authentication, and Precision

| Field | Official type | Meaning |
|---|---|---|
| `click_paydoc_id` | bigint | CLICK payment ID, Prepare/Confirm |
| `attempt_trans_id` | bigint | Request-attempt ID, Prepare/Confirm |
| `service_id` | int | Configured CLICK service |
| `action` | int | 0 Getinfo, 1 Prepare, 2 Confirm |
| `params` | object | Configured account/order/payment key-value pairs |
| `merchant_prepare_id` | bigint | Optional Prepare response ID; in Confirm request table |
| `merchant_confirm_id` | bigint | Optional Confirm response ID |
| `sign_time` | varchar | `YYYY-MM-DD HH:mm:ss`, Prepare/Confirm |
| `sign_string` | varchar | Verification hash, Prepare/Confirm |

The documented signature is:

```text
MD5(click_paydoc_id + attempt_trans_id + service_id + SECRET_KEY + params + action + sign_time)
```

The provider defines `params` precisely as **“all values of the params object's pairs in the transmitted order”** (`все значения пар объекта params в переданном порядке`). Concatenate values without keys or separators; do not sign `JSON.stringify(params)`, sort keys, reformat amounts, or add `merchant_prepare_id`/the response `split` list to this formula. The Confirm field table points to this same formation section.

Preserve the original request value representation/order through verification. The source does not specify a canonical serialization for nested objects, arrays, null/boolean values, or numeric lexical normalization. Use an agreed flat schema and provider signing fixtures; do not invent these conversions. JavaScript parsing/enumeration can change numeric lexemes or integer-like key order. Verify with a constant-time comparison, check the configured service, and keep the secret on the backend.

Getinfo's table/example has **no** signature fields or `Auth` header. This omission does not authorize publishing customer data to unauthenticated callers. Confirm the intended access restrictions with Click; neither fabricate a required Merchant API SHA1 header nor expose a secret to the frontend. There is no documented signature on the merchant's response `split` list; do not invent one.

`bigint` means preserve full-width IDs. All examples below deliberately use safely representable small JSON numbers. A full-width JavaScript service cannot first call ordinary `JSON.parse`/Express JSON parsing and then recover a rounded ID using `BigInt` or a reviver. Use lossless parsing before conversion and exact decimal-string/native 64-bit storage; serialize numeric JSON tokens where the protocol requires numbers. Native `BigInt` is not directly JSON-stringifiable, and converting every ID to a quoted JSON string is not a documented protocol change. No new package is prescribed here. See [the standard SHOP precision boundary](02-shop-api-requests.md#id-precision), which guards raw form values only, not already-rounded JSON values.

## Action 0 — Optional Getinfo

Request: `service_id`, `action=0`, `params`. This lookup displays information before payment; it must not credit an account or create a paid outcome.

Official request:

```json
{
  "action": 0,
  "service_id": 123,
  "params": {"contract": "***", "full_name": "***", "service_type": "***"}
}
```

Response: `params` for display, `error`, `error_note`. Official success:

```json
{
  "error": 0,
  "error_note": "Успешно",
  "params": {"licshet": "***", "fio": "***", "address": "***", "period": "***"}
}
```

The source refers to a parameter dictionary without defining every service's required keys; [Advanced Shop's dictionary](16-advanced-shop.md#parameter-dictionary-and-errors) is useful context, not proof that a particular Split service requires all entries or aliases `licshet`/`fio` to other names. Official lookup failure: `{"error":-250,"error_note":"Абонент не найден"}`.

## Action 1 — Prepare and Allocate

Prepare obtains the information needed for payment (branches, payment details, transit and settlement accounts) and returns the counterparties' allocation. Validate the account/order, amount, and allocation before marking anything paid. Persist the verified payment identity, parameters, expected total, allocation, and prepared billing ID so a repeated request does not recalculate different recipients or reserve/credit twice.

Official request (`***` is the source's redacted data/hash marker; a real request needs the computed signature):

```json
{
  "action": 1,
  "click_paydoc_id": 123,
  "attempt_trans_id": 345,
  "service_id": 123,
  "sign_time": "2020-03-04 17:00:00",
  "sign_string": "***",
  "params": {"contract": "***", "full_name": "***", "service_type": "***", "amount": 1000}
}
```

Prepare response fields:

| Field | Official type | Meaning |
|---|---|---|
| `click_paydoc_id` | bigint | Echo payment ID |
| `attempt_trans_id` | bigint | Echo attempt ID |
| `merchant_prepare_id` | bigint | Billing ID for confirmation; optional in response table |
| `params` | object | Information needed to carry out payment |
| `split` | item[] | Counterparties participating in allocation |
| `error` | int | 0 success, negative error |
| `error_note` | varchar | Request result description |

### Allocation Invariants

| Item field | Official type | Meaning |
|---|---|---|
| `cntrg_id` | int | Counterparty ID in CLICK |
| `amount` | float | Amount allocated to this counterparty |

**The sum of all counterparty amounts must equal the payment amount.** The source explicitly warns that item field names may differ (`Наименования полей могут отличаться`); `cntrg_id`/`amount` are the documented sample names. Agree the live service's field names and registered counterparty IDs with Click instead of inventing recipient identifiers.

Amounts are in so'm. Calculate/compare allocations using exact decimal or integer minor units internally (e.g. 45000 + 55000 = 100000 tiyin for a 1000-so'm payment); convert back to the documented so'm values on the wire. Do not silently accept a floating-point remainder or apply fiscalization/Telegram's tiyin scale to the Split JSON response.

Official success, with 450 + 550 = 1000:

```json
{
  "click_paydoc_id": 123,
  "attempt_trans_id": 345,
  "merchant_prepare_id": 12345,
  "error": 0,
  "error_note": "Успешно",
  "params": {
    "branch_id": 123,
    "payment_account": "***",
    "payment_mfo": "***",
    "transit_account": "***",
    "transit_mfo": "***"
  },
  "split": [
    {"cntrg_id": 123, "amount": 450},
    {"cntrg_id": 321, "amount": 550}
  ]
}
```

Official failure echoes both payment identifiers and returns `error=-250`, `error_note`, with no `split` or payment-detail object. The -250 example is not enumerated in the standard -1…-9 table; confirm service-specific error codes rather than treating it as universally prescribed.

## Action 2 — Confirm

Confirm request fields: all Prepare request fields with `action=2`, plus `merchant_prepare_id` from Prepare. Its signature is the same formula; the prepare ID is **not** added to the hash by analogy with standard SHOP.

```json
{
  "action": 2,
  "click_paydoc_id": 123,
  "attempt_trans_id": 345,
  "merchant_prepare_id": 12345,
  "service_id": 123,
  "sign_time": "2020-03-04 17:00:00",
  "sign_string": "***",
  "params": {"contract": "***", "full_name": "***", "service_type": "***", "amount": 1000}
}
```

Correlate the prepared billing record, payment identity, amount, and saved allocation; perform the billing confirmation atomically. Prepare does not itself prove debit/settlement. Confirm must not duplicate fulfillment or allocations on retries.

Confirm response fields: `click_paydoc_id` (bigint), `attempt_trans_id` (bigint), optional `merchant_confirm_id` (bigint), `params` (object), `error` (int), `error_note` (varchar). Official success:

```json
{
  "click_paydoc_id": 123,
  "attempt_trans_id": 345,
  "merchant_confirm_id": 12345,
  "error": 0,
  "error_note": "Успешно",
  "params": {}
}
```

Official failure shape:

```json
{
  "click_paydoc_id": 123,
  "attempt_trans_id": 345,
  "error": -250,
  "error_note": "Описание ошибки"
}
```

### Retry Identity

The source repeats: **a repeated attempt has a new ID; `click_paydoc_id` and `attempt_trans_id` together form a unique value**. Keep both, not just the payment ID or only the attempt ID. Correlate Prepare and Confirm within that identity and stored billing state; action distinguishes the phase, not a second payment. A new attempt is distinct from retransmission of an existing phase, but a new attempt ID must not cause the same already-paid order to be credited again. Preserve payment-level history across attempts and use the saved prepare ID rather than assuming a new attempt reuses it.

## Errors and Unresolved Cancellation Input

| `error` | Official note | Meaning |
|---|---|---|
| 0 | Success | Request succeeded |
| -1 | SIGN CHECK FAILED! | Invalid signature |
| -2 | Incorrect parameter amount | Incorrect amount |
| -3 | Action not found | Unsupported action |
| -4 | Already paid | Previously confirmed transaction; table mentions attempted confirmation/cancellation |
| -5 | User does not exist by params | Account/order not found from `params` |
| -6 | Transaction does not exist | Check `merchant_prepare_id` |
| -7 | Failed to update user | Billing update failure |
| -8 | Error in request from click | Missing/invalid request data |
| -9 | Transaction cancelled | Previously cancelled transaction |

The source's incoming-error table says a negative CLICK error requires billing cancellation and `-9`, but its Prepare/Confirm request field tables and examples contain **no `error` field**. Its -4 description also mentions cancellation of a confirmed transaction. Do not fabricate a cancellation action or transplant standard SHOP's `error` input/order from its Postman tests. Obtain the actual Split service cancellation payload and precedence from Click before production. [Standard SHOP's explicit contradiction](03-shop-api-errors.md#upstream-cancellation-contradiction) remains documented separately; it is not a resolved rule for this JSON protocol.

Validate these payloads with Click's agreed service fixtures. The [official 15-scenario Postman generator](04-shop-api-testing.md) targets standard form-urlencoded SHOP, not Split allocation/Confirm. Never label a passing standard suite as proof of Split correctness.
