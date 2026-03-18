# Auntie Som Lab - API Testing Report

This folder contains API test cases for Auntie Som's Noodle Stall backend.

## What This Lab Covers

The tests focus on:

1. Authentication behavior
2. Order input validation
3. Stock handling
4. Price calculation
5. Error status code correctness

## Backend Under Test

- API reference: https://github.com/foname/api-testing-course?tab=readme-ov-file#-api-reference
- Local backend file used for analysis: ../api-testing-course/auntie-som-noodle-api/app.py

## Quick Bug Summary

### 1. Missing quantity validation can crash the server

`POST /orders` uses `quantity` in arithmetic without fully validating missing, null, or wrong-type values.

```python
item["stock"] -= quantity
```

If `quantity` is missing or `null`, this can trigger a `TypeError` and return HTTP 500 instead of a clean 4xx validation response.

### 2. Invalid quantity values are accepted

Negative and float quantities are not rejected explicitly before stock update.

```json
{ "itemId": 1, "quantity": -1 }
{ "itemId": 1, "quantity": 1.5 }
```

This can corrupt stock values.

### 3. Over-quantity ordering is not blocked

The API checks only whether stock is already `<= 0`, but does not validate requested amount against available stock.

```json
{ "itemId": 1, "quantity": 99 }
```

Result: order may succeed and stock can become negative.

### 4. totalPrice formula may not match expected business rules

Observed formula:

```python
totalPrice = (price * quantity) - 5
```

If no discount requirement exists, this is a business logic defect.

### 5. Not-found order returns non-standard status code

For `GET /orders/<order_id>` with unknown id, current behavior returns:

```http
HTTP/1.1 200 OK
{"error": "Order not found"}
```

Expected REST-style behavior is usually:

```http
HTTP/1.1 404 Not Found
{"error": "Order not found"}
```

### 6. Missing itemId is treated like not-found item

When `itemId` is missing, API falls through to `Item not found (404)`.
Semantically better behavior is `400 Bad Request`, for example:

```json
{"error": "itemId is required"}
```

## Test Files and What They Check

### Authentication

- `01_Auth-NullUserName.yml`: login with null username
- `02_Auth-WrongPass.yml`: login with wrong password
- `04_Auth-Correct.yml`: login success path
- `03_Order-NoAuth.yml`: create order without token, should return 401

### Quantity Validation

- `08_Order-QuantityIsLess0.yml`: quantity less than 0
- `09_Order-NegativeQuantity.yml`: negative quantity
- `10_Order-ZeroQuantity.yml`: zero quantity
- `11_Order-InvalidQuantityType.yml`: string quantity, for example `"one"`
- `15_Order-MissingQuantity.yml`: quantity omitted
- `16_Order-NullQuantity.yml`: quantity is `null`
- `17_Order-FloatQuantity.yml`: float quantity, for example `1.5`

Example invalid payloads:

```json
{ "itemId": 1 }
{ "itemId": 1, "quantity": null }
{ "itemId": 1, "quantity": "one" }
```

### Stock Integrity

- `06_Order-OverQuantity.yml`: quantity larger than stock, should be rejected
- `07_Menu-CheckStock.yml`: verifies stock changes after ordering

### Valid Order and Price Behavior

- `05_Order-Valid.yml`: valid order flow and `totalPrice` verification

### Resource Lookup and Required Fields

- `12_Order-NotFound.yml`: fetch unknown order id
- `13_Order-MissingItemId.yml`: missing itemId field
- `14_Order-InvalidItemId.yml`: unknown item id, for example `999`

## Why Some Assertions Are Flexible

Some tests currently allow both `400` and `500` to make backend instability visible while still producing useful test results.

After backend fixes are complete, tighten each case to one strict expected status and one strict error schema.
