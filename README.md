# 🐷 Piggy Saving System — API Reference

> **Version:** 1.0.0  
> **Base URL:** `https://<project-ref>.supabase.co`  
> **Protocol:** HTTPS only  
> **Content-Type:** `application/json`  
> **Last Updated:** 2026-03-11

---

## Table of Contents

1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Authentication](#authentication)
4. [Edge Functions](#edge-functions)
   - [Process Transfer](#1-process-transfer)
   - [Calculate Interest](#2-calculate-interest)
5. [Data Transfer Objects (DTOs)](#data-transfer-objects-dtos)
6. [Data Model](#data-model)
7. [Security](#security)
8. [QR Code Payment Flow](#qr-code-payment-flow)
9. [Error Handling](#error-handling)
10. [Rate Limiting & Idempotency](#rate-limiting--idempotency)
11. [Admin Operations](#admin-operations)
12. [Client SDK Usage](#client-sdk-usage)

---

## Overview

The Piggy Saving System is a digital savings platform providing:

- **Main Accounts** — Primary wallet seeded with $1,000 on registration
- **Piggy Goals** — Locked/unlocked goal-based savings accounts with compound interest
- **P2P Transfers** — Peer-to-peer money transfers between users
- **Contributions** — Crowdfunded contributions to any user's saving goal
- **QR Payments** — TLV-encoded QR codes for contactless P2P and contributions
- **Double-Entry Ledger** — Bank-grade audit trail for every financial movement
- **Admin Controls** — Transaction reversals, system settings, and user management

All financial mutations are processed through **server-side Edge Functions** to guarantee atomic balance updates and ledger integrity.

---

## Architecture

```
┌─────────────┐     ┌──────────────────────┐     ┌──────────────┐
│  React SPA  │────▶│  Edge Functions       │────▶│  PostgreSQL  │
│  (Frontend) │     │  (Business Logic)     │     │  (Database)  │
└─────────────┘     └──────────────────────┘     └──────────────┘
       │                     │                          │
       │              ┌──────┴───────┐                  │
       │              │  Service     │                  │
       │              │  Role Key    │──────────────────┘
       │              └──────────────┘
       │
       └── JWT Bearer Token (User Auth)
```

**Key Design Principles:**
- All balance mutations happen server-side via service role
- Double-entry bookkeeping for every transaction
- Idempotency keys prevent duplicate processing
- RLS policies enforce data isolation at the database level

---

## Authentication

### Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/auth/v1/signup` | Register new user |
| `POST` | `/auth/v1/token?grant_type=password` | Login (email + password) |
| `POST` | `/auth/v1/logout` | Invalidate current session |

---

### `POST /auth/v1/signup`

**Request DTO:**

```typescript
interface SignUpRequest {
  email: string;          // Required. Valid email address.
  password: string;       // Required. Minimum 6 characters.
  data: {
    full_name: string;    // Required. User's display name.
    phone: string;        // Optional. Phone number for P2P lookup.
  };
}
```

```json
{
  "email": "john.doe@example.com",
  "password": "Str0ngP@ss!",
  "data": {
    "full_name": "John Doe",
    "phone": "+1234567890"
  }
}
```

**Response DTO (201 Created):**

```typescript
interface SignUpResponse {
  access_token: string;
  token_type: "bearer";
  expires_in: number;       // Seconds until expiry
  expires_at: number;       // Unix timestamp
  refresh_token: string;
  user: {
    id: string;             // UUID
    email: string;
    created_at: string;     // ISO 8601
    user_metadata: {
      full_name: string;
      phone: string;
    };
  };
}
```

```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIs...",
  "token_type": "bearer",
  "expires_in": 3600,
  "expires_at": 1710000000,
  "refresh_token": "v1.MjA0...",
  "user": {
    "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "email": "john.doe@example.com",
    "created_at": "2026-03-11T10:00:00.000Z",
    "user_metadata": {
      "full_name": "John Doe",
      "phone": "+1234567890"
    }
  }
}
```

**Database Trigger — `handle_new_user()`:**

On successful sign-up, the following records are automatically created:

| Table | Record |
|-------|--------|
| `profiles` | `{ user_id, full_name, phone }` |
| `user_roles` | `{ user_id, role: 'user' }` |
| `accounts` | `{ user_id, account_type: 'main', balance: 1000.00, currency: 'USD' }` |

---

### `POST /auth/v1/token?grant_type=password`

**Request DTO:**

```typescript
interface LoginRequest {
  email: string;
  password: string;
}
```

```json
{
  "email": "john.doe@example.com",
  "password": "Str0ngP@ss!"
}
```

**Response DTO (200 OK):**

```typescript
interface LoginResponse {
  access_token: string;
  token_type: "bearer";
  expires_in: number;
  expires_at: number;
  refresh_token: string;
  user: {
    id: string;
    email: string;
    role: string;
    last_sign_in_at: string;
  };
}
```

**Error Response (400):**

```json
{
  "error": "invalid_grant",
  "error_description": "Invalid login credentials"
}
```

---

### PIN Security Layer

The application implements a secondary authentication layer via a 6-digit PIN:

| Operation | Description |
|-----------|-------------|
| **Setup** | User sets PIN → hashed and stored in `profiles.pin_hash` |
| **Unlock** | App locks after inactivity; PIN required to re-enter |
| **Confirm** | PIN confirmation required before any financial transfer |

> PIN hashing is performed client-side using SHA-256 before storage.

---

## Edge Functions

### 1. Process Transfer

| Property | Value |
|----------|-------|
| **Function Name** | `process-transfer` |
| **Method** | `POST` |
| **Authentication** | Bearer token (JWT) required |
| **Authorization** | Authenticated users only |

Handles all financial movements atomically. Each request produces:
- ✅ Transaction record
- ✅ Double-entry ledger entries (debit + credit)
- ✅ Balance updates (source and destination)
- ✅ User notifications
- ✅ Idempotency key for duplicate prevention

---

#### Action: `transfer` — Main → Own Piggy

Transfer funds from the user's main account to one of their own piggy saving goals.

**Request DTO:**

```typescript
interface TransferRequest {
  action: "transfer";
  amount: number;          // Required. Must be > 0.
  piggy_goal_id: string;   // Required. UUID of the target piggy goal.
}
```

```json
{
  "action": "transfer",
  "amount": 50.00,
  "piggy_goal_id": "b7e4f2a1-3c8d-4e9f-a1b2-c3d4e5f67890"
}
```

**Response DTO (200 OK):**

```typescript
interface TransferResponse {
  message: "Transfer successful";
  transaction_id: string;   // UUID of the created transaction
}
```

```json
{
  "message": "Transfer successful",
  "transaction_id": "f1e2d3c4-b5a6-7890-cdef-1234567890ab"
}
```

**Validation Rules:**

| Rule | Error Message |
|------|---------------|
| `amount` must be > 0 | `"Invalid amount"` |
| `piggy_goal_id` required | `"Missing piggy goal ID"` |
| Sufficient main balance | `"Insufficient balance"` |
| Goal belongs to user | `"Piggy account not found"` |
| Goal status = `active` | `"Goal is not active"` |

**Side Effects:**

| Effect | Detail |
|--------|--------|
| Main balance | Decreased by `amount` |
| Piggy balance | Increased by `amount` |
| Ledger | Debit main → Credit piggy |
| Auto-complete | If `piggy_balance >= target_amount` → goal status = `completed` |
| Notification | `"$50.00 transferred to 'Goal Name'"` |

---

#### Action: `p2p` — Main → Another User's Main

Send money directly to another user's main account.

**Request DTO:**

```typescript
interface P2PRequest {
  action: "p2p";
  amount: number;              // Required. Must be > 0.
  recipient_user_id: string;   // Required. UUID of the recipient user.
}
```

```json
{
  "action": "p2p",
  "amount": 25.00,
  "recipient_user_id": "c3d4e5f6-a1b2-7890-cdef-abcdef123456"
}
```

**Response DTO (200 OK):**

```typescript
interface P2PResponse {
  message: "P2P transfer successful";
  transaction_id: string;
}
```

```json
{
  "message": "P2P transfer successful",
  "transaction_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
}
```

**Validation Rules:**

| Rule | Error Message |
|------|---------------|
| `amount` must be > 0 | `"Invalid amount"` |
| `recipient_user_id` required | `"Missing recipient"` |
| Cannot send to self | `"Cannot transfer to yourself"` |
| Sufficient balance | `"Insufficient balance"` |
| Recipient has main account | `"Recipient account not found"` |

**Side Effects:**

| Effect | Detail |
|--------|--------|
| Sender balance | Decreased by `amount` |
| Recipient balance | Increased by `amount` |
| Ledger | Debit sender → Credit recipient |
| Sender notification | `"$25.00 sent to John Doe"` |
| Recipient notification | `"$25.00 received from Jane Smith"` |

---

#### Action: `contribute` — Main → Any User's Piggy Goal

Contribute funds to any user's active saving goal (crowdfunding).

**Request DTO:**

```typescript
interface ContributeRequest {
  action: "contribute";
  amount: number;                      // Required. Must be > 0.
  recipient_piggy_goal_id: string;     // Required. UUID of the target goal.
}
```

```json
{
  "action": "contribute",
  "amount": 10.00,
  "recipient_piggy_goal_id": "d4e5f6a1-b2c3-7890-abcd-ef1234567890"
}
```

**Response DTO (200 OK):**

```typescript
interface ContributeResponse {
  message: "Contribution successful";
  transaction_id: string;
}
```

```json
{
  "message": "Contribution successful",
  "transaction_id": "e5f6a1b2-c3d4-7890-abcd-ef1234567890"
}
```

**Validation Rules:**

| Rule | Error Message |
|------|---------------|
| `amount` must be > 0 | `"Invalid amount"` |
| `recipient_piggy_goal_id` required | `"Missing piggy goal"` |
| Sufficient balance | `"Insufficient balance"` |
| Goal status = `active` | `"Goal not active"` |

**Side Effects:**

| Effect | Detail |
|--------|--------|
| Contributor balance | Decreased by `amount` |
| Goal piggy balance | Increased by `amount` |
| Ledger | Debit contributor → Credit piggy |
| Auto-complete | If `piggy_balance >= target_amount` → goal completed |
| Contributor notification | `"$10.00 contributed to 'Goal Name'"` |
| Goal owner notification | `"Jane contributed $10.00 to 'Goal Name' 🎉"` |

---

#### Action: `break` — Break a Piggy Goal

Liquidate a piggy goal and return funds to the main account. Early breaks (before lock period expires) incur a penalty.

**Request DTO:**

```typescript
interface BreakRequest {
  action: "break";
  break_piggy_goal_id: string;   // Required. UUID of the goal to break.
}
```

```json
{
  "action": "break",
  "break_piggy_goal_id": "f6a1b2c3-d4e5-7890-abcd-ef1234567890"
}
```

**Response DTO (200 OK):**

```typescript
interface BreakResponse {
  message: "Piggy broken";
  returnAmount: number;      // Amount returned to main (after penalty)
  penalty: number;           // Penalty amount deducted (0 if no penalty)
  transaction_id: string;
}
```

```json
{
  "message": "Piggy broken",
  "returnAmount": 95.00,
  "penalty": 5.00,
  "transaction_id": "1a2b3c4d-5e6f-7890-abcd-ef1234567890"
}
```

**Validation Rules:**

| Rule | Error Message |
|------|---------------|
| `break_piggy_goal_id` required | `"Missing piggy goal"` |
| Goal belongs to user | `"Goal not active or not found"` |
| Goal status = `active` | `"Goal not active or not found"` |
| Piggy balance > 0 | `"No balance to break"` |

**Penalty Calculation:**

```
lock_expiry = goal.created_at + (lock_period_days × 86,400,000ms)

IF current_time < lock_expiry:
    penalty = balance × (early_break_penalty_pct / 100)    // Default: 5%
    returnAmount = balance - penalty
ELSE:
    penalty = 0
    returnAmount = balance
```

**Side Effects:**

| Effect | Detail |
|--------|--------|
| Piggy balance | Set to `0` |
| Main balance | Increased by `returnAmount` |
| Ledger (break) | Debit piggy (full balance) → Credit main (returnAmount) |
| Ledger (fee) | If penalty > 0: separate fee transaction + ledger entry |
| Goal status | Updated to `"broken"`, `broken_at` timestamp set |
| Notification | Includes penalty details if applicable |

---

### 2. Calculate Interest

| Property | Value |
|----------|-------|
| **Function Name** | `calculate-interest` |
| **Method** | `POST` |
| **Authentication** | Service-level (no user JWT) |
| **Trigger** | Daily cron job / manual invocation |

Calculates and credits daily compound interest to all active piggy accounts.

**Request DTO:**

```typescript
// No request body required
interface CalculateInterestRequest {}
```

**Response DTO (200 OK):**

```typescript
interface CalculateInterestResponse {
  message: "Interest calculated" | "No active piggy accounts";
  processed: number;          // Number of accounts that received interest
  totalInterest: number;      // Sum of all interest credited
}
```

```json
{
  "message": "Interest calculated",
  "processed": 42,
  "totalInterest": 3.67
}
```

**Algorithm:**

```
annual_rate = system_settings['interest_rate_piggy']   // Default: 3.50%
daily_rate  = annual_rate / 100 / 365

FOR EACH piggy_account WHERE balance > 0 AND goal.status = 'active':
    IF already_paid_today(account_id):
        SKIP
    
    interest = ROUND(balance × daily_rate, 2)
    
    IF interest <= 0:
        SKIP
    
    INSERT interest_payment { account_id, piggy_goal_id, principal, rate, amount }
    INSERT transaction { to_account_id, amount, type: 'interest', idempotency_key }
    INSERT ledger_entry { transaction_id, account_id, credit: interest }
    UPDATE account.balance += interest
```

**Idempotency Key Format:** `interest-{accountId}-{YYYY-MM-DD}`

> ✅ Safe to invoke multiple times per day — duplicate processing is prevented.

**Error Response (500):**

```json
{
  "error": "Interest calculation error: <detail>"
}
```

---

## Data Transfer Objects (DTOs)

### Unified Request DTO

```typescript
interface TransferRequest {
  action: "transfer" | "p2p" | "contribute" | "break";
  amount?: number;                      // Required for: transfer, p2p, contribute
  piggy_goal_id?: string;               // Required for: transfer
  recipient_user_id?: string;           // Required for: p2p
  recipient_piggy_goal_id?: string;     // Required for: contribute
  break_piggy_goal_id?: string;         // Required for: break
}
```

### Unified Response DTO

```typescript
interface TransferResult {
  message: string;
  transaction_id?: string;
  returnAmount?: number;     // Only for: break
  penalty?: number;          // Only for: break
  error?: string;            // Only on failure
}
```

### Account DTO

```typescript
interface Account {
  id: string;                // UUID
  user_id: string;           // UUID
  account_type: "main" | "piggy";
  balance: number;           // Decimal, 2 places
  currency: string;          // Default: "USD"
  piggy_goal_id: string | null;
  created_at: string;        // ISO 8601
  updated_at: string;        // ISO 8601
}
```

### Piggy Goal DTO

```typescript
interface PiggyGoal {
  id: string;                    // UUID
  user_id: string;               // UUID
  name: string;                  // Goal display name
  target_amount: number;         // Decimal target
  lock_period_days: number | null; // Lock duration (null = no lock)
  hide_balance: boolean;         // Hide balance from non-owners
  status: "active" | "completed" | "broken";
  created_at: string;            // ISO 8601
  completed_at: string | null;   // ISO 8601, set on completion
  broken_at: string | null;      // ISO 8601, set when broken
}
```

### Transaction DTO

```typescript
interface Transaction {
  id: string;                    // UUID
  from_account_id: string | null;
  to_account_id: string | null;
  amount: number;
  type: TransactionType;
  status: TransactionStatus;
  description: string | null;
  idempotency_key: string | null;
  created_at: string;            // ISO 8601
  completed_at: string | null;   // ISO 8601
}

type TransactionType =
  | "transfer"        // Main → own piggy
  | "p2p"             // Main → another user's main
  | "contribution"    // Main → any user's piggy goal
  | "interest"        // System-generated daily interest
  | "fee"             // Early break penalty
  | "break"           // Piggy → main (goal liquidation)
  | "deposit"         // Initial signup deposit
  | "reversal";       // Admin reversal

type TransactionStatus =
  | "pending"         // Created, not yet processed
  | "completed"       // Successfully executed
  | "failed"          // Execution failed
  | "reversed";       // Reversed by admin
```

### Ledger Entry DTO

```typescript
interface LedgerEntry {
  id: string;              // UUID
  transaction_id: string;  // FK → transactions
  account_id: string;      // FK → accounts
  debit: number;           // Money leaving account
  credit: number;          // Money entering account
  created_at: string;      // ISO 8601
}
```

### Interest Payment DTO

```typescript
interface InterestPayment {
  id: string;              // UUID
  account_id: string;      // FK → accounts
  piggy_goal_id: string;   // FK → piggy_goals
  principal: number;       // Balance at time of calculation
  rate: number;            // Daily rate applied
  amount: number;          // Interest credited
  created_at: string;      // ISO 8601
}
```

### Notification DTO

```typescript
interface Notification {
  id: string;              // UUID
  user_id: string;         // UUID
  title: string;
  message: string;
  type: "info" | "success" | "warning" | "error";
  read: boolean;           // Default: false
  created_at: string;      // ISO 8601
}
```

### Profile DTO

```typescript
interface Profile {
  id: string;                  // UUID
  user_id: string;             // UUID
  full_name: string | null;
  phone: string | null;
  pin_hash: string | null;     // SHA-256 hashed PIN
  last_activity: string | null; // ISO 8601
  created_at: string;          // ISO 8601
  updated_at: string;          // ISO 8601
}
```

### System Settings DTO

```typescript
interface SystemSetting {
  id: string;              // UUID
  key: string;             // Unique setting identifier
  value: string;           // String-encoded value
  updated_at: string;      // ISO 8601
}
```

---

## Data Model

### Entity Relationship Diagram

```
┌──────────┐       ┌──────────────┐       ┌──────────────┐
│  Users   │──1:1──│   Profiles   │       │  User Roles  │
│ (auth)   │──1:N──│              │       │              │
└──────────┘       └──────────────┘       └──────────────┘
     │                                          │
     │ 1:N                                      │
     ▼                                          │
┌──────────────┐       ┌──────────────┐         │
│   Accounts   │──N:1──│ Piggy Goals  │         │
│ (main/piggy) │       │              │         │
└──────────────┘       └──────────────┘         │
     │                       │                  │
     │ 1:N                   │ 1:N              │
     ▼                       ▼                  │
┌──────────────┐   ┌──────────────────┐         │
│ Transactions │   │Interest Payments │         │
└──────────────┘   └──────────────────┘         │
     │                                          │
     │ 1:N                                      │
     ▼                                          │
┌──────────────┐   ┌──────────────────┐         │
│Ledger Entries│   │  System Settings │─────────┘
└──────────────┘   └──────────────────┘
                   ┌──────────────────┐
                   │  Notifications   │
                   └──────────────────┘
```

### Tables Summary

| Table | Purpose | Primary Key | Key Relationships |
|-------|---------|-------------|-------------------|
| `profiles` | User profile data | `id` (UUID) | `user_id` → auth.users |
| `user_roles` | RBAC role assignments | `id` (UUID) | `user_id` → auth.users |
| `accounts` | Financial accounts | `id` (UUID) | `user_id`, `piggy_goal_id` → piggy_goals |
| `piggy_goals` | Saving goal definitions | `id` (UUID) | `user_id` → auth.users |
| `transactions` | All money movements | `id` (UUID) | `from_account_id`, `to_account_id` → accounts |
| `ledger_entries` | Double-entry audit trail | `id` (UUID) | `transaction_id` → transactions, `account_id` → accounts |
| `interest_payments` | Interest accrual log | `id` (UUID) | `account_id` → accounts, `piggy_goal_id` → piggy_goals |
| `notifications` | User notifications | `id` (UUID) | `user_id` → auth.users |
| `system_settings` | Admin configuration | `id` (UUID) | — |

### System Settings (Defaults)

| Key | Default Value | Type | Description |
|-----|---------------|------|-------------|
| `interest_rate_piggy` | `3.50` | Percentage | Annual compound interest rate for piggy accounts |
| `early_break_penalty_pct` | `5` | Percentage | Penalty for breaking locked goals before expiry |

---

## Security

### Authentication Flow

```
Client                    Edge Function                  Database
  │                            │                            │
  │── Bearer JWT ─────────────▶│                            │
  │                            │── Verify JWT ─────────────▶│
  │                            │◀── User Object ────────────│
  │                            │                            │
  │                            │── Service Role Key ───────▶│
  │                            │   (bypasses RLS)           │
  │                            │◀── Query Results ──────────│
  │◀── JSON Response ─────────│                            │
```

### Row-Level Security (RLS) Matrix

All tables have RLS **enabled**. Policies enforce data isolation:

| Table | `SELECT` | `INSERT` | `UPDATE` | `DELETE` |
|-------|----------|----------|----------|----------|
| `profiles` | Own + Admin + Auth search | Own | Own | ✗ |
| `user_roles` | Own + Admin | Admin | Admin | Admin |
| `accounts` | Own + Admin + Auth | Own | Own + Auth | ✗ |
| `piggy_goals` | Own + Admin + Active | Own | Own | ✗ |
| `transactions` | Own (via account) + Admin | Auth | Admin | ✗ |
| `ledger_entries` | Own (via account) + Admin | Auth | ✗ | ✗ |
| `interest_payments` | Own (via account) | Service | ✗ | ✗ |
| `notifications` | Own | Auth | Own | ✗ |
| `system_settings` | Public (read) | Admin | Admin | Admin |

### Admin Role Verification

```sql
-- Security definer function (bypasses RLS, prevents recursion)
CREATE FUNCTION public.has_role(_user_id UUID, _role app_role)
RETURNS BOOLEAN
LANGUAGE SQL STABLE SECURITY DEFINER
SET search_path = 'public'
AS $$
  SELECT EXISTS (
    SELECT 1 FROM public.user_roles
    WHERE user_id = _user_id AND role = _role
  )
$$;
```

### Double-Entry Ledger Invariant

Every transaction produces balanced ledger entries:

```
∀ transaction_id: SUM(debit) = SUM(credit)
```

| Operation | Debit Account | Credit Account |
|-----------|---------------|----------------|
| Transfer (main→piggy) | Main | Piggy |
| P2P | Sender Main | Recipient Main |
| Contribute | Contributor Main | Goal Piggy |
| Break | Piggy (full) | Main (return) |
| Fee | Piggy (penalty) | — (system) |
| Interest | — | Piggy |

---

## QR Code Payment Flow

### TLV Payload Structure

The QR payload uses a **Tag-Length-Value** encoding scheme compatible with EMV/KHQR standards:

| Tag | Field | Description |
|-----|-------|-------------|
| `00` | Format Indicator | Always `01` |
| `01` | Point of Initiation | `11` (static) or `12` (dynamic) |
| `26` | Merchant Account Info | Nested TLV: user/piggy data |
| `58` | Country Code | `KH` |
| `62` | Additional Data | Nested TLV: expiry timestamp |
| `63` | CRC | CRC-16 checksum (CCITT-FALSE) |

**Merchant Account (Tag 26) Sub-fields:**

| Sub-Tag | Field | Description |
|---------|-------|-------------|
| `00` | Type | `main` (P2P) or `piggy` (contribution) |
| `01` | User ID | UUID of the recipient |
| `02` | Account ID | UUID of the account (optional) |
| `03` | Piggy ID | UUID of the piggy goal (optional) |
| `04` | Name | Display name (optional) |

### QR Payment Sequence

```
Generator                Scanner                 Edge Function
    │                       │                         │
    │── Generate QR ────────│                         │
    │   (TLV payload)       │                         │
    │                       │── Scan/Decode ───────── │
    │                       │   Validate expiry       │
    │                       │   Validate recipient    │
    │                       │                         │
    │                       │── Enter Amount ─────────│
    │                       │── Confirm PIN ──────────│
    │                       │                         │
    │                       │── process-transfer ────▶│
    │                       │   action: p2p/contribute│
    │                       │◀── Result ──────────────│
    │                       │                         │
```

---

## Error Handling

### Error Response Format

All errors return a consistent JSON structure:

```typescript
interface ErrorResponse {
  error: string;   // Human-readable error message
}
```

### HTTP Status Codes

| Code | Usage |
|------|-------|
| `200` | Successful operation |
| `400` | Client error (validation, insufficient funds, etc.) |
| `401` | Missing or invalid authentication |
| `500` | Server error (interest calculation failures) |

### Error Catalog

| Error Message | Action(s) | Cause |
|---------------|-----------|-------|
| `Missing authorization header` | All | No `Authorization` header provided |
| `Unauthorized` | All | Invalid or expired JWT token |
| `Server configuration error` | All | Missing environment variables |
| `Main account not found` | All | User has no main account |
| `Invalid amount` | transfer, p2p, contribute | `amount` ≤ 0 or missing |
| `Insufficient balance` | transfer, p2p, contribute | Not enough funds in main account |
| `Missing piggy goal ID` | transfer | `piggy_goal_id` not provided |
| `Piggy account not found` | transfer, contribute, break | No account linked to goal |
| `Goal is not active` | transfer | Goal status ≠ `active` |
| `Goal not active` | contribute | Goal status ≠ `active` |
| `Missing recipient` | p2p | `recipient_user_id` not provided |
| `Cannot transfer to yourself` | p2p | Sender = recipient |
| `Recipient account not found` | p2p | Recipient has no main account |
| `Missing piggy goal` | contribute, break | Goal ID not provided |
| `Goal not active or not found` | break | Goal doesn't exist, not owned by user, or not active |
| `No balance to break` | break | Piggy account balance = 0 |
| `Invalid action` | — | Unrecognized action type |

---

## Rate Limiting & Idempotency

### Idempotency

Every financial transaction is assigned a unique `idempotency_key` (UUID v4) to prevent duplicate processing:

| Operation | Key Format | Scope |
|-----------|------------|-------|
| Transfer | Random UUID | Per request |
| P2P | Random UUID | Per request |
| Contribute | Random UUID | Per request |
| Break | Random UUID | Per request |
| Fee | Random UUID | Per break with penalty |
| Interest | `interest-{accountId}-{YYYY-MM-DD}` | Per account per day |

### Concurrency Considerations

- Balance is **re-fetched** immediately before each mutation to minimize stale reads
- The system uses service-role operations (bypassing RLS) for atomic updates
- For production-grade guarantees, the intended architecture uses **pessimistic locking** (`SELECT ... FOR UPDATE`)

---

## Admin Operations

Admins (users with `role = 'admin'` in `user_roles`) have elevated access:

### Capabilities

| Operation | Description | Mechanism |
|-----------|-------------|-----------|
| View all users | Access all profiles and accounts | RLS: `has_role(auth.uid(), 'admin')` |
| View all transactions | Full transaction history | RLS policy |
| View ledger entries | Complete audit trail | RLS policy |
| Reverse transactions | Mark transaction as `reversed`, create reversal entry | Client-side mutation |
| Manage settings | Update interest rates, penalty percentages | Direct table update |
| Manage roles | Promote/demote users | Direct table update |

### Transaction Reversal Flow

```typescript
// 1. Mark original transaction as reversed
UPDATE transactions SET status = 'reversed' WHERE id = txId;

// 2. Create reversal transaction (swap from/to)
INSERT INTO transactions {
  from_account_id: original.to_account_id,
  to_account_id: original.from_account_id,
  amount: original.amount,
  type: 'reversal',
  status: 'completed',
  description: 'Reversal of <txId>'
};

// 3. Adjust account balances
UPDATE accounts SET balance = balance + amount WHERE id = original.from_account_id;
UPDATE accounts SET balance = balance - amount WHERE id = original.to_account_id;
```

---

## Client SDK Usage

### Setup

```typescript
import { supabase } from "@/integrations/supabase/client";
```

### Transfer to Own Piggy

```typescript
const { data, error } = await supabase.functions.invoke('process-transfer', {
  body: {
    action: 'transfer',
    amount: 50.00,
    piggy_goal_id: 'uuid-of-goal'
  }
});
```

### P2P Transfer

```typescript
const { data, error } = await supabase.functions.invoke('process-transfer', {
  body: {
    action: 'p2p',
    amount: 25.00,
    recipient_user_id: 'uuid-of-recipient'
  }
});
```

### Contribute to Goal

```typescript
const { data, error } = await supabase.functions.invoke('process-transfer', {
  body: {
    action: 'contribute',
    amount: 10.00,
    recipient_piggy_goal_id: 'uuid-of-goal'
  }
});
```

### Break Piggy Goal

```typescript
const { data, error } = await supabase.functions.invoke('process-transfer', {
  body: {
    action: 'break',
    break_piggy_goal_id: 'uuid-of-goal'
  }
});
```

### Trigger Interest Calculation

```typescript
const { data, error } = await supabase.functions.invoke('calculate-interest');
```

### Helper Wrapper (Recommended)

```typescript
import { processTransfer } from "@/lib/transfers";

// Handles auth validation, error extraction, and typing
const result = await processTransfer({
  action: 'transfer',
  amount: 50,
  piggy_goal_id: 'uuid'
});
```

---

## Appendix

### Environment Variables

| Variable | Scope | Description |
|----------|-------|-------------|
| `SUPABASE_URL` | Edge Functions | Project API URL |
| `SUPABASE_SERVICE_ROLE_KEY` | Edge Functions | Service role key (bypasses RLS) |
| `SUPABASE_ANON_KEY` | Edge Functions | Anonymous/public key |
| `VITE_SUPABASE_URL` | Frontend | Project API URL |
| `VITE_SUPABASE_PUBLISHABLE_KEY` | Frontend | Public anon key |

### Technology Stack

| Layer | Technology |
|-------|------------|
| Frontend | React 18 + Vite + TypeScript + Tailwind CSS |
| Backend | Supabase Edge Functions (Deno) |
| Database | PostgreSQL with RLS |
| Auth | Supabase Auth (JWT) |
| State | TanStack React Query |
| UI Components | shadcn/ui + Radix UI |

---

*This document is auto-generated from the system's edge functions, database schema, and RLS policies. For the full database DDL, see [`docs/full-database-schema.sql`](./full-database-schema.sql).*
