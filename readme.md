# 🍜 API Testing Lab: Auntie Som’s Noodle Stall

> **Objective:** Prove that Nephew Lek’s "it works on my machine" attitude is a recipe for disaster.

---

## 📖 The Backstory

Auntie Som runs a **legendary noodle stall**. Her Tom Yum is world-class, her customers are loyal, and her business is booming.  

Recently, her nephew **Lek** (who just finished a 2-week coding bootcamp) decided to "digitize" the business. He built a backend API to handle online orders. Auntie Som is thrilled, but the code… well, Lek says:  

> *"It’s fine lah Auntie! I tested it once with my own phone. No need for professional testing!"*

**You are the Last Line of Defense.**  
Before Auntie Som launches this app to 1,000+ hungry customers, you must audit the system, find the logic flaws, and automate the validation using **Bruno**.

---

## 🎯 Learning Objectives

By the end of this lab, you should be able to:
- 🖋️ Write **Bruno test scripts** for automated validation.
- 📦 Extract values from API responses using **Post-response scripts**.
- 🔐 Manage and use **Environment Variables**.
- 🧠 Verify **Business Rules**, not just HTTP status codes.
- 🚀 Catch regressions automatically before they hit production.

---

## 🛠️ Setup Instructions

### 1️⃣ Prepare the Project
- **Fork this repository** to your own GitHub account.
- **Clone it** locally to your machine.

### 2️⃣ Start the API Server
The backend is a lightweight Flask application.
- **Requirements:** Python 3.10+ installed.
- **Navigate** to the `auntie-som-noodle-api` folder.
- **Install dependencies:**  
  ```bash
  pip install flask
  ```
- **Run the server:**  
  ```bash
  python app.py
  ```
- The server will run at: `http://localhost:5000`

### 3️⃣ Setup Bruno
- Download and install [Bruno](https://www.usebruno.com).
- Create a **New Collection** named `Auntie Som Lab`.
- Create an **Environment** (e.g., `local`) and set a variable `baseUrl` to `http://localhost:5000`.

---

## 📑 API Reference

### 🔐 Authentication
`POST /auth/login` - Get an access token.
- **Body (JSON):**
  ```json
  {
    "username": "student",
    "password": "hungry"
  }
  ```
- **Response (200 OK):**
  ```json
  {
    "accessToken": "uuid-token-string",
    "expiresIn": 3600
  }
  ```

---

### 🍜 Menu
`GET /menu` - View available items and stock levels.
- **Response (200 OK):**
  ```json
  [
    { "id": 1, "name": "Tom Yum Noodles", "price": 50, "stock": 2 },
    { "id": 2, "name": "Fried Rice", "price": 45, "stock": 1 }
  ]
  ```

---

### 🛒 Orders
`POST /orders` - Place a new order.
- **Headers:** `Authorization: Bearer <token>`
- **Body (JSON):**
  ```json
  {
    "itemId": 1,
    "quantity": 1
  }
  ```
- **Response (200 OK):**
  ```json
  {
    "orderId": "ORD-abc123",
    "status": "created",
    "totalPrice": 45
  }
  ```

`GET /orders/<orderId>` - Retrieve order details.
- **Response (200 OK):**
  ```json
  {
    "orderId": "ORD-abc123",
    "status": "created",
    "totalPrice": 45
  }
  ```

---

## 🕵️ Your Mission: The Silent Auditor

Your task is to create a comprehensive Bruno collection that validates the entire workflow.  

⚠️ **The Catch:** Nephew Lek left several **logic bugs** in the system. Some endpoints might return `200 OK` even when the business logic is completely broken.

**Your collection must include:**
1.  **Auth Flow**: Automatically store the `accessToken` from login and use it for subsequent requests.
2.  **Order Validation**: Ensure orders can only be placed with valid auth, valid items, and available stock.
3.  **Data Integrity**: Check that the `totalPrice` calculated by the API matches your expectations.
4.  **Edge Cases**: What happens if you order more than what's in stock? What happens if you request a non-existent order?

> [!TIP]
> **Clicking "Send" is not testing.** Your tests must contain scripts (Assertions or Javascript) that **FAIL** when the system behaves incorrectly.

---

## 🧠 Rules of the Lab
- ❌ **Do NOT** modify the `app.py` backend code.
- ❌ **Do NOT** rely on manual checking.
- ✅ **Use environment variables** for the Base URL and Tokens.
- ✅ **Use Scripts/Assertions** for all validations.

---

## 📦 What to Submit

1.  **Push your Bruno collection folder** to your GitHub repository.
2.  **Update your README.md** (the one in your fork) with:
    - A summary of the bugs you discovered.
    - How your tests detect these bugs.
4.  **Send the GitHub link** to your instructor via Discord.

---
*Good luck. Auntie Som is counting on you!* 🍜🔥

---
# Bruno

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
