# DB College Project - Fitness Tracker

A full-stack Flask web app for tracking daily fitness habits across food, gym, sleep, and wellbeing, with account authentication and date-based analytics dashboards.

## 1) Project Overview

This project is a monolithic Python Flask application with:
- Server-side rendered HTML templates (Jinja2)
- SQLite database (`fitness.db`)
- Authentication with `Flask-Login`
- Custom schema migration logic at app startup
- Dashboard + report pages with Chart.js charts
- Theme toggle and responsive UI

Core use case:
- Register/login
- Log food, sleep, wellbeing, and gym activity per date
- View daily metrics + weekly trends
- View workout streak and health score

## 2) Tech Stack

- Backend: Python + Flask
- Auth/session: Flask-Login + Werkzeug password hashing
- Database: SQLite3
- Frontend templating: Jinja2 (Flask templates)
- Frontend styling: CSS (`static/style.css`)
- Frontend charts: Chart.js (CDN)
- Icons/fonts: Google Fonts (Outfit), Font Awesome (CDN in some pages)

## 3) Repository Structure

- `app.py`: Main Flask app, routes, business logic, DB helpers, migrations
- `schema.sql`: Base schema (tables + constraints)
- `requirements.txt`: Python dependencies (`Flask`, `Flask-Login`)
- `fitness.db`: SQLite database file
- `templates/`: Jinja templates for all pages
- `static/style.css`: Global styling

Template highlights:
- `templates/index.html`: Dashboard (daily score + weekly charts)
- `templates/food.html`: Food logging + edit/delete flows
- `templates/gym.html`: Multi-step workout logger + streaks
- `templates/sleep.html`: Sleep tracking (upsert per day)
- `templates/wellbeing.html`: Wellbeing activity logging
- `templates/report.html`: Detailed daily report + weekly chart summary
- `templates/login.html`, `register.html`, `forgot_password.html`, `reset_password.html`
- `templates/profile.html`: Personal details (name, age, height, weight)
- `templates/_navbar.html`, `_flash_messages.html`: Shared partials

## 4) How the Application Works

### 4.1 Startup Flow
On startup (`app.py`):
1. Flask app is created and configured.
2. `Flask-Login` is initialized.
3. `init_db()` is called immediately.
4. `init_db()` runs `migrate_schema()`:
   - Executes `schema.sql`
   - Ensures required columns/indexes exist
   - Migrates older table variants (`users`, `sleep_log`) safely

This means schema setup/migration is automatic when the app starts.

### 4.2 Authentication & User Model
- Passwords are hashed with Werkzeug (`generate_password_hash` / `check_password_hash`).
- Sessions and login state are handled by Flask-Login.
- Unauthorized requests are redirected to `/login`, with the original URL saved and restored after login.
- First registered user becomes `admin` (or any email in `ADMIN_EMAILS` env var).

### 4.3 Password Reset
- `/forgot-password` generates a one-time token (30-minute expiry).
- Reset link is printed to server console (not emailed).
- `/reset-password/<token>` validates token and updates password.
- Old active tokens are invalidated after reset.

### 4.4 Date Handling
- Most pages use a selected date (`YYYY-MM-DD`) via query/form.
- Dates are normalized and capped to today (no future date logging).
- Dashboard/report also compute a rolling 7-day window ending on selected date.

### 4.5 Dashboard Logic (`/`)
Computes for selected date:
- Food totals: calories/protein/fat/sugar
- Gym calories + duration
- Net calories = intake - gym burn
- Health score (0-100) based on calorie goal, protein, sugar, gym activity, sleep
- Weekly arrays for charts: calories, sleep, wellbeing minutes, gym calories

### 4.6 Food Logging (`/food`)
- Uses built-in `FOOD_DATA` dictionary grouped by meal.
- Supports predefined food items or `OTHERS` custom macro entry.
- Logs `qty` and nutrient totals into `food_log`.
- Supports edit/delete per row.

### 4.7 Sleep Tracking (`/sleep`)
- One sleep record per user per date (`UNIQUE(user_id, date)`).
- Uses upsert behavior (`ON CONFLICT DO UPDATE`).
- Supports deletion for selected date.

### 4.8 Wellbeing Tracking (`/wellbeing`)
- Logs activity + minutes for selected date.
- Supports custom “OTHERS” activity name.
- Supports deletion per entry.

### 4.9 Gym Tracking (`/gym`)
- Multi-step form in UI (muscle group -> exercise -> metrics -> intensity).
- Supports both strength and cardio entries.
- Cardio calories use MET-based estimation with optional speed/incline adjustments.
- Strength calories use volume-based formula with muscle/intensity multipliers.
- Calculates gym streak from distinct workout dates.
- “Repeat Last Workout” prefill behavior exists in UI.

### 4.10 Reports (`/report`)
- Meal-wise macro summary for selected date.
- Gym entries + total burn for selected date.
- Weekly trend charts for calories, sleep, wellbeing, gym burn.

## 5) Database Schema

Main tables:
- `users`: account + role + profile fields + active status
- `password_reset_tokens`: reset token lifecycle
- `food_log`: nutrition entries
- `sleep_log`: one row per user/date
- `wellbeing_log`: activity duration entries
- `gym_log`: strength/cardio workout entries

Performance indexes are created for `(user_id, date)` access patterns.

Schema source: `schema.sql` plus runtime migration helpers in `app.py`.

## 6) Run Instructions

### 6.1 Prerequisites
- Python 3.9+ (recommended)
- `pip`

### 6.2 Install
```bash
cd /Users/sj/Desktop/db/DB_CollegeProject
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

### 6.3 Environment Variables (optional but recommended)
```bash
export SECRET_KEY="your-strong-secret"
export ADMIN_EMAILS="admin@example.com,owner@example.com"
```

- `SECRET_KEY`: Flask session signing key (defaults to insecure `change-me-in-production` if unset)
- `ADMIN_EMAILS`: comma-separated emails that should get `admin` role

### 6.4 Start the App
```bash
python3 app.py
```

App runs on:
- `http://127.0.0.1:5002`

### 6.5 First Use
1. Open the app in browser.
2. Register a user.
3. First account becomes admin unless overridden by `ADMIN_EMAILS`.
4. Start logging fitness data.

## 7) Key Routes

Auth/Profile:
- `GET/POST /login`
- `GET/POST /register`
- `POST /logout`
- `GET/POST /profile`
- `GET/POST /forgot-password`
- `GET/POST /reset-password/<token>`

Tracking:
- `GET /` dashboard
- `GET/POST /food`
- `POST /food/delete/<item_id>`
- `GET/POST /food/edit/<item_id>`
- `GET/POST /sleep`
- `POST /sleep/delete`
- `GET/POST /wellbeing`
- `POST /wellbeing/delete/<item_id>`
- `GET/POST /gym`
- `POST /gym/delete/<item_id>`
- `GET /report`

## 8) Security & Operational Notes

- Password reset links are printed to server stdout (developer workflow, not production email flow).
- Default `SECRET_KEY` is not safe for production.
- SQLite file is local; no multi-instance DB setup.
- App runs with `debug=True` in `__main__`.
- No automated test suite is present in current repository.

## 9) Known Data/Logic Characteristics

- Food nutrient values are hardcoded in `FOOD_DATA`.
- Daily goals are hardcoded in `GOALS`.
- Future dates are blocked by date normalization.
- Cardio/strength calorie formulas are estimation models, not medical-grade calculations.

## 10) Suggested Improvements

- Move config to `.env` and disable debug in production.
- Add email integration for password reset links.
- Add automated tests (unit + route tests).
- Add DB migration tooling (e.g., Flask-Migrate/Alembic).
- Add validation constraints and form libraries (e.g., WTForms).
- Add API layer if mobile/client separation is needed.

## 11) Quick Command Reference

Install deps:
```bash
pip install -r requirements.txt
```

Run app:
```bash
python3 app.py
```

Reset local DB (destructive):
```bash
rm -f fitness.db
python3 app.py
```

## 12) Current Project Status Summary

This is a complete, functional student-level fitness tracker web app with:
- Working authentication
- Multi-domain health logging
- Useful dashboards/charts
- Solid SQLite persistence
- Basic role model and password reset flow

It is ready for local development/demo use, and can be productionized with security/config/testing hardening.
