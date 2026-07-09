# Square API Automation Suite

![Postman](https://img.shields.io/badge/Postman-FF6C37?style=flat&logo=postman&logoColor=white)
![Newman](https://img.shields.io/badge/Newman-CLI-orange)
![Square API](https://img.shields.io/badge/Square-Sandbox%20API-3E4348?logo=square&logoColor=white)
![Status](https://img.shields.io/badge/Status-Passing-brightgreen)

## Project Overview

This is a Postman test automation suite built against Square's Sandbox API, covering the Orders, Payments, and Refunds endpoints. It goes beyond basic request/response checks by chaining requests together to simulate a real transaction lifecycle, and by including negative tests, concurrency checks, and security/auth boundary tests.

The suite runs the same way through the Postman Runner and through the Newman CLI, so it can be executed manually or wired into a CI/CD pipeline.

## Objectives

- Validate a complete order → payment → refund lifecycle, with state passed automatically between requests
- Confirm the API correctly rejects invalid input, stale data, and unauthorized access
- Demonstrate data-driven testing without relying on Postman's paid "Test Data File" feature
- Provide a suite that runs identically in both the Postman Runner and from Newman CLI

## Collection Structure and Workflow

The collection is organized into 3 folders (referred to here as "Gates"), containing 11 requests total.

**Gate 1 — End-to-End Ledger Transactions**
Chained requests representing a full transaction. Each request depends on data saved by the one before it.

| # | Request | Method | Purpose |
|---|---------|--------|---------|
| 1 | Authorize Order | POST | Creates an order using data-driven item name/price |
| 2 | Fetch Order State | GET | Confirms the order persisted and is in `OPEN` state |
| 3 | Modify Order Records | PUT | Adds a line item using optimistic concurrency (`version`) |
| 4 | Capture Payment Gateway | POST | Charges a Sandbox test card for the exact order total |
| 5 | Execute Refund Request | POST | Refunds a partial amount off the payment |

**Gate 2 — Data Integrity & Operational Boundaries**
Independent negative and boundary-condition tests. These depend on Gate 1 having run first, but not on each other.

| # | Request | Method | Purpose |
|---|---------|--------|---------|
| 6 | Decline Exception Handling | POST | Empty required field — validates the error contract via JSON Schema |
| 7 | Refund Balance Limit Check | POST | Confirms refund amounts can't exceed the remaining refundable balance |
| 8 | Volatility Boundary Check | PUT | Stale `version` — confirms optimistic concurrency control blocks it |

**Gate 3 — Gateway Security & Compliance**
Authentication and protocol boundary tests.

| # | Request | Method | Purpose |
|---|---------|--------|---------|
| 9 | Unauthorized Access Invalidation | GET | Invalid bearer token — expects `401` |
| 10 | Missing Authorization Header Enforcement | GET | No Authorization header at all — expects `401` |
| 11 | Missing Content-Type Enforcement | POST | Malformed body, no Content-Type — expects `400`/`415` |

**Variable flow:** `Authorize Order` saves `savedOrderId` and `currentOrderVersion`. `Modify Order Records` updates the version and saves the real order total (`currentOrderTotal`). `Capture Payment Gateway` uses that total and saves `savedPaymentId`. `Execute Refund Request` saves the remaining refundable balance, used by `Refund Balance Limit Check` in Gate 2.

## Tools and Technologies

- **Postman** — request building, chaining, and test scripting
- **Newman** — CLI test runner, for local automation and CI/CD compatibility
- **JavaScript** (`pm.*` API) — test assertions and pre-request logic
- **tv4** — JSON Schema validation for one request's error contract
- **Square Sandbox API** — Orders, Payments, and Refunds endpoints

## Key Highlights

- Full chained transaction lifecycle (order → payment → refund), with state passed automatically via environment variables
- Data-driven testing implemented with an embedded dataset and a pre-request script, as a workaround for Postman's paid-only Test Data File feature
- One request validated against a full JSON Schema, in addition to standard manual assertions
- Optimistic concurrency tested directly, by deliberately sending a stale version number
- Three separate authentication/security boundary tests: bad token, missing token, and missing Content-Type
- Verified to run identically in the Postman Runner and via Newman CLI

## Test Execution Results
![image alt](https://github.com/user-attachments/assets/16368589-1066-4aac-9b82-f07760a770fb)
Latest full run (4 iterations, data-driven):

| Metric | Result |
|---|---|
| Iterations | 4 |
| Requests executed | 44 |
| Test scripts run | 132 |
| Assertions | 128 |
| Failures | 0 |

All 11 requests pass across all 4 dataset iterations, including the 3 requests that are expected to return non-2xx status codes as their correct outcome (`Decline Exception Handling`, `Volatility Boundary Check`, `Unauthorized Access Invalidation`).

## How to Use

**1. Clone the repository**
```bash
git clone https://github.com/QuantumRay-code/Square-API-Automation-Suite.git
```

**2. Create a free Square Developer account**

Sign up at the [Square Developer Dashboard](https://developer.squareup.com/apps) (free). Create a new application, open its **Sandbox** tab, and note your Access Token and Location ID.

**3. (Running via Newman) Fill in placeholders directly in the JSON files**

- `Square Sandbox Environment.json` → replace the placeholder values for `access_token` and `locationId`
- `Square API Automation Suite.json` → find `auth_secret_16jo` in the `"variable"` array near the bottom of the file, and replace its placeholder with your real Sandbox access token

`auth_secret_17j8` already ships with its intended value (`EXPIRED-TOKEN-FOR-SECURITY-TESTING`) — no change needed there.

**4. Install Newman**
```bash
npm install -g newman
```

**5. (Optional — running via the Postman app instead) Import both files into Postman**

- In Postman → **Environments** → **Square Sandbox Environment** → fill in `access_token` and `locationId`
- In Postman → click the collection name → **Variables** tab → set:
  - `auth_secret_16jo` → your real Sandbox access token
  - `auth_secret_17j8` → `EXPIRED-TOKEN-FOR-SECURITY-TESTING` (or any invalid string)

These two variables live at the **collection level**, not inside any individual request — you won't find them by opening `Unauthorized Access Invalidation` or `Missing Content-Type Enforcement` directly.

Skip this step entirely if you're running via Newman only.

**6. (Optional) Adjust the data-driven dataset**

The collection variable `testDataset` holds a sample array of 4 rows:
```json
[
  {"itemName":"Standard Line Item","amount":2500},
  {"itemName":"Premium Line Item","amount":5000},
  {"itemName":"Discount Line Item","amount":1000},
  {"itemName":"Bulk Order Item","amount":8000}
]
```
Add, remove, or edit rows as needed. Set the **iteration count to match** your row count when you run the collection (step 7).

**7. Run the collection**
```bash
newman run "Square API Automation Suite.json" -e "Square Sandbox Environment.json" -n 4
```
(Change `-n 4` if you edited the dataset in step 6.)

**8. (Optional) Generate an HTML report**

install newman HTML reporter (if not installed)
```bash
npm install -g newman-reporter-htmlextra
```
then run
```bash
newman run "Square API Automation Suite.json" -e "Square Sandbox Environment.json" -n 4 -r html
```
