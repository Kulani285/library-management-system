# Software Requirements Specification (SRS)
## Library Management System


**Stack:** React (frontend) · Node.js/Express (backend) · PostgreSQL (database)

---

## 1. Introduction

### 1.1 Purpose
This document defines the functional and non-functional requirements for a Library Management System (LMS) that allows a library to manage its book catalog, members, and borrowing operations digitally.

### 1.2 Project Scope
The system will support two user roles, **Admin (Librarian)** and **Member**, and will cover the full lifecycle of book management: cataloging, searching, borrowing, returning, fine calculation, and basic reporting.

### 1.3 Intended Audience
This document is intended as a reference for design, development, and test case creation throughout the SDLC/STLC of this project.

---

## 2. Overall Description

### 2.1 User Roles

| Role | Description |
|---|---|
| **Admin/Librarian** | Manages catalog, members, loans, and views reports |
| **Member** | Browses catalog, borrows/returns books, views their own loan history and fines |

### 2.2 Assumptions & Constraints
- Single library branch (no multi-branch support in v1)
- Payment processing will use a third-party gateway in **test/sandbox mode** (e.g., Stripe Test Mode)
- AI features will use a third-party LLM API (e.g., Anthropic Claude API or OpenAI API) on a metered/limited-budget basis; usage will be rate-limited and cached where possible to control cost
- Email notifications are in scope for v1 (see FR-29 below)

---

## 3. Functional Requirements

Each requirement has an ID so it can be traced directly into test cases later (e.g., FR-01 → TC-01, TC-02...).

### 3.1 Authentication & Authorization
- **FR-01**: System shall allow a user to register as a Member with name, email, and password.
- **FR-02**: System shall allow login via email/password, returning a session token (JWT).
- **FR-03**: System shall restrict Admin-only actions (catalog management, member management, reports) to users with the Admin role.
- **FR-04**: System shall allow logout, invalidating the session token.

### 3.2 Catalog Management (Admin)
- **FR-05**: Admin shall be able to add a new book (title, author, ISBN, category, total copies).
- **FR-06**: Admin shall be able to edit an existing book's details.
- **FR-07**: Admin shall be able to delete a book, provided it has no active loans.
- **FR-08**: System shall prevent duplicate ISBN entries.

### 3.3 Catalog Browsing & Search (Member + Admin)
- **FR-09**: Users shall be able to search books by title, author, ISBN, or category.
- **FR-10**: Search results shall display availability (copies available vs. total copies).
- **FR-11**: Users shall be able to view a single book's detail page.

### 3.4 Borrowing & Returning
- **FR-12**: Member shall be able to borrow an available book (creates a Loan record with due date = borrow date + 14 days).
- **FR-13**: System shall prevent borrowing if zero copies are available.
- **FR-14**: System shall prevent a Member from borrowing more than 5 books at once.
- **FR-15**: Member shall be able to return a borrowed book.
- **FR-16**: System shall update available copy count immediately on borrow/return.

### 3.5 Fines & Payments
- **FR-17**: System shall calculate a fine of $0.50/day for each day a book is returned after its due date.
- **FR-18**: Member shall be able to view their current outstanding fines.
- **FR-19**: Member shall be able to pay an outstanding fine via integrated payment gateway (Stripe, test mode).
- **FR-19a**: System shall create a payment session/intent with the gateway and redirect/embed the checkout for the fine amount.
- **FR-19b**: System shall mark a fine as "paid" only after receiving a confirmed payment webhook from the gateway (not on client-side redirect alone).
- **FR-19c**: System shall handle failed or cancelled payments gracefully, leaving the fine status as "unpaid" and showing an error message to the Member.
- **FR-19d**: System shall store a payment record (amount, gateway transaction ID, status, timestamp) for each payment attempt, linked to the relevant fine.
- **FR-20**: System shall prevent a Member with unpaid fines over $10 from borrowing additional books.
- **FR-20a**: Admin shall be able to view a log of all payment transactions, including failed attempts.
- **FR-20b**: Admin shall be able to manually mark a fine as "waived" (e.g., for disputes), separate from "paid", with a required reason note.

### 3.6 Member Management (Admin)
- **FR-21**: Admin shall be able to view a list of all members.
- **FR-22**: Admin shall be able to view a single member's loan history and fine history.
- **FR-23**: Admin shall be able to deactivate a member account.

### 3.7 Reporting (Admin)
- **FR-24**: Admin shall be able to view a report of currently overdue books.
- **FR-25**: Admin shall be able to view a report of most-borrowed books (top 10).

### 3.8 AI-Powered Features

**Natural-Language Search & Recommendations**
- **FR-26**: Member shall be able to enter a free-text query (e.g., "a short mystery novel for a rainy afternoon") and receive relevant book matches.
- **FR-26a**: System shall use an LLM to extract structured intent (genre, length, mood, similar titles) from the query, then filter/rank the catalog using that structured data, rather than relying on the LLM alone to "know" the catalog.
- **FR-26b**: If the LLM service is unavailable, search shall fall back to standard keyword search (see NFR-10).

**AI-Generated Book Summaries**
- **FR-27**: Admin shall be able to request an AI-generated short description for a book by providing title and author; the LLM-generated text pre-fills the description field for the Admin to review and edit before saving.
- **FR-27a**: System shall cache generated summaries so the same title/author pair is never re-generated unnecessarily (controls API cost).

**AI Chatbot Assistant (Member-facing)**
- **FR-28**: Member shall be able to open a chat widget and ask natural-language questions (e.g., "what do I have borrowed?", "when is it due?", "recommend something like [book]").
- **FR-28a**: Chatbot shall have read access to the logged-in member's own loan and fine history only, never another member's data.
- **FR-28b**: Chatbot shall be read-only for v1; it cannot borrow, return, or pay on the member's behalf (write actions are a Future Enhancement).
- **FR-28c**: System shall log chatbot conversations (associated with member ID) for debugging/quality review.

### 3.9 Email Notifications
- **FR-29**: System shall send an email to a Member when they successfully borrow a book, including title and due date.
- **FR-29a**: System shall send a reminder email to a Member 2 days before a book's due date.
- **FR-29b**: System shall send an overdue notice email to a Member once a book becomes overdue, including the accruing fine amount.
- **FR-29c**: System shall send a confirmation email to a Member after a fine payment is successfully processed.
- **FR-29d**: System shall use a transactional email provider (e.g., SendGrid, Postmark) rather than sending directly from the application server.
- **FR-29e**: Member shall be able to opt out of non-critical email notifications (reminders) in their account settings; critical emails (overdue notices, payment confirmations) cannot be disabled.

---

## 4. Non-Functional Requirements

| ID | Requirement |
|---|---|
| **NFR-01** | API response time shall be under 500ms for 95% of requests under normal load |
| **NFR-02** | Passwords shall be hashed (bcrypt) before storage; never stored in plaintext |
| **NFR-03** | System shall be usable on screen widths from 375px (mobile) to 1920px (desktop) |
| **NFR-04** | Database schema shall enforce referential integrity (e.g., a Loan cannot reference a non-existent Book or Member) |
| **NFR-05** | System shall handle at least 50 concurrent users without degraded performance |
| **NFR-06** | All Admin actions on members' fines shall be logged for audit purposes |
| **NFR-07** | System shall never store raw card details; all card data is handled exclusively by the payment gateway (PCI-DSS scope reduction via tokenization) |
| **NFR-08** | Payment webhook endpoints shall verify the gateway's signature to prevent spoofed payment confirmations |
| **NFR-09** | All payment-related API calls shall use HTTPS only |
| **NFR-10** | AI-dependent features (search, chatbot) shall degrade gracefully if the LLM API is unavailable: search falls back to keyword search; chatbot shows a clear "currently unavailable" state rather than failing silently |
| **NFR-11** | AI-feature endpoints shall be rate-limited per user to control API cost and prevent abuse |
| **NFR-12** | All user input passed into LLM prompts shall be sanitized/scoped so it cannot be used to manipulate the system beyond the intended feature (prompt-injection mitigation) |
| **NFR-13** | AI feature responses (search, chatbot) shall target under 3 seconds end-to-end, acknowledging LLM latency is higher than standard API calls (see NFR-01) |
| **NFR-14** | Failure to send a notification email shall be logged and retried (at least once) rather than silently dropped |

---

## 5. Out of Scope (v1)
- Multi-branch library support
- Real (live/production) payment processing; sandbox/test mode only
- Book reservations/holds queue
- Mobile native app
- Chatbot performing write actions on a member's behalf (borrowing, returning, paying); read-only assistant in v1

---

## 6. Traceability Note
Every FR/NFR ID above will map to one or more test cases in the **Test Case Design** document (STLC Phase 3), so coverage can be verified at test closure.
