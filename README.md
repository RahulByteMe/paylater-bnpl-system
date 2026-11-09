# PayLater — Buy Now, Pay Later (BNPL) System

An in-memory BNPL platform designed as a backend machine-coding project. The system supports product inventory, user registration and credit limits, prepaid and BNPL purchases, partial/full repayments, due tracking, order history, available credit, and blacklisting after three defaults.

**Initial stack:** Node.js, Express.js, JavaScript  
**Storage:** In-memory `Map` collections; no database  
**Money representation:** Integer paise to avoid floating-point errors

## Documentation

- [Product Requirements Document](docs/PRD.md)
- [System Design & Data Model](docs/SYSTEM_DESIGN.md)
- [API Documentation](docs/API_DOCUMENTATION.md)

## Core flows

### Prepaid purchase
`User → Select product → Validate user/product/stock → Record prepaid payment → Create PAID order → Decrease stock`

### BNPL purchase
`User → Select product → Validate user → Check blacklist → Check available credit → Check stock → Create order and due → Decrease stock`

### Repayment
`User → Select due → Validate outstanding balance → Record payment → Reduce outstanding → Restore available credit → Update due status`

### Default handling
A due becomes overdue when `dueDate` has passed and `outstandingAmount > 0`. A due contributes at most one default. A user is blacklisted after three distinct defaulted dues.

## Important business rules

1. `availableCredit = creditLimit - totalOutstandingBNPL`.
2. PREPAID purchases do not consume BNPL credit.
3. BNPL is rejected for blacklisted users or insufficient credit.
4. Partial payments reduce outstanding and restore the same amount of available credit.
5. A payment must be greater than zero and cannot exceed the due's outstanding amount.
6. Stock is decremented only after all purchase validations pass.
7. The same due cannot be counted as a default more than once.
8. Blacklisted users cannot make new BNPL purchases. PREPAID purchases remain allowed in this educational project.
9. All amounts are stored as integer paise.
10. Payment processing is simulated; no real payment gateway, credit bureau, or lending decision is integrated.

## Project status

This repository currently contains the product and technical design. Implementation should follow the phases in the PRD.

## Planned local setup

Once the application code is added:

```bash
npm install
npm run dev
```
