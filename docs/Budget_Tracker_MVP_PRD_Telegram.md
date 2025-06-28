**Budget Tracker (MVP) PRD**

## 1. Objective
Launch a minimal viable expense-tracking system using n8n workflows and Supabase, enabling households to log expenses via Telegram bot, store them centrally, set a monthly budget, and receive weekly summaries.

## 2. MVP Scope & Out of Scope
**In Scope** (n8n + Supabase only):
- **Bot Commands** over Telegram:
  - `/createhousehold <Name>` → generate a household and join code
  - `/joinhousehold <Code> <Name>` → add user to household
  - `/expense <amount> - <place>` → log an expense
  - `/setbudget <amount>` → set current month’s budget
- **Automations**:
  - Daily Cron: seed new monthly budget record in Supabase
  - Weekly Cron: generate and send weekly spending summary back to Telegram
- **Data Storage** in Supabase (no UI):
  - Tables: `households`, `users`, `expenses`, `budgets`
  - Simple schema for expense records and budgets

**Out of Scope**:
- Web dashboard or frontend
- Predictive forecasting
- Recurring/annual expenses
- AI categorization (use simple keyword rules)

## 3. Core User Flows

### 3.1 Create & Join Household
1. **User** sends `/createhousehold FamilyName` → n8n creates record in `households` with a 6-char `join_code` and replies with code.
2. **Second user** sends `/joinhousehold <code> MyName` → n8n looks up `households`, inserts into `users` table.

### 3.2 Log Expense
- In group or 1:1 chat, user sends `/expense 150 - Cafe`. n8n parses `amount` and `place`, applies keyword lookup for category (or tags as "Uncategorized"), and inserts into `expenses` with timestamps and `user_id`.

### 3.3 Set Monthly Budget
- User sends `/setbudget 5000`. n8n upserts a row in `budgets` for the current month and `household_id`.

### 3.4 Weekly Summary
- Every Sunday at 18:00, a Cron node triggers:
  1. Query Supabase for this household’s expenses in the past 7 days
  2. Sum totals and group by category
  3. Post a summary message to the corresponding Telegram chat

## 4. Supabase Schema
```sql
-- Households
CREATE TABLE households (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  name text NOT NULL,
  join_code text UNIQUE NOT NULL,
  salary_day int NOT NULL DEFAULT 1
);

-- Users
CREATE TABLE users (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  telegram_id text UNIQUE NOT NULL,
  name text,
  household_id uuid REFERENCES households(id)
);

-- Expenses
CREATE TABLE expenses (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  household_id uuid REFERENCES households(id),
  submitted_by uuid REFERENCES users(id),
  amount numeric NOT NULL,
  place text,
  category text DEFAULT 'Uncategorized',
  created_at timestamp DEFAULT now()
);

-- Budgets
CREATE TABLE budgets (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  household_id uuid REFERENCES households(id),
  month text NOT NULL,
  amount numeric NOT NULL,
  UNIQUE(household_id, month)
);
```

## 5. n8n Workflow Components
1. **Webhook Trigger** for Telegram messages  
2. **Command Switch**: route `/createhousehold`, `/joinhousehold`, `/setbudget`, or `/expense` messages  
3. **Function Nodes**: parse text payloads and implement keyword categorization  
4. **Supabase Nodes**: insert/select/upsert for households, users, expenses, budgets  
5. **Cron Nodes**:  
   - Daily: upsert budget row for new month  
   - Weekly: compile and send summary  


**Note:** Telegram is the primary platform for interactive flows like setting budgets and logging expenses. WhatsApp is under consideration for limited use in sending weekly summaries only.