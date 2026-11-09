# API Documentation

Base URL (local): `http://localhost:3000/api`

All request/response examples use JSON. Money values are integer paise. For example, ₹1,250 is `125000`.

## Common response shape

Success:

```json
{
  "success": true,
  "data": {}
}
```

Error:

```json
{
  "success": false,
  "error": {
    "code": "INSUFFICIENT_CREDIT",
    "message": "Available credit is insufficient for this purchase."
  }
}
```

## Users

### Register user

`POST /users`

Request:

```json
{
  "name": "Aarav",
  "creditLimit": 1000000
}
```

Response `201`:

```json
{
  "success": true,
  "data": {
    "id": "usr_example",
    "name": "Aarav",
    "creditLimit": 1000000,
    "status": "ACTIVE",
    "defaultCount": 0
  }
}
```

### Get user

`GET /users/:userId`

### Get credit summary

`GET /users/:userId/credit`

Example response:

```json
{
  "success": true,
  "data": {
    "creditLimit": 1000000,
    "totalOutstanding": 300000,
    "availableCredit": 700000,
    "defaultCount": 0,
    "status": "ACTIVE"
  }
}
```

## Products and inventory

### Create product

`POST /products`

```json
{
  "name": "Headphones",
  "price": 250000,
  "stock": 10
}
```

### List products

`GET /products`

### Get product

`GET /products/:productId`

### Adjust stock

`PATCH /products/:productId/stock`

```json
{
  "quantityChange": 5
}
```

`quantityChange` may be positive or negative, but the resulting stock cannot be negative.

## Orders

### PREPAID purchase

`POST /orders/prepaid`

```json
{
  "userId": "usr_example",
  "items": [
    {
      "productId": "prd_example",
      "quantity": 1
    }
  ]
}
```

Expected behavior:
- Validate user, products, quantities, active state, and stock.
- Simulate immediate payment.
- Create a `PAID` order.
- Decrease stock.
- Do not create a due or consume BNPL credit.

### BNPL purchase

`POST /orders/bnpl`

```json
{
  "userId": "usr_example",
  "items": [
    {
      "productId": "prd_example",
      "quantity": 1
    }
  ]
}
```

Optional configuration may set the due duration; default is 14 days.

Expected behavior:
- Reject blacklisted users.
- Reject insufficient credit.
- Validate all stock before mutation.
- Create one order and one due for the cart.
- Decrease stock and available credit (through increased outstanding).

### Get order

`GET /orders/:orderId`

### User order history

`GET /users/:userId/orders`

Returns newest-first order history.

## Payments and dues

### Make a partial/full repayment

`POST /payments`

```json
{
  "userId": "usr_example",
  "dueId": "due_example",
  "amount": 100000
}
```

Rules:
- Amount must be a positive integer paise value.
- Due must belong to the user.
- Amount cannot exceed outstanding.
- A successful payment creates a payment record and reduces outstanding.
- If outstanding reaches zero, due and order become `PAID`.
- A partial payment on an already overdue due leaves it `OVERDUE`.

### List user's dues

`GET /users/:userId/dues`

### Get due details

`GET /dues/:dueId`

### List payment history

`GET /users/:userId/payments`

## Admin / operations

### Evaluate overdue dues

`POST /admin/dues/evaluate-overdue`

This operation scans all dues using the current time, marks eligible dues overdue, and counts a default once per due. It blacklists users when they reach three distinct defaults.

For deterministic testing, the service layer should accept an injected date/time. Do not expose arbitrary clock manipulation in a production API.

## Status values

- User: `ACTIVE`, `BLACKLISTED`
- Payment mode: `PREPAID`, `BNPL`
- Order: `PAID`, `PENDING`, `PARTIALLY_PAID`, `OVERDUE`
- Due: `PENDING`, `PARTIALLY_PAID`, `OVERDUE`, `PAID`

## Suggested HTTP status mapping

| HTTP | Use |
|---|---|
| 200 | Read/update operation succeeded |
| 201 | User/product/order/payment created |
| 400 | Invalid input |
| 404 | Entity not found |
| 409 | Insufficient stock/credit, blacklisted BNPL user, overpayment, invalid state transition |
| 500 | Unexpected server error |

## Validation checklist

- Required IDs and fields are present.
- `quantity` is a positive integer.
- `price`, `creditLimit`, and payment amount are safe non-negative/positive integers as appropriate.
- `items` is a non-empty array.
- Duplicate product lines in one cart are either consolidated or validated using aggregate quantity.
- All products and stock are validated before any inventory mutation.
- Payment due belongs to the requesting user.
- A due with zero outstanding cannot be paid again.
- No operation may produce negative stock, outstanding, or available credit.
