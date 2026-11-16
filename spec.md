# EV Charging Expenses — Product Specification

## 1. About this document

This document is the source of truth for what EV Charging Expenses does. It defines behaviour, data and rules. It does not prescribe a technology stack; that is covered by a separate implementation document.

`index.html` in this repository is the **visual reference** for layout, styling, interaction feel and wording. Where the HTML and this document disagree about behaviour, this document wins. Section 11 lists the places where they are known to differ.

The words **must** and **must not** mark requirements.

## 2. Product overview

EV Charging Expenses lets an electric-vehicle owner log each charging session and see what charging costs them. A user records the vehicle, date, cost, energy and where they charged. The app summarises spending by month, by home versus public charging, and by charging provider.

**In scope**

- Signing in and out.
- A dashboard of spending figures for a chosen period, a monthly budget and a six-month chart.
- A filterable list of charging sessions, with add, edit and delete.
- Management of vehicles, charging providers and provider branches.
- A light and dark appearance.
- English and Thai, switchable in the app.

**Out of scope**

- Sharing data between users.
- Import, export or integration with charging networks.
- Currencies other than Thai baht.
- Tablet and desktop layouts beyond the phone-width column.

## 3. Users and access

- A user must sign in with a username and password before seeing any data.
- Incorrect credentials must be rejected with an error message, and the user stays on the sign-in screen.
- A signed-in user stays signed in across page reloads until they sign out.
- Each user sees and changes only their own vehicles, providers, branches and sessions.
- Signing out returns to the sign-in screen and no user data remains visible.

How accounts are created is defined in the implementation document.

## 4. Data model

Every record has a stable unique `id`. Records refer to each other by `id`, never by name.

### 4.1 Vehicle

| Field | Type | Rules |
|---|---|---|
| `name` | text | Required. Unique per user after trimming. The display name. |
| `brand` | text | Optional. |
| `model` | text | Optional. |
| `plate` | text | Optional. License plate. |

### 4.2 Provider

| Field | Type | Rules |
|---|---|---|
| `name` | text | Required. Unique per user after trimming. |

A provider owns zero or more branches.

### 4.3 Branch

| Field | Type | Rules |
|---|---|---|
| `provider` | reference | Required. The owning provider. |
| `name` | text | Required. Unique within its provider after trimming. |

### 4.4 Session

| Field | Type | Rules |
|---|---|---|
| `vehicle` | reference | Required. |
| `date` | date | Required. A calendar date with no time of day. |
| `cost` | decimal | Required. THB, zero or more, at most two decimal places. |
| `kwh` | decimal | Required. Zero or more, at most one decimal place. |
| `type` | choice | Required. `Home` or `Public`. |
| `provider` | reference | Required when `type` is `Public`. Must be empty when `type` is `Home`. |
| `branch` | reference | Optional when `type` is `Public`. Must belong to `provider`. Must be empty when `type` is `Home`. |
| `start` | whole number | Optional. Battery percentage at start, 0–100. |
| `end` | whole number | Optional. Battery percentage at end, 0–100. |
| `duration` | whole number | Optional. Minutes, zero or more. |
| `note` | text | Optional. |

### 4.5 Starting data for a new user

A new user starts with:

- No vehicles and no sessions.
- The default provider list below, which they can edit or delete like any other provider.

| Provider | Branches |
|---|---|
| EV Station PluZ | Bangna, Vibhavadi, Rama 9 |
| EA Anywhere | Central World, Siam Paragon, Mega Bangna |
| PEA VOLTA | Bangkok, Chiang Mai, Phuket |
| EleX by EGAT | EGAT HQ, Lat Krabang |
| SHARGE | 7-Eleven Sukhumvit, Shell Rama 2 |
| OneCharge | Cultural Center, Green Bus |
| iGreen+ | Suvarnabhumi, Ramkhamhaeng |
| Evolt | Office Park, Mall |
| Altervim | Lotus, 7-Eleven |
| Spark | Nakhon Sawan, Highway |

### 4.6 Persistence

- Vehicles, providers, branches and sessions must persist until the user deletes them.
- The monthly budget must persist with the user's account.
- The dark mode choice and the language choice must persist on the device.
- The vehicle selection, the dashboard period, the session filters and the expanded provider are view state. They reset when the app is reloaded.

### 4.7 User settings

| Field | Type | Rules |
|---|---|---|
| `monthlyBudget` | decimal | Optional. THB, greater than zero, at most two decimal places. Empty means no budget is set. |

There is one budget per user. It covers all of the user's vehicles together.

## 5. Navigation

- The sign-in screen is shown first unless the user is already signed in.
- A bottom navigation bar has three tabs: **Dashboard**, **Sessions** and **Settings**. The active tab is highlighted.
- A floating add button (`+`) sits above the navigation bar on every signed-in screen and opens the Add Session dialog.
- **Vehicles** and **Chargers** are sub-screens of Settings. Each has a Back button that returns to Settings. The Settings tab stays highlighted while a sub-screen is open.
- The navigation bar and the add button are hidden on the sign-in screen.

## 6. Screens

### 6.1 Sign in

- Fields: Username and Password.
- A Login button submits the credentials (see section 3).
- On success the Dashboard opens.

### 6.2 Dashboard

The header shows the app title and a **Log out** button.

A **vehicle selector** offers "All vehicles" plus each of the user's vehicles. The selection filters the Dashboard and also the Sessions screen. If the selected vehicle is deleted, the selection returns to "All vehicles".

A **period selector** sits beside the vehicle selector. It sets the date range for the period figures below. It does not affect the Sessions screen.

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
| By Provider (Public) | Yes | Yes | Cost per provider for public sessions in the period, highest first, top five only. The provider name is on the left and the cost on the right. Shows "No public sessions" when empty. |
| Monthly Spending | No | Yes | Bar chart of the last six calendar months, ending with the current month, oldest on the left. |
| Recent Sessions | No | Yes | The five most recent sessions as session cards. Shows "No sessions yet" when empty. |

Card labels do not repeat the period. The period selector is the only place it is named.

**Change against the comparison range**

- Shown under the Total Cost figure as an arrow, a percentage and the comparison name, for example `▲ 12% vs last month`.
- The percentage is the difference between the two totals divided by the comparison total, rounded to a whole number.
- An increase in spending is shown in the destructive colour with `▲`. A decrease is shown in the accent colour with `▼`. No change shows `0%` in muted text with no arrow.
- Nothing is shown when the period is All time or when the comparison total is zero.

**Home vs Public**

- The card is a small table: the type name on the left, and its cost with its average cost per kWh on the right, for example `Home` · `820` · `5.40/kWh`.
- A type with no energy in the period shows `—` for its rate.
- When both types have a rate, a final row labelled "Public vs Home" shows the public rate minus the home rate as a signed figure, for example `+2.35/kWh`, or `−0.40/kWh` when public is cheaper.
- The difference is shown as a signed number, not a sentence, so that it reads the same in every language.

**Monthly Budget card**

- The card always covers the current calendar month and all vehicles, whatever the period and vehicle selectors show. Its label says so: "Monthly Budget · All vehicles".
- It shows the amount spent, the budget, and what remains, for example `1,240 of 2,000 · 760 left`.
- When spending is over the budget, it shows the excess, for example `2,240 of 2,000 · 240 over`.
- A progress bar shows spending as a share of the budget. It is full when spending reaches or passes the budget.
- The bar uses the accent colour below 80% of the budget, the warning colour from 80% up to 100%, and the destructive colour above 100%.
- Tapping the card opens the budget dialog (section 6.6).

**Monthly Spending chart**

- Each of the six months appears exactly once.
- Bar height is proportional to the month with the highest spending. A month with no spending still shows a minimal bar.
- The current month's bar uses the accent colour. The other bars are neutral.
- Each bar is labelled with the short month name.
- When a budget is set and the vehicle selector shows "All vehicles", a horizontal line marks the budget level. The line is hidden when the budget is more than twice the highest month. When the line is shown and the budget is above the highest month, the bars scale to the budget so the line stays inside the chart.

### 6.3 Sessions

- Sessions are listed as session cards, newest date first. Sessions on the same date are ordered newest-created first.
- Three filters combine with AND:
  - **Month:** "All months" plus every month that has at least one session, newest first.
  - **Type:** "All types", Home, Public.
  - **Provider:** "All providers" plus every provider used by at least one session.
- The Dashboard's vehicle selection also applies.
- Shows "No sessions match the filters" when the result is empty.

### 6.4 Session card

Used on the Dashboard and the Sessions screen. Each session is a separate card.

- **Left:** the vehicle name in bold, then `date · energy kWh · location`.
  - Location is "Home" for home sessions.
  - Location is `provider · branch` for public sessions, or the provider alone when no branch is set.
- **Right:** the cost, and a type badge (green for Home, blue for Public).
- Battery percentages, duration and note are not shown on the card. They are visible in the Edit Session dialog.

Cards have no visible buttons. They respond to:

| Action | Result |
|---|---|
| Swipe left-to-right | The card follows the finger up to 100px and a red "Delete" background fades in. Releasing after more than 80px opens the delete confirmation. A shorter swipe snaps back. |
| Long press (900ms without moving) | Opens the Edit Session dialog. The device vibrates briefly where supported. |
| Right-click | Opens the Edit Session dialog. |

### 6.5 Add / Edit Session dialog

A bottom sheet covering 75% of the screen height over a dimmed backdrop. The form scrolls inside the sheet. **Cancel** and **Save Session** stay fixed side by side at the bottom. A `✕` button in the header also closes the sheet. Closing without saving discards changes.

| Field | Control | Notes |
|---|---|---|
| Vehicle | Select | Required. Lists the user's vehicles. |
| Date | Date picker | Required. Defaults to today. |
| Cost (THB) | Number | Required. |
| Energy (kWh) | Number | Required. |
| Type | Select | Home (default) or Public. |
| Provider | Select | Shown only for Public. Required. |
| Branch | Select | Shown only for Public. Lists the chosen provider's branches. Shows "No branches yet" when the provider has none. |
| + Add new branch | Button | Shown only for Public. Asks for a name, adds the branch to the chosen provider and selects it. Section 6.8 naming rules apply. |
| Start Battery % | Number | Optional. |
| End Battery % | Number | Optional. |
| Duration minutes | Number | Optional. |
| Note | Text area | Optional. |

- **Add mode:** the title is "Add Session" and the form is blank apart from the defaults.
- **Edit mode:** the title is "Edit Session" and the form is filled from the session. A **Delete** action is available in the sheet and opens the delete confirmation.
- Saving must validate the fields against section 4.4. Invalid fields are flagged and nothing is saved.
- After a successful save the sheet closes and every screen reflects the change.
- Changing the type from Public to Home clears the provider and branch.
- If the user has no vehicles, the add button must direct them to add a vehicle first.

### 6.6 Settings

Five cards:

- **Dark Mode:** a toggle switch.
- **Language:** a selector listing each language by its own name ("English", "ไทย"). The change applies immediately, without a reload.
- **Monthly Budget:** shows the current budget, or "Not set". Opens the budget dialog.
- **Vehicles:** opens the Vehicles screen.
- **Chargers:** opens the Chargers screen.

**Budget dialog** (centred modal): one amount field in THB, with Cancel and Save.

- Saving an amount greater than zero sets the budget.
- Saving an empty field removes the budget.
- Zero, a negative amount or more than two decimal places is rejected with an error message and the dialog stays open.
- The budget applies to the current month and every later month until it is changed. Changing it does not alter how past months are shown.

### 6.7 Vehicles

- One card per vehicle in a single column: an icon, the display name, then `brand · model · plate`, with **Edit** and **Delete** buttons. Empty details are left out along with their separators.
- **+ Add** in the header opens the vehicle dialog.
- Shows "No vehicles yet" when empty.

**Vehicle dialog** (centred modal): Display Name, Brand, Model, License Plate, with Cancel and Save.

- Display Name is required. A missing or duplicate name is rejected with an error message and the dialog stays open.
- Renaming a vehicle changes the name shown on all of its sessions.

**Deleting a vehicle**

- Requires confirmation.
- A vehicle with no sessions is deleted outright.
- A vehicle that has sessions cannot be deleted. The confirmation explains this and tells the user how many sessions use it.

### 6.8 Chargers

- One card per provider, showing its name and branch count, with **Edit** and **Delete** buttons.
- Tapping a provider expands it to list its branches. Only one provider is expanded at a time.
- Each branch has **Edit** and **Delete** buttons. An expanded provider also has **+ Add Branch**.
- **+ Add** in the header opens the provider dialog.
- Shows "No providers yet" when empty, and "No branches yet" inside a provider with none.

**Provider dialog** and **branch dialog** (centred modals) each have one name field, with Cancel and Save.

- A missing or duplicate name is rejected with an error message and the dialog stays open.
- Renaming a provider or branch changes the name shown on all sessions that use it.

**Deleting a provider or branch**

- Requires confirmation.
- Deleting a provider also deletes its branches.
- A provider or branch that is used by sessions cannot be deleted. The confirmation explains this and tells the user how many sessions use it.

### 6.9 Delete confirmation

One shared centred modal is used for sessions, vehicles, providers and branches. It shows a title, a message identifying the item, a **Cancel** button and a red **Delete** button. Nothing is deleted until Delete is chosen.

For a session, the message shows its date, vehicle and cost.

## 7. Calculation and display rules

- **Current month** means the current calendar month in the user's local time zone.
- A session belongs to the month of its `date`.
- Section 6.2 states which Dashboard elements follow the vehicle selection and which follow the period.
- A period's date range includes both its first and last day.
- A comparison range that is cut to "the same number of days" starts on the first day of the earlier period and never runs past that period's last day.
- Costs are stored exactly as entered. Rounding is for display only. Percentages and differences are calculated from the stored values, not the rounded ones.

Amounts are shown without a currency symbol. All amounts are in Thai baht, and input labels say so, for example "Cost (THB)".

Numbers and month names follow the selected language's conventions. The examples below are for English.

| Value | Format | Example |
|---|---|---|
| Cost totals and session card cost | Whole baht, rounded half up | `1,240` |
| Average cost per kWh | Two decimals | `6.85` |
| Change against comparison | Whole percentage | `12%` |
| Energy | One decimal, followed by `kWh` | `32.5 kWh` |
| Date | `YYYY-MM-DD` | `2026-11-14` |
| Month filter option | `YYYY-MM` | `2026-11` |

## 8. Visual design

`index.html` is the reference for the look of every screen. The key rules are:

- **Layout:** a single column capped at 430px wide and centred.
- **Accent colour:** green `#16a34a` for primary buttons, key figures, the active tab, the add button and the current-month bar.
- **Destructive colour:** red `#dc2626` for delete confirmation, the swipe background, a spending increase and an exceeded budget.
- **Warning colour:** amber `#d97706` for a budget that is at least 80% used.
- **Cards:** 16px corner radius with a soft elevated shadow.
- **Typography:** the system font. Screen titles are 20px bold and key figures are 28px bold.
- **Form controls:** full width, 12px corner radius, 16px text.
- **Selectors:** every drop-down uses the app's own control, never the browser's default. Closed, it looks like the other form controls, with the chosen value and a chevron. Tapping it opens a bottom sheet listing the options, with the current one in the accent colour and ticked. Choosing an option, tapping outside the sheet or pressing Escape closes it.

**Themes**

| Token | Light | Dark |
|---|---|---|
| Background | `#f5f7fa` | `#111827` |
| Text | `#1a1a1a` | `#f3f4f6` |
| Card | `#ffffff` | `#1f2937` |
| Border | `#e0e0e0` | `#374151` |
| Muted text | `#666666` | `#9ca3af` |
| Secondary surface | `#f0f0f0` | `#374151` |

Every surface, including all dialogs and the expanded branch list, must follow the active theme.

## 9. Content safety

All user-entered text (names, plates, notes) must be displayed as plain text. It must never be interpreted as markup or code.

## 10. Accessibility and input

- Every action must be reachable without touch gestures. Edit is reachable by right-click, and delete from the Edit Session dialog.
- The page must allow the user to zoom.
- Dialogs must be closable with their Cancel or `✕` control.

## 11. Differences from the visual reference

`index.html` is a mockup. It differs from this specification in the following ways, and this specification takes precedence in each case.

| Area | Visual reference | Specification |
|---|---|---|
| Sign in | Accepts anything. Has a "Fill demo credentials" button. | Credentials are checked. No demo button. |
| Data | Resets to random sample data on every reload. | Persists per user. No sample sessions or vehicles. |
| Dark mode | Resets on reload. Some dialogs stay light. | Persists. All surfaces are themed. |
| References | Sessions store vehicle, provider and branch by name. | Records are linked by `id`. |
| Deleting in-use items | Allowed. Sessions keep the old name. | Blocked while sessions use the item. |
| Session delete | Swipe only. | Swipe, plus a Delete action in the Edit Session dialog. |
| Sub-screens | No tab highlighted. | Settings tab stays highlighted. |
| Monthly budget | Starts at 2,000 and resets on reload. | Starts empty and persists per user. |
| Add new branch | Browser prompt. | In-app input. |
| Errors | Browser alerts, or silent. | In-app error messages. |
| Vehicle details | Shows empty separators. | Empty details are left out. |
| Zoom | Disabled. | Allowed. |

## 12. Languages and translation

The app supports English (the default) and Thai. Section 6.6 defines how the user switches.

**What is translated**

- All interface text: labels, buttons, messages, placeholders, empty states and errors.
- Number separators and month names, which follow the selected language.

**What is not translated**

- Anything the user typed: vehicle names, provider and branch names, plates and notes.
- The default provider and branch names in section 4.5.
- Dates, which stay `YYYY-MM-DD`, and the unit `kWh`.
- Stored values. A session's type is stored as `Home` or `Public` whatever language is shown.

**Rules for interface text**

These keep the app translatable and apply to every language added later.

- Every visible phrase must come from a single string table per language. No interface text may be written directly in screen code.
- A phrase must be stored whole, with named placeholders for the values inside it, for example `{spent} of {budget} · {rest} left`. Sentences must not be assembled from fragments, because word order differs between languages.
- Where wording depends on a condition, each outcome must be its own complete phrase. "left" and "over" on the budget card are two phrases, not one phrase with a swapped word.
- Where wording depends on a count, the phrase must use the language's plural rules, for example `1 branch` and `2 branches`.
- A value should be shown as a labelled figure in preference to a sentence wherever that is clear enough.
- A missing translation falls back to English.
- Layout must not depend on text length. Labels may wrap, and figures stay aligned to the right.

Adding a language must require only a new string table and an entry in the language selector.

The Thai wording in `index.html` is a first draft and needs review by a native speaker before release.
