# CuberDesk

Slack-based **desk and parking booking system** for multiple offices.

CuberDesk allows employees to:

- Book desks via Slack
- See who is attending each office
- Reserve parking where available
- Receive weekly and daily booking reminders
- Allow admins to manage bookings for the entire organisation

The system integrates directly into Slack using **App Home UI, DM reminders, and admin tools**.

---

# Table of Contents

- Overview  
- Tech Stack  
- Architecture  
- Offices  
- Database Schema  
- Database RPC Functions  
- Project Structure  
- Core System Components  
- Booking Workflows  
- Scheduled Jobs  
- Admin Tools  
- Date Handling (Critical)  
- Configuration  
- Error Handling  
- Race Condition Handling  
- Adding Features  
- Troubleshooting  
- Security  
- Performance Considerations  
- Testing  
- Deployment  
- Production Checklist  
- Logs  
- System Flow  
- Quick Reference  

---

# Overview

**Purpose**

CuberDesk provides a **centralised desk booking system inside Slack**.

Employees can:

- Book desks for specific dates
- View who else is attending
- Reserve parking (if available)
- Receive reminders if they haven't booked

Admins can:

- Book desks for users
- Cancel bookings
- View office attendance
- Manage parking allocation

---

# Tech Stack

Backend:

- Node.js
- @slack/bolt (Socket Mode)
- Supabase (PostgreSQL)
- node-cron

UI:

- Slack Block Kit
- Slack App Home
- Slack Modals
- Slack Slash Commands

---

# Architecture

```
Slack UI
   ↓
Socket Mode Bot
(index.js / admin.js / homeView.js)
   ↓
Database Layer
(db.js)
   ↓
Supabase PostgreSQL
```

---

# Offices

| Office | Parking |
|------|------|
| London | No |
| Cardiff | No |
| Exeter | Yes (2 slots) |
| Manchester | No |
| Paris | No |
| Barcelona | No |

Only **Exeter currently supports parking reservations**.

---

# Database Schema

## user_preferences

Stores the user's default office.

| Column | Type | Notes |
|------|------|------|
| user_id | TEXT | Slack user ID (Primary Key) |
| default_office | TEXT | User preferred office |

---

## day_bookings

Stores desk reservations.

| Column | Type |
|------|------|
| user_id | TEXT |
| work_date | TEXT |
| office | TEXT |

Primary Key:

```
(user_id, work_date)
```

Ensures **one booking per user per day**.

---

## exeter_parking

Stores parking slot reservations.

| Column | Type |
|------|------|
| work_date | TEXT |
| slot | INTEGER |
| user_id | TEXT |

Primary Key:

```
(work_date, slot)
```

Prevents parking slot conflicts.

---

## dm_buttons

Stores references to DM messages containing booking buttons.

| Column | Type |
|------|------|
| user_id | TEXT |
| work_date | TEXT |
| channel | TEXT |
| ts | TEXT |

Primary Key:

```
(user_id, work_date)
```

Allows the bot to **update DM messages when bookings change**.

---

# Database RPC Functions

## book_desk_with_optional_parking

Atomic transaction that inserts:

- desk booking
- optional parking slot

Parameters:

- `p_user_id`
- `p_work_date`
- `p_office`
- `p_slot`

Fails if:

- user already booked
- parking slot already taken

Prevents race conditions.

---

## cancel_booking

Deletes booking and associated parking.

Parameters:

```
p_user_id
p_work_date
```

---

## pop_dm_button

Returns and deletes DM button reference.

Parameters:

```
p_user_id
p_work_date
```

---

# Project Structure

```
├── index.js
├── admin.js
├── homeView.js
├── db.js
└── utils
    ├── dates.js
    └── strings.js
```

---

# Core System Components

## index.js

Main bot application.

Responsibilities:

- Slack event handlers
- booking logic
- cron jobs
- UI refresh
- DM updates

---

### Office Configuration

```javascript
const OFFICE_CONFIG = {
  exeter: { parking_is_optional: true, parking_slots: 2 },
  london: { parking_is_optional: false },
  cardiff: { parking_is_optional: false },
  manchester: { parking_is_optional: false },
  paris: { parking_is_optional: false },
  barcelona: { parking_is_optional: false }
};
```

---

### publishHome(client, userId)

Rebuilds and publishes the Slack **App Home interface**.

Steps:

1. Get user's default office
2. Fetch bookings for next 2 weeks
3. Build UI using `buildHomeView()`
4. Publish view via Slack API

---

### fetchBookingsMap()

Returns bookings grouped by **date and office**.

Example:

```json
{
  "2025-01-20": {
    "exeter": ["<@U123>", "<@U456>"],
    "london": ["<@U789>"]
  }
}
```

Used by the Home view to display **attendance lists**.

---

# Booking Workflows

## User Books Desk (No Parking)

1. User clicks **Join**
2. `join_{date}` action triggered
3. `db.bookDesk()`
4. `completeBooking()`
5. `publishHome()`

User immediately sees **Cancel button**.

---

## User Books Exeter (With Parking)

1. User clicks **Join**
2. Parking slots checked
3. Modal displayed

Options:

- Slot 1
- Slot 2
- No parking

4. Modal submitted  
5. Booking stored  
6. UI updated  

---

## User Cancels Booking

1. User clicks **Cancel**
2. Confirmation dialog appears
3. `cancelBooking()`
4. DM button updated
5. Home view refreshed

---

# Scheduled Jobs

## Weekly Digest

Runs **Monday at 8:00am**

Cron:

```
0 8 * * 1
```

Flow:

1. Fetch user preferences
2. Fetch Mon–Fri bookings
3. Build weekly schedule blocks
4. DM each user
5. Store button references

---

## Daily Reminder

Runs **Monday–Thursday at 4:00pm**

Cron:

```
0 16 * * 1-4
```

Flow:

1. Find users not booked tomorrow
2. Check attendance at their office
3. Check parking availability
4. Send reminder DM with Join button

---

# Admin Tools

Admins defined via environment variable:

```
ADMIN_USER_IDS
```

Admin features:

- Book for user
- Cancel booking
- View bookings

---

# Slash Commands

| Command | Example |
|------|------|
| `/book-for` | `/book-for @user 2025-01-20 exeter slot_1` |
| `/cancel-for` | `/cancel-for @user 2025-01-20` |
| `/view-bookings` | `/view-bookings 2025-01-20 exeter` |

---

# Date Handling (CRITICAL)

All dates use:

```
Europe/London
```

Never use:

```
date.toISOString()
```

Instead use helper:

```javascript
toLocaleISOString(date)
```

Example output:

```
2025-01-20
```

---

# Configuration

Environment variables:

```
SLACK_BOT_TOKEN=
SLACK_SIGNING_SECRET=
SLACK_APP_TOKEN=

SUPABASE_URL=
SUPABASE_SERVICE_ROLE_KEY=

ADMIN_USER_IDS=U123,U456
```

---

# Error Handling

Common database errors:

| Error | Meaning |
|------|------|
| `booking_pkey` | User already booked |
| `exeter_parking_pkey` | Parking slot taken |
| `PGRST116` | No rows found |

---

# Race Condition Handling

Problem:

Two users try to book the **last parking slot simultaneously**.

Solution:

```
BEGIN;
INSERT booking;
INSERT parking;
COMMIT;
```

If parking fails → transaction rolls back.

---

# Deployment

Requirements:

- Node.js 16+
- Supabase project
- Slack App with Socket Mode enabled

Install dependencies:

```
npm install
```

Run bot:

```
node index.js
```

---

# Production Deployment

Recommended process manager:

```
pm2 start index.js
```

---

# Production Checklist

- Environment variables set
- Supabase tables created
- RPC functions deployed
- Slack commands configured
- Socket mode enabled
- Admin users configured
- Logging enabled
- Monitoring enabled

---

# Quick Reference

### index.js

- publishHome()
- completeBooking()
- swapBtn()
- fetchBookingsMap()
- weeklyDigestJob()
- dailyReminderJob()

### db.js

- bookDesk()
- cancelBooking()
- getDefaultOffice()
- listBookingsRange()
- listSlotStatus()

### admin.js

- isAdmin()
- buildAdminSection()
- formatBookingsForDate()
- setupAdminCommands()

### homeView.js

- buildHomeView()

---

# Key Principles

- Always refresh UI after database changes
- Always use London timezone
- Admin actions refresh both admin and target user UI
- Database constraints prevent booking conflicts
- `completeBooking()` synchronizes UI state

---

**CuberDesk**  
Slack-powered desk booking for distributed offices.
