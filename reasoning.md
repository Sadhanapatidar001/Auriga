# Technical Reasoning

## 1. Problem Understanding

The AV Room manages multiple types of equipment such as cameras, projectors, microphones, tripods, speakers, and laptops.

The existing paper-based process creates several problems:

* Equipment availability is difficult to track.
* The same equipment can be requested by multiple people.
* Equipment may not be returned on time.
* Late fees are difficult to calculate manually.
* Deposits and refunds are difficult to track.
* Physical equipment units are not individually traceable.
* There is no clear history of who currently has an item.
* Borrowed equipment needs to be transferable when responsibility changes.

The system is therefore designed around the main workflow:

```text
Borrow → Track → Return → Calculate Fees → Refund
```

Additional features such as notifications, borrowing limits, and loan transfers support this core workflow.

---

# 2. Technology Choices

## Frontend: React + TypeScript + Vite

React is used to build a component-based user interface.

TypeScript provides type safety and helps reduce errors while working with equipment, loans, users, and API responses.

Vite provides a simple and fast development environment.

## Backend: Node.js + Express + TypeScript

Node.js is used for server-side development.

Express provides routing and middleware for APIs.

TypeScript provides type safety across backend services and API logic.

## Database: MySQL

MySQL is used because the application contains structured and relational data.

Examples of relationships include:

```text
User → Loan
Equipment → EquipmentUnit
Loan → LoanUnit
Loan → Deposit
Loan → LoanTransfer
User → Notification
```

These relationships are naturally represented using a relational database.

## ORM: Prisma

Prisma provides:

* Type-safe database access
* Schema management
* Database migrations
* Relationship handling
* Prisma Client for database queries

---

# 3. Why Equipment and EquipmentUnit Are Separate

An equipment type can have multiple physical units.

For example:

```text
Projector
├── PRJ-001
├── PRJ-002
└── PRJ-003
```

Therefore, `Equipment` represents the equipment type and quantity, while `EquipmentUnit` represents each physical item.

### Equipment

Stores information such as:

* Name
* Category
* Total quantity
* Deposit amount
* Late fee
* Description

### EquipmentUnit

Stores information such as:

* Asset code
* Serial number
* Condition
* Current status

This allows the system to track individual physical assets instead of treating all units as one generic item.

---

# 4. Availability Logic

Availability must be calculated from actual equipment units and active bookings.

For example, if there are:

```text
Total Projectors = 3
Currently Borrowed = 2
```

then:

```text
Available = 3 - 2 = 1
```

The system should not allow a request for more units than are available.

For date-based bookings, two booking periods overlap when:

```text
requestedStart < existingEnd
AND
requestedEnd > existingStart
```

If this condition is true, the bookings overlap.

If the condition is false, the booking periods do not overlap.

This prevents double booking of the same physical resources.

---

# 5. Why Loan and LoanUnit Are Separate

A loan represents a borrowing transaction.

A loan may contain multiple physical equipment units.

For example:

```text
Loan #101
Quantity = 2

    ↓

Projector PRJ-001
Projector PRJ-002
```

`LoanUnit` creates the relationship between the loan and the actual physical units.

This makes it possible to identify exactly which equipment units are assigned to a borrower.

---

# 6. Loan Status Design

Loans use explicit statuses to represent their lifecycle.

```text
REQUESTED
    ↓
APPROVED
    ↓
ACTIVE
    ↓
RETURNED
```

Other possible states include:

```text
REJECTED
CANCELLED
OVERDUE
```

Using explicit statuses makes business rules easier to implement.

For example:

* Only ACTIVE loans can be returned.
* Only ACTIVE loans can be transferred.
* OVERDUE loans can trigger reminders and late fees.
* RETURNED loans cannot be transferred.

---

# 7. Late Fee Calculation

The late fee is based on the number of days between the due date and actual return date.

```text
Late Days = max(0, Actual Return Date - Due Date)

Late Fee = Late Days × Late Fee Per Day
```

The `max(0, ...)` rule ensures that returning an item early does not create a negative late fee.

### Example

```text
Due Date:          10 September
Return Date:       13 September
Late Fee Per Day:  ₹20

Late Days = 3

Late Fee = 3 × ₹20
         = ₹60
```

---

# 8. Deposit and Refund Logic

A refundable deposit is associated with the loan.

When the equipment is returned:

```text
Refund Amount = max(0, Deposit Amount - Late Fee)
```

### Example

```text
Deposit = ₹1000
Late Fee = ₹200

Refund = ₹800
```

If the late fee is greater than the deposit, the refund becomes zero.

The financial information is stored with the loan/deposit records so that the transaction history remains available.

---

# 9. Borrowing Limit

A borrower should not be able to reserve an unreasonable amount of equipment.

The system therefore checks the number of active units already associated with the borrower.

Default limit:

```text
5 active units
```

Before approving a new request:

```text
Current Active Units + Requested Units
```

must not exceed the configured limit.

This is checked before creating/approving the borrowing transaction.

---

# 10. Loan Transfer Design

The system has an additional requirement:

> An active loan can be transferred from one borrower to another.

The important rule is that the transfer is a change of responsibility, not a new booking.

Therefore, the existing `Loan` record is updated:

```text
Old Borrower → New Borrower
```

The following values remain unchanged:

```text
Loan ID
Equipment
Quantity
Booking Date
Borrow Date
Due Date
Status
Loan Units
Deposit / Financial History
```

No new loan is created.

---

# 11. Why Availability Does Not Change During Transfer

A transfer does not mean that equipment was returned and borrowed again.

For example:

```text
Projector PRJ-001
        ↓
Currently with Student A
        ↓
Transferred to Student B
```

The projector is still physically borrowed.

Therefore:

```text
Before Transfer:
Available = 2

After Transfer:
Available = 2
```

The availability calculation must remain unchanged.

This avoids accidentally releasing equipment during a borrower transfer.

---

# 12. Why the Original Due Date Is Preserved

A transfer changes the responsible borrower, not the borrowing period.

Example:

```text
Original Borrower: Student A
Due Date: 20 September

Transferred to: Student B
```

The due date remains:

```text
20 September
```

It is not extended or restarted.

This ensures that a transfer cannot be used to reset the borrowing period.

---

# 13. Loan Transfer Audit Trail

A separate `LoanTransfer` record is created for every successful transfer.

It stores information such as:

```text
loanId
fromBorrowerId
toBorrowerId
transferredAt
transferredById
reason
```

This provides an audit history.

Example:

```text
Loan #25

Student A
    ↓
Transferred to Student B

Reason:
Responsibility changed to another team member
```

The original borrower and new borrower can therefore be identified later.

---

# 14. Why the Transfer Is Transactional

Loan transfer changes important relational data.

The following operations should succeed together:

```text
Update Loan Borrower
        +
Create LoanTransfer Audit Record
        +
Create/Send Notification
```

A database transaction is used for the critical database changes.

If an important database operation fails, the transfer should not leave the loan in a partially updated state.

---

# 15. Transfer Authorization

Loan transfers are restricted to authorized users, preferably administrators.

The backend verifies:

```text
Authentication
      ↓
Authorization
      ↓
Loan exists
      ↓
Loan is ACTIVE
      ↓
New borrower exists
      ↓
New borrower is different
      ↓
Borrowing limit is valid
      ↓
Transfer
```

The backend performs these checks instead of relying only on frontend validation.

---

# 16. Notifications

Notifications are used to remind borrowers about upcoming and overdue returns.

Important reminder cases include:

```text
Due Tomorrow
Due Today
Overdue
Loan Transferred
```

Notifications help reduce forgotten returns without changing the core borrowing workflow.

---

# 17. Data Integrity

The database uses relationships and constraints to maintain valid data.

Examples:

* A loan must reference a valid borrower.
* An equipment unit must belong to valid equipment.
* A loan unit must reference a valid loan and equipment unit.
* A deposit must reference a valid loan.
* A transfer must reference a valid loan and users.

This prevents orphaned or inconsistent records.

---

# 18. Core Design Principle

The system prioritizes the actual AV room workflow before secondary features.

Priority order:

```text
1. Borrowing
2. Availability
3. Returns
4. Deposits
5. Borrowing Limits
6. Notifications
7. Loan Transfers
```

The architecture keeps these concerns separated so that additional features can be added without changing the fundamental borrowing and availability logic.

---

# 19. Example End-to-End Scenario

Assume the AV room has:

```text
Projectors = 3
```

Student A requests:

```text
2 Projectors
```

After approval:

```text
Available = 1
```

Student B requests:

```text
2 Projectors
```

The request cannot be approved because only one projector is available.

Student B can instead request:

```text
1 Projector
```

After borrowing:

```text
Available = 0
```

If Student A's active loan is transferred to Student C:

```text
Student A → Student C
```

the projector availability remains:

```text
Available = 0
```

The original due date also remains unchanged.

When the equipment is returned, the system calculates:

```text
Late Days
     ↓
Late Fee
     ↓
Refundable Deposit
     ↓
Equipment Available Again
```

This represents the complete lifecycle of an AV room equipment transaction.
