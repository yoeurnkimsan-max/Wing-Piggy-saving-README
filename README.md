# Piggy Saving System — API Design
## Overview
This system provides a digital savings platform with main accounts, piggy (goal-based) savings accounts, P2P transfers, QR-based payments, daily compound interest, and admin controls. All financial operations go through server-side edge functions to ensure atomic balance updates and double-entry ledger integrity.
---
## Authentication
| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/auth/v1/signup` | Register with email + password. Auto-creates profile, user role, and main account (seeded with $1,000). |
| `POST` | `/auth/v1/token?grant_type=password` | Login with email + password. Returns JWT access + refresh tokens. |
| `POST` | `/auth/v1/logout` | Invalidate session. |
### Sign Up Payload
```json
{
  "email": "user@example.com",
  "password": "secure123",
  "data": {
    "full_name": "John Doe",
    "phone": "+1234567890"
  }
}
```
### On Sign Up (Database Trigger: `handle_new_user`)
Automatically creates:
1. `profiles` row (full_name, phone)
2. `user_roles` row (role = `user`)
3. `accounts` row (type = `main`, balance = `1000.00`, currency = `USD`)
### PIN Security Layer
- PIN hash stored in `profiles.pin_hash`
- App locks after inactivity; PIN required to unlock
- PIN confirmation required before transfers
---
## Edge Functions
### 1. `process-transfer`
**Auth:** Bearer token required (JWT)  
**Method:** `POST`  
**Invocation:** `supabase.functions.invoke('process-transfer', { body })`
Handles all financial movements atomically. Each action creates a transaction record, double-entry ledger entries, updates balances, and sends notifications.
---
#### Action: `transfer` — Main → Own Piggy
Move money from your main account to one of your own saving goals.
**Request:**
```json
{
  "action": "transfer",
  "amount": 50.00,
  "piggy_goal_id": "uuid-of-goal"
}
```
**Response (200):**
```json
{
  "message": "Transfer successful",
  "transaction_id": "uuid"
}
```
**Validations:**
- `amount` > 0
- Sufficient main account balance (re-fetched at execution time)
- Goal must exist, belong to user, and have `status = "active"`
**Side Effects:**
- Main balance decreased, piggy balance increased
- Ledger: debit main, credit piggy
- If `piggy_balance >= target_amount` → goal auto-completes
- Notification: "Transfer Successful"
---
#### Action: `p2p` — Main → Another User's Main
Send money to another user's main account.
**Request:**
```json
{
  "action": "p2p",
  "amount": 25.00,
  "recipient_user_id": "uuid-of-recipient"
}
```
**Response (200):**
```json
{
  "message": "P2P transfer successful",
  "transaction_id": "uuid"
}
```
**Validations:**
- `amount` > 0
- Cannot send to yourself
- Sufficient balance
- Recipient must have a main account
**Side Effects:**
- Sender balance decreased, recipient balance increased
- Ledger: debit sender, credit recipient
- Notification to both sender ("P2P Sent") and recipient ("P2P Received")
---
#### Action: `contribute` — Main → Someone's Piggy Goal
Contribute money to any user's active saving goal (including your own).
**Request:**
```json
{
  "action": "contribute",
  "amount": 10.00,
  "recipient_piggy_goal_id": "uuid-of-goal"
}
```
**Response (200):**
```json
{
  "message": "Contribution successful",
  "transaction_id": "uuid"
}
```
**Validations:**
- `amount` > 0
- Sufficient balance
- Goal must be `active`
**Side Effects:**
- Sender main balance decreased, goal's piggy account increased
- Ledger: debit sender, credit piggy
- If goal completed → auto-updates status
- Notifications to both contributor and goal owner
---
#### Action: `break` — Break a Piggy Goal
Break your own saving goal and return balance to main account. Early breaks incur a penalty.
**Request:**
```json
{
  "action": "break",
  "break_piggy_goal_id": "uuid-of-goal"
}
```
**Response (200):**
```json
{
  "message": "Piggy broken",
  "returnAmount": 95.00,
  "penalty": 5.00,
  "transaction_id": "uuid"
}
```
**Validations:**
- Goal must belong to user and be `active`
- Piggy balance must be > 0
**Penalty Logic:**
- If `lock_period_days` is set and lock hasn't expired:
  - Penalty = `balance × early_break_penalty_pct` (from `system_settings`, default 5%)
- If lock expired or no lock: penalty = 0
**Side Effects:**
- Piggy balance → 0, main balance += returnAmount
- Ledger: debit piggy (full balance), credit main (returnAmount)
- If penalty > 0: separate `fee` transaction + ledger entry
- Goal status → `"broken"`, `broken_at` set
- Notification with penalty details
---
#### Error Response (400):
```json
{
  "error": "Insufficient balance"
}
```
Common errors:
| Error | Cause |
|-------|-------|
| `Missing authorization header` | No Bearer token |
| `Unauthorized` | Invalid/expired token |
| `Main account not found` | User has no main account |
| `Insufficient balance` | Not enough funds |
| `Goal is not active` | Goal already broken/completed |
| `Cannot transfer to yourself` | P2P self-transfer attempt |
| `Invalid amount` | Amount ≤ 0 or missing |
| `Invalid action` | Unknown action type |
---
### 2. `calculate-interest`
**Auth:** Service-level (no user JWT required)  
**Method:** `POST`  
**Purpose:** Daily batch job — calculates and credits compound interest to all active piggy accounts.
**Request:** Empty body (no parameters needed)
**Response (200):**
```json
{
  "message": "Interest calculated",
  "processed": 42,
  "totalInterest": 3.67
}
```
**Logic:**
1. Read `interest_rate_piggy` from `system_settings` (default: 3.50% APR)
2. Compute daily rate: `annualRate / 100 / 365`
3. For each piggy account with `balance > 0` and an `active` goal:
   - Skip if interest already paid today (idempotency check)
   - Interest = `balance × dailyRate` (rounded to 2 decimals)
   - Insert `interest_payments` record
   - Create `interest` transaction with idempotency key: `interest-{accountId}-{YYYY-MM-DD}`
   - Insert ledger entry (credit to piggy)
   - Update account balance
**Idempotency:** Safe to call multiple times per day — skips accounts already processed.
---
## Data Model (Quick Reference)
### Core Tables
| Table | Purpose | Key Fields |
|-------|---------|------------|
| `profiles` | User info | `user_id`, `full_name`, `phone`, `pin_hash` |
| `user_roles` | Role-based access | `user_id`, `role` (enum: `user`, `admin`) |
| `accounts` | Financial accounts | `user_id`, `account_type` (main/piggy), `balance`, `piggy_goal_id` |
| `piggy_goals` | Saving goals | `user_id`, `name`, `target_amount`, `lock_period_days`, `status` |
| `transactions` | All money movements | `from_account_id`, `to_account_id`, `amount`, `type`, `status` |
| `ledger_entries` | Double-entry audit trail | `transaction_id`, `account_id`, `debit`, `credit` |
| `interest_payments` | Interest accrual log | `account_id`, `piggy_goal_id`, `principal`, `rate`, `amount` |
| `notifications` | User notifications | `user_id`, `title`, `message`, `type`, `read` |
| `system_settings` | Admin config | `key`, `value` |
### Transaction Types
| Type | Description |
|------|-------------|
| `transfer` | Main → own piggy |
| `p2p` | Main → another user's main |
| `contribution` | Main → any user's piggy goal |
| `interest` | System-generated daily interest credit |
| `fee` | Early break penalty |
| `break` | Piggy → main (goal broken) |
| `deposit` | Initial deposit (signup) |
| `reversal` | Admin reversal of a transaction |
### Transaction Statuses
| Status | Description |
|--------|-------------|
| `pending` | Created but not yet processed |
| `completed` | Successfully executed |
| `failed` | Execution failed |
| `reversed` | Reversed by admin |
### System Settings
| Key | Default | Description |
|-----|---------|-------------|
| `interest_rate_piggy` | `3.50` | Annual interest rate (%) for piggy accounts |
| `early_break_penalty_pct` | `5` | Penalty percentage for breaking locked goals early |
---
## Security
### Row-Level Security (RLS)
All tables have RLS enabled. Key rules:
| Table | SELECT | INSERT | UPDATE | DELETE |
|-------|--------|--------|--------|--------|
| `profiles` | Own + admin + authenticated search | Own | Own | ✗ |
| `accounts` | Own + admin + authenticated | Own | Own + authenticated | ✗ |
| `transactions` | Own (via account) + admin | Authenticated | Admin only | ✗ |
| `ledger_entries` | Own (via account) + admin | Authenticated | ✗ | ✗ |
| `piggy_goals` | Own + admin + active goals | Own | Own | ✗ |
| `notifications` | Own | Authenticated | Own | ✗ |
| `system_settings` | Anyone | Admin | Admin | Admin |
| `user_roles` | Own + admin | Admin | Admin | Admin |
### Admin Role Check
```sql
-- Security definer function (bypasses RLS)
SELECT public.has_role(auth.uid(), 'admin');
```
### Double-Entry Ledger
Every transaction produces balanced ledger entries:
- **Debit** = money leaving an account
- **Credit** = money entering an account
- For every transaction: `SUM(debits) = SUM(credits)`
---
## QR Code Payment Flow
### QR Payload Format (TLV-encoded)
```
Tag 01: recipient_user_id (P2P)
Tag 02: piggy_goal_id (Contribution)
Tag 03: amount (optional, pre-filled)
Tag 99: expiry timestamp
```
### Flow
1. **Generator** creates QR with TLV-encoded payload
2. **Scanner** decodes QR → validates payload → shows confirmation
3. **User** enters amount (if not pre-filled) → confirms with PIN
4. **System** calls `process-transfer` with `action: "p2p"` or `"contribute"`
---
## Client-Side Invocation
```typescript
import { supabase } from "@/integrations/supabase/client";
// All transfers
const { data, error } = await supabase.functions.invoke('process-transfer', {
  body: {
    action: 'transfer',
    amount: 50,
    piggy_goal_id: 'uuid'
  }
});
// Interest calculation (typically cron-triggered)
const { data } = await supabase.functions.invoke('calculate-interest');
```
---
## Admin Capabilities
Admins (users with `role = 'admin'` in `user_roles`) can:
1. **View all users** — profiles, accounts, goals
2. **View all transactions** — full transaction history
3. **View ledger entries** — complete audit trail
4. **Reverse transactions** — marks original as `reversed`, creates reversal entry, adjusts balances
5. **Manage system settings** — interest rates, penalty percentages
6. **Manage user roles** — promote/demote users
