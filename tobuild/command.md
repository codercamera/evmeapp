# Build Instructions — EV Charging Expenses

**Status:** Nothing is built yet. Work starts on the command **"Start"**.

## 1. Sources

- `tobuild/spec.md` is the source of truth for features, data and rules.
- The HTML mockup (`index.html`) is a visual reference only.
- If this file and `tobuild/spec.md` disagree, stop and ask. Section 7 lists the spec changes this plan needs.

## 2. Technology stack

- **Backend:** PHP 8.1 or later with Slim Framework 4.
- **Frontend:** React, pre-built into static assets.
- **Database:** SQLite, through `pdo_sqlite`.
- **Hosting:** shared hosting. There is no command line, no Composer and no scheduled jobs on the server. Everything must be built before upload, and the `vendor` folder ships inside the package.
- **Required PHP extensions:** `pdo_sqlite`, `phar`, `zlib`, `gd`, `fileinfo`, `mbstring`, `json`, `openssl`.

## 3. Deployment

### 3.1 What is uploaded

Two files are uploaded to the application directory:

- `bootstrap.php`
- `assets.tar.gz`, which holds the whole application: PHP source, `vendor`, the React build and `index.php`.

The archive is a gzipped tar. Extraction uses PHP's `PharData`, which needs the `phar` extension, so `phar` is added to the required list.

### 3.2 What `bootstrap.php` does

1. Checks the PHP version and the required extensions, and checks that the directory is writable. If anything is missing, it lists what is missing and stops.
2. Refuses to run if `config.php` already exists.
3. Verifies `assets.tar.gz` against a SHA-256 checksum built into `bootstrap.php`.
4. Extracts `assets.tar.gz` into the current directory, including `index.php`.
5. Deletes `assets.tar.gz` and then deletes itself. If it cannot delete itself, it tells the user to remove it by hand.
6. Redirects to the setup wizard.

### 3.3 Directory layout after extraction

| Path | Content | Reachable from the web |
|---|---|---|
| `index.php` | Front controller for the API and the app | Yes |
| `.htaccess` | Rewrite rules and access blocks | No |
| `app/` | React build (static files) | Yes |
| `src/`, `vendor/`, `migrations/` | PHP code and libraries | No |
| `config.php` | Settings written by the wizard | No |
| `data/` | SQLite database and `data/images/` | No |

- Every "No" path is blocked by `.htaccess` rules, and each of those folders also gets its own deny-all `.htaccess`.
- `config.php` and all PHP files outside `index.php` start with a guard that exits when the file is requested directly.
- If the host allows it, `data/` is placed outside the web root. Its location is a setting in `config.php`.
- These blocks rely on Apache. On a host without `.htaccess` support, the wizard must warn that `data/` is exposed and stop.

## 4. Setup wizard

The wizard runs only while no `config.php` exists. Once setup finishes, every wizard address returns "not found".

**Step 1 — Site address.** The wizard shows the address detected from the request and asks the user to confirm or correct it. The confirmed value is saved in `config.php`. The detected value is never trusted without confirmation.

**Step 2 — Database.** Two choices:

- **Start new.** The user sets a username and password. The wizard creates the database, runs all migrations and creates this user as the administrator, with the starting data the spec defines.
- **Restore a backup.** The user uploads one backup archive (section 6). The wizard checks that it is a real backup from this system and that its schema version is not newer than the application, then restores the database and the photo files and runs any pending migrations.

**Database file.** The database is stored as `data/<random name>.sqlite`. The random name is generated at setup and saved in `config.php`.

## 5. Accounts (decision needed — see section 7)

- The user created by the wizard is the **administrator**.
- There is no public registration. The administrator creates the other users.
- There is no password reset by email. The administrator sets a new password for a user from a **Users** screen in Settings.
- Any user can change their own password in Settings.
- Sign-in attempts are rate-limited using a table in the SQLite database, since no other storage is available.

## 6. Backup and maintenance

Because the host has no command line or scheduled jobs, maintenance is done from an administrator-only **Maintenance** screen in Settings:

- **Download backup:** one zip holding the SQLite database and the `data/images` folder.
- **Restore:** done through the setup wizard on a fresh install.
- **Clean up photos:** removes image files with no database row and reports rows with no file, as the spec requires.
- **Migrations:** run automatically when the application version is newer than the database's schema version.

## 7. Spec changes this plan needs

These points differ from `tobuild/spec.md` today. Each needs a decision before "Start".

| # | Topic | Spec today | This plan |
|---|---|---|---|
| 1 | Creating accounts | Anyone can register from the sign-in screen. | No public registration. The administrator creates users. |
| 2 | Roles | All users are equal. | One administrator role. Needs an `is_admin` column on `users`. |
| 3 | Password change | Not covered. | Users change their own password. The administrator resets others. |
| 4 | Users screen | Not present. | New screen under Settings, administrator only. |
| 5 | Maintenance screen | Not present. Clean-up task has no defined trigger. | New screen under Settings, administrator only. |
| 6 | Backup | "Back up the database and images together." | One downloadable zip, restorable from the setup wizard. |

If #1 is refused, public registration stays as in the spec, and the wizard's first user is still the administrator for the Maintenance screen.

## 8. Confirmed decisions

| Topic | Decision |
|---|---|
| Source of truth | `tobuild/spec.md`. On conflict with the mockup, follow the spec. |
| Sign-in | Username and password, as in the spec. Log out stays in the Dashboard header. |
| Money | Whole satang, as in the spec. Energy in whole watt-hours, as in the spec. |
| Photos | Files under `data/images/<first 3 characters>/<guid>.jpg`, as in the spec. The server re-encodes each upload with `gd` before saving. |
| Site address | Detected, confirmed by the user in the wizard, saved in `config.php`. |
| Database location | `data/` with a random file name, blocked from the web, outside the web root where possible. |
| Paths | Relative everywhere, so the app works in a sub-folder. |

## 9. Build sequence after "Start"

1. PHP + Slim project structure, with SQLite migrations taken from the spec's schema.
2. `bootstrap.php` and the setup wizard.
3. The API, enforcing every rule in the spec on the server.
4. The React frontend, matching the spec and the mockup, built into static assets.
5. Automated tests for the API rules and the schema constraints.
6. Packaging: `assets.tar.gz` with its checksum written into `bootstrap.php`.
7. A short install guide: upload two files, open `bootstrap.php`, follow the wizard.

**Type "Start" when ready.**
