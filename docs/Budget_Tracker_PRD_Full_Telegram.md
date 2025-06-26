**Product Requirements Document (PRD)**

## 1. Executive Summary

A multi-tenant, Telegram-integrated expense tracking and forecasting platform ("MoneyBuddy"), enabling households to log expenses via chatbot, view real-time budgets and forecasts through a web dashboard, and receive notifications. The MVP supports secure household management, AI-powered categorization, monthly budgets, weekly summaries, and predictive analytics.

## 2. Problem Statement

Households struggle to consistently track shared expenses, leading to overspending and lack of visibility. Telegram-based summaries are clumsy for detailed analysis; manual categorization is tedious. Users need a seamless, shared system that adapts to salary cycles and predicts end-of-month spend.

## 3. Goals & Objectives

- **Fast data entry** via Telegram bot or web UI
- **Accurate categorization** using AI/keyword models
- **Shared household view** for multiple members
- **Dynamic budgets** matching salary schedules per household
- **Predictive forecasting** updating throughout the month
- **Intuitive dashboard** with drill-downs and historical trends

## 4. Stakeholders

- **End Users**: Couples/families managing shared budgets
- **Product Owner**: Defines features and prioritizes backlog
- **Engineering Team**: Implements backend, frontend, and integrations
- **Design Team**: UI/UX for web dashboard and bot messages
- **Operations**: Deploy and monitor services

## 5. User Personas

1. **Eyal (Primary User)**: Tech-savvy backend engineer, needs quick entry and detailed analytics.
2. **Michal (Secondary User)**: Busy professional, prefers mobile-first simplicity and clear summaries.
3. **Future Admins**: Small households or flatmates requiring occasional invite flows.

## 6. Use Cases & User Stories

- **UC1: Create Household**: As a user, I can `/createhousehold <Name>` to start a new budget group and receive a join code.
- **UC2: Join Household**: As a user, I can `/joinhousehold <Code> <MyName>` to join an existing household.
- **UC3: Log Expense via Bot**: As a household member, I can send `<amount> - <place>` to the bot to record an expense.
- **UC4: Set Monthly Budget**: I can `/setbudget <amount>` to update the current month’s budget.
- **UC5: Weekly Summary**: The system sends a summary every Sunday with totals and category breakdown.
- **UC6: Forecast**: On the dashboard, view predicted month-end spend based on current pace.
- **UC7: Web Dashboard**: Users can log in to see expense list, budgets, recurring items, and forecasts.

## 7. Functional Requirements

1. **Authentication & Authorization**
   - Supabase Auth with email/magic link
   - RLS policies ensure users see only their household data

2. **Bot Integration**
   - Webhook endpoint for incoming Telegram messages (Green API/Twilio)
   - Slash commands: `/createhousehold`, `/joinhousehold`, `/setbudget`, `/addrecurring`, `/listrecurring`, `/removerecurring`, `/setrecurring`, `/summary`

3. **Database Schema**
   - Tables: `households`, `users`, `expenses`, `recurring_expenses`, `budgets`, `forecasts`
   - Columns per earlier schema

4. **Expense Ingestion**
   - Parse amount & place
   - AI categorization via Gemini or keyword logic
   - Insert into `expenses` with timestamp and `submitted_by`

5. **Budget Management**
   - Store per-household `salary_day`
   - Daily Cron to seed new month’s `budgets` record
   - `/setbudget` command to overwrite budget

6. **Recurring & Annual Expenses**
   - Creation of `recurring_expenses` with frequency (monthly/yearly)
   - Automatic expense generation on due dates
   - Slash commands to manage recurring items

7. **Summaries & Forecasts**
   - Weekly summary job
   - Forecast job: moving average or exponential smoothing
   - APIs to retrieve summary, forecast for frontend

8. **Web Dashboard**
   - React/Angular app
   - Expense table with filters, virtual scroll
   - Budget vs spent gauge
   - Forecast line chart
   - Recurring expenses management UI
   - Settings: invite code, budget, salary day

## 8. Recurring & Annual Expenses

- Support creation of fixed recurring expenses (e.g., rent) with frequency = "monthly".
- Support annual or other periodic expenses (e.g., car insurance) with frequency = "yearly" and automatic division by 12 for budgeting.
- Automatically schedule and insert these into the `expenses` table on each due date.
- Provide UI and slash-commands (`/addrecurring`, `/listrecurring`, `/removerecurring`, `/setrecurring`) to manage recurring expense definitions.
- **Note:** The `/setrecurring` command requires the recurring expense's unique ID (listed via `/listrecurring`), making it less user-friendly; full editing capabilities (e.g., changing amount, frequency) are recommended through the web UI for a smoother experience.

## 9. Non-Functional Requirements

- **Scalability**: Support 1000+ households, 10k+ expenses/month
- **Reliability**: 99.9% uptime for ingestion and cron jobs
- **Security**: Encrypt data in transit (HTTPS), RLS and JWT for API
- **Performance**: API responses <200ms, dashboard load <2s
- **Maintainability**: Use migrations (Knex/Supabase), unit tests for key logic
- **Compliance**: GDPR for personal data; opt-in for newsletters

## 10. System Architecture

```text
┌─────────────┐    Telegram    ┌───────────┐   Ingest   ┌─────────────┐
│  Telegram   │ ─────────────▶│  n8n /     │ ─────────▶ │  Ingestion  │
│    Bot      │               │  or Backend│            │  Service    │
└─────────────┘               └───────────┘            └─────┬───────┘
                                                           │
                                                           │ writes
                                                           ▼
                    ┌──────────┐    queries    ┌───────────────┐
                    │ Supabase │◀─────────────▶│  Analytics &  │
                    │ Postgres │               │  Forecast     │
                    └──────────┘                └─────┬─────────┘
                                                           │
                                                     serves │ REST
                                                           ▼
                                                 ┌────────────────┐
                                                 │   Angular      │
                                                 │   Dashboard    │
                                                 └────────────────┘
```

## 11. Data Model Overview

- **households**: id, name, join_code, salary_day
- **users**: id, telegram_id, name, household_id
- **expenses**: id, household_id, submitted_by, amount, place, category, subcategory, created_at
- **recurring_expenses**: id, household_id, amount, place, category, subcategory, frequency (monthly/yearly), next_due, created_at
- **budgets**: id, household_id, month, amount, created_at
- **forecasts**: household_id, month, predicted_total, updated_at

## 12. API Endpoints

- `POST /webhook/expense` — Ingest bot messages
- `POST /webhook/command` — Handle slash commands
- `GET /api/household/:id/summary` — Return current spend vs budget
- `GET /api/household/:id/forecast` — Return forecast data
- `GET /api/household/:id/expenses` — Paginated expense list
- `POST /api/household/:id/recurring-expenses` — Create a new recurring expense definition
- `GET /api/household/:id/recurring-expenses` — List all recurring expense definitions
- `DELETE /api/household/:id/recurring-expenses/:recurringId` — Remove a recurring expense definition
- `POST /api/household/:id/trigger-recurring` — Manually trigger generation of due recurring expenses

## 13. UI Wireframes & Mockups

- **Login** → magic link
- **Home** → summary cards + chart
- **Expenses** → filterable list
- **Recurring Expenses** → manage rent, insurance, subscriptions (add/edit/delete)
- **Forecast** → line chart with actual vs predicted
- **Settings** → budget, salary day, invite code

## 14. Milestones & Timeline

| Phase              | Deliverables                               | ETA (Weeks) |
| ------------------ | ------------------------------------------ | ----------- |
| Phase 1: Setup     | DB schema, Auth, Bot webhook ingest        | 2           |
| Phase 2: Core      | Expense ingestion, categorization, budgets | 3           |
| Phase 3: Dashboard | API & frontend MVP (login, list, summary)  | 4           |
| Phase 4: Forecast  | Forecast algorithm + UI integration        | 2           |
| Phase 5: Polish    | Tests, docs, deploy, beta launch           | 2           |

_Total: ~13 weeks._

## 15. Success Metrics

- **Adoption**: 100 households onboarded in first month
- **Engagement**: 80% weekly usage rate per household
- **Accuracy**: >90% correct auto-categorization
- **Retention**: <10% churn after 3 months

## 16. Risks & Mitigations

- **Message parsing errors**: Provide `/help` and strict formatter; fallback to manual correction.
- **Budget drift**: Forecast model may mispredict—start simple and refine with real data.
- **User confusion**: Clear in-bot messaging and UI tooltips.
- **Data privacy**: Enforce RLS, audit logs.

*End of PRD.*


**Note:** All bot interactions are designed for Telegram. WhatsApp is being considered only for passive summaries.