# Advanced Shop — JSON Getinfo, Prepare, Complete, Check, Compare

> Source: [official Advanced Shop](https://docs.click.uz/additional/get-info) — last verified 2026-10-05.
> Exact source evidence: [official route module](https://docs.click.uz/assets/js/ae4eca94.2228372e.js), metadata `additional/get-info`, source `@site/docs/additional/get-info.md`. The [route manifest](https://docs.click.uz/assets/js/main.f7b73d63.js) and [asset resolver](https://docs.click.uz/assets/js/runtime~main.a2df20c5.js) identify this module. The live site is a JavaScript SPA; a static deep-route fetch can return homepage content instead.

## Select This Protocol Deliberately

Advanced Shop is a **separate JSON callback contract**, not standard SHOP with extra fields. Confirm which service protocol Click has enabled before configuring your merchant endpoint. Requests come from CLICK to the supplier; responses go back to CLICK with `Content-Type: application/json; charset=utf-8`.

| Contract | Request encoding | Actions | Payment identity |
|---|---|---|---|
| [Standard SHOP](02-shop-api-requests.md) | form-urlencoded | 0 Prepare, 1 Complete | `click_trans_id`, `click_paydoc_id`, `merchant_trans_id` |
| Advanced Shop | JSON | 0 Getinfo, 1 Prepare, 2 Complete, 3 Check, 4 Compare | `click_paydoc_id`, `attempt_trans_id`, account/order in `params` |
| [Split Shop](17-split-shop.md) | JSON | 0 optional Getinfo, 1 Prepare, 2 Confirm | Same named payment/attempt fields, plus Prepare response `split` |

Do not dispatch solely on an action number across protocols: Advanced `action=0` is an account lookup, not a reservation/payment Prepare. The source heading says **Complete** for action 2, although its field description says “Confirm”; this guide uses the heading's Complete name. Standard SHOP signatures and its form-urlencoded test collection are not interchangeable with this protocol.

## Wire Fields and Signature

| Field | Official Advanced type | Meaning / applicable actions |
|---|---|---|
| `service_id` | int | Service identifier; all actions |
| `action` | int | Action 0–4 |
| `params` | object | Merchant-specific payment/display key-value pairs; actions 0–3 |
| `click_paydoc_id` | int | CLICK payment identifier; actions 1–3 |
| `attempt_trans_id` | int | Attempt identifier; actions 1–3 |
| `merchant_prepare_id` | varchar | Prepare response ID, optional there; present in Complete/Check request tables |
| `sign_time` | varchar | Payment time, `YYYY-MM-DD HH:mm:ss`; actions 1–3 |
| `sign_string` | varchar | MD5 verification hash; actions 1–3 |
| `from_date`, `till_date` | varchar | Compare date-range boundaries; action 4 |

Prepare, Complete, and Check document the **same** formula, without separators:

```text
MD5(click_paydoc_id + attempt_trans_id + service_id + SECRET_KEY + params + action + sign_time)
```

The exact provider definition of `params` is **“all values of the params object, transmitted in the original order”** (`все значения объекта params, переданные в исходном порядке`). Concatenate the values, not keys or an entire JSON serialization. Do not sort keys, join with separators, or add `merchant_prepare_id` to this formula. In particular, do not reuse standard SHOP's Complete formula.

Preserve the original value representation/order until verification. The documentation does not define nested-object/array serialization, null/boolean conversion, numeric lexical normalization, or encoding beyond the UTF-8 content type. A parsed object followed by `JSON.stringify`, `Object.values`, or a newly formatted amount is not a documented canonicalization: parsing can change numeric representation and JavaScript property enumeration can reorder integer-like keys. Agree the flat parameter schema and exact signing fixtures with Click; obtain clarification for any unsupported value form rather than inventing a rule. Compare the resulting 32-character hex hash in constant time and validate the configured service before changing billing state. Keep `SECRET_KEY` on the backend.

### Authentication Omissions Are Not Permissions

Getinfo and Compare request tables/examples contain **no** `sign_time`, `sign_string`, or `Auth` header. That is what the source documents; it does not establish that public unauthenticated disclosure of account information or a payment register is safe. Confirm Click's intended authentication/network restrictions for these actions. Do not fabricate a mandatory Merchant API SHA1 `Auth` header, nor expose lookup/reconciliation data without an agreed access boundary.

### ID Precision and Type Inconsistencies

This page labels CLICK/attempt IDs as `int`, whereas Split Shop uses `bigint` and standard SHOP documents 64-bit CLICK IDs. Preserve exact identifiers in storage/signatures rather than imposing a guessed 32-bit ceiling. Advanced merchant IDs are `varchar` in tables, but official examples show numeric `12345`; confirm the service's accepted wire type instead of silently coercing them.

All numeric examples below are deliberately below `Number.MAX_SAFE_INTEGER`. Ordinary JavaScript `JSON.parse`/Express JSON parsing can round larger JSON integer tokens **before** verification; `BigInt(roundedNumber)` or a reviver cannot repair that. A full-width deployment needs lossless parsing before numeric conversion and exact string/native 64-bit storage. Keep documented JSON numbers numeric on the wire where required; do not send native `BigInt` through `JSON.stringify`, or quote every ID without provider agreement. The [standard SHOP precision note](02-shop-api-requests.md#id-precision) explains the bounded Node.js example; its guard for raw form strings does not fix an already-parsed JSON request.

## Action 0 — Getinfo (Optional, Before Payment)

Request fields: `service_id`, `action=0`, `params`. Return `params` for customer display plus `error` and `error_note`; lookup must not create a paid transaction.

Official request shape:

```json
{
  "action": 0,
  "service_id": 123,
  "params": {"contract": "***", "full_name": "***", "service_type": "***"}
}
```

Official success shape:

```json
{
  "error": 0,
  "error_note": "Успешно",
  "params": {"licshet": "***", "fio": "***", "address": "***", "period": "***"}
}
```

`licshet`/`fio` appear in the display example, although the parameter dictionary lists `account`/`full_name`; do not treat that difference as a universal aliasing rule. An official lookup failure example is `{"error":-250,"error_note":"Абонент не найден"}`. The `-250` sample is not part of the listed -1…-9 standard error table; agree service-specific codes with Click.

## Action 1 — Prepare

Prepare validates the account/order and amount in `params`, obtains payment details, and creates a durable pending billing record, not a paid outcome. Persist the payment/attempt identifiers, original verified parameters, expected amount, and returned billing ID so later Complete/Check can correlate them.

Official request shape (the `***` markers are the provider's redacted example; compute a real hash before sending any request):

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

Prepare response fields: `click_paydoc_id`, `attempt_trans_id`, optional `merchant_prepare_id` (table: varchar), `params`, `error`, `error_note`. Official success:

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
  }
}
```

The amounts here are in so'm, not fiscalization/Telegram tiyin. Use exact decimal/minor-unit arithmetic internally without changing the original signed value. No `split` list is documented for Advanced Prepare; use [Split Shop](17-split-shop.md) when that is the enabled contract.

## Action 2 — Complete

The request has all Prepare request fields with `action=2`, plus `merchant_prepare_id`. It uses the same params-value signature formula. Official example:

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

Validate correlation with the prepared billing record and perform the billing transition atomically. Do not deliver twice when a request is repeated or a reply is lost. Successful response fields are `click_paydoc_id`, `attempt_trans_id`, optional `merchant_confirm_id` (table: varchar), `params`, `error`, `error_note`:

```json
{
  "click_paydoc_id": 123,
  "attempt_trans_id": 345,
  "merchant_confirm_id": 12345,
  "error": 0,
  "error_note": "Успешно",
  "params": {"***": "***"}
}
```

Do not add standard SHOP's required request `error` field by analogy: the Advanced action 1–3 tables/examples do not include it. The page's error section nevertheless describes incoming negative CLICK errors; this omission needs a service-specific clarification before implementing cancellation input handling.

## Action 3 — Check

Check has the same request fields as Complete, with `action=3`, and the same signature. It asks for the **persisted processing outcome**, not permission to repeat fulfillment or a way to pretend an unknown transaction succeeded.

Response fields: `click_paydoc_id`, `attempt_trans_id`, `params`, `error`, `error_note`, `status`.

| `status` | Official meaning | CLICK behavior |
|---|---|---|
| 0 | Request has not yet been processed | Retry the payment attempt |
| 1 | Processing attempt was unsuccessful | Cancel the payment |
| 2 | Request was successfully processed | Mark the payment successful |

Official success example:

```json
{
  "click_paydoc_id": 123,
  "attempt_trans_id": 345,
  "merchant_confirm_id": 12345,
  "error": 0,
  "error_note": "Успешно",
  "status": 2,
  "params": {"***": "***"}
}
```

`merchant_confirm_id` appears in this example but is absent from the Check response table; confirm whether your service needs it. Do not equate top-level `error=0` with payment success independently of `status`. The official error example for actions 1–3 echoes both identifiers with `error=-250` and `error_note`, omitting `params` and, for Check, `status`; do not invent a mandatory error-response status.

## Action 4 — Compare (Reconciliation)

Request fields: `service_id`, `action=4`, `from_date`, `till_date`.

```json
{
  "action": 4,
  "service_id": 123,
  "from_date": "2020-03-03 00:00:00",
  "till_date": "2020-03-04 00:00:00"
}
```

Response fields: `requests` (an **object**, not an array), each payment's `params`, `error`, `error_note`. The official register is keyed by payment ID:

```json
{
  "error": 0,
  "error_note": "Успешно",
  "requests": {
    "123": {
      "click_paydoc_id": 123,
      "params": {"contract": "***", "full_name": "***", "service_type": "***", "amount": 1000}
    }
  }
}
```

Build the register from persisted billing data; Compare must not create/confirm payments. The source does not specify timezone, inclusive/exclusive boundaries, pagination, or which cancelled/pending records to include. Agree those semantics with Click before claiming reconciliation is complete. An official failure shape is `{"error":-250,"error_note":"Описание ошибки"}`.

## Parameter Dictionary and Errors

The source dictionary lists payment-detail keys `branch_id`, `payment_account`, `payment_mfo`, `transit_account`, `transit_mfo`; account/customer keys `account`, `address`, `birthday`, `caller_id`, `contract`, `email`, `full_name`, `login`, `phone_num`, `TIN`; and service/order keys `act_num`, `amount`, `apart_num`, `card_num`, `category`, `credit_id`, `cross_phone`, `date`, `house_num`, `internet_package`, `invoice`, `order_num`, `property_id`, `receipt_num`, `region`, `security_code`, `service_type`. They are **dictionary entries, not all required fields**. Use only the configured service's schema; do not collect sensitive card/security values merely because a key is listed.

| `error` | Official note | Meaning |
|---|---|---|
| 0 | Success | Request succeeded |
| -1 | SIGN CHECK FAILED! | Signature failure |
| -2 | Incorrect parameter amount | Wrong amount |
| -3 | Action not found | Unknown action |
| -4 | Already paid | Previously confirmed transaction |
| -5 | User does not exist by params | Account/order not found using `params` |
| -6 | Transaction does not exist | Billing transaction not found |
| -7 | Failed to update user | Billing update failure |
| -8 | Error in request from click | Malformed/incomplete request |
| -9 | Transaction cancelled | Cancelled transaction |

The page says an incoming negative CLICK error requires billing cancellation and return `-9`, but omits that input from its action request tables. The [standard SHOP cancellation contradiction](03-shop-api-errors.md#upstream-cancellation-contradiction) and its [Postman scenario 8](04-shop-api-testing.md#cancellation-ambiguity-preserve-scenario-8) must not be silently transplanted into a different JSON workflow. Obtain the enabled service's cancellation/authentication contract and preserve an audit trail of provider-reported failure versus merchant fulfillment failure.
