# Daybook

A bilingual (ខ្មែរ / English) bookkeeping app for small businesses: daily income and expenses, money lent and borrowed, and monthly reports.

| Part      | Tech                                                        | Folder      |
|-----------|-------------------------------------------------------------|-------------|
| Web app   | Next.js 16 (App Router), React 19, TypeScript               | `frontend/` |
| API       | Spring Boot 3.5, Java 21, Spring Security (JWT), JPA, Flyway | `backend/`  |
| Database  | PostgreSQL 16                                               | via Docker  |

## Run everything with Docker

```bash
cp .env.example .env
# put a long random value in JWT_SECRET, e.g. the output of: openssl rand -base64 48
docker compose up --build
```

Open http://localhost:3000 and create an account.

## Run for development

You need Java 21, Maven 3.9+, Node 20+ and a PostgreSQL database.

```bash
# 1. Database only
docker compose up db

# 2. API on http://localhost:8080 (Flyway creates the tables on first start)
cd backend
mvn spring-boot:run

# 3. Web app on http://localhost:3000
cd frontend
npm install
npm run dev
```

The web app proxies every `/api/*` request to the API (see `frontend/next.config.ts`), so the browser only talks to one origin. Point it elsewhere with `BACKEND_URL=http://host:port` at build or dev time.

## Configuration (API)

| Variable        | Default                                        | Notes |
|-----------------|------------------------------------------------|-------|
| `DB_URL`        | `jdbc:postgresql://localhost:5432/daybook`     | |
| `DB_USER`       | `daybook`                                      | |
| `DB_PASSWORD`   | `daybook`                                      | |
| `JWT_SECRET`    | a dev-only value                               | **Required in production**, at least 32 characters |
| `COOKIE_SECURE` | `false`                                        | Set `true` when served over HTTPS |
| `CORS_ORIGINS`  | `http://localhost:3000`                        | Only needed if a browser calls the API directly |

## How sign-in works

Signing in returns a JWT in an `httpOnly`, `SameSite=Strict` cookie that lasts 30 days. Page scripts can't read it, and other sites can't send it. Passwords are hashed with BCrypt. API clients other than the browser can send the same token as `Authorization: Bearer <token>`.

## API

All routes except `/api/auth/*` need a signed-in user, and only ever see that user's data.

| Method | Path | Purpose |
|--------|------|---------|
| POST | `/api/auth/register` | `{email, password, language}` → creates account and signs in |
| POST | `/api/auth/login` | `{email, password}` |
| POST | `/api/auth/logout` | Clears the cookie |
| GET | `/api/me` | Current user and settings |
| PUT | `/api/me/settings` | `{businessName, currency, language}` |
| DELETE | `/api/me/data` | Erase all entries and loans (keeps the account) |
| GET | `/api/entries?month=YYYY-MM` | One month, newest first. Without `month`: everything (CSV export) |
| POST | `/api/entries` | `{kind: IN\|OUT, amount, note, category, payMethod, date}` |
| DELETE | `/api/entries/{id}` | |
| GET | `/api/reports/monthly?month=YYYY-MM` | Totals, daily bars, category and payment breakdowns |
| GET | `/api/loans` | All loans with their payments |
| GET | `/api/loans/{id}` | |
| POST | `/api/loans` | `{direction: IN\|OUT, name, amount, note, lentOn, dueOn}` (IN = they owe me) |
| PUT | `/api/loans/{id}` | Same body |
| DELETE | `/api/loans/{id}` | |
| POST | `/api/loans/{id}/payments` | `{amount, paidOn, countAsEntry, entryNote}` |
| DELETE | `/api/loans/{id}/payments/{paymentId}` | Also removes the income/expense entry it created |

Errors come back as `{"error": "code"}` (for example `email_taken`, `amount_too_high`, `due_before_loan`), which the web app translates.

## Languages

All interface text lives in `frontend/lib/i18n.ts`. Categories and payment methods are stored in the database as fixed English keys (`Sale`, `Rent`, `Cash`…) and translated when shown, so switching language never changes your data. Each user's language is saved to their account and follows them to other devices.

## Project layout

```
backend/src/main/java/com/daybook/
  auth/      register, sign in, JWT cookie
  user/      settings, erase data
  entry/     income and expenses
  loan/      loans and repayments
  report/    monthly report
  config/    security, properties
  common/    errors, shared types
backend/src/main/resources/db/migration/   database schema (Flyway)

frontend/
  app/login/          sign in / create account
  app/(app)/          signed-in pages: Today, History, Loans, Loan detail, Report, Setup
  components/         shared UI, language + toast providers
  lib/                API client, translations, formatting
```
