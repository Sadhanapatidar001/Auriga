# AV Room Management System

A full-stack web application for managing college AV room equipment such as cameras, projectors, microphones, tripods, speakers, and laptops.

The system helps students and administrators manage equipment borrowing, availability, returns, deposits, late fees, and loan transfers digitally instead of using a paper register.

## Features

* User authentication with JWT
* Student and Admin roles
* Browse available AV equipment
* Check equipment availability
* Request equipment borrowing
* Admin approval/rejection of requests
* Track active and returned loans
* Equipment return management
* Automatic late-fee calculation
* Refundable security deposit tracking
* Borrowing limit enforcement
* Due-date and overdue notifications
* Transfer an active loan from one borrower to another
* Original due date remains unchanged after loan transfer
* Equipment availability remains unaffected during transfer
* Track individual physical equipment units
* Equipment status management: Available, Borrowed, Maintenance, Lost, Damaged
* Loan transfer audit history

## Tech Stack

### Frontend

* React
* TypeScript
* Vite

### Backend

* Node.js
* Express.js
* TypeScript
* JWT Authentication

### Database

* MySQL
* Prisma ORM

## Project Structure

```text
av-room-management/
│
├── backend/
│   ├── prisma/
│   │   ├── migrations/
│   │   ├── schema.prisma
│   │   └── seed.ts
│   │
│   └── src/
│       ├── controllers/
│       ├── routes/
│       ├── middleware/
│       └── ...
│
├── frontend/
│   └── src/
│       ├── components/
│       ├── pages/
│       ├── services/
│       └── ...
│
├── .gitignore
└── README.md
```

## Database Models

The application uses the following main entities:

* User
* Equipment
* EquipmentUnit
* Loan
* LoanUnit
* Deposit
* Notification
* LoanTransfer

## Loan Transfer

An active loan can be transferred from one borrower to another.

During a transfer:

* The existing loan record is updated with the new borrower.
* The original due date remains unchanged.
* Borrow and booking dates remain unchanged.
* Equipment quantity remains unchanged.
* Physical equipment units are not released or reassigned.
* Equipment availability does not change.
* Deposit and financial history are preserved.
* A transfer audit record is created.

## Late Fee

Late fees are calculated based on the number of days after the due date.

```text
Late Days = Actual Return Date - Due Date

Late Fee = Late Days × Late Fee Per Day
```

The refundable amount is calculated as:

```text
Refund Amount = Deposit Amount - Late Fee
```

The refund cannot become negative.

## Borrowing Limit

The system limits the number of active equipment units a borrower can have at one time.

The default borrowing limit is **5 units**.

## Installation

### Prerequisites

Make sure the following are installed:

* Node.js
* npm
* MySQL
* Git

### 1. Clone the Repository

```bash
git clone https://github.com/Sadhanapatidar001/Auriga.git
cd Auriga
```

### 2. Backend Setup

```bash
cd backend
npm install
```

Create a `.env` file inside the `backend` directory and configure the database connection and JWT secret.

Example:

```env
DATABASE_URL="mysql://USERNAME:PASSWORD@localhost:3306/av_room_db"
JWT_SECRET="your-secret-key"
PORT=5000
```

### 3. Database Setup

Create the MySQL database:

```sql
CREATE DATABASE av_room_db;
```

Run Prisma migrations:

```bash
npx prisma migrate deploy
```

Generate Prisma Client:

```bash
npx prisma generate
```

If seed data is configured:

```bash
npx prisma db seed
```

### 4. Start Backend

```bash
npm run dev
```

Backend runs on:

```text
http://localhost:5000
```

### 5. Frontend Setup

Open another terminal:

```bash
cd frontend
npm install
npm run dev
```

Frontend runs on:

```text
http://localhost:5173
```

## Environment Variables

Sensitive environment variables should **not** be committed to GitHub.

The `.env` file is ignored using `.gitignore`.

Use `.env.example` to document required environment variables without exposing passwords, secrets, or credentials.

## Main Workflow

```text
User Login
    ↓
Browse Equipment
    ↓
Check Availability
    ↓
Request Equipment
    ↓
Admin Approval
    ↓
Equipment Borrowed
    ↓
Active Loan
    ↓
Return Equipment
    ↓
Late Fee Calculation
    ↓
Deposit Refund
```

## Loan Transfer Workflow

```text
Active Loan
    ↓
Admin Selects New Borrower
    ↓
Validate New Borrower
    ↓
Check Borrowing Limit
    ↓
Transfer Existing Loan
    ↓
Create Transfer Audit Record
    ↓
Notify New Borrower
```

## Future Improvements

* Email notifications
* Automated reminder scheduler
* QR/barcode scanning for equipment
* Advanced admin dashboard
* Equipment usage analytics
* Fine-grained permission management

## License

This project is developed as a college/company build-round project.
