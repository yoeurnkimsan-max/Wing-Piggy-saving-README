# 🐷 Piggy Saving System — API Design Specification

> **Version:** 2.0.0  
> **Status:** Draft  
> **Author:** Piggy Engineering Team  
> **Last Updated:** 2026-03-12  
> **Stack:** Spring Boot 3.x · Java 17 · PostgreSQL 15  
> **Protocol:** HTTPS (TLS 1.3) · REST · JSON

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [Conventions](#2-conventions)
3. [Authentication & Authorization](#3-authentication--authorization)
4. [API Endpoints](#4-api-endpoints)
   - 4.1 [Auth Module](#41-auth-module)
   - 4.2 [Account Module](#42-account-module)
   - 4.3 [Piggy Goal Module](#43-piggy-goal-module)
   - 4.4 [Transfer Module](#44-transfer-module)
   - 4.5 [Interest Module](#45-interest-module)
   - 4.6 [Notification Module](#46-notification-module)
   - 4.7 [Admin Module](#47-admin-module)
   - 4.8 [QR Payment Module](#48-qr-payment-module)
   - 4.9 [Profile Module](#49-profile-module)
5. [Data Transfer Objects (DTOs)](#5-data-transfer-objects-dtos)
6. [Data Model](#6-data-model)
7. [Security Architecture](#7-security-architecture)
8. [Error Handling](#8-error-handling)
9. [Idempotency & Concurrency](#9-idempotency--concurrency)
10. [QR Code Payload Specification](#10-qr-code-payload-specification)
11. [Business Rules & Invariants](#11-business-rules--invariants)
12. [Appendix](#12-appendix)

---

## 1. Introduction

The **Piggy Saving System** is a digital savings platform that allows users to manage a main wallet, create goal-based savings accounts ("Piggies") with optional lock periods and compound interest, perform peer-to-peer transfers, contribute to other users' goals, and generate/scan QR codes for contactless payments.

### 1.1 Core Capabilities

| Capability | Description |
|---|---|
| **Main Account** | Primary wallet, seeded with $1,000.00 USD on registration |
| **Piggy Goals** | Goal-based savings with optional lock periods and daily compound interest |
| **P2P Transfer** | Direct money transfer between users via phone lookup |
| **Goal Contributions** | Crowdfunded contributions to any user's active saving goal |
| **QR Payments** | TLV-encoded QR codes for contactless P2P and contribution flows |
| **Double-Entry Ledger** | Immutable audit trail for every financial movement |
| **Interest Engine** | Daily compound interest calculation on active piggy accounts |
| **Admin Dashboard** | Transaction reversals, system settings, user/role management |

### 1.2 Architecture Overview

```
┌──────────────┐      ┌─────────────────────────────┐      ┌──────────────┐
│              │      │     Spring Boot 3.x          │      │              │
│   Frontend   │─────▶│                              │─────▶│  PostgreSQL  │
│   (SPA/App)  │ REST │  ┌─────────┐ ┌───────────┐  │ JPA  │     15       │
│              │◀─────│  │  Auth   │ │  Service   │  │◀─────│              │
└──────────────┘      │  │  Filter │ │  Layer     │  │      └──────────────┘
                      │  └─────────┘ └───────────┘  │
                      │  ┌─────────┐ ┌───────────┐  │
                      │  │   DTO   │ │ Repository │  │
                      │  │  Layer  │ │   Layer    │  │
                      │  └─────────┘ └───────────┘  │
                      └─────────────────────────────┘
```

---

## 2. Conventions

### 2.1 HTTP Methods

| Method | Usage |
|--------|-------|
| `GET` | Retrieve resource(s). Never mutates state. |
| `POST` | Create resource or execute action. |
| `PUT` | Full replacement of resource. |
| `PATCH` | Partial update of resource. |
| `DELETE` | Remove resource (soft-delete preferred). |

### 2.2 URL Structure

```
/api/v1/{module}/{resource}
/api/v1/{module}/{resource}/{id}
/api/v1/{module}/{resource}/{id}/{sub-resource}
```

### 2.3 Request Headers

| Header | Required | Description |
|--------|----------|-------------|
| `Authorization` | Yes* | `Bearer <JWT>` — required for all protected endpoints |
| `Content-Type` | Yes | `application/json` |
| `Accept` | Recommended | `application/json` |
| `Idempotency-Key` | Yes** | UUID v4 — required for all financial mutation endpoints |
| `X-Request-ID` | Optional | Client-generated trace ID for debugging |

### 2.4 Pagination

All list endpoints support cursor-based pagination:

```json
{
  "data": [...],
  "pagination": {
    "page": 1,
    "size": 20,
    "total_elements": 150,
    "total_pages": 8,
    "has_next": true,
    "has_previous": false
  }
}
```

**Query Parameters:**
- `page` — Page number (0-indexed, default: `0`)
- `size` — Items per page (default: `20`, max: `100`)
- `sort` — Sort field (e.g., `created_at`)
- `direction` — `asc` or `desc` (default: `desc`)

### 2.5 Timestamp Format

All timestamps follow **ISO 8601** with timezone: `2026-03-12T14:30:00.000Z`

### 2.6 Currency Format

All monetary values are represented as `BigDecimal` with **2 decimal places** and stored as `DECIMAL(15,2)` in the database. Currency code follows **ISO 4217** (default: `USD`).

---

## 3. Authentication & Authorization

### 3.1 Authentication Flow

```
┌────────┐                    ┌────────────┐                  ┌──────────┐
│ Client │                    │ Auth       │                  │ Database │
│        │                    │ Controller │                  │          │
└───┬────┘                    └─────┬──────┘                  └────┬─────┘
    │  POST /api/v1/auth/signup     │                              │
    │──────────────────────────────▶│  INSERT profiles             │
    │                               │─────────────────────────────▶│
    │                               │  INSERT user_roles (user)    │
    │                               │─────────────────────────────▶│
    │                               │  INSERT accounts (main,$1k)  │
    │                               │─────────────────────────────▶│
    │  201 { user, token }          │                              │
    │◀──────────────────────────────│                              │
    │                               │                              │
    │  POST /api/v1/auth/login      │                              │
    │──────────────────────────────▶│  Validate credentials        │
    │                               │─────────────────────────────▶│
    │  200 { access_token, ... }    │                              │
    │◀──────────────────────────────│                              │
```

### 3.2 JWT Token Structure

```json
{
  "sub": "550e8400-e29b-41d4-a716-446655440000",
  "email": "john@example.com",
  "roles": ["user"],
  "iat": 1741784400,
  "exp": 1741788000
}
```

### 3.3 Role-Based Access Control (RBAC)

| Role | Permissions |
|------|-------------|
| `user` | CRUD own resources, transfer, contribute, view own data |
| `admin` | All `user` permissions + view all users, reverse transactions, manage settings, manage roles |

Roles are stored in a **separate `user_roles` table** (not on the profile) to prevent privilege escalation.

---

## 4. API Endpoints

### 4.1 Auth Module

#### `POST /api/v1/auth/signup`

Register a new user account. Automatically provisions profile, role, and main account.

**Request Body:**

```json
{
  "email": "john.doe@example.com",
  "password": "Str0ngP@ss!",
  "full_name": "John Doe",
  "phone": "+855123456789"
}
```

**Response — `201 Created`:**

```json
{
  "status": "success",
  "data": {
    "user_id": "550e8400-e29b-41d4-a716-446655440000",
    "email": "john.doe@example.com",
    "access_token": "eyJhbGciOiJIUzI1NiIs...",
    "refresh_token": "v1.MjQ1Njc4OTAx...",
    "expires_in": 3600
  },
  "message": "Account created successfully"
}
```

**Side Effects:**
1. Profile record created with `full_name` and `phone`
2. User role `user` assigned in `user_roles`
3. Main account created with `balance = 1000.00 USD`

**Validation Rules:**

| Field | Rule |
|-------|------|
| `email` | Required. Valid email format. Must be unique. |
| `password` | Required. Minimum 6 characters. |
| `full_name` | Required. 1–100 characters. |
| `phone` | Optional. Valid phone format. |

---

#### `POST /api/v1/auth/login`

Authenticate and receive JWT tokens.

**Request Body:**

```json
{
  "email": "john.doe@example.com",
  "password": "Str0ngP@ss!"
}
```

**Response — `200 OK`:**

```json
{
  "status": "success",
  "data": {
    "user_id": "550e8400-e29b-41d4-a716-446655440000",
    "email": "john.doe@example.com",
    "full_name": "John Doe",
    "access_token": "eyJhbGciOiJIUzI1NiIs...",
    "refresh_token": "v1.MjQ1Njc4OTAx...",
    "token_type": "Bearer",
    "expires_in": 3600
  }
}
```

**Error — `401 Unauthorized`:**

```json
{
  "status": "error",
  "error": {
    "code": "AUTH_INVALID_CREDENTIALS",
    "message": "Invalid email or password"
  }
}
```

---

#### `POST /api/v1/auth/logout`

Invalidate the current session.

**Headers:** `Authorization: Bearer <token>`

**Response — `200 OK`:**

```json
{
  "status": "success",
  "message": "Logged out successfully"
}
```

---

#### `POST /api/v1/auth/refresh`

Refresh an expired access token.

**Request Body:**

```json
{
  "refresh_token": "v1.MjQ1Njc4OTAx..."
}
```

**Response — `200 OK`:**

```json
{
  "status": "success",
  "data": {
    "access_token": "eyJhbGciOiJIUzI1NiIs...",
    "refresh_token": "v1.new_refresh_token...",
    "expires_in": 3600
  }
}
```

---

### 4.2 Account Module

#### `GET /api/v1/accounts`

Retrieve all accounts belonging to the authenticated user.

**Response — `200 OK`:**

```json
{
  "status": "success",
  "data": [
    {
      "id": "acc-001",
      "user_id": "550e8400-...",
      "account_type": "main",
      "balance": 850.00,
      "currency": "USD",
      "piggy_goal_id": null,
      "created_at": "2026-03-01T00:00:00.000Z",
      "updated_at": "2026-03-12T10:30:00.000Z"
    },
    {
      "id": "acc-002",
      "user_id": "550e8400-...",
      "account_type": "piggy",
      "balance": 150.00,
      "currency": "USD",
      "piggy_goal_id": "goal-001",
      "created_at": "2026-03-05T00:00:00.000Z",
      "updated_at": "2026-03-12T10:30:00.000Z"
    }
  ]
}
```

---

#### `GET /api/v1/accounts/{id}`

Retrieve a specific account by ID.

**Response — `200 OK`:**

```json
{
  "status": "success",
  "data": {
    "id": "acc-001",
    "user_id": "550e8400-...",
    "account_type": "main",
    "balance": 850.00,
    "currency": "USD",
    "piggy_goal_id": null,
    "created_at": "2026-03-01T00:00:00.000Z",
    "updated_at": "2026-03-12T10:30:00.000Z"
  }
}
```

---

#### `GET /api/v1/accounts/main`

Shortcut to retrieve the authenticated user's main account.

**Response — `200 OK`:**

```json
{
  "status": "success",
  "data": {
    "id": "acc-001",
    "user_id": "550e8400-...",
    "account_type": "main",
    "balance": 850.00,
    "currency": "USD",
    "created_at": "2026-03-01T00:00:00.000Z",
    "updated_at": "2026-03-12T10:30:00.000Z"
  }
}
```

---

### 4.3 Piggy Goal Module

#### `POST /api/v1/piggy-goals`

Create a new saving goal and its associated piggy account.

**Request Body:**

```json
{
  "name": "New Laptop",
  "target_amount": 1500.00,
  "lock_period_days": 90,
  "hide_balance": false
}
```

**Response — `201 Created`:**

```json
{
  "status": "success",
  "data": {
    "goal": {
      "id": "goal-001",
      "user_id": "550e8400-...",
      "name": "New Laptop",
      "target_amount": 1500.00,
      "lock_period_days": 90,
      "hide_balance": false,
      "status": "active",
      "created_at": "2026-03-12T14:30:00.000Z",
      "lock_expires_at": "2026-06-10T14:30:00.000Z",
      "broken_at": null,
      "completed_at": null
    },
    "account": {
      "id": "acc-003",
      "account_type": "piggy",
      "balance": 0.00,
      "currency": "USD",
      "piggy_goal_id": "goal-001"
    }
  },
  "message": "Piggy goal created successfully"
}
```

**Validation Rules:**

| Field | Rule |
|-------|------|
| `name` | Required. 1–100 characters. |
| `target_amount` | Required. Must be > 0. |
| `lock_period_days` | Optional. If set, must be ≥ 1. |
| `hide_balance` | Optional. Default: `false`. |

**Side Effects:**
1. `piggy_goals` record created with `status = 'active'`
2. `accounts` record created with `account_type = 'piggy'`, `balance = 0`, linked via `piggy_goal_id`

---

#### `GET /api/v1/piggy-goals`

List all piggy goals for the authenticated user.

**Query Parameters:**
- `status` — Filter by status: `active`, `completed`, `broken` (optional)

**Response — `200 OK`:**

```json
{
  "status": "success",
  "data": [
    {
      "id": "goal-001",
      "name": "New Laptop",
      "target_amount": 1500.00,
      "current_balance": 450.25,
      "progress_percentage": 30.02,
      "lock_period_days": 90,
      "lock_expires_at": "2026-06-10T14:30:00.000Z",
      "is_locked": true,
      "hide_balance": false,
      "status": "active",
      "created_at": "2026-03-12T14:30:00.000Z"
    }
  ]
}
```

---

#### `GET /api/v1/piggy-goals/{id}`

Get details of a specific piggy goal including account balance, interest history, and lock status.

**Response — `200 OK`:**

```json
{
  "status": "success",
  "data": {
    "id": "goal-001",
    "name": "New Laptop",
    "target_amount": 1500.00,
    "current_balance": 450.25,
    "progress_percentage": 30.02,
    "lock_period_days": 90,
    "lock_expires_at": "2026-06-10T14:30:00.000Z",
    "is_locked": true,
    "hide_balance": false,
    "status": "active",
    "total_interest_earned": 2.45,
    "created_at": "2026-03-12T14:30:00.000Z",
    "account_id": "acc-003"
  }
}
```

---

#### `PATCH /api/v1/piggy-goals/{id}`

Update piggy goal metadata (name, hide_balance only — not target_amount or lock_period).

**Request Body:**

```json
{
  "name": "Gaming Laptop",
  "hide_balance": true
}
```

**Response — `200 OK`:**

```json
{
  "status": "success",
  "data": {
    "id": "goal-001",
    "name": "Gaming Laptop",
    "hide_balance": true,
    "updated_at": "2026-03-12T15:00:00.000Z"
  },
  "message": "Goal updated successfully"
}
```

---

### 4.4 Transfer Module

All transfer operations are **atomic** and processed within a database transaction using `SELECT ... FOR UPDATE` (pessimistic locking).

#### `POST /api/v1/transfers/to-piggy`

Transfer funds from main account to own piggy goal.

**Headers:** `Idempotency-Key: <uuid-v4>`

**Request Body:**

```json
{
  "piggy_goal_id": "goal-001",
  "amount": 100.00
}
```

**Response — `200 OK`:**

```json
{
  "status": "success",
  "data": {
    "transaction_id": "tx-001",
    "from_account_id": "acc-001",
    "to_account_id": "acc-003",
    "amount": 100.00,
    "type": "transfer",
    "description": "Transfer to New Laptop",
    "new_main_balance": 750.00,
    "new_piggy_balance": 250.00,
    "goal_completed": false,
    "completed_at": "2026-03-12T14:35:00.000Z"
  },
  "message": "Transfer successful"
}
```

**Validation Rules:**

| Rule | Description |
|------|-------------|
| `amount > 0` | Transfer amount must be positive |
| `main_balance >= amount` | Insufficient balance check |
| `goal.status == 'active'` | Goal must be active |
| `goal.user_id == auth_user` | Must own the goal |

**Side Effects:**
1. Main account debited by `amount`
2. Piggy account credited by `amount`
3. Transaction record created (`type: transfer`, `status: completed`)
4. Two ledger entries created (debit + credit)
5. Notification sent to user
6. If `new_piggy_balance >= target_amount` → goal status updated to `completed`

---

#### `POST /api/v1/transfers/p2p`

Send funds from main account to another user's main account.

**Headers:** `Idempotency-Key: <uuid-v4>`

**Request Body:**

```json
{
  "recipient_user_id": "660e8400-e29b-41d4-a716-446655440001",
  "amount": 50.00
}
```

**Response — `200 OK`:**

```json
{
  "status": "success",
  "data": {
    "transaction_id": "tx-002",
    "from_account_id": "acc-001",
    "to_account_id": "acc-010",
    "amount": 50.00,
    "type": "p2p",
    "recipient_name": "Jane Smith",
    "description": "P2P to Jane Smith",
    "new_main_balance": 700.00,
    "completed_at": "2026-03-12T14:40:00.000Z"
  },
  "message": "P2P transfer successful"
}
```

**Validation Rules:**

| Rule | Description |
|------|-------------|
| `amount > 0` | Transfer amount must be positive |
| `main_balance >= amount` | Insufficient balance check |
| `recipient_user_id != auth_user` | Cannot transfer to yourself |
| Recipient exists | Recipient must have a main account |

**Side Effects:**
1. Sender main account debited by `amount`
2. Recipient main account credited by `amount`
3. Transaction record created (`type: p2p`)
4. Two ledger entries created
5. Notification sent to **both** sender and recipient

---

#### `POST /api/v1/transfers/contribute`

Contribute funds to another user's active piggy goal.

**Headers:** `Idempotency-Key: <uuid-v4>`

**Request Body:**

```json
{
  "piggy_goal_id": "goal-005",
  "amount": 25.00
}
```

**Response — `200 OK`:**

```json
{
  "status": "success",
  "data": {
    "transaction_id": "tx-003",
    "from_account_id": "acc-001",
    "to_account_id": "acc-020",
    "amount": 25.00,
    "type": "contribution",
    "goal_name": "Birthday Fund",
    "goal_owner": "Jane Smith",
    "description": "Contribution to \"Birthday Fund\"",
    "new_main_balance": 675.00,
    "goal_completed": false,
    "completed_at": "2026-03-12T14:45:00.000Z"
  },
  "message": "Contribution successful"
}
```

**Validation Rules:**

| Rule | Description |
|------|-------------|
| `amount > 0` | Contribution amount must be positive |
| `main_balance >= amount` | Insufficient balance check |
| `goal.status == 'active'` | Goal must be active |

**Side Effects:**
1. Contributor's main account debited
2. Goal owner's piggy account credited
3. Transaction record created (`type: contribution`)
4. Two ledger entries created
5. Notifications sent to **both** contributor and goal owner
6. If goal completed → status updated to `completed`

---

#### `POST /api/v1/transfers/break-piggy`

Break a piggy goal and return funds to main account. Early break incurs a penalty.

**Headers:** `Idempotency-Key: <uuid-v4>`

**Request Body:**

```json
{
  "piggy_goal_id": "goal-001"
}
```

**Response — `200 OK` (with penalty):**

```json
{
  "status": "success",
  "data": {
    "transaction_id": "tx-004",
    "piggy_goal_id": "goal-001",
    "goal_name": "New Laptop",
    "original_balance": 250.00,
    "penalty_percentage": 5.00,
    "penalty_amount": 12.50,
    "return_amount": 237.50,
    "new_main_balance": 912.50,
    "was_early_break": true,
    "completed_at": "2026-03-12T15:00:00.000Z"
  },
  "message": "Piggy broken with early break penalty"
}
```

**Response — `200 OK` (no penalty — lock expired):**

```json
{
  "status": "success",
  "data": {
    "transaction_id": "tx-005",
    "piggy_goal_id": "goal-002",
    "goal_name": "Vacation Fund",
    "original_balance": 500.00,
    "penalty_percentage": 0,
    "penalty_amount": 0,
    "return_amount": 500.00,
    "new_main_balance": 1350.00,
    "was_early_break": false,
    "completed_at": "2026-03-12T15:05:00.000Z"
  },
  "message": "Piggy broken successfully"
}
```

**Business Logic:**

```
lock_expiry = goal.created_at + (lock_period_days × 86400000 ms)
is_early_break = NOW() < lock_expiry

IF is_early_break:
    penalty_pct = system_settings['early_break_penalty_pct'] (default: 5%)
    penalty = ROUND(balance × penalty_pct / 100, 2)
    return_amount = balance - penalty
ELSE:
    penalty = 0
    return_amount = balance
```

**Side Effects:**
1. Piggy account balance set to `0`
2. Main account credited by `return_amount`
3. Break transaction created (`type: break`)
4. If penalty > 0: fee transaction created (`type: fee`)
5. Ledger entries for all transactions
6. Goal status updated to `broken`, `broken_at` set
7. Notification sent (warning type if penalty applied)

---

### 4.5 Interest Module

#### `POST /api/v1/interest/calculate`

**Access:** Admin only or scheduled CRON job.

Calculates and applies daily compound interest to all active piggy accounts with balance > 0.

**Response — `200 OK`:**

```json
{
  "status": "success",
  "data": {
    "processed": 42,
    "total_interest_paid": 3.67,
    "annual_rate": 3.50,
    "daily_rate": 0.00009589,
    "calculation_date": "2026-03-12",
    "skipped_already_paid": 5,
    "skipped_inactive": 3
  },
  "message": "Interest calculated for 42 accounts"
}
```

**Business Logic:**

```
annual_rate  = system_settings['interest_rate_piggy'] (default: 3.50%)
daily_rate   = annual_rate / 100 / 365
interest     = ROUND(balance × daily_rate, 2)
new_balance  = balance + interest
```

**Per-Account Side Effects:**
1. `interest_payments` record created
2. Transaction created (`type: interest`)
3. Ledger entry (credit to piggy account)
4. Account balance updated

**Idempotency:**
- Interest key format: `interest-{account_id}-{YYYY-MM-DD}`
- Duplicate key → skipped (already paid today)

---

#### `GET /api/v1/interest/history/{piggyGoalId}`

Retrieve interest payment history for a specific piggy goal.

**Query Parameters:**
- `page`, `size` — Pagination

**Response — `200 OK`:**

```json
{
  "status": "success",
  "data": [
    {
      "id": "ip-001",
      "account_id": "acc-003",
      "piggy_goal_id": "goal-001",
      "principal": 450.00,
      "rate": 0.00009589,
      "amount": 0.04,
      "created_at": "2026-03-12T00:00:00.000Z"
    },
    {
      "id": "ip-002",
      "account_id": "acc-003",
      "piggy_goal_id": "goal-001",
      "principal": 450.04,
      "rate": 0.00009589,
      "amount": 0.04,
      "created_at": "2026-03-11T00:00:00.000Z"
    }
  ],
  "pagination": {
    "page": 0,
    "size": 20,
    "total_elements": 30
  }
}
```

---

### 4.6 Notification Module

#### `GET /api/v1/notifications`

Retrieve all notifications for the authenticated user.

**Query Parameters:**
- `read` — Filter: `true`, `false`, or omit for all
- `page`, `size` — Pagination

**Response — `200 OK`:**

```json
{
  "status": "success",
  "data": [
    {
      "id": "ntf-001",
      "title": "P2P Received",
      "message": "$50.00 received from John Doe",
      "type": "success",
      "read": false,
      "created_at": "2026-03-12T14:40:00.000Z"
    }
  ],
  "pagination": {
    "page": 0,
    "size": 20,
    "total_elements": 15
  },
  "meta": {
    "unread_count": 3
  }
}
```

---

#### `PATCH /api/v1/notifications/{id}/read`

Mark a single notification as read.

**Response — `200 OK`:**

```json
{
  "status": "success",
  "message": "Notification marked as read"
}
```

---

#### `PATCH /api/v1/notifications/read-all`

Mark all unread notifications as read.

**Response — `200 OK`:**

```json
{
  "status": "success",
  "data": {
    "updated_count": 3
  },
  "message": "All notifications marked as read"
}
```

---

### 4.7 Admin Module

All admin endpoints require `admin` role. Validated via `has_role(user_id, 'admin')`.

#### `GET /api/v1/admin/users`

List all user profiles with account summaries.

**Response — `200 OK`:**

```json
{
  "status": "success",
  "data": [
    {
      "user_id": "550e8400-...",
      "email": "john@example.com",
      "full_name": "John Doe",
      "phone": "+855123456789",
      "main_balance": 850.00,
      "active_goals": 2,
      "total_transactions": 15,
      "created_at": "2026-03-01T00:00:00.000Z",
      "last_activity": "2026-03-12T10:00:00.000Z"
    }
  ],
  "pagination": { ... }
}
```

---

#### `GET /api/v1/admin/transactions`

List all transactions across the system.

**Query Parameters:**
- `type` — Filter: `transfer`, `p2p`, `contribution`, `interest`, `fee`, `break`, `reversal`
- `status` — Filter: `completed`, `reversed`, `pending`, `failed`
- `from_date`, `to_date` — Date range filter
- `page`, `size`, `sort`, `direction` — Pagination & sorting

**Response — `200 OK`:**

```json
{
  "status": "success",
  "data": [
    {
      "id": "tx-001",
      "from_account_id": "acc-001",
      "from_user_name": "John Doe",
      "to_account_id": "acc-003",
      "to_user_name": "John Doe",
      "amount": 100.00,
      "type": "transfer",
      "status": "completed",
      "description": "Transfer to New Laptop",
      "idempotency_key": "550e8400-...",
      "created_at": "2026-03-12T14:35:00.000Z",
      "completed_at": "2026-03-12T14:35:00.000Z"
    }
  ],
  "pagination": { ... }
}
```

---

#### `POST /api/v1/admin/transactions/{id}/reverse`

Reverse a completed transaction. Creates a counter-transaction and adjusts balances.

**Request Body:**

```json
{
  "reason": "Customer dispute — incorrect amount"
}
```

**Response — `200 OK`:**

```json
{
  "status": "success",
  "data": {
    "original_transaction_id": "tx-001",
    "reversal_transaction_id": "tx-006",
    "amount": 100.00,
    "type": "reversal",
    "reason": "Customer dispute — incorrect amount",
    "completed_at": "2026-03-12T16:00:00.000Z"
  },
  "message": "Transaction reversed successfully"
}
```

**Side Effects:**
1. Original transaction status → `reversed`
2. New reversal transaction created (swapped from/to)
3. Account balances restored
4. Ledger entries for reversal recorded

---

#### `GET /api/v1/admin/ledger`

View the immutable double-entry ledger.

**Query Parameters:**
- `account_id` — Filter by account
- `transaction_id` — Filter by transaction
- `from_date`, `to_date` — Date range
- `page`, `size` — Pagination

**Response — `200 OK`:**

```json
{
  "status": "success",
  "data": [
    {
      "id": "le-001",
      "transaction_id": "tx-001",
      "account_id": "acc-001",
      "debit": 100.00,
      "credit": 0.00,
      "created_at": "2026-03-12T14:35:00.000Z"
    },
    {
      "id": "le-002",
      "transaction_id": "tx-001",
      "account_id": "acc-003",
      "debit": 0.00,
      "credit": 100.00,
      "created_at": "2026-03-12T14:35:00.000Z"
    }
  ]
}
```

---

#### `GET /api/v1/admin/settings`

Retrieve all system settings.

**Response — `200 OK`:**

```json
{
  "status": "success",
  "data": [
    { "key": "interest_rate_base", "value": "0.10", "updated_at": "2026-03-01T00:00:00.000Z" },
    { "key": "interest_rate_piggy", "value": "3.50", "updated_at": "2026-03-01T00:00:00.000Z" },
    { "key": "early_break_penalty_pct", "value": "5.00", "updated_at": "2026-03-01T00:00:00.000Z" },
    { "key": "early_closure_fee", "value": "5.00", "updated_at": "2026-03-01T00:00:00.000Z" },
    { "key": "dormancy_fee", "value": "6.00", "updated_at": "2026-03-01T00:00:00.000Z" }
  ]
}
```

---

#### `PUT /api/v1/admin/settings/{key}`

Update a system setting.

**Request Body:**

```json
{
  "value": "4.00"
}
```

**Response — `200 OK`:**

```json
{
  "status": "success",
  "data": {
    "key": "interest_rate_piggy",
    "value": "4.00",
    "updated_at": "2026-03-12T16:30:00.000Z"
  },
  "message": "Setting updated successfully"
}
```

---

### 4.8 QR Payment Module

#### `POST /api/v1/qr/generate`

Generate a QR code payload for receiving payments.

**Request Body:**

```json
{
  "type": "main",
  "piggy_goal_id": null,
  "expiry_minutes": 30
}
```

**Response — `200 OK`:**

```json
{
  "status": "success",
  "data": {
    "payload": "0002010102122653000213550e8400-e29...630482AF",
    "type": "main",
    "user_id": "550e8400-...",
    "user_name": "John Doe",
    "expires_at": "2026-03-12T15:30:00.000Z"
  }
}
```

---

#### `POST /api/v1/qr/decode`

Decode a scanned QR payload and return payment details.

**Request Body:**

```json
{
  "payload": "0002010102122653000213550e8400-e29...630482AF"
}
```

**Response — `200 OK`:**

```json
{
  "status": "success",
  "data": {
    "type": "main",
    "recipient_user_id": "550e8400-...",
    "recipient_name": "John Doe",
    "piggy_goal_id": null,
    "piggy_goal_name": null,
    "is_expired": false,
    "expires_at": "2026-03-12T15:30:00.000Z"
  }
}
```

After decoding, the client submits the payment via `POST /api/v1/transfers/p2p` or `POST /api/v1/transfers/contribute`.

---

### 4.9 Profile Module

#### `GET /api/v1/profiles/me`

Get the authenticated user's profile.

**Response — `200 OK`:**

```json
{
  "status": "success",
  "data": {
    "id": "profile-001",
    "user_id": "550e8400-...",
    "email": "john@example.com",
    "full_name": "John Doe",
    "phone": "+855123456789",
    "pin_configured": true,
    "created_at": "2026-03-01T00:00:00.000Z",
    "last_activity": "2026-03-12T10:00:00.000Z"
  }
}
```

---

#### `PATCH /api/v1/profiles/me`

Update profile fields.

**Request Body:**

```json
{
  "full_name": "John M. Doe",
  "phone": "+855987654321"
}
```

**Response — `200 OK`:**

```json
{
  "status": "success",
  "data": {
    "full_name": "John M. Doe",
    "phone": "+855987654321",
    "updated_at": "2026-03-12T16:00:00.000Z"
  },
  "message": "Profile updated successfully"
}
```

---

#### `POST /api/v1/profiles/search`

Search for a user by phone number (used for P2P recipient lookup).

**Request Body:**

```json
{
  "phone": "+855123456789"
}
```

**Response — `200 OK`:**

```json
{
  "status": "success",
  "data": {
    "user_id": "660e8400-...",
    "full_name": "Jane Smith",
    "phone": "+855123456789"
  }
}
```

**Response — `404 Not Found`:**

```json
{
  "status": "error",
  "error": {
    "code": "USER_NOT_FOUND",
    "message": "No user found with the provided phone number"
  }
}
```

---

#### `POST /api/v1/profiles/pin/setup`

Set or update the user's PIN code.

**Request Body:**

```json
{
  "pin": "123456"
}
```

**Response — `200 OK`:**

```json
{
  "status": "success",
  "message": "PIN configured successfully"
}
```

---

#### `POST /api/v1/profiles/pin/verify`

Verify the user's PIN (used before financial operations).

**Request Body:**

```json
{
  "pin": "123456"
}
```

**Response — `200 OK`:**

```json
{
  "status": "success",
  "data": {
    "valid": true
  }
}
```

---

## 5. Data Transfer Objects (DTOs)

### 5.1 Request DTOs

```
┌─────────────────────────────────────────────────────────────┐
│                     REQUEST DTOs                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  SignUpRequest                                              │
│  ├── email: String              @NotBlank @Email            │
│  ├── password: String           @NotBlank @Size(min=6)      │
│  ├── full_name: String          @NotBlank @Size(max=100)    │
│  └── phone: String              @Pattern(phone)             │
│                                                             │
│  LoginRequest                                               │
│  ├── email: String              @NotBlank @Email            │
│  └── password: String           @NotBlank                   │
│                                                             │
│  CreatePiggyGoalRequest                                     │
│  ├── name: String               @NotBlank @Size(max=100)    │
│  ├── target_amount: BigDecimal  @Positive                   │
│  ├── lock_period_days: Integer  @Min(1) (nullable)          │
│  └── hide_balance: Boolean      (default: false)            │
│                                                             │
│  TransferToPiggyRequest                                     │
│  ├── piggy_goal_id: UUID        @NotNull                    │
│  └── amount: BigDecimal         @Positive                   │
│                                                             │
│  P2PTransferRequest                                         │
│  ├── recipient_user_id: UUID    @NotNull                    │
│  └── amount: BigDecimal         @Positive                   │
│                                                             │
│  ContributionRequest                                        │
│  ├── piggy_goal_id: UUID        @NotNull                    │
│  └── amount: BigDecimal         @Positive                   │
│                                                             │
│  BreakPiggyRequest                                          │
│  └── piggy_goal_id: UUID        @NotNull                    │
│                                                             │
│  QRGenerateRequest                                          │
│  ├── type: String               @Pattern("main|piggy")      │
│  ├── piggy_goal_id: UUID        (nullable)                  │
│  └── expiry_minutes: Integer    @Min(1) (default: 30)       │
│                                                             │
│  QRDecodeRequest                                            │
│  └── payload: String            @NotBlank                   │
│                                                             │
│  PhoneSearchRequest                                         │
│  └── phone: String              @NotBlank                   │
│                                                             │
│  PinSetupRequest                                            │
│  └── pin: String                @Size(min=4, max=6)         │
│                                                             │
│  PinVerifyRequest                                           │
│  └── pin: String                @NotBlank                   │
│                                                             │
│  UpdateSettingRequest                                       │
│  └── value: String              @NotBlank                   │
│                                                             │
│  ReverseTransactionRequest                                  │
│  └── reason: String             @NotBlank                   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 5.2 Response DTOs

```
┌─────────────────────────────────────────────────────────────┐
│                     RESPONSE DTOs                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ApiResponse<T>                 (Standard wrapper)          │
│  ├── status: String             "success" | "error"         │
│  ├── data: T                    (nullable)                  │
│  ├── message: String            (nullable)                  │
│  ├── error: ErrorDetail         (nullable, on error only)   │
│  └── pagination: Pagination     (nullable, on lists only)   │
│                                                             │
│  ErrorDetail                                                │
│  ├── code: String               e.g. "INSUFFICIENT_BALANCE" │
│  ├── message: String            Human-readable message      │
│  ├── field: String              (nullable) Failing field    │
│  └── timestamp: String          ISO 8601                    │
│                                                             │
│  Pagination                                                 │
│  ├── page: Integer                                          │
│  ├── size: Integer                                          │
│  ├── total_elements: Long                                   │
│  ├── total_pages: Integer                                   │
│  ├── has_next: Boolean                                      │
│  └── has_previous: Boolean                                  │
│                                                             │
│  AuthResponse                                               │
│  ├── user_id: UUID                                          │
│  ├── email: String                                          │
│  ├── full_name: String                                      │
│  ├── access_token: String                                   │
│  ├── refresh_token: String                                  │
│  ├── token_type: String         "Bearer"                    │
│  └── expires_in: Integer        seconds                     │
│                                                             │
│  AccountResponse                                            │
│  ├── id: UUID                                               │
│  ├── user_id: UUID                                          │
│  ├── account_type: String       "main" | "piggy"            │
│  ├── balance: BigDecimal                                    │
│  ├── currency: String           ISO 4217                    │
│  ├── piggy_goal_id: UUID        (nullable)                  │
│  ├── created_at: Instant                                    │
│  └── updated_at: Instant                                    │
│                                                             │
│  PiggyGoalResponse                                          │
│  ├── id: UUID                                               │
│  ├── name: String                                           │
│  ├── target_amount: BigDecimal                              │
│  ├── current_balance: BigDecimal                            │
│  ├── progress_percentage: Double                            │
│  ├── lock_period_days: Integer  (nullable)                  │
│  ├── lock_expires_at: Instant   (nullable)                  │
│  ├── is_locked: Boolean                                     │
│  ├── hide_balance: Boolean                                  │
│  ├── status: String             "active"|"completed"|"broken│
│  ├── total_interest_earned: BigDecimal                      │
│  ├── account_id: UUID                                       │
│  ├── created_at: Instant                                    │
│  ├── completed_at: Instant      (nullable)                  │
│  └── broken_at: Instant         (nullable)                  │
│                                                             │
│  TransactionResponse                                        │
│  ├── id: UUID                                               │
│  ├── from_account_id: UUID      (nullable)                  │
│  ├── to_account_id: UUID        (nullable)                  │
│  ├── amount: BigDecimal                                     │
│  ├── type: String               (see Transaction Types)     │
│  ├── status: String             (see Transaction Statuses)  │
│  ├── description: String        (nullable)                  │
│  ├── idempotency_key: String    (nullable)                  │
│  ├── created_at: Instant                                    │
│  └── completed_at: Instant      (nullable)                  │
│                                                             │
│  TransferResultResponse                                     │
│  ├── transaction_id: UUID                                   │
│  ├── from_account_id: UUID                                  │
│  ├── to_account_id: UUID                                    │
│  ├── amount: BigDecimal                                     │
│  ├── type: String                                           │
│  ├── description: String                                    │
│  ├── new_main_balance: BigDecimal                           │
│  ├── new_piggy_balance: BigDecimal (nullable)               │
│  ├── goal_completed: Boolean    (nullable)                  │
│  └── completed_at: Instant                                  │
│                                                             │
│  BreakResultResponse                                        │
│  ├── transaction_id: UUID                                   │
│  ├── piggy_goal_id: UUID                                    │
│  ├── goal_name: String                                      │
│  ├── original_balance: BigDecimal                           │
│  ├── penalty_percentage: Double                             │
│  ├── penalty_amount: BigDecimal                             │
│  ├── return_amount: BigDecimal                              │
│  ├── new_main_balance: BigDecimal                           │
│  ├── was_early_break: Boolean                               │
│  └── completed_at: Instant                                  │
│                                                             │
│  NotificationResponse                                       │
│  ├── id: UUID                                               │
│  ├── title: String                                          │
│  ├── message: String                                        │
│  ├── type: String               "info"|"success"|"warning"  │
│  ├── read: Boolean                                          │
│  └── created_at: Instant                                    │
│                                                             │
│  InterestPaymentResponse                                    │
│  ├── id: UUID                                               │
│  ├── account_id: UUID                                       │
│  ├── piggy_goal_id: UUID                                    │
│  ├── principal: BigDecimal                                  │
│  ├── rate: Double                                           │
│  ├── amount: BigDecimal                                     │
│  └── created_at: Instant                                    │
│                                                             │
│  ProfileResponse                                            │
│  ├── id: UUID                                               │
│  ├── user_id: UUID                                          │
│  ├── email: String                                          │
│  ├── full_name: String                                      │
│  ├── phone: String                                          │
│  ├── pin_configured: Boolean                                │
│  ├── created_at: Instant                                    │
│  └── last_activity: Instant                                 │
│                                                             │
│  SystemSettingResponse                                      │
│  ├── key: String                                            │
│  ├── value: String                                          │
│  └── updated_at: Instant                                    │
│                                                             │
│  LedgerEntryResponse                                        │
│  ├── id: UUID                                               │
│  ├── transaction_id: UUID                                   │
│  ├── account_id: UUID                                       │
│  ├── debit: BigDecimal                                      │
│  ├── credit: BigDecimal                                     │
│  └── created_at: Instant                                    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 6. Data Model

### 6.1 Entity Relationship Diagram

```
                          ┌──────────────┐
                          │    users     │ (auth-managed)
                          │──────────────│
                          │ id (PK)      │
                          │ email        │
                          └──────┬───────┘
                                 │
              ┌──────────────────┼──────────────────┐
              │                  │                   │
     ┌────────▼────────┐ ┌──────▼───────┐  ┌───────▼────────┐
     │   profiles      │ │  user_roles  │  │   accounts     │
     │─────────────────│ │──────────────│  │────────────────│
     │ id (PK)         │ │ id (PK)      │  │ id (PK)        │
     │ user_id (FK,UQ) │ │ user_id (FK) │  │ user_id (FK)   │
     │ full_name       │ │ role (ENUM)  │  │ account_type   │
     │ phone           │ │              │  │ balance        │
     │ pin_hash        │ └──────────────┘  │ currency       │
     │ last_activity   │                   │ piggy_goal_id  │──┐
     └─────────────────┘                   └────────┬───────┘  │
                                                    │          │
                              ┌──────────────────────┤          │
                              │                      │          │
                    ┌─────────▼──────────┐  ┌───────▼────────┐ │
                    │   transactions     │  │ interest_      │ │
                    │───────────────────│  │ payments       │ │
                    │ id (PK)           │  │────────────────│ │
                    │ from_account_id   │  │ id (PK)        │ │
                    │ to_account_id     │  │ account_id(FK) │ │
                    │ amount            │  │ piggy_goal_id  │ │
                    │ type              │  │ principal      │ │
                    │ status            │  │ rate           │ │
                    │ description       │  │ amount         │ │
                    │ idempotency_key   │  └────────────────┘ │
                    └─────────┬─────────┘                     │
                              │                               │
                    ┌─────────▼──────────┐         ┌─────────▼─────────┐
                    │  ledger_entries    │         │   piggy_goals     │
                    │───────────────────│         │──────────────────│
                    │ id (PK)           │         │ id (PK)          │
                    │ transaction_id(FK)│         │ user_id (FK)     │
                    │ account_id (FK)   │         │ name             │
                    │ debit             │         │ target_amount    │
                    │ credit            │         │ lock_period_days │
                    └───────────────────┘         │ hide_balance     │
                                                  │ status           │
     ┌──────────────────┐  ┌──────────────────┐   │ broken_at        │
     │  notifications   │  │ system_settings  │   │ completed_at     │
     │─────────────────│  │─────────────────│   └──────────────────┘
     │ id (PK)          │  │ id (PK)         │
     │ user_id (FK)     │  │ key (UQ)        │
     │ title            │  │ value           │
     │ message          │  └─────────────────┘
     │ type             │
     │ read             │
     └─────────────────┘
```

### 6.2 Table Summary

| Table | Description | Key Constraints |
|-------|-------------|-----------------|
| `profiles` | User profile data (name, phone, PIN hash) | `user_id` UNIQUE, FK → `auth.users` |
| `user_roles` | RBAC roles (separate table for security) | `(user_id, role)` UNIQUE |
| `accounts` | Main & piggy wallets | `balance DEFAULT 0`, `account_type IN ('main','piggy')` |
| `piggy_goals` | Saving goal metadata | `status IN ('active','broken','completed')` |
| `transactions` | All financial movements | `type` + `status` enumerations, `idempotency_key` UNIQUE |
| `ledger_entries` | Immutable double-entry records | `(debit > 0 AND credit = 0) OR (credit > 0 AND debit = 0)` |
| `interest_payments` | Daily interest accrual log | FK → `accounts`, `piggy_goals` |
| `notifications` | User notification inbox | `read DEFAULT false` |
| `system_settings` | Admin-configurable system constants | `key` UNIQUE |

### 6.3 Enumerations

**Transaction Types:**

| Value | Description |
|-------|-------------|
| `transfer` | Main → own piggy |
| `p2p` | Main → another user's main |
| `contribution` | Main → another user's piggy |
| `interest` | System → piggy (daily compound) |
| `fee` | Piggy → system (early break penalty) |
| `break` | Piggy → own main |
| `deposit` | System → main (initial seed) |
| `reversal` | Counter-transaction by admin |

**Transaction Statuses:**

| Value | Description |
|-------|-------------|
| `pending` | Awaiting processing |
| `completed` | Successfully processed |
| `failed` | Processing failed |
| `reversed` | Reversed by admin |

**Goal Statuses:**

| Value | Description |
|-------|-------------|
| `active` | Accepting transfers and earning interest |
| `completed` | Balance reached target amount |
| `broken` | Manually broken by user |

**User Roles:**

| Value | Description |
|-------|-------------|
| `user` | Standard user (default on registration) |
| `admin` | System administrator |

**Notification Types:**

| Value | Description |
|-------|-------------|
| `info` | Informational (sent, broken without penalty) |
| `success` | Positive outcome (received, goal completed) |
| `warning` | Caution (early break penalty applied) |

### 6.4 Default System Settings

| Key | Default Value | Description |
|-----|---------------|-------------|
| `interest_rate_base` | `0.10` | Base interest rate (%) — reserved |
| `interest_rate_piggy` | `3.50` | Annual piggy interest rate (%) |
| `early_break_penalty_pct` | `5.00` | Early break penalty percentage |
| `early_closure_fee` | `5.00` | Early closure flat fee ($) — reserved |
| `dormancy_fee` | `6.00` | Annual dormancy fee ($) — reserved |

---

## 7. Security Architecture

### 7.1 Authentication

| Layer | Implementation |
|-------|----------------|
| Password Storage | BCrypt (cost factor 12) |
| Token Format | JWT (HS256 or RS256) |
| Token Lifetime | Access: 1 hour, Refresh: 7 days |
| PIN Storage | SHA-256 hash (never stored in plaintext) |

### 7.2 Authorization Matrix

| Resource | `user` | `admin` |
|----------|--------|---------|
| Own profile | Read, Update | Read, Update all |
| Own accounts | Read | Read all |
| Own transactions | Read | Read all, Reverse |
| Own piggy goals | CRUD | Read all |
| Own notifications | Read, Update | Read all |
| System settings | Read | Read, Update |
| User roles | Read own | CRUD all |
| Ledger entries | Read own | Read all |

### 7.3 Financial Security

| Control | Description |
|---------|-------------|
| Pessimistic Locking | `SELECT ... FOR UPDATE` on account rows during transfers |
| Double-Entry Ledger | Every debit has a matching credit — `SUM(debits) = SUM(credits)` |
| Idempotency Keys | UUID per mutation — duplicate keys rejected |
| Balance Validation | Server-side check: `balance >= amount` before every debit |
| Atomic Transactions | All balance mutations wrapped in database transactions |
| Immutable Ledger | Ledger entries are INSERT-only — never updated or deleted |

### 7.4 Row-Level Security (RLS)

All tables have RLS enabled. Key policies:

| Table | Policy | Rule |
|-------|--------|------|
| `profiles` | Owner access | `user_id = auth.uid()` |
| `accounts` | Owner access | `user_id = auth.uid()` |
| `piggy_goals` | Owner + public active | `user_id = auth.uid() OR status = 'active'` |
| `transactions` | Participant access | `from_account OR to_account belongs to user` |
| `ledger_entries` | Account owner | `account_id IN (user's accounts)` |
| `notifications` | Owner access | `user_id = auth.uid()` |
| `system_settings` | Public read | `SELECT` for all, `UPDATE` for admin |
| `user_roles` | Owner read | `user_id = auth.uid()`, admin full access |

---

## 8. Error Handling

### 8.1 Standard Error Response

```json
{
  "status": "error",
  "error": {
    "code": "INSUFFICIENT_BALANCE",
    "message": "Your main account balance ($50.00) is insufficient for this transfer ($100.00)",
    "field": null,
    "timestamp": "2026-03-12T14:35:00.000Z"
  }
}
```

### 8.2 Validation Error Response (400)

```json
{
  "status": "error",
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Request validation failed",
    "details": [
      { "field": "amount", "message": "must be greater than 0" },
      { "field": "piggy_goal_id", "message": "must not be null" }
    ],
    "timestamp": "2026-03-12T14:35:00.000Z"
  }
}
```

### 8.3 HTTP Status Codes

| Code | Usage |
|------|-------|
| `200` | Successful operation |
| `201` | Resource created |
| `400` | Validation error or bad request |
| `401` | Missing or invalid authentication |
| `403` | Insufficient permissions (wrong role) |
| `404` | Resource not found |
| `409` | Conflict (duplicate idempotency key, duplicate email) |
| `422` | Business rule violation (insufficient balance, inactive goal) |
| `429` | Rate limit exceeded |
| `500` | Internal server error |

### 8.4 Error Code Catalog

| Code | HTTP | Description |
|------|------|-------------|
| `AUTH_INVALID_CREDENTIALS` | 401 | Wrong email or password |
| `AUTH_TOKEN_EXPIRED` | 401 | JWT has expired |
| `AUTH_UNAUTHORIZED` | 401 | Missing Authorization header |
| `AUTH_FORBIDDEN` | 403 | User lacks required role |
| `VALIDATION_ERROR` | 400 | Request body validation failed |
| `USER_NOT_FOUND` | 404 | No user with given identifier |
| `ACCOUNT_NOT_FOUND` | 404 | Account does not exist |
| `GOAL_NOT_FOUND` | 404 | Piggy goal does not exist |
| `GOAL_NOT_ACTIVE` | 422 | Goal is not in `active` status |
| `INSUFFICIENT_BALANCE` | 422 | Sender balance < transfer amount |
| `SELF_TRANSFER` | 422 | Cannot P2P transfer to yourself |
| `NO_BALANCE_TO_BREAK` | 422 | Piggy account has zero balance |
| `DUPLICATE_IDEMPOTENCY_KEY` | 409 | Idempotency key already processed |
| `DUPLICATE_EMAIL` | 409 | Email already registered |
| `INTEREST_ALREADY_PAID` | 409 | Interest already calculated today |
| `TRANSACTION_ALREADY_REVERSED` | 409 | Transaction was already reversed |
| `RATE_LIMIT_EXCEEDED` | 429 | Too many requests |
| `INTERNAL_ERROR` | 500 | Unexpected server error |

---

## 9. Idempotency & Concurrency

### 9.1 Idempotency Keys

All financial mutation endpoints require an `Idempotency-Key` header (UUID v4).

| Endpoint | Key Format | Behavior on Duplicate |
|----------|------------|----------------------|
| `POST /transfers/to-piggy` | Client-generated UUID | Return original result, no re-execution |
| `POST /transfers/p2p` | Client-generated UUID | Return original result, no re-execution |
| `POST /transfers/contribute` | Client-generated UUID | Return original result, no re-execution |
| `POST /transfers/break-piggy` | Client-generated UUID | Return original result, no re-execution |
| `POST /interest/calculate` | `interest-{accountId}-{YYYY-MM-DD}` | System-generated, auto-dedup |

### 9.2 Concurrency Control

```
┌──────────────────────────────────────────────────────────────┐
│                  Transfer Flow (Pessimistic Lock)            │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  BEGIN TRANSACTION                                           │
│  │                                                           │
│  ├── SELECT * FROM accounts WHERE id = ? FOR UPDATE          │
│  │   (Lock sender account row)                               │
│  │                                                           │
│  ├── SELECT * FROM accounts WHERE id = ? FOR UPDATE          │
│  │   (Lock receiver account row)                             │
│  │                                                           │
│  ├── VALIDATE balance >= amount                              │
│  │                                                           │
│  ├── UPDATE accounts SET balance = balance - amount          │
│  │   WHERE id = sender_id                                    │
│  │                                                           │
│  ├── UPDATE accounts SET balance = balance + amount          │
│  │   WHERE id = receiver_id                                  │
│  │                                                           │
│  ├── INSERT INTO transactions (...)                          │
│  │                                                           │
│  ├── INSERT INTO ledger_entries (...) -- debit               │
│  ├── INSERT INTO ledger_entries (...) -- credit              │
│  │                                                           │
│  COMMIT                                                      │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### 9.3 Lock Ordering

To prevent deadlocks, always lock accounts in **ascending UUID order**:

```
account_ids = sort([sender_id, receiver_id])
FOR EACH id IN account_ids:
    SELECT * FROM accounts WHERE id = ? FOR UPDATE
```

---

## 10. QR Code Payload Specification

### 10.1 TLV (Tag-Length-Value) Format

QR payloads follow a simplified EMV/KHQR-inspired TLV encoding.

**Structure:**

```
[Tag: 2 chars][Length: 2 chars][Value: N chars]
```

### 10.2 Top-Level Tags

| Tag | Name | Description |
|-----|------|-------------|
| `00` | Format Indicator | Always `01` |
| `01` | Point of Initiation | `12` (dynamic QR) |
| `26` | Merchant Account Info | Nested TLV with payment details |
| `58` | Country Code | `KH` (Cambodia) |
| `62` | Additional Data | Optional nested TLV |
| `63` | CRC | CRC-16/CCITT-FALSE checksum (4 hex chars) |

### 10.3 Merchant Account (Tag 26) Sub-Tags

| Sub-Tag | Name | Description |
|---------|------|-------------|
| `00` | Type | `main` or `piggy` |
| `02` | User ID | Recipient's user UUID |
| `03` | Account ID | Recipient's account UUID (optional) |
| `04` | Piggy Goal ID | For contribution QR (optional) |
| `05` | Display Name | Recipient's display name (optional) |

### 10.4 Additional Data (Tag 62) Sub-Tags

| Sub-Tag | Name | Description |
|---------|------|-------------|
| `05` | Expiry | Unix timestamp (seconds) |

### 10.5 CRC Calculation

```
1. Build payload string (all tags EXCEPT 63)
2. Append "6304" (tag 63, length 04)
3. Compute CRC-16/CCITT-FALSE over the string
4. Append 4-character uppercase hex result
```

### 10.6 Example Payload

```
000201                           → Format: 01
010212                           → POI: 12 (dynamic)
2653                             → Merchant Info (53 chars)
  0004main                       → Type: main
  0236550e8400-e29b-41d4-a716-44 → User ID
  0508John Doe                   → Name: John Doe
5802KH                           → Country: KH
6216                             → Additional Data (16 chars)
  0510 1741784400                → Expiry timestamp
6304 82AF                        → CRC checksum
```

---

## 11. Business Rules & Invariants

### 11.1 Financial Invariants

| # | Rule | Enforcement |
|---|------|-------------|
| 1 | `SUM(all debits) = SUM(all credits)` | Double-entry ledger |
| 2 | `account.balance >= 0` always | Server-side validation before debit |
| 3 | Every transaction has ≥ 1 ledger entry | Application logic |
| 4 | Ledger entries are immutable | No UPDATE/DELETE on `ledger_entries` |
| 5 | Completed transactions cannot be modified | Only admin reversal (creates new tx) |

### 11.2 Registration Side Effects

On user signup, the following are **atomically** created:
1. `profiles` row (full_name, phone)
2. `user_roles` row (role = `user`)
3. `accounts` row (type = `main`, balance = `1000.00`, currency = `USD`)

### 11.3 Interest Calculation Rules

1. Runs once daily (scheduled CRON or manual trigger)
2. Only applies to `piggy` accounts with `balance > 0`
3. Only applies to goals with `status = 'active'`
4. Idempotent — skips accounts already paid on the same calendar day
5. Formula: `interest = ROUND(balance × (annual_rate / 100 / 365), 2)`

### 11.4 Early Break Penalty Rules

1. `lock_expiry = created_at + (lock_period_days × 86,400,000 ms)`
2. If current time < lock_expiry → **early break**
3. Penalty = `ROUND(balance × early_break_penalty_pct / 100, 2)`
4. Return amount = `balance - penalty`
5. Penalty recorded as separate `fee` transaction
6. If no lock period → no penalty

### 11.5 Goal Completion Rules

1. After any credit to a piggy account, check: `new_balance >= target_amount`
2. If true → update goal `status = 'completed'`, set `completed_at`
3. Completed goals stop earning interest

---

## 12. Appendix

### 12.1 Technology Stack

| Component | Technology |
|-----------|------------|
| Backend Framework | Spring Boot 3.x |
| Language | Java 17 |
| Database | PostgreSQL 15 |
| ORM | Spring Data JPA / Hibernate |
| Authentication | Spring Security + JWT |
| Validation | Jakarta Bean Validation |
| API Documentation | SpringDoc OpenAPI (Swagger UI) |
| Build Tool | Maven / Gradle |
| Testing | JUnit 5 + Mockito + Testcontainers |

### 12.2 Suggested Package Structure

```
com.piggy.saving
├── config/              # Security, CORS, JWT configuration
├── controller/          # REST controllers (one per module)
├── dto/
│   ├── request/         # Request DTOs with validation annotations
│   └── response/        # Response DTOs
├── entity/              # JPA entities
├── enums/               # TransactionType, GoalStatus, etc.
├── exception/           # Custom exceptions + GlobalExceptionHandler
├── mapper/              # Entity ↔ DTO mappers (MapStruct)
├── repository/          # Spring Data JPA repositories
├── security/            # JWT filter, UserDetailsService
├── service/             # Business logic layer
│   ├── AccountService
│   ├── TransferService
│   ├── InterestService
│   ├── PiggyGoalService
│   └── NotificationService
├── scheduler/           # @Scheduled interest calculation
└── util/                # QR TLV encoder/decoder, helpers
```

### 12.3 API Base URL

```
Production:  https://api.piggy-saving.com/api/v1
Staging:     https://staging-api.piggy-saving.com/api/v1
Development: http://localhost:8080/api/v1
```

---

> **Document Version History**
>
> | Version | Date | Author | Changes |
> |---------|------|--------|---------|
> | 2.0.0 | 2026-03-12 | Engineering Team | Full rewrite — Spring Boot architecture, comprehensive DTOs |
> | 1.0.0 | 2026-03-11 | Engineering Team | Initial API reference |

---

*© 2026 Piggy Saving System. All rights reserved.*
