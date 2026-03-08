# Piggy Saving Account — API Design

## Base URL
```
https://<project-ref>.supabase.co
```

## Authentication
All endpoints require `Authorization: Bearer <access_token>` header (JWT from Supabase Auth).

---

## 1. Authentication

### POST `/auth/v1/signup`
Register a new user.
```json
// Request
{
  "email": "user@example.com",
  "password": "securePass123",
  "data": {
    "full_name": "John Doe",
    "phone": "+855123456789"
  }
}
// Response 200
{
  "id": "uuid",
  "email": "user@example.com",
  "created_at": "2026-03-08T00:00:00Z"
}
```
**Side effects:** Trigger creates profile, user_role (user), and main account ($1000).

### POST `/auth/v1/token?grant_type=password`
Sign in.
```json
// Request
{ "email": "user@example.com", "password": "securePass123" }
// Response 200
{
  "access_token": "eyJ...",
  "refresh_token": "xxx",
  "user": { "id": "uuid", "email": "..." }
}
```

### POST `/auth/v1/logout`
Sign out. Invalidates session.

---

## 2. Profiles

### GET `/rest/v1/profiles?user_id=eq.{uid}`
Get current user's profile.
```json
// Response 200
{
  "id": "uuid",
  "user_id": "uuid",
  "full_name": "John Doe",
  "phone": "+855123456789",
  "pin_hash": "hashed_value",
  "last_activity": "2026-03-08T12:00:00Z"
}
```

### PATCH `/rest/v1/profiles?user_id=eq.{uid}`
Update profile fields.
```json
// Request
{ "full_name": "Jane Doe", "phone": "+855987654321" }
```

### PATCH `/rest/v1/profiles?user_id=eq.{uid}`
Set/update PIN hash.
```json
// Request
{ "pin_hash": "hashed_new_pin" }
```

---

## 3. Accounts

### GET `/rest/v1/accounts?user_id=eq.{uid}&account_type=eq.main`
Get main account.
```json
// Response 200
{
  "id": "uuid",
  "user_id": "uuid",
  "account_type": "main",
  "balance": 1000.00,
  "currency": "USD"
}
```

### GET `/rest/v1/accounts?user_id=eq.{uid}&account_type=eq.piggy`
Get all piggy accounts for user.

---

## 4. Piggy Goals

### GET `/rest/v1/piggy_goals?user_id=eq.{uid}&select=*,accounts!fk_piggy_goal(*)`
List user's piggy goals with linked accounts.
```json
// Response 200
[
  {
    "id": "uuid",
    "name": "Vacation Fund",
    "target_amount": 500.00,
    "lock_period_days": 90,
    "hide_balance": false,
    "status": "active",
    "created_at": "2026-01-01T00:00:00Z",
    "accounts": [{ "id": "uuid", "balance": 150.00, ... }]
  }
]
```

### POST `/rest/v1/piggy_goals`
Create a new piggy goal.
```json
// Request
{
  "user_id": "uuid",
  "name": "Emergency Fund",
  "target_amount": 1000.00,
  "lock_period_days": 30,
  "hide_balance": false
}
// Response 201
{ "id": "new-goal-uuid", ... }
```
**Note:** Client must also insert a linked piggy account:
```json
POST /rest/v1/accounts
{
  "user_id": "uuid",
  "account_type": "piggy",
  "piggy_goal_id": "new-goal-uuid",
  "balance": 0,
  "currency": "USD"
}
```

### PATCH `/rest/v1/piggy_goals?id=eq.{goal_id}`
Update goal (e.g., name, hide_balance).

---

## 5. Transfers (Edge Function)

### POST `/functions/v1/process-transfer`
Central money movement endpoint. Requires `Authorization` header.

#### Action: `transfer` (Main → Own Piggy)
```json
// Request
{ "action": "transfer", "amount": 50.00, "piggy_goal_id": "uuid" }
// Response 200
{ "message": "Transfer successful", "transaction_id": "uuid" }
```

#### Action: `p2p` (Main → Another User's Main)
```json
// Request
{ "action": "p2p", "amount": 25.00, "recipient_user_id": "uuid" }
// Response 200
{ "message": "P2P transfer successful", "transaction_id": "uuid" }
```

#### Action: `contribute` (Main → Another User's Piggy)
```json
// Request
{
  "action": "contribute",
  "amount": 10.00,
  "recipient_user_id": "uuid",
  "recipient_piggy_goal_id": "uuid"
}
// Response 200
{ "message": "Contribution successful", "transaction_id": "uuid" }
```

#### Action: `break` (Break Piggy → Return to Main)
```json
// Request
{ "action": "break", "break_piggy_goal_id": "uuid" }
// Response 200
{
  "message": "Piggy broken",
  "transaction_id": "uuid",
  "returnAmount": 142.50,
  "penalty": 7.50
}
```

#### Error Responses
```json
// 400
{ "error": "Insufficient balance" }
{ "error": "Goal not found or not active" }
{ "error": "Recipient not found" }
// 401
{ "error": "Not authenticated" }
```

---

## 6. Interest Calculation (Edge Function)

### POST `/functions/v1/calculate-interest`
Scheduled/manual trigger for daily compound interest on active piggy accounts.
```json
// Response 200
{
  "message": "Interest calculated",
  "processed": 15,
  "totalInterest": 1.23
}
```
**Logic:** Reads `interest_rate_piggy` from settings, applies daily rate, skips already-paid accounts, creates transaction + ledger entry.

---

## 7. Transactions

### GET `/rest/v1/transactions?or=(from_account_id.in.({ids}),to_account_id.in.({ids}))&order=created_at.desc&limit=50`
List user's transactions.
```json
// Response 200
[
  {
    "id": "uuid",
    "from_account_id": "uuid",
    "to_account_id": "uuid",
    "amount": 50.00,
    "type": "transfer",       // transfer|p2p|contribution|interest|fee|break|deposit|reversal
    "status": "completed",    // pending|completed|failed|reversed
    "description": "To Vacation Fund",
    "created_at": "2026-03-08T10:00:00Z"
  }
]
```

---

## 8. Ledger Entries

### GET `/rest/v1/ledger_entries?account_id=in.({ids})&order=created_at.desc`
Immutable double-entry records.
```json
// Response 200
[
  {
    "id": "uuid",
    "transaction_id": "uuid",
    "account_id": "uuid",
    "debit": 50.00,
    "credit": 0,
    "created_at": "2026-03-08T10:00:00Z"
  }
]
```

---

## 9. Interest Payments

### GET `/rest/v1/interest_payments?account_id=eq.{id}&order=created_at.desc&limit=30`
Interest history for a piggy account.
```json
// Response 200
[
  {
    "id": "uuid",
    "account_id": "uuid",
    "piggy_goal_id": "uuid",
    "principal": 150.00,
    "rate": 0.0000959,
    "amount": 0.01,
    "created_at": "2026-03-08T00:00:00Z"
  }
]
```

---

## 10. Notifications

### GET `/rest/v1/notifications?user_id=eq.{uid}&order=created_at.desc&limit=20`
List notifications.

### PATCH `/rest/v1/notifications?id=eq.{id}`
Mark as read.
```json
{ "read": true }
```

---

## 11. System Settings (Admin Only)

### GET `/rest/v1/system_settings`
Read all settings (public).
```json
// Response 200
[
  { "key": "interest_rate_piggy", "value": "3.50" },
  { "key": "early_break_penalty_pct", "value": "5.00" },
  { "key": "early_closure_fee", "value": "5.00" },
  { "key": "dormancy_fee", "value": "6.00" }
]
```

### PATCH `/rest/v1/system_settings?key=eq.{key}`
Update setting (admin only).
```json
{ "value": "4.00" }
```

---

## 12. User Roles (Admin Only)

### GET `/rest/v1/user_roles?user_id=eq.{uid}`
Check user role.

### POST `/rest/v1/rpc/has_role`
RPC to check role (avoids RLS recursion).
```json
// Request
{ "_user_id": "uuid", "_role": "admin" }
// Response 200
true
```

---

## Error Codes

| Code | Meaning |
|------|---------|
| 200  | Success |
| 201  | Created |
| 400  | Bad request / validation error |
| 401  | Not authenticated |
| 403  | Forbidden (RLS denied) |
| 404  | Not found |
| 409  | Conflict (idempotency key) |
| 500  | Server error |

## Rate Limits
- Auth endpoints: 30 req/hour per IP
- REST API: 1000 req/sec per project
- Edge Functions: 100 concurrent invocations

## Idempotency
All money movement operations use a unique `idempotency_key` (UUID) to prevent double-spending. Duplicate keys return the original transaction result.
