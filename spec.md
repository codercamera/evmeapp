# EV Charging Expenses — Specification

This document specifies the app as implemented on branch `test2` (commit `02effc9`). It describes current behaviour, so the code in `index.html` is the reference where the two disagree. Known defects and undecided points are listed in section 10 rather than written into the requirements.

## 1. Overview

EV Charging Expenses is a mobile-first web mockup for logging electric-vehicle charging sessions and seeing what they cost. A user records each charge (vehicle, date, cost, energy, home or public charger) and the app summarises spending by month, by charger type and by provider.

**In scope**

- A demo login gate.
- A dashboard of spending figures and a six-month chart.
- A filterable list of charging sessions, with add, edit and delete.
- Management of vehicles, charging providers and provider branches.
- A light/dark appearance toggle.

**Out of scope**

- Real authentication, user accounts or a backend.
- Saving data between page loads (see section 7).
- Tablet or desktop layouts.
- Import, export, sync or multi-currency support.

## 2. Platform and delivery

| Item | Requirement |
|---|---|
| Delivery | One static file, `index.html`, with inline CSS and JavaScript. No build step and no external dependencies. |
| Hosting | Opens directly from disk or from any static host. |
| Layout | Single column, capped at 430px wide and centred. Pinch zoom is disabled. |
| Language | English UI. Currency is Thai baht, shown with the `฿` symbol. |
| Input | Designed for touch. Mouse support is partial (see section 10). |

## 3. Data model

All data lives in one in-memory `state` object.

### 3.1 Vehicle

| Field | Type | Rules |
|---|---|---|
| `name` | string | Required. Unique. Acts as the vehicle's identifier. |
| `brand` | string | Optional. |
| `model` | string | Optional. |
| `plate` | string | Optional. |

### 3.2 Provider and branch

Providers are stored as a map from provider name to a list of branch names.

- A provider name is unique and acts as its identifier.
- A branch name is unique within its provider.
- A provider may have no branches.

### 3.3 Session

| Field | Type | Rules |
|---|---|---|
| `id` | number | Timestamp in milliseconds, assigned at creation. |
| `vehicle` | string | Required. The vehicle's name at the time of saving. |
| `date` | string | Required. `YYYY-MM-DD`. |
| `cost` | number | Required. THB, zero or more, up to two decimals. |
| `kwh` | number | Required. Zero or more, in steps of 0.1. |
| `type` | string | `Home` or `Public`. |
| `provider` | string or null | Set only when `type` is `Public`. |
| `branch` | string or null | Set only when `type` is `Public`. |
| `start` | number or null | Optional. Battery percentage at start, 0–100. |
| `end` | number or null | Optional. Battery percentage at end, 0–100. |
| `duration` | number or null | Optional. Minutes, zero or more. |
| `note` | string | Optional. Empty string when not given. |

Sessions reference vehicles, providers and branches by name, not by id. The session list is kept sorted by date, newest first.

### 3.4 Seed data

Every page load starts from the same seed:

- **Vehicles:** "Daily Driver" (Tesla Model Y, กท 1234) and "Weekend Tripper" (BYD Atto 3, ขข 5678).
- **Providers:** ten Thai charging networks, each with two or three branches (EV Station PluZ, EA Anywhere, PEA VOLTA, EleX by EGAT, SHARGE, OneCharge, iGreen+, Evolt, Altervim, Spark).
- **Sessions:** 18 randomly generated sessions dated within the last 70 days. Each has 15–45 kWh, a rate of ฿4.5–6.5 per kWh for home or ฿6.5–9.5 for public, a duration of 20–70 minutes, and about a 30% chance of the note "Fast charge". Only public sessions get start and end battery percentages.

## 4. Navigation

- **Login screen** is shown first unless the user is already logged in.
- **Bottom navigation** has three tabs: Dashboard, Sessions and Settings. The active tab is highlighted in green.
- **Floating add button** (`+`) sits above the navigation on every logged-in screen and opens the Add Session dialog.
- **Vehicles** and **Chargers** are sub-screens of Settings. Each has a Back button that returns to Settings. No tab is highlighted while a sub-screen is open.
- The navigation and the add button are hidden on the login screen.
- Screens are not reflected in the URL. The browser's back button does not move between screens.

## 5. Screens

### 5.1 Login

- Fields: Username and Password.
- "Fill demo credentials" fills in `demo` / `1234`.
- "Login" enters the app and opens the Dashboard. Credentials are not checked.
- The logged-in flag is stored so that a page refresh stays logged in (see section 7).

### 5.2 Dashboard

The header shows the app title and a "Log out" button. Logging out clears the logged-in flag and returns to the login screen.

A **vehicle selector** offers "All vehicles" plus each vehicle. The selection filters every figure on the Dashboard and also the Sessions screen.

| Element | Content |
|---|---|
| Total Cost This Month | Sum of cost for sessions dated in the current calendar month, rounded to whole baht. |
| All-time Total | Sum of cost for all sessions, rounded to whole baht. |
| Avg. Cost / kWh | Current month's cost divided by current month's energy, to two decimals. Shows `฿0` when the month has no energy. |
| Home vs Public | All-time cost for each type, rounded to whole baht. |
| By Provider (Public) | All-time cost per provider for public sessions, highest first, top five only. Shows "No public sessions" when empty. |
| Monthly Spending | Bar chart of the last six calendar months, oldest on the left. Bar height is proportional to the highest month, with a 4% minimum. The current month is green and the rest are grey. Each bar is labelled with the short month name. |
| Recent Sessions | The five most recent sessions as session cards. Shows "No sessions yet" when empty. |

### 5.3 Sessions

- Three filters, combined with AND: month, type (Home or Public) and provider.
- The Dashboard's vehicle selection also applies here.
- Month and provider options are built from all sessions, whatever the vehicle selection.
- Shows "No sessions match the filters" when the result is empty.

### 5.4 Session card

Used on both the Dashboard and the Sessions screen. Each session is a separate card with a soft elevated shadow.

- **Left:** vehicle name in bold, then `date · kWh · location`. Location is "Home" for home sessions and `provider · branch` for public ones.
- **Right:** cost rounded to whole baht, and a type badge (green for Home, blue for Public).
- Battery percentages, duration and note are stored but not shown on the card.

Cards have no visible buttons. They respond to gestures:

| Gesture | Result |
|---|---|
| Swipe left-to-right | The card follows the finger up to 100px and a red "Delete" background fades in. Releasing after more than 80px opens the delete confirmation. A shorter swipe snaps back. |
| Long press (500ms, under 10px of movement) | Opens the Edit Session dialog. The device vibrates briefly where supported. |
| Right-click or context menu | Opens the Edit Session dialog. |

### 5.5 Add / Edit Session dialog

A bottom sheet that covers 75% of the screen height over a dimmed backdrop. The form scrolls inside the sheet. Cancel and Save Session stay fixed side by side at the bottom, and a `✕` button in the header also closes the sheet.

| Field | Control | Notes |
|---|---|---|
| Vehicle | Select | Required. Lists current vehicles. |
| Date | Date picker | Required. Defaults to today. |
| Cost (THB) | Number | Required. Minimum 0, step 0.01. |
| Energy (kWh) | Number | Required. Minimum 0, step 0.1. |
| Type | Select | Home (default) or Public. |
| Provider | Select | Shown only for Public. |
| Branch | Select | Shown only for Public. Lists the chosen provider's branches, or "No branches yet". |
| + Add new branch | Button | Shown only for Public. Asks for a name and adds it to the chosen provider, then selects it. |
| Start Battery % | Number | Optional. 0–100. |
| End Battery % | Number | Optional. 0–100. |
| Duration minutes | Number | Optional. Minimum 0. |
| Note | Text area | Optional. |

- In add mode the title is "Add Session" and the form is blank.
- In edit mode the title is "Edit Session" and the form is filled from the session.
- Saving validates required fields, adds or replaces the session, re-sorts the list, closes the sheet and refreshes all screens.
- Changing the type to Home on save clears the provider and branch.

### 5.6 Settings

Three cards:

- **Dark Mode:** a toggle switch.
- **Vehicles:** opens the Vehicles screen.
- **Chargers:** opens the Chargers screen.

### 5.7 Vehicles

- One card per vehicle, in a single column: an icon, the display name, then `brand · model · plate`, with Edit and Delete buttons.
- "+ Add" in the header opens the vehicle dialog.
- Shows "No vehicles yet" when empty.

**Vehicle dialog** (centred modal) has Display Name, Brand, Model and License Plate.

- Display Name is required. Saving with an empty name does nothing.
- A duplicate name is rejected with an alert.
- Renaming a vehicle updates the vehicle name on all of its sessions.

**Deleting a vehicle** asks for confirmation. Its sessions are kept and continue to show the old name.

### 5.8 Chargers

- One card per provider, showing its name and branch count, with Edit and Delete buttons.
- Tapping a provider expands it to list its branches. Only one provider is expanded at a time.
- Each branch has Edit and Delete buttons. An expanded provider also has "+ Add Branch".
- "+ Add" in the header opens the provider dialog.
- Shows "No providers yet" when empty, and "No branches yet" inside an empty provider.

**Provider dialog** and **branch dialog** (centred modals) each have a single name field.

- An empty name does nothing. A duplicate name is rejected with an alert.
- Renaming a provider or branch updates the matching sessions.

**Deleting a provider** removes it and all its branches after confirmation. **Deleting a branch** removes only that branch. In both cases existing sessions keep the old names.

### 5.9 Delete confirmation dialog

One shared centred modal is used for sessions, vehicles, providers and branches. It shows a title, a message naming the item, a Cancel button and a red Delete button. Nothing is deleted until Delete is tapped.

## 6. Calculation rules

- The "current month" is the `YYYY-MM` prefix of the current date in UTC. A session belongs to a month when its date starts with that prefix.
- All Dashboard figures use the vehicle-filtered session set.
- Totals are shown rounded to whole baht. Only the average cost per kWh shows two decimals.
- Stored costs keep their decimals. Rounding is for display only.

## 7. Persistence

| Data | Stored | Lifetime |
|---|---|---|
| Logged-in flag | `localStorage` key `evmeapp-loggedin` = `"1"` | Until logout. |
| Vehicles, providers, branches, sessions | Memory only | Lost on refresh. The seed data is regenerated, with new random sessions. |
| Dark mode | Memory only | Always off after a refresh. |
| Selected vehicle, filters, expanded provider | Memory only | Lost on refresh. |

Data is deliberately not saved. This is a mockup, and each refresh returns it to a clean demo state.

## 8. Visual design

- **Accent colour:** green `#16a34a` for primary buttons, key figures, the active tab, the add button and the current-month bar.
- **Destructive colour:** red `#dc2626` for delete confirmation and the swipe background.
- **Cards:** white (or dark grey in dark mode), 16px radius, soft shadow.
- **Themes:** light and dark are driven by CSS variables for background, text, card, border, muted text, secondary surface and navigation.
- **Typography:** the system font stack. Screen titles are 20px bold and key figures are 28px bold.
- **Form controls:** full width, 12px radius, 16px text to avoid zoom on focus in iOS.

## 9. Non-functional requirements

- Works offline once loaded, since there are no network requests.
- No third-party libraries. The chart is drawn with plain HTML elements.
- All screens re-render after every data change. This is acceptable at mockup data sizes.

## 10. Known issues and open questions

These are behaviours in the current code that look unintended or undecided. Each needs a decision before the next round of work.

**Defects**

- **Dark mode is incomplete.** The delete, vehicle, provider and branch dialogs have a hard-coded white background, and the expanded branch list is hard-coded light grey. Text in those areas becomes hard to read in dark mode.
- **No delete on desktop.** Delete is reachable only by touch swipe. Edit works with right-click, but a mouse user cannot delete a session.
- **User text is not escaped.** Vehicle, provider and branch names and notes are inserted into the page as HTML. A name containing markup or a double quote can break the layout or the buttons.
- **Deleted references in edit.** Editing a session whose vehicle, provider or branch has been deleted opens the form without that value selected, so saving changes it to something else.
- **Month boundaries use UTC.** In Thailand (UTC+7), sessions logged between midnight and 07:00 on the first of a month count towards the previous month's "this month" total.
- **Chart months can repeat or skip.** The six-month chart steps back from today's day of the month, so on the 29th–31st a shorter month can be skipped and another shown twice.
- **Optional numbers change type.** Battery percentages and duration are numbers in the seed data but text when entered through the form.
- **Confirmation message line breaks are lost.** The delete messages contain line breaks that the dialog shows as spaces.

**Open questions**

- **Should data persist?** An earlier version saved everything to `localStorage`. The current version resets on refresh by design. The right answer depends on whether this stays a demo.
- **Should login check anything?** Any input, including empty fields, logs in.
- **Should the optional fields be shown?** Battery percentages, duration and note are collected but never displayed.
- **Should "Avg. Cost / kWh" be monthly or all-time?** It is monthly today, while the neighbouring "Home vs Public" card is all-time, and neither is labelled.
- **Should deleting a vehicle or provider touch its sessions?** Today the sessions are kept with the old name, which leaves entries that no longer match anything in Settings.
- **The README is out of date.** It describes the `main` branch app ("EV Charge Manager", data saved in `localStorage`), not this one.
