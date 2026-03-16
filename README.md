# CuberDesk

System Overview
Purpose: Desk and parking booking system across multiple offices via Slack.

Tech Stack:

Node.js + @slack/bolt (Socket Mode)

Supabase (PostgreSQL)

node-cron for scheduled jobs

Offices: London, Cardiff, Exeter, Manchester, Paris, Barcelona Parking: Only Exeter has optional parking (2 slots)

Architecture


Slack (UI) - Socket Mode
Bot (index.js, admin.js, homeView.js)
Database Layer (db.js)
Supabase (PostgreSQL)
Database Schema
Tables
user_preferences



- user_id (TEXT, PK) - Slack user ID
- default_office (TEXT) - User's preferred office
Stores each user's default office selection.

day_bookings



- user_id (TEXT)
- work_date (TEXT) - Format: YYYY-MM-DD
- office (TEXT)
- PRIMARY KEY (user_id, work_date)
One booking per user per day. Constraint prevents double-booking.

exeter_parking



- work_date (TEXT)
- slot (INTEGER) - 1 or 2
- user_id (TEXT)
- PRIMARY KEY (work_date, slot)
Tracks parking slot assignments. Constraint prevents slot conflicts.

dm_buttons



- user_id (TEXT)
- work_date (TEXT)
- channel (TEXT) - DM channel ID
- ts (TEXT) - Message timestamp
- PRIMARY KEY (user_id, work_date)
Tracks DM messages with buttons so they can be updated when bookings change.

Database Functions (RPC)
book_desk_with_optional_parking(p_user_id, p_work_date, p_office, p_slot)

Atomic transaction that inserts booking + parking (if slot provided)

Prevents race conditions

Returns error if user already booked or slot taken

cancel_booking(p_user_id, p_work_date)

Atomic transaction that deletes booking + parking

Cascades deletion to both tables

pop_dm_button(p_user_id, p_work_date)

Returns and deletes DM button record

Used when updating DM messages

Core Files
1. index.js (Main Bot Logic)
Key Sections:

Configuration:



const OFFICE_CONFIG = {
  'exeter': { parking_is_optional: true, parking_slots: 2 },
  'london': { parking_is_optional: false },
  // ... other offices
};
Defines which offices have parking.

publishHome(client, userId)

Fetches user's bookings for next 2 weeks

Calls buildHomeView() to generate UI blocks

Publishes to user's app home

fetchBookingsMap()

Gets all bookings for next 2 weeks

Groups by date and office

Returns: { "2025-01-20": { "exeter": ["<@U123>", "<@U456>"] } }

Event Handlers:

app_home_opened - User opens app → Publish home view

default_office_select - User changes office → Update DB → Refresh home

join_{date} - User clicks Join → Check parking → Book or show modal

cancel_{date} - User clicks Cancel → Delete booking → Refresh UI

Exeter Parking Flow:



// If exeter and parking available:
1. Show modal with parking options (Slot 1, Slot 2, No parking)
2. User submits → bookDesk(user, date, exeter, slot)
3. completeBooking() updates UI
completeBooking(userId, date, office, slot, client)

Updates user's DM button (if exists) using swapBtn()

Refreshes user's app home

Called after every successful booking

swapBtn(blocks, iso, toJoin, office, slot)

Modifies Slack blocks to change Join ↔ Cancel button

Adds office/parking context blocks if booking exists

Returns modified blocks for chat.update()

Scheduled Jobs:

weeklyDigestJob() - Mondays 8am



1. Get all user preferences
2. Get bookings for Mon-Fri
3. For each user:
   - Build blocks showing their 5-day status
   - Post DM with Join/Cancel buttons
   - Save button references to dm_buttons table
dailyReminderJob() - Mon-Thu 4pm



1. Get users NOT booked for tomorrow
2. Show who IS booked at their office
3. Show parking availability (if Exeter)
4. Post DM with Join button
5. Save button reference
2. db.js (Database Layer)
All functions are async and throw errors on failure.

User Preferences:

getDefaultOffice(userId) → Returns office string or null

setDefaultOffice(userId, office) → Upserts preference

getAllUserPreferences() → Returns Map of userId → office

Bookings:

bookDesk(userId, date, office, slot=null) → Calls RPC, handles parking

cancelBooking(userId, date) → Calls RPC, deletes booking+parking

listBookingsRange(startDate, endDate) → Array of all bookings in range

getUserBookingForDate(userId, date) → Single booking or null

DM Buttons:

saveButton(userId, date, channel, ts) → Upserts button reference

popButton(userId, date) → Gets and deletes button (for updates)

Parking:

listSlotStatus(date) → Returns {1: userId, 2: null} (slot availability)

listAllParkingForRange(start, end) → Map of date → slot status

3. homeView.js (UI Generator)
buildHomeView({ userId, defaultOffice, bookingsByDate })

Builds Slack Block Kit JSON for app home:



Structure:
1. Header with user greeting + office dropdown
2. Divider
3. Admin tools (if admin)
4. 10 days of calendar (2 weeks, Mon-Fri only):
   - Date header
   - Office attendance list
   - Join or Cancel button for user
   - Divider
Returns Slack home view JSON.

4. admin.js (Admin Functions)
Setup:



const ADMIN_USER_IDS = process.env.ADMIN_USER_IDS.split(',');
isAdmin(userId) → Boolean
UI Component:



buildAdminSection(userId) → Returns blocks with dropdown:
  - 📅 Book for User
  - ❌ Cancel Booking
  - 👀 View Bookings
Dropdown Handler:



admin_action_select → Routes to modal:
  - "admin_booking" → openAdminBookingModal()
  - "admin_cancel" → openAdminCancelModal()
  - "admin_view" → openAdminViewModal()
Admin Booking Flow:

Admin selects user + date + office + parking

Submit → bookDesk(targetUserId, ...)

completeBooking(targetUserId, ...) - Updates target's UI

publishHome(adminUserId) - Refreshes admin's UI

Send confirmation DM to admin

Admin Cancel Flow:

Admin selects user + date

Submit → cancelBooking(targetUserId, date)

publishHome(targetUserId) - Updates target's UI

publishHome(adminUserId) - Refreshes admin's UI

Send confirmation DM to admin

View Bookings:



formatBookingsForDate(db, date, office, config):
1. Fetch bookings for date
2. Filter by office (if specified)
3. Group by office
4. Fetch parking status for offices with parking
5. Build formatted Slack blocks showing:
   - Office header with count
   - User list
   - Parking assignments
   - Total count
Slash Commands:

/book-for @user YYYY-MM-DD office [slot_1|slot_2|no_parking]

/cancel-for @user YYYY-MM-DD

/view-bookings YYYY-MM-DD [office]

All three mirror modal functionality.

Key Workflows
User Books Desk (No Parking)


1. User clicks [Join] → join_{date} action
2. Check if office has parking:
   - No parking → bookDesk(user, date, office, null)
3. completeBooking():
   - Update DM button (if exists)
   - publishHome(user)
4. User sees [Cancel] button immediately
User Books Exeter (With Parking)


1. User clicks [Join] → join_{date} action
2. Check exeter + available slots
3. Show modal with radio buttons (Slot 1, Slot 2, No parking)
4. User submits → exeter_booking_submit
5. bookDesk(user, date, exeter, slot)
6. completeBooking() + publishHome()
7. User sees [Cancel] with parking info
User Cancels Booking


1. User clicks [Cancel] → cancel_{date} action
2. Confirmation dialog appears
3. User confirms → cancelBooking(user, date)
4. popButton() to get DM reference
5. Update DM button: Cancel → Join
6. publishHome(user)
7. User sees [Join] button
Admin Books for Someone


1. Admin opens dropdown → "📅 Book for User"
2. Modal: Select user, date, office, parking
3. Submit → bookDesk(targetUser, date, office, slot)
4. completeBooking(targetUser, ...) - Target's UI updates
5. publishHome(adminUser) - Admin's UI updates
6. Target user sees booking with [Cancel] button
7. Target can cancel anytime (no difference from self-booking)
Weekly Digest (Monday 8am)


1. Get all user preferences (userId → office)
2. Get all bookings for Mon-Fri this week
3. Get parking status for whole week
4. For each user:
   a. Build 5 rows (Mon-Fri)
   b. Each row shows: Date | Status | Join/Cancel button
   c. Post DM with all 5 days
   d. Save button references for each day
Daily Reminder (Mon-Thu 4pm)


1. Get users NOT booked for tomorrow
2. For each unbooked user:
   a. Check who IS booked at their office tomorrow
   b. Check parking availability (if Exeter)
   c. Build message: "Nobody booked yet" or "John, Jane are coming"
   d. Post DM with [Join office] button
   e. Save button reference
Important Functions Explained
swapBtn(blocks, iso, toJoin, office=null, slot=null)
Purpose: Toggle DM message button between Join and Cancel

How it works:

Deep clone blocks (avoid mutation)

Find status block by block_id: day_status_{iso}

Find actions block by block_id: day_actions_{iso}

Update status text: "Not booked" ↔ "You're in ✅"

Swap button: Join ↔ Cancel

If booking (not join), insert context blocks:

Office emoji + name

Parking slot (if applicable)

Return modified blocks

Why needed: DM messages persist. When booking changes, we update the existing message so user sees current state.

completeBooking(userId, date, office, slot, client)
Purpose: Sync UI after booking is created

Flow:

Get DM button reference: popButton(userId, date)

If button exists:

Fetch original message

Call swapBtn() to change Join → Cancel

Update message with new blocks

Save new timestamp

Refresh app home: publishHome(client, userId)

Critical: This is called after EVERY booking (user or admin) to ensure UI consistency.

publishHome(client, userId)
Purpose: Rebuild and publish user's app home

Flow:

Fetch user's default office

Call fetchBookingsMap() for next 2 weeks

Call buildHomeView() with data

Publish to Slack via client.views.publish()

When called:

User opens app home

User changes default office

User books/cancels

Admin books/cancels for user

Any action affecting user's bookings

fetchBookingsMap()
Purpose: Get all bookings organized by date and office

Returns:



{
  "2025-01-20": {
    "exeter": ["<@U123>", "<@U456>"],
    "london": ["<@U789>"]
  },
  "2025-01-21": {
    "exeter": ["<@U123>"]
  }
}
Used by: Home view to show "who's in" for each day

formatBookingsForDate(db, date, office, config)
Purpose: Generate rich booking report for admins

Flow:

Fetch all bookings for date

Filter by office (if specified)

Group by office: { exeter: [...], london: [...] }

For each office with parking, fetch slot status

Build Slack blocks showing:

Header with date

Per-office sections with user lists

Parking slot assignments

Total count

Returns: { blocks: [...], text: "Bookings for..." }

Date Handling
CRITICAL: All dates use Europe/London timezone.

Key Utilities (utils/dates.js):



toLocaleISOString(date) 
// Returns: "YYYY-MM-DD" in London timezone
// Used for ALL database date keys
mondayOfWeek(date)
// Returns Date object for Monday of given week
addDays(date, n)
// Adds n days to date, returns new Date
friendlyDate(iso)
// "2025-01-20" → "Monday, 20 January 2025"
DAY_NAMES = ["Sunday", "Monday", ...]
Why this matters:

User in London books "today" at 11pm → Uses London date, not UTC

UTC might already be next day

All dates stored as strings, parsed in London timezone

Configuration
Environment Variables (.env)


# Slack
SLACK_BOT_TOKEN=xoxb-...
SLACK_SIGNING_SECRET=...
SLACK_APP_TOKEN=xapp-...
PORT=3000
# Supabase
SUPABASE_URL=https://....supabase.co
SUPABASE_SERVICE_ROLE_KEY=...
# Admin users (comma-separated Slack user IDs)
ADMIN_USER_IDS=U01ABC123,U02DEF456
Office Configuration (index.js)


const OFFICE_CONFIG = {
  'exeter': { parking_is_optional: true, parking_slots: 2 },
  'london': { parking_is_optional: false },
  'cardiff': { parking_is_optional: false },
  'manchester': { parking_is_optional: false },
  'paris': { parking_is_optional: false },
  'barcelona': { parking_is_optional: false }
};
To add parking to another office:

Add slots to OFFICE_CONFIG

Create parking table (see exeter_parking schema)

Update RPC functions to handle new table

Update admin.js parking options

Cron Schedules


// Weekly digest - Mondays at 8am London time
cron.schedule("0 8 * * 1", weeklyDigestJob, { timezone: "Europe/London" });
// Daily reminder - Mon-Thu at 4pm London time
cron.schedule("0 16 * * 1-4", dailyReminderJob, { timezone: "Europe/London" });
Error Handling
Database Errors
Common errors:

booking_pkey - User already has booking for this date

exeter_parking_pkey - Parking slot already taken

PGRST116 - No rows found (not always an error)

Pattern:



try {
  await db.bookDesk(...);
} catch (err) {
  if (err.message.includes('booking_pkey')) {
    // User already booked
  } else if (err.message.includes('exeter_parking_pkey')) {
    // Slot taken (race condition)
  }
  // Show error to user
}
Race Conditions
Problem: Two users try to book last parking slot simultaneously

Solution: Database-level transactions in RPC functions



-- book_desk_with_optional_parking uses:
BEGIN;
INSERT INTO day_bookings ...;
INSERT INTO exeter_parking ...; -- Fails if slot taken
COMMIT;
-- If any fails, both rollback
Missing publishHome
Symptom: UI doesn't refresh after action

Cause: publishHome not passed to setupAdminCommands()

Fix:



admin.setupAdminCommands(
  app, db, completeBooking, openDm, 
  OFFICE_CONFIG, 
  publishHome // ← Must be defined before this call
);
Adding New Features
Add a New Office
Update homeView.js:



options: ["london","cardiff","exeter","manchester","paris","barcelona","NEWOFFICE"]
Update index.js:



const OFFICE_CONFIG = {
  // ...
  'newoffice': { parking_is_optional: false }
};
Done! Office appears in dropdowns and works immediately.

Add Parking to an Office
Create parking table:



CREATE TABLE newoffice_parking (
  work_date TEXT,
  slot INTEGER,
  user_id TEXT,
  PRIMARY KEY (work_date, slot)
);
Update RPC function:



-- In book_desk_with_optional_parking:
IF p_office = 'newoffice' AND p_slot IS NOT NULL THEN
  INSERT INTO newoffice_parking ...;
END IF;
-- In cancel_booking:
DELETE FROM newoffice_parking WHERE user_id = p_user_id AND work_date = p_work_date;
Update OFFICE_CONFIG:



'newoffice': { parking_is_optional: true, parking_slots: 3 }
Update db.js: Add query logic in listSlotStatus() and listAllParkingForRange() for new table.

Add a New Admin Command
Add to admin.js:



app.command("/my-command", async ({ ack, command, client }) => {
  await ack();
  if (!isAdmin(command.user_id)) return;
  // Your logic
});
Configure in Slack:

Go to api.slack.com/apps → Your app → Slash Commands

Add new command with your endpoint

Modify Weekly Digest Days
Current: Mon-Fri

Change to Mon-Thu:



// In weeklyDigestJob()
for (let i = 0; i < 4; i++) { // Was: i < 5
  const d = addDays(monday, i);
  // ...
}
Troubleshooting
"No bookings showing in app home"
Check:

fetchBookingsMap() returning data?

Add: console.log('Bookings:', bookingsByDate)

Date format correct?

Must be "YYYY-MM-DD" in London timezone

Database has data?

Query: SELECT * FROM day_bookings WHERE work_date >= '2025-01-20'

"Buttons don't update in DMs"
Check:

dm_buttons table has records?

SELECT * FROM dm_buttons WHERE user_id = 'U123'

popButton() returning data?

Message timestamp still valid?

Slack messages expire after ~90 days

"Admin tools not showing"
Check:

User in ADMIN_USER_IDS?

console.log(ADMIN_USER_IDS)

Environment variable set?

echo $ADMIN_USER_IDS

Bot restarted after .env change?

"Parking slot conflict"
Cause: Race condition - two requests at exact same time

Solution: Already handled by database constraint

First request wins

Second request gets error

Error shown to user: "Slot just taken, try again"

"Home view not refreshing"
Check:

publishHome passed to admin setup?

publishHome defined before setupAdminCommands() call?

Try-catch swallowing errors?

Check logs for errors

Security Considerations
Admin Access
Admin user IDs stored in environment variable

Checked on every admin action: if (!isAdmin(userId)) return;

No admin data in database (stateless admin check)

To add admin: Add their Slack user ID to .env and restart bot

Database Access
Uses Supabase service role key (full access)

All queries server-side (users never access DB directly)

RPC functions enforce data integrity with constraints

Slack Tokens
Bot token: Read/write access to workspace

Signing secret: Verifies requests from Slack

App token: Required for Socket Mode

All must be kept secret (never commit to git)

Performance Considerations
Database Queries
Optimized:

fetchBookingsMap() - Single query for 2 weeks

listSlotStatus() - Single query per date

Indexes on (user_id, work_date) for fast lookups

Potential bottleneck:

Weekly digest: Loops through all users

Solution: Currently fast (<100 users), add batching if needed

Memory
DM Button Storage:

dm_buttons table grows over time

Old messages (>90 days) can't be updated anyway

Cleanup: Periodically delete old records:



DELETE FROM dm_buttons WHERE work_date < NOW() - INTERVAL '90 days';
Testing
Manual Testing Checklist
User Flows:

[ ] Book desk (no parking office)

[ ] Book desk with parking

[ ] Cancel booking

[ ] Change default office

[ ] Open app home

[ ] Receive weekly digest

[ ] Receive daily reminder

[ ] Click buttons in DMs

Admin Flows:

[ ] Book for another user (modal)

[ ] Book for another user (slash command)

[ ] Cancel booking (modal)

[ ] Cancel booking (slash command)

[ ] View bookings (all offices)

[ ] View bookings (filtered)

Edge Cases:

[ ] Try to double-book (should error)

[ ] Try to book same parking slot (should error)

[ ] Cancel non-existent booking (should handle gracefully)

[ ] Book for past date (currently allowed - add validation if needed)

Test Data Setup


-- Add test user
INSERT INTO user_preferences (user_id, default_office) 
VALUES ('U_TEST_123', 'exeter');
-- Add test booking
INSERT INTO day_bookings (user_id, work_date, office)
VALUES ('U_TEST_123', '2025-01-20', 'exeter');
-- Add test parking
INSERT INTO exeter_parking (work_date, slot, user_id)
VALUES ('2025-01-20', 1, 'U_TEST_123');
Deployment
Requirements
Node.js 16+

npm packages: @slack/bolt, @supabase/supabase-js, node-cron, dotenv

Supabase project with schema set up

Slack app with Socket Mode enabled

Setup Steps
Supabase project

Created tables (user_preferences, day_bookings, exeter_parking, dm_buttons)

Created RPC functions

URL and service role key

Created Slack app

Enabled Socket Mode

Added bot scopes: chat:write, users:read, commands

Installed to workspace

Got tokens

Configure environment

.env

Add admin user IDs

Run bot



npm install
node index.js
Verify

Check logs for "⚡️ CuberDesk bot running"

Open app home in Slack

Test booking a desk

Production Deployment
 PM2 for process management

Monitoring:

all actions logged (already implemented)

Monitored Supabase connection

Alert on bot disconnection

Quick Reference
Key Functions by File
index.js:

publishHome(client, userId) - Rebuild app home

fetchBookingsMap() - Get all bookings for 2 weeks

completeBooking(...) - Sync UI after booking

swapBtn(...) - Toggle DM button

weeklyDigestJob() - Monday 8am job

dailyReminderJob() - Mon-Thu 4pm job

db.js:

bookDesk(user, date, office, slot) - Create booking

cancelBooking(user, date) - Delete booking

getDefaultOffice(user) - Get preference

listBookingsRange(start, end) - Query bookings

listSlotStatus(date) - Get parking availability

admin.js:

isAdmin(userId) - Check admin status

buildAdminSection(userId) - Generate UI blocks

formatBookingsForDate(...) - Generate booking report

setupAdminCommands(...) - Register all handlers

homeView.js:

buildHomeView({ userId, defaultOffice, bookingsByDate }) - Generate home UI

Common Patterns
Update UI after action:



await db.bookDesk(...);
await completeBooking(...);
await publishHome(client, userId);
Admin action with dual refresh:



await db.bookDesk(targetUser, ...);
await completeBooking(targetUser, ...);
await publishHome(client, targetUserId); // Target's UI
await publishHome(client, adminUserId);  // Admin's UI
Error handling:



try {
  await db.bookDesk(...);
} catch (err) {
  if (err.message.includes('_pkey')) {
    // Constraint violation
  }
  // Show error to user
}
Summary
This bot manages desk bookings via Slack using:

Database: Supabase with transactional RPC functions

UI: Slack Block Kit (app home + DMs)

Automation: Cron jobs for digests and reminders

Admin: Dropdown menu with full booking management

Key concepts:

All dates in Europe/London timezone

Database constraints prevent conflicts

DM buttons updated via saved references

publishHome() refreshes UI after every change

Admin actions update both admin and target user UIs

Entry points:

index.js (start here)

Read db.js for database operations

Read admin.js for admin features

Read homeView.js for UI generation

For questions, check logs and database state first!

CuberDesk - Developer Quick Reference
File Structure


├── index.js          Main bot logic, event handlers, cron jobs
├── admin.js          Admin dropdown, modals, slash commands
├── homeView.js       App home UI generator
├── db.js             Supabase database wrapper
└── utils/
    ├── dates.js      Date utilities (London timezone)
    └── strings.js    String formatters
Database Tables
Table

Purpose

Key Constraint

user_preferences

Default office per user

PK: user_id

day_bookings

Desk reservations

PK: (user_id, work_date)

exeter_parking

Parking slots

PK: (work_date, slot)

dm_buttons

DM message references

PK: (user_id, work_date)

Key Functions
User Actions


// Book desk
await db.bookDesk(userId, "2025-01-20", "exeter", 1);
await completeBooking(userId, date, office, slot, client);
// Cancel booking
await db.cancelBooking(userId, date);
await publishHome(client, userId);
// Change office
await db.setDefaultOffice(userId, "london");
await publishHome(client, userId);
Admin Actions


// Book for someone (refresh BOTH UIs)
await db.bookDesk(targetUserId, date, office, slot);
await completeBooking(targetUserId, ...);
await publishHome(client, targetUserId);  // Target
await publishHome(client, adminUserId);   // Admin
// Cancel for someone (refresh BOTH UIs)
await db.cancelBooking(targetUserId, date);
await publishHome(client, targetUserId);  // Target
await publishHome(client, adminUserId);   // Admin
UI Updates


// Refresh app home
await publishHome(client, userId);
// Update DM button
const rec = await db.popButton(userId, date);
await client.chat.update({
  channel: rec.channel,
  ts: rec.ts,
  blocks: swapBtn(blocks, date, isJoining, office, slot)
});
Common Issues
Problem

Solution

UI not refreshing

Check publishHome passed to setupAdminCommands

Date format wrong

Use toLocaleISOString(date) from utils/dates

Parking conflict

Database constraint handles it - show error to user

DM button not updating

Check dm_buttons table has record

Admin tools not showing

Verify user in ADMIN_USER_IDS env var

Configuration
.env Required Variables


SLACK_BOT_TOKEN=xoxb-...
SLACK_SIGNING_SECRET=...
SLACK_APP_TOKEN=xapp-...
SUPABASE_URL=...
SUPABASE_SERVICE_ROLE_KEY=...
ADMIN_USER_IDS=U123,U456
Office Config (index.js)


const OFFICE_CONFIG = {
  'exeter': { parking_is_optional: true, parking_slots: 2 },
  'london': { parking_is_optional: false }
};
Cron Schedules
Weekly Digest: Monday 8am (0 8 * * 1)

Daily Reminder: Mon-Thu 4pm (0 16 * * 1-4)

Date Handling (CRITICAL)


// Always use Europe/London timezone
const { toLocaleISOString, mondayOfWeek, addDays } = require("./utils/dates");
// Get today in London
const today = toLocaleISOString(new Date()); // "2025-01-20"
// Never use date.toISOString() directly!
Slash Commands
Command

Admin Only

Example

/book-for

Yes

/book-for @user 2025-01-20 exeter slot_1

/cancel-for

Yes

/cancel-for @user 2025-01-20

/view-bookings

Yes

/view-bookings 2025-01-20 exeter

Event Handlers
Event/Action

Handler

Purpose

app_home_opened

index.js

Show calendar

default_office_select

index.js

Update preference

join_{date}

index.js

Book desk

cancel_{date}

index.js

Cancel booking

admin_action_select

admin.js

Route dropdown

admin_booking_submit

admin.js

Create booking

admin_cancel_submit

admin.js

Delete booking

Error Messages


// Common database errors
'booking_pkey'         → User already booked for this date
'exeter_parking_pkey'  → Parking slot already taken
'PGRST116'             → No rows found (not always error)
Adding Features
New Office
Add to homeView.js options array

Add to OFFICE_CONFIG in index.js

New Admin Command
Add handler in admin.js

Configure in Slack app settings

New Scheduled Job


cron.schedule("0 9 * * 5", fridayJob, { timezone: "Europe/London" });
Testing Commands


// Trigger jobs manually (index.js exports these)
const { runWeeklyDigestNow, runReminderNow } = require('./index');
runWeeklyDigestNow();
Logs to Watch


[event] app_home_opened by U123
[action] join_2025-01-20 by U123
[booking] User U123 booked desk at exeter for 2025-01-20
[admin_booking] U456 booked U123 at exeter for 2025-01-20 (slot: 1)
Flow Diagram


User Click → Event Handler → db.js → Supabase
                ↓
           completeBooking()
                ↓
           Update DM + publishHome()
                ↓
           User sees updated UI
Database RPC Functions


book_desk_with_optional_parking(user_id, work_date, office, slot)
  → Atomic: INSERT booking + parking
cancel_booking(user_id, work_date)
  → Atomic: DELETE booking + parking
pop_dm_button(user_id, work_date)
  → Return and DELETE button record
Quick Debug


// Check booking exists
SELECT * FROM day_bookings WHERE user_id = 'U123' AND work_date = '2025-01-20';
// Check parking
SELECT * FROM exeter_parking WHERE work_date = '2025-01-20';
// Check DM buttons
SELECT * FROM dm_buttons WHERE user_id = 'U123';
// Check admin status
console.log(ADMIN_USER_IDS.includes('U123'));
Production Checklist
[ ] All .env variables set

[ ] Admin user IDs configured

[ ] Supabase tables created

[ ] RPC functions deployed

[ ] Slack slash commands configured

[ ] Bot has required OAuth scopes

[ ] Socket Mode enabled

[ ] Cron timezone set to Europe/London

[ ] Process manager (PM2/systemd) configured

[ ] Logs being captured

[ ] Error monitoring set up

Support Resources
Slack API docs: api.slack.com

Block Kit builder: api.slack.com/block-kit

Supabase docs: http://supabase.com/docs 

Bolt framework: slack.dev/bolt-js

Key Takeaways
Always refresh UI after database changes

Use London timezone for all dates

publishHome both users for admin actions

Database constraints prevent conflicts

completeBooking() syncs everything

 

 
