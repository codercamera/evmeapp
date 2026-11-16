# EV Charging Expenses — System Specification

## 1. About this document

This document is the source of truth for building EV Charging Expenses as a real system. It defines the features, the data model, the rules and the behaviour of every screen. An implementer, human or AI agent, must be able to build the system from this document alone.

- The words **must** and **must not** mark requirements.
- The HTML mockup in this repository is a **visual reference only**. Use it for layout, spacing, colours and interaction feel. Do not copy its code, its data handling or its sign-in, which are placeholders. Where the mockup and this document disagree, this document wins.
- The data model is given as SQLite DDL in section 4. The rest of the technology stack is the implementer's choice, within the constraints in section 14.

## 2. Product overview

EV Charging Expenses lets an electric-vehicle owner log each charging session and see what charging costs. A user records the cost, the energy, the vehicle, the date and the place, with an optional photo. The app summarises spending by period, by home versus public charging, and by charging provider, and tracks it against a monthly budget.

**In scope**

- Accounts: register, sign in, sign out.
- A dashboard of spending figures for a chosen period, a monthly budget and a six-month chart.
- A filterable list of charging sessions, with add, edit and delete.
- A quick way to add a session: defaults from the last session, recent places, and an in-app keypad.
- One optional photo per session, cropped with a square frame.
- Management of vehicles, charging providers and provider branches.
- A preconfigured list of charging providers for every new account.
- Light and dark themes, four font sizes, and English and Thai.
- Installation on a phone's home screen as a progressive web app.

**Out of scope**

- Sharing data between users.
- Import, export or integration with charging networks.
- Currencies other than Thai baht.
- Layouts beyond the phone-width column.
- Password reset by email.

## 3. Accounts and access

### 3.1 Register

- The sign-in screen has a "Create account" link that opens the register form: Username, Password, Confirm password.
- Username: 3 to 32 characters; letters, digits, `.`, `_` and `-`; unique without regard to letter case.
- Password: at least 8 characters. Password and confirmation must match.
- On success the account is created with its starting data (section 4.3), the user is signed in, and the Dashboard opens.
- Each failed rule is reported next to its field. A taken username says so.

### 3.2 Sign in and sign out

- Fields: Username and Password, with a Login button.
- Wrong credentials are rejected with one message that does not reveal which field was wrong. The user stays on the sign-in screen.
- A signed-in user stays signed in across reloads and app restarts for 30 days from their last use, or until they sign out.
- Sign out ends the session on the server, returns to the sign-in screen, and leaves no user data visible or cached on screen.

### 3.3 Security rules

- Passwords must be stored only as a salted hash from a password-hashing algorithm (Argon2id or bcrypt). They must never be logged or returned.
- Sign-in tokens must be random, at least 256 bits, and stored on the server only as a hash.
- Every read and write must be limited to the signed-in user's own rows. A request for another user's record must behave as "not found".
- All validation in this document must be enforced on the server. Client-side checks are for convenience only.
- Sign-in must be rate-limited per username and per address to slow guessing.

## 4. Data model

### 4.1 Conventions

- The database is SQLite, with `PRAGMA foreign_keys = ON` on every connection.
- Every table has an integer primary key `id`. Records refer to each other by `id`, never by name.
- **Money** is stored as whole satang in `INTEGER` columns (1 baht = 100 satang). 285.50 baht is stored as `28550`.
- **Energy** is stored as whole watt-hours in `INTEGER` columns. 18.5 kWh is stored as `18500`. Input allows one decimal of a kWh, so stored values are multiples of 100.
- **Dates** are `TEXT` in `YYYY-MM-DD` form and carry no time of day. **Timestamps** are `TEXT` in UTC ISO 8601 form, for example `2026-11-16T08:30:00Z`.
- Names are trimmed of leading and trailing spaces before saving. Uniqueness of names ignores letter case.
- **Photos** are stored as files on disk, not in the database. The database holds one row per photo that links the session to its file (section 4.6).

### 4.2 Schema

```sql
PRAGMA foreign_keys = ON;

-- Accounts
CREATE TABLE users (
  id             INTEGER PRIMARY KEY AUTOINCREMENT,
  username       TEXT    NOT NULL COLLATE NOCASE UNIQUE
                         CHECK (length(username) BETWEEN 3 AND 32),
  password_hash  TEXT    NOT NULL,
  created_at     TEXT    NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%SZ', 'now'))
);

-- Sign-in sessions (one row per signed-in device)
CREATE TABLE auth_sessions (
  id            INTEGER PRIMARY KEY AUTOINCREMENT,
  user_id       INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  token_hash    TEXT    NOT NULL UNIQUE,
  created_at    TEXT    NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%SZ', 'now')),
  last_used_at  TEXT    NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%SZ', 'now')),
  expires_at    TEXT    NOT NULL
);
CREATE INDEX idx_auth_sessions_user ON auth_sessions(user_id);

-- Per-user settings that follow the account
CREATE TABLE user_settings (
  user_id                INTEGER PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,
  monthly_budget_satang  INTEGER CHECK (monthly_budget_satang IS NULL OR monthly_budget_satang > 0),
  updated_at             TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%SZ', 'now'))
);

-- Vehicles
CREATE TABLE vehicles (
  id          INTEGER PRIMARY KEY AUTOINCREMENT,
  user_id     INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  name        TEXT    NOT NULL COLLATE NOCASE CHECK (length(trim(name)) > 0),
  brand       TEXT,
  model       TEXT,
  plate       TEXT,
  created_at  TEXT    NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%SZ', 'now')),
  UNIQUE (user_id, name)
);

-- Charging providers (each user owns their own list)
CREATE TABLE providers (
  id          INTEGER PRIMARY KEY AUTOINCREMENT,
  user_id     INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  name        TEXT    NOT NULL COLLATE NOCASE CHECK (length(trim(name)) > 0),
  created_at  TEXT    NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%SZ', 'now')),
  UNIQUE (user_id, name)
);

-- Branches of a provider
CREATE TABLE branches (
  id           INTEGER PRIMARY KEY AUTOINCREMENT,
  provider_id  INTEGER NOT NULL REFERENCES providers(id) ON DELETE CASCADE,
  name         TEXT    NOT NULL COLLATE NOCASE CHECK (length(trim(name)) > 0),
  created_at   TEXT    NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%SZ', 'now')),
  UNIQUE (provider_id, name),
  UNIQUE (id, provider_id)            -- target for the composite key on sessions
);

-- Charging sessions
CREATE TABLE sessions (
  id             INTEGER PRIMARY KEY AUTOINCREMENT,
  user_id        INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  vehicle_id     INTEGER NOT NULL REFERENCES vehicles(id) ON DELETE RESTRICT,
  session_date   TEXT    NOT NULL CHECK (session_date GLOB '[0-9][0-9][0-9][0-9]-[0-9][0-9]-[0-9][0-9]'),
  cost_satang    INTEGER NOT NULL CHECK (cost_satang >= 0),
  energy_wh      INTEGER NOT NULL CHECK (energy_wh >= 0 AND energy_wh % 100 = 0),
  type           TEXT    NOT NULL CHECK (type IN ('Home', 'Public')),
  provider_id    INTEGER REFERENCES providers(id) ON DELETE RESTRICT,
  branch_id      INTEGER,
  start_percent  INTEGER CHECK (start_percent BETWEEN 0 AND 100),
  end_percent    INTEGER CHECK (end_percent BETWEEN 0 AND 100),
  duration_min   INTEGER CHECK (duration_min >= 0),
  note           TEXT    NOT NULL DEFAULT '',
  created_at     TEXT    NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%SZ', 'now')),
  updated_at     TEXT    NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%SZ', 'now')),
  -- Home has no provider or branch. Public needs a provider; branch is optional.
  CHECK ((type = 'Home'   AND provider_id IS NULL AND branch_id IS NULL)
      OR (type = 'Public' AND provider_id IS NOT NULL)),
  -- A branch must belong to the session's provider.
  FOREIGN KEY (branch_id, provider_id) REFERENCES branches(id, provider_id) ON DELETE RESTRICT
);
CREATE INDEX idx_sessions_user_date ON sessions(user_id, session_date DESC, created_at DESC);
CREATE INDEX idx_sessions_vehicle   ON sessions(vehicle_id);
CREATE INDEX idx_sessions_provider  ON sessions(provider_id);
CREATE INDEX idx_sessions_branch    ON sessions(branch_id);

-- One optional photo per session. The image itself is a file on disk
-- (section 4.6); this table links the session to that file.
CREATE TABLE session_photos (
  session_id  INTEGER PRIMARY KEY REFERENCES sessions(id) ON DELETE CASCADE,
  guid        TEXT    NOT NULL UNIQUE
                      CHECK (length(guid) = 36 AND guid = lower(guid)),
  mime_type   TEXT    NOT NULL DEFAULT 'image/jpeg' CHECK (mime_type = 'image/jpeg'),
  width       INTEGER NOT NULL CHECK (width  BETWEEN 1 AND 1024),
  height      INTEGER NOT NULL CHECK (height BETWEEN 1 AND 1024),
  size_bytes  INTEGER NOT NULL CHECK (size_bytes BETWEEN 1 AND 1048576),
  created_at  TEXT    NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%SZ', 'now'))
);
```

Rules the schema cannot express, which the server must enforce:

- A session's vehicle and provider must belong to the same user as the session.
- A session's `session_date` must be a real calendar date.
- `updated_at` is set on every change to a session or to the settings row.
- Lists of sessions need only whether a photo exists, which is whether a `session_photos` row exists. They must not read image files.

### 4.3 Starting data for a new account

Creating an account must, in the same transaction:

1. Insert the `users` row.
2. Insert a `user_settings` row with no budget.
3. Insert the preconfigured providers below into `providers` for that user, with no branches.

| Preconfigured provider |
|---|
| OneCharge |
| EV Station PluZ |
| EA Anywhere |
| iGreen+ |
| Evolt |
| Spark |
| Elexa |

- The preconfigured providers are ordinary rows owned by the user. The user can rename them, delete them and add branches to them like any other provider.
- A new account has no vehicles, no branches and no sessions.

### 4.4 What is stored where

| Data | Where | Lifetime |
|---|---|---|
| Account, vehicles, providers, branches, sessions | Database | Until the user deletes them. |
| Photo files | `data/images` on the server, linked from `session_photos` | Until the photo or its session is deleted. |
| Monthly budget | Database (`user_settings`) | Until changed. Follows the account to every device. |
| Theme, language, font size, "swipe tip seen" | The device | Until changed. Not sent to the server. |
| Vehicle selection, dashboard period, session filters, expanded provider | Memory | Reset when the app is reloaded. |

### 4.5 Delete rules

| Record | Rule |
|---|---|
| Session | Deleted outright after confirmation, with its photo row and photo file. |
| Vehicle | Cannot be deleted while any session uses it. |
| Provider | Cannot be deleted while any session uses it. Otherwise it is deleted with its branches. |
| Branch | Cannot be deleted while any session uses it. |

When a delete is blocked, the confirmation dialog explains why and states how many sessions use the item. It offers only a Close button.

### 4.6 Photo files

Photo images are kept as files under `data/images`, not in the database.

**Naming and location**

- When a photo is saved, the server generates a new GUID for it: a random version 4 UUID, written in lowercase with hyphens, for example `3f2a9c1e-7b4d-4e8a-9c2f-1a6b5d0e4f77`. The client never chooses the GUID.
- The file name is the GUID with the extension `.jpg`.
- The file is placed in a sub-folder named after the first three characters of the GUID. The sub-folder is created if it does not exist.
- The full path is therefore `data/images/<first 3 characters>/<guid>.jpg`.

| GUID | File path |
|---|---|
| `3f2a9c1e-7b4d-4e8a-9c2f-1a6b5d0e4f77` | `data/images/3f2/3f2a9c1e-7b4d-4e8a-9c2f-1a6b5d0e4f77.jpg` |
| `a07d55b2-1c3e-4f6a-8b9d-2e4f6a8c0b1d` | `data/images/a07/a07d55b2-1c3e-4f6a-8b9d-2e4f6a8c0b1d.jpg` |

- The three characters are hexadecimal (`0`–`9`, `a`–`f`), so there are at most 4,096 sub-folders.
- The path is always derived from the GUID. It is not stored in the database.

**Link to the database**

- Each photo has one row in `session_photos`, holding the session it belongs to and the file's GUID. That row is the only link between a session and its file.
- A GUID is never reused. Replacing a session's photo creates a new GUID and a new file, and the old file is deleted.

**Keeping files and rows in step**

- Saving: the server writes the file first, then inserts or updates the row in the same transaction as the session. If the transaction fails, the server deletes the file it just wrote.
- Deleting: when a photo is removed, or its session is deleted, the server deletes the row first and removes the file after the transaction commits.
- Deleting an account deletes all of that user's photo files.
- A file with no row, or a row with no file, is an error state. The server must provide a clean-up task that removes files with no row and reports rows with no file.

**Access**

- `data/images` must not be served directly to the web. A photo is returned only by the server, and only to the signed-in user who owns its session.
- The database file and `data/images` together hold the system's data and must be backed up together.

## 5. Navigation

- The sign-in screen is shown first unless the user is already signed in.
- A floating bottom navigation bar has three tabs: **Dashboard**, **Sessions** and **Settings**. It is a rounded pill, inset 16px from the screen edges and the bottom, with a soft shadow. The active tab has a green tint behind its icon and label.
- A floating round add button (`+`) sits above the navigation bar on every signed-in screen and opens the Add Session sheet.
- **Vehicles** and **Chargers** are sub-screens of Settings. Each has a Back button that returns to Settings. The Settings tab stays highlighted while a sub-screen is open.
- The navigation bar and the add button are hidden on the sign-in and register screens.

## 6. Screens

### 6.1 Sign in and register

As defined in section 3. Both screens show the app name and a centred form card.

### 6.2 Dashboard

The header shows the app title and a **Log out** button.

A **vehicle selector** offers "All vehicles" plus each of the user's vehicles. The selection filters the Dashboard and also the Sessions screen. If the selected vehicle is deleted, the selection returns to "All vehicles".

A **period selector** sits beside the vehicle selector. It sets the date range for the period figures. It does not affect the Sessions screen.

| Period | Date range | Compared with |
|---|---|---|
| This month (default) | First day of the current month to today. | The same number of days at the start of last month. |
| Last month | The whole previous calendar month. | The whole month before it. |
| Last 3 months | The two previous calendar months plus the current month to date. | The three calendar months before that range, cut to the same number of days. |
| This year | 1 January to today. | The same number of days at the start of last year. |
| All time | Every session. | No comparison. |

The Dashboard shows these elements, in this order:

| Element | Follows period | Follows vehicle | Content |
|---|---|---|---|
| Total Cost | Yes | Yes | Sum of cost in the period, with the change against the comparison range. A second line shows the all-time total, except when the period is All time. |
| Monthly Budget | No | No | Progress against the budget for the current month. Shown only when a budget is set. |
| Avg. Cost / kWh | Yes | Yes | Cost in the period divided by energy in the period. Shows `—` when the period has no energy. |
| Home vs Public | Yes | Yes | For each type: the cost in the period and the average cost per kWh. |
| By Provider (Public) | Yes | Yes | Cost per provider for public sessions in the period, highest first, top five only. Provider name on the left, cost on the right. Shows "No public sessions" when empty. |
| Monthly Spending | No | Yes | Bar chart of the last six calendar months, ending with the current month, oldest on the left. |
| Recent Sessions | No | Yes | The five most recent sessions as session cards. Shows "No sessions yet" when empty. |

Card labels do not repeat the period. The period selector is the only place it is named.

**Change against the comparison range**

- Shown under the Total Cost figure as an arrow, a percentage and the comparison name, for example `▲ 12% vs last month`.
- The percentage is the difference between the two totals divided by the comparison total, rounded to a whole number.
- An increase is shown in the destructive colour with `▲`. A decrease is shown in the accent colour with `▼`. No change shows `0%` in muted text with no arrow.
- Nothing is shown when the period is All time or when the comparison total is zero.

**Home vs Public**

- The card is a small table: the type name on the left, and its cost with its average cost per kWh on the right, for example `Home` · `820` · `5.40/kWh`.
- A type with no energy in the period shows `—` for its rate.
- When both types have a rate, a final row labelled "Public vs Home" shows the public rate minus the home rate as a signed figure, for example `+2.35/kWh`, or `−0.40/kWh` when public is cheaper.

**Monthly Budget card**

- The card always covers the current calendar month and all vehicles, whatever the selectors show. Its label says so: "Monthly Budget · All vehicles".
- It shows the amount spent, the budget and what remains, for example `1,240 of 2,000 · 760 left`.
- When spending is over the budget, it shows the excess, for example `2,240 of 2,000 · 240 over`.
- A progress bar shows spending as a share of the budget. It is full when spending reaches or passes the budget.
- The bar uses the accent colour below 80% of the budget, the warning colour from 80% up to 100%, and the destructive colour above 100%.
- Tapping the card opens the budget dialog (section 6.7).

**Monthly Spending chart**

- Each of the six months appears exactly once.
- Bar height is proportional to the month with the highest spending. A month with no spending still shows a minimal bar.
- The current month's bar uses the accent colour. The other bars are neutral.
- Each bar is labelled with the short month name.
- When a budget is set and the vehicle selector shows "All vehicles", a dashed horizontal line marks the budget level. The line is hidden when the budget is more than twice the highest month. When the line is shown and the budget is above the highest month, the bars scale to the budget so the line stays inside the chart.

### 6.3 Sessions

- Sessions are listed as session cards, newest date first. Sessions on the same date are ordered newest-created first.
- Three filters combine with AND:
  - **Month:** "All months" plus every month that has at least one session, newest first.
  - **Type:** "All types", Home, Public.
  - **Provider:** "All providers" plus every provider used by at least one session.
- The Dashboard's vehicle selection also applies.
- Shows "No sessions match the filters" when the result is empty.

**Swipe tip**

- A one-line tip sits above the list: "Tap a session to edit it. Swipe right to delete, or left to edit."
- The tip has a close button. It disappears for good on that device when closed or after the user's first completed swipe.

### 6.4 Session card

Used on the Dashboard and the Sessions screen. Each session is a separate card.

- **Left:** the vehicle name in bold, then `date · energy kWh · place`.
  - Place is "Home" for home sessions.
  - Place is `provider · branch` for public sessions, or the provider alone when no branch is set.
- **Right:** a 40px photo thumbnail when the session has a photo, then the cost, a type badge (green for Home, blue for Public) and a chevron (›).
- Battery percentages, duration and note are not shown on the card.

| Action | Result |
|---|---|
| Tap or click the card | Opens the Edit Session sheet. This is the main way to edit, and to delete through the sheet's Delete action. |
| Tap the thumbnail | Opens the photo full screen on a dark backdrop. It does not open the session. Tapping anywhere closes it. |
| Swipe left-to-right | The card follows the finger up to 100px and a red "Delete" background appears on the left. Releasing after 80px or more opens the delete confirmation. A shorter swipe snaps back. |
| Swipe right-to-left | The card follows the finger up to 100px and a green "Edit" background appears on the right. Releasing after 80px or more opens the Edit Session sheet. A shorter swipe snaps back. |
| Right-click | Opens the Edit Session sheet. |

- A touch that begins by moving vertically scrolls the list and must not move the card.
- A touch that begins by moving horizontally moves the card and must not scroll the list.
- A tap that ends a swipe must not also open the sheet.
- Long press does nothing.
- The swipes are shortcuts. Every action they offer is also reachable by tapping the card.

### 6.5 Add Session sheet (quick add)

Opened by the add button. It is a bottom sheet over a dimmed backdrop, titled "Add Session", with a `✕` button. It is only as tall as its content, up to 75% of the screen height. **Cancel** and **Save Session** sit side by side directly below the fields. Closing without saving discards everything.

If the user has no vehicles, the add button must not open the sheet. It tells the user to add a vehicle first and offers to open the Vehicles screen.

**Defaults when the sheet opens**

- Vehicle, type, provider and branch are copied from the user's most recent session (newest date, then newest created). With no sessions yet, the vehicle is the first vehicle and the type is Home.
- Date is today.
- Cost, energy, the optional fields and the photo are empty.

**Content, top to bottom**

1. **Recent places.** A heading "Recent charges" with a "Show all (N)" link, then up to three tiles in one row.
   - The tiles are the three most recently used distinct places. A place is Home, or one provider-and-branch pair. A place used many times appears once.
   - Each tile shows the place name only, wrapping to at most two lines.
   - Tapping a tile sets the type, provider and branch to that place and the vehicle to the vehicle of its most recent session. Cost, energy and date are not changed. The tapped tile is highlighted.
   - The section is hidden when the user has no sessions.
2. **Place row.** Labelled "Place", showing the chosen place and a chevron. Tapping it opens "Choose a place" (section 6.6).
3. **Cost (THB).** Required. Opens the keypad.
4. **Energy (kWh)** with the **photo tile** beside it, on the same line. Energy is required and opens the keypad.
5. **Keypad**, shown only while Cost or Energy is active.
6. **More.** A link that reveals the remaining fields and grows the sheet to 75% of the screen height. It toggles to "Less".
7. **Cancel** and **Save Session.**

**Fields behind More**

| Field | Control | Rules |
|---|---|---|
| Vehicle | Selector | Required. Lists the user's vehicles. |
| Date | Date picker | Required. |
| Start Battery % | Number | Optional. 0–100. |
| End Battery % | Number | Optional. 0–100. |
| Duration minutes | Number | Optional. Zero or more. |
| Note | Text area | Optional. |

**Keypad**

- Tapping the Cost or Energy field, or the keyboard icon at its right edge, opens a keypad inside the sheet, below the two fields. The phone's own keyboard must not appear for these fields.
- Keys: digits 0–9, a decimal point, a backspace key, and **Done**.
- The keypad types into the field that was tapped. That field shows a green border while active. Tapping the other field moves the keypad to it. Done closes the keypad.
- Cost accepts at most two decimals. Energy accepts at most one. Each accepts at most seven digits in total.
- A leading zero is replaced by the next digit. A decimal point typed first becomes `0.`.
- If the keypad makes the content taller than the sheet, the form scrolls inside the sheet. Save must always stay reachable.

**Saving**

- Save validates every field against section 4. Invalid fields are flagged and nothing is saved.
- After a successful save the sheet closes and every screen reflects the new session.

### 6.6 Choose a place

A screen inside the session sheet, titled "Choose a place", with a **Back** link that returns without changing anything. It opens from the Place row and from "Show all (N)".

**Pickers, in a box at the top**

| Control | Rules |
|---|---|
| Type | Selector: Home or Public. |
| Provider | Shown only for Public. Selector listing the user's providers. Required for Public. |
| Branch | Shown only for Public. Selector listing the chosen provider's branches, plus "No branch". Shows "No branches yet" when the provider has none. |
| + Add new branch | Shown only for Public. Opens a small in-app input for the name. The branch is added to the chosen provider and selected. The naming rules of section 6.9 apply. |
| Use this place | Button. Applies the pickers to the session and returns to the form. |

**Places you have used**

- Below the box, a list of every distinct place the user has charged at, most recently used first. N in "Show all (N)" is the number of places.
- Each row shows the place, then the vehicle and date of the most recent session there, with that session's cost and energy on the right.
- Tapping a row chooses that place, exactly as a recent tile does, and returns to the form at once.

### 6.7 Photo

Each session may have one photo, typically of the charger's screen or the receipt.

**Photo tile**

- A square tile, about 56px wide and as tall as the Energy field, sits to the right of the Energy field. Empty, it shows a camera icon and the word "Photo" inside a dashed green border. With a photo, it shows the photo.
- Tapping the tile opens a menu: **Take photo**, **Choose from gallery**, **Remove photo** (only when a photo is attached) and **Cancel**.

**Crop**

- A chosen or taken picture opens a full-screen crop view on a black background, with **Cancel**, the title "Crop photo" and **Done**.
- The whole picture is shown, scaled to fit, never cut off.
- A square frame lies over the picture. Everything outside the frame is dimmed.
- The user drags the frame to move it. It cannot leave the picture.
- Three buttons, **S**, **M** and **L**, set the frame's side to 45%, 65% and 85% of the picture's shorter side. M is the default. Changing size keeps the frame's position.
- **Done** keeps only what is inside the frame, scaled to 1024 × 1024 pixels and saved as JPEG. **Cancel** keeps the previous photo, or none.

**Storage and display**

- The cropped photo is uploaded when the session is saved. It must be at most 1 MB.
- The server must check that the upload is a JPEG no larger than 1024 × 1024 pixels and must reject anything else.
- The server saves the image as a file under `data/images` and records it in `session_photos`, as defined in section 4.6.
- Removing the photo and saving deletes its row and its file.
- A session with a photo shows the thumbnail on its card (section 6.4).

### 6.8 Edit Session sheet

Opened by tapping a session card, by the edit swipe or by right-click. It is the same sheet as Add Session, with these differences:

- The title is "Edit Session", and the fields are filled from the session.
- The sheet is 75% of the screen height. The fields behind More are always shown, and there is no More link.
- The recent places section is hidden. The Place row is shown and opens "Choose a place".
- The photo tile shows the session's photo, if any.
- A red **Delete Session** button sits at the foot of the form. It closes the sheet and opens the delete confirmation.

### 6.9 Settings

Six cards, in this order:

- **Dark Mode:** a toggle switch.
- **Language:** a selector listing each language by its own name ("English", "ไทย"). The change applies at once, without a reload.
- **Font Size:** a selector with 90%, 100% (default), 120% and 140%. It scales all text, spacing and controls together, and applies at once.
- **Monthly Budget:** shows the current budget, for example "2,000 / month", or "Not set". Opens the budget dialog.
- **Vehicles:** opens the Vehicles screen.
- **Chargers:** opens the Chargers screen.

**Budget dialog** (centred modal): one amount field in THB, with Cancel and Save.

- Saving an amount greater than zero sets the budget.
- Saving an empty field removes the budget.
- Zero, a negative amount or more than two decimal places is rejected with an error message, and the dialog stays open.
- There is one budget per user. It covers all vehicles together and applies to every month until changed.

**Vehicles screen**

- One card per vehicle in a single column: an icon, the display name, then `brand · model · plate`, with **Edit** and **Delete** buttons. Empty details are left out along with their separators.
- **+ Add** in the header opens the vehicle dialog. Shows "No vehicles yet" when empty.
- **Vehicle dialog** (centred modal): Display Name, Brand, Model, License Plate, with Cancel and Save.
- Display Name is required and must be unique for the user. A missing or duplicate name is rejected with an error message, and the dialog stays open.
- Renaming a vehicle changes the name shown on all of its sessions.

**Chargers screen**

- One card per provider, showing its name and branch count, with **Edit** and **Delete** buttons.
- Tapping a provider expands it to list its branches. Only one provider is expanded at a time.
- Each branch has **Edit** and **Delete** buttons. An expanded provider also has **+ Add Branch**.
- **+ Add** in the header opens the provider dialog.
- Shows "No providers yet" when empty, and "No branches yet" inside a provider with none.
- **Provider dialog** and **branch dialog** (centred modals) each have one name field, with Cancel and Save.
- A provider name must be unique for the user. A branch name must be unique within its provider. A missing or duplicate name is rejected with an error message, and the dialog stays open.
- Renaming a provider or branch changes the name shown on all sessions that use it.

### 6.10 Delete confirmation

One shared centred modal is used for sessions, vehicles, providers and branches. It shows a title, a message identifying the item, a **Cancel** button and a red **Delete** button. Nothing is deleted until Delete is chosen.

- For a session, the message shows its date, vehicle and cost.
- For an item that cannot be deleted, the dialog follows section 4.5.

## 7. Calculation and display rules

- **Today** and **current month** are taken from the user's device time zone.
- A session belongs to the month of its `session_date`.
- A period's date range includes both its first and last day.
- A comparison range that is cut to "the same number of days" starts on the first day of the earlier period and never runs past that period's last day.
- Section 6.2 states which Dashboard elements follow the vehicle selection and which follow the period.
- Totals, averages, percentages and differences are calculated from stored satang and watt-hours. Rounding happens only when a value is displayed.
- Average cost per kWh is total satang ÷ total watt-hours × 10, which gives baht per kWh.

Amounts are shown without a currency symbol. All amounts are in Thai baht, and input labels say so, for example "Cost (THB)".

Numbers and month names follow the selected language's conventions. The examples are for English.

| Value | Format | Example |
|---|---|---|
| Cost totals and session card cost | Whole baht, rounded half up, with thousands separators | `1,240` |
| Average cost per kWh | Two decimals | `6.85` |
| Change against comparison | Whole percentage | `12%` |
| Energy | Up to one decimal, followed by `kWh` | `32.5 kWh` |
| Date | `YYYY-MM-DD` | `2026-11-14` |
| Month filter option | `YYYY-MM` | `2026-11` |

## 8. Visual design

The mockup is the reference for the look of every screen. The fixed rules are:

- **Layout:** a single column capped at 430px wide and centred.
- **Accent colour:** green `#16a34a` for primary buttons, key figures, the active tab, the add button and the current-month bar.
- **Destructive colour:** red `#dc2626` for delete actions, the delete swipe, a spending increase and an exceeded budget.
- **Warning colour:** amber `#d97706` for a budget that is at least 80% used, and for the budget line on the chart.
- **Cards:** 16px corner radius with a soft elevated shadow.
- **Typography:** the system font. Screen titles are 20px bold and key figures are 28px bold.
- **Form controls:** full width, 12px corner radius, 16px text.
- **Selectors:** every drop-down uses the app's own control, never the browser's default. Closed, it looks like the other form controls, with the chosen value and a chevron. Tapping it opens a bottom sheet listing the options, with the current one in the accent colour and ticked. Inside a form, the sheet shows the field's label above the list. Choosing an option, tapping outside the sheet or pressing Escape closes it.
- **Bottom sheets:** rounded top corners of 20px, over a dimmed backdrop.

**Themes**

| Token | Light | Dark |
|---|---|---|
| Background | `#f5f7fa` | `#111827` |
| Text | `#1a1a1a` | `#f3f4f6` |
| Card | `#ffffff` | `#1f2937` |
| Border | `#e0e0e0` | `#374151` |
| Muted text | `#666666` | `#9ca3af` |
| Secondary surface | `#f0f0f0` | `#374151` |

Every surface, including every dialog, sheet and list, must follow the active theme. Light is the default.

## 9. Languages and translation

The app supports English (the default) and Thai.

**What is translated**

- All interface text: labels, buttons, messages, placeholders, empty states and errors.
- Number separators and month names, which follow the selected language.

**What is not translated**

- Anything the user typed: vehicle, provider and branch names, plates and notes.
- The preconfigured provider names in section 4.3.
- Dates, which stay `YYYY-MM-DD`, and the unit `kWh`.
- Stored values. A session's type is stored as `Home` or `Public` whatever language is shown.

**Rules for interface text**

- Every visible phrase must come from a single string table per language. No interface text may be written directly in screen code.
- A phrase must be stored whole, with named placeholders for the values inside it, for example `{spent} of {budget} · {rest} left`. Sentences must not be assembled from fragments, because word order differs between languages.
- Where wording depends on a condition, each outcome must be its own complete phrase.
- Where wording depends on a count, the phrase must use the language's plural rules, for example `1 branch` and `2 branches`.
- A value should be shown as a labelled figure in preference to a sentence wherever that is clear.
- A missing translation falls back to English.
- Layout must not depend on text length. Labels may wrap, and figures stay aligned to the right.
- Server error messages must be returned as codes, and the client shows the phrase for the code in the selected language.

Adding a language must require only a new string table and an entry in the language selector. The Thai wording must be reviewed by a native speaker before release.

## 10. Content safety

All user-entered text (names, plates, notes) must be displayed as plain text. It must never be interpreted as markup or code. Database access must use parameterised queries.

## 11. Accessibility and input

- Every action must be reachable without touch gestures, by tap or click.
- The page must allow the user to zoom.
- Dialogs and sheets must be closable with their Cancel, Back or `✕` control, and with the Escape key.
- While any dialog, sheet or selector list is open, the page behind it must not scroll, and it must return to the same place when the dialog closes. Scrolling inside the dialog must not scroll the page.
- Controls must have a touch target of at least 40px.

## 12. Errors

- Validation errors appear in the app, next to the field or inside the dialog. The browser's own alert and prompt boxes must not be used.
- If a save fails because the server cannot be reached, the form stays open with its content, and a message says the change was not saved.
- If the sign-in session has expired, the app returns to the sign-in screen and says so.

## 13. Installable app

The app must be installable on a phone's home screen as a progressive web app.

**Installing**

- The app must be served over HTTPS.
- On Android, the browser offers "Install app" or "Add to Home screen". On iPhone and iPad, the user installs from Safari with Share, then "Add to Home Screen".
- The installed app is named "EV Charging" under its icon and "EV Charging Expenses" in full.

**When installed**

- The app opens in its own window with no browser address bar, in portrait.
- The phone's status bar matches the app background and follows the theme.
- The navigation bar, the add button and the bottom sheets must stay clear of the phone's home indicator and rounded corners.

**Icons**

- A green rounded square with a white lightning bolt.
- Provided as PNG at 192px and 512px, a 512px maskable version with the bolt inside the safe zone, and a 180px version for iPhone.

**Offline**

- After the first visit, the app's screens must open with no connection.
- When online, the app always loads the latest version. The saved copy is used only when the network is unavailable.
- Without a connection the app says it is offline. Adding, editing and deleting need a connection.
- All paths must be relative, so the app works at a domain root or in a sub-folder.

## 14. System constraints

- The system is a web client and a server. The server owns the SQLite database and is the only component that reads or writes it.
- The client talks to the server over HTTPS. The shape of the interface between them is the implementer's choice, as long as every rule in this document is enforced on the server.
- Creating an account with its starting data must be a single transaction. Saving a session with its photo row must be a single transaction, with the photo file handled as section 4.6 defines.
- The location of the `data` folder must be configurable. The server must be the only process that writes to it.
- Dashboard figures may be calculated on the server or on the client, but both must give the results defined in sections 6.2 and 7.
- Schema changes after the first release must be applied by versioned migrations.
