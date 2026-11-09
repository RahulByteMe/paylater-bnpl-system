# Product Requirements Document (PRD)
## PayLater — Buy Now, Pay Later System

| Field | Value |
|---|---|
| Project | PayLater BNPL System |
| Version | 1.0 |
| Status | Design / implementation-ready |
| Target | Backend machine-coding project |
| Initial time constraint | 90 minutes |
| Runtime | Node.js + Express.js |
| Language | JavaScript |
| Persistence | In-memory only; no database |

---

## 1. Overview

PayLater is a simplified e-commerce BNPL platform. A registered customer can purchase products by paying immediately (PREPAID) or by using an assigned credit limit (BNPL). BNPL purchases create outstanding dues that can be repaid partially or fully. The system tracks pending and overdue dues and blacklists customers after three distinct defaults.

The project is intended to demonstrate backend engineering fundamentals: domain modelling, data structures, validation, business-rule enforcement, transaction consistency, REST API design, and testing.

## 2. Problem statement

An e-commerce platform needs a small payment and credit subsystem that can:

- Maintain product stock.
- Register customers and assign credit limits.
- Support immediate prepaid purchases and pay-later purchases.
- Track outstanding amounts and repayments.
- Show order history and remaining credit.
- Identify overdue payments and prevent repeat BNPL access for defaulting customers.

The implementation must run entirely in memory. Data resets when the process restarts; persistence is explicitly out of scope for the first version.

## 3. Goals

### 3.1 In scope

1. Register users with a non-negative credit limit.
2. Add, list, and update products and stock.
3. Place PREPAID orders.
4. Place BNPL orders subject to credit and blacklist checks.
5. Create a due record for each BNPL order.
6. Accept partial and full payments against a due.
7. Track PENDING, PARTIALLY_PAID, PAID, and OVERDUE due states.
8. Detect defaults without double-counting the same due.
9. Blacklist a user after three distinct defaults.
10. Retrieve order history, due history, and available credit.
11. Validate input and return consistent errors.
12. Test success paths, failure paths, and boundary cases.

### 3.2 Out of scope

- Database or durable persistence.
- Authentication, authorization, OTP, or password management.
- Real payment gateway integration, refunds, chargebacks, or settlement.
- Interest, late fees, EMI schedules, and multiple instalments.
- Credit bureau checks, KYC, fraud scoring, or regulatory compliance.
- Distributed locking, multi-instance deployment, and production-grade concurrency.
- Frontend UI and notifications.

## 4. Actors

| Actor | Capabilities |
|---|---|
| Customer | Place prepaid/BNPL orders, view own orders/dues/credit, make repayments |
| Admin / operator | Register users, manage products and stock, trigger overdue evaluation, inspect records |

For the initial machine-coding version, API calls may identify a user through `userId`; this is not authentication and must be documented as a simulation shortcut.

## 5. Functional requirements

### FR-01: User registration

- Accept a name and credit limit.
- Generate a unique user ID.
- Initialize the user as ACTIVE with zero defaults.
- Initialize total outstanding to zero.
- Reject missing names, negative limits, invalid types, and duplicate identifiers if IDs are supplied by the caller.

**User fields:** `id`, `name`, `creditLimit`, `status`, `defaultCount`, `createdAt`.

### FR-02: Product and inventory management

- Create a product with name, unit price, and non-negative stock quantity.
- List products and retrieve a product by ID.
- Update stock through an explicit stock adjustment operation.
- Reject negative prices, negative stock, unknown products, and purchases exceeding stock.

**Product fields:** `id`, `name`, `price`, `stock`, `active`.

### FR-03: PREPAID purchase

1. Validate user and product.
2. Validate quantity is a positive integer.
3. Calculate total = unit price × quantity.
4. Verify product is active and stock is sufficient.
5. Simulate successful immediate payment.
6. Create an order with `paymentMode = PREPAID` and `status = PAID`.
7. Decrease stock only after validations pass.
8. Do not create a BNPL due and do not change available BNPL credit.

### FR-04: BNPL purchase

1. Validate user and product.
2. Reject if user is BLACKLISTED.
3. Validate quantity and calculate total.
4. Verify product is active and stock is sufficient.
5. Calculate available credit.
6. Reject if available credit is less than the order total.
7. Create an order with `paymentMode = BNPL` and `status = PENDING`.
8. Create exactly one due record for the BNPL order.
9. Decrease stock.
10. Available credit decreases by the order total because outstanding increases by that amount.

### FR-05: Credit calculation

`availableCredit = creditLimit - sum(outstandingAmount for all unpaid BNPL dues)`

Equivalent implementation may maintain `totalOutstanding` on the user, provided every order/payment mutation updates it consistently. Prefer deriving it from dues in the initial implementation to avoid duplicated state.

Rules:
- Available credit must never be negative.
- PREPAID orders do not affect credit.
- A BNPL purchase consumes credit.
- A repayment restores credit by the amount actually paid.
- A default does not automatically erase the debt or restore credit.

### FR-06: Repayments

- Accept `dueId` and `amount`.
- Amount must be a positive integer number of paise.
- Due must exist and belong to the specified user.
- Reject payments greater than outstanding.
- Record each accepted payment as a separate payment transaction.
- Reduce due outstanding by payment amount.
- If outstanding becomes zero, set due status to PAID and linked order status to PAID.
- If outstanding remains positive, set due status to PARTIALLY_PAID unless it is already overdue; overdue status remains OVERDUE until fully paid.
- Restore available credit by the payment amount.
- A paid due cannot accept another payment.

### FR-07: Due and overdue tracking

Each BNPL order has one due record:
- `dueDate` is assigned at purchase time using a configurable duration (default: 14 days).
- `outstandingAmount` begins at the order total.
- Initial status is PENDING.
- A due is overdue when `now > dueDate` and `outstandingAmount > 0`.
- A due with zero outstanding is PAID, even if its due date has passed.

Overdue evaluation can be triggered by an admin endpoint or a service function. The core logic must accept an injected `now` value so tests do not depend on wall-clock timing.

### FR-08: Default tracking and blacklist

- A default occurs when an unpaid due crosses its due date.
- A due may contribute at most one default.
- Store a `defaultedAt` timestamp or boolean on the due.
- Increment the user's default count only on the first transition of that due into default.
- When `defaultCount >= 3`, set user status to BLACKLISTED.
- A blacklisted user cannot place new BNPL orders.
- Existing dues remain payable.
- PREPAID orders remain allowed in this project.
- Repaying a defaulted due does not erase its historical default count in v1.

### FR-09: Order history

Return a user's orders newest first, including:
- Order ID and creation time.
- Product ID, product name snapshot, quantity, and total amount.
- Payment mode and order status.
- Linked due ID when payment mode is BNPL.

### FR-10: Credit summary

Return:
- Credit limit.
- Total outstanding.
- Available credit.
- Default count.
- User status.

### FR-11: Error handling

Use consistent error responses. Suggested HTTP status codes:
- `400` invalid input or business-rule violation.
- `404` user/product/order/due not found.
- `409` insufficient stock, duplicate operation, or conflicting state.
- `500` unexpected internal error.

Do not mutate stock, credit, orders, or payments when a validation fails.

## 6. Domain states

### Order status

| Status | Meaning |
|---|---|
| PAID | PREPAID order successfully paid, or BNPL order fully repaid |
| PENDING | BNPL order has an unpaid due and is not yet partially paid |
| PARTIALLY_PAID | BNPL order has received a partial repayment |
| OVERDUE | BNPL order has an outstanding due past its due date |

### Due status

| Status | Meaning |
|---|---|
| PENDING | No repayment yet; due date not crossed |
| PARTIALLY_PAID | Some amount repaid; balance remains and due date not crossed |
| OVERDUE | Due date crossed and outstanding is greater than zero |
| PAID | Outstanding amount is zero |

### User status

- `ACTIVE`
- `BLACKLISTED`

## 7. End-to-end user journeys

### Journey A — Register and prepaid purchase

1. Admin registers customer with ₹10,000 credit limit.
2. Admin adds a product priced at ₹2,000 with stock 5.
3. Customer purchases quantity 1 using PREPAID.
4. System validates stock and records payment.
5. Order is PAID and stock becomes 4.
6. Available BNPL credit remains ₹10,000.

### Journey B — BNPL purchase and partial repayment

Assume the customer has a ₹10,000 credit limit.

| Step | Event | Outstanding | Available credit |
|---|---|---:|---:|
| 0 | Initial state | ₹0 | ₹10,000 |
| 1 | BNPL purchase ₹3,000 | ₹3,000 | ₹7,000 |
| 2 | Repay ₹1,000 | ₹2,000 | ₹8,000 |
| 3 | Repay ₹2,000 | ₹0 | ₹10,000 |

The order and due become PAID after step 3.

### Journey C — Three defaults and blacklist

1. Customer places three BNPL orders, each producing a separate due.
2. All three due dates pass with positive outstanding balances.
3. Overdue evaluation marks each due overdue.
4. Each due is counted once; user's default count reaches 3.
5. User status becomes BLACKLISTED.
6. A new BNPL purchase is rejected.
7. Existing dues can still be repaid; PREPAID remains allowed.

## 8. Acceptance criteria

- [ ] User registration creates a unique ACTIVE user.
- [ ] Product creation rejects invalid price and stock.
- [ ] PREPAID order reduces stock and does not consume BNPL credit.
- [ ] BNPL order reduces available credit and creates one due.
- [ ] BNPL purchase is rejected for insufficient credit.
- [ ] BNPL purchase is rejected for a blacklisted user.
- [ ] Purchase is rejected if stock is insufficient.
- [ ] Failed purchase leaves stock and credit unchanged.
- [ ] Partial repayment reduces outstanding and restores credit correctly.
- [ ] Full repayment changes due and order to PAID.
- [ ] Overpayment and zero/negative payment are rejected.
- [ ] Overdue due is detected based on due date.
- [ ] Re-running overdue evaluation does not increment defaults again.
- [ ] Three distinct defaults blacklist the user.
- [ ] Blacklisted user can still repay existing dues.
- [ ] Order history returns the user's orders only.
- [ ] Available credit never exceeds credit limit or falls below zero.

## 9. Non-functional requirements

- **Correctness:** Enforce business invariants in service methods, not only in routes.
- **Maintainability:** Separate routes/controllers, services, models, and in-memory repositories.
- **Testability:** Inject clock and ID generation where practical.
- **Clarity:** Use explicit enums/constants for statuses and payment modes.
- **Consistency:** Validate all preconditions before mutating state.
- **Performance:** In-memory lookups should be O(1) by ID using Maps; history queries may be O(n).

## 10. Suggested implementation plan

| Phase | Time budget (90 min) | Deliverable |
|---|---:|---|
| 1. Setup + models | 10 min | Express app, Maps, constants |
| 2. Users + products | 10 min | Registration and inventory |
| 3. Purchase flows | 20 min | PREPAID and BNPL |
| 4. Repayments | 15 min | Partial/full payments |
| 5. Due/default logic | 15 min | Overdue detection and blacklist |
| 6. Read APIs | 10 min | History and credit summary |
| 7. Tests + cleanup | 10 min | Edge cases and clean responses |

For a portfolio implementation, expand beyond the 90-minute exercise with unit tests, request validation, API examples, and a clear README.

## 11. Risks and design decisions

- **In-memory reset:** All data disappears on restart; acceptable for the exercise.
- **Date testing:** Inject a clock instead of sleeping in tests.
- **Money precision:** Store paise as integers.
- **Double defaulting:** Persist a per-due default marker in memory.
- **Partial payment state:** Keep overdue state when a late partial repayment leaves a balance.
- **Atomicity:** In a single-process synchronous implementation, validate first and mutate in one service operation. This is not a substitute for database transactions or distributed locking in production.
- **Credit policy:** This project uses a simple fixed credit limit; it is not a real credit underwriting system.

## 12. Definition of done

The project is ready for its first release when all acceptance criteria pass, the documented API works through Postman/curl, and a clean restart correctly demonstrates the documented in-memory reset behavior.
