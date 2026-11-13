# Predetermined deployment mode

A second, install-time-only deployment flavor, alongside the normal **standard** mode every deployment runs in by default. It exists for one purpose: running a small user-interaction study on one module with a handful of known, pre-vetted testers, where self-registration and mandatory MFA are pure friction and there's no real security concern — only interaction data is being collected, not facility operations or real user data.

**This is always a completely separate, standalone deployment** — its own install, own database, own server. It is never a mode switch applied to an existing production instance, and the two modes are not interchangeable on a running deployment — see "Switching modes" below.

## What actually differs from standard mode

Exactly two behavioral differences. Nothing else in core changes.

| Route | Standard mode | Predetermined mode |
|---|---|---|
| `POST /auth/register-smtp` | normal self-registration | `403`, disabled |
| `POST /auth/register-mail` | normal self-registration | `403`, disabled |
| `GET /api/config` | no `deploymentMode` field before this change | `deploymentMode: "standard"` \| `"predetermined"` |
| `POST /auth/login` | `totp_enabled` users get an MFA challenge | unchanged route/logic — predetermined users simply have `totp_enabled=0`, so this path already never triggers for them |
| *(frontend only, no route)* `/register` page, `ProtectedRoute`'s forced-MFA screen | reachable / enforced | hidden / skipped |

Two install-only CLI entry points also exist for this mode — commands, not HTTP routes, since neither is reachable over the network:
- `install/setup-database.js` — reads `DEPLOYMENT_MODE` from `.env` and records it once (see "Switching modes" below).
- `install/generate-predetermined-users.js` — bulk-creates the tester roster (see below).

Everything else — roles, groups, preferences, files, modules, tickets, page access, the self-service "🔒 Security" MFA section a user could still optionally enable for themselves — is completely untouched.

## Setup sequence

1. Set `DEPLOYMENT_MODE=predetermined` in `App_backend/.env` **before** running the database setup (see `.env.example`).
2. Run `install/setup-database.js` as usual — this creates the database, imports `schema.sql`, and records the mode once in `core_deployment_mode`.
3. Run `install/create-admin-user.js` as usual to create the real administrator account — this script is entirely unaffected by deployment mode; it's how you (the person running the study) log in to actually administer this deployment.
4. Generate the tester roster — two equivalent ways, same underlying account-creation logic
   (`App_backend/src/core/services/predeterminedUsers.js`) either way:
   - **Admin UI** (no server shell access needed): log in as the administrator account from step 3, go to
     Setup → Preferences → 🧪 Testers (only shown on a deployment whose `/api/config` already reports
     `deploymentMode: "predetermined"`), enter a count and role, click **Generate & Download Excel**. Downloads
     one `.xlsx` with one row per account: its QR code image, the plain login link, username, password, role,
     and a blank Email column for you to fill in yourself before sending each row to its tester.
   - **CLI**, if you'd rather run it from the server directly:
     ```
     node install/generate-predetermined-users.js --count 120 --role facility_user
     ```
     `--count` is required (no baked-in default — 120 is just an example). `--role` is optional, defaults to
     `facility_user`, and must already exist in `setup_roles`. Writes one `<username>.png` QR code per account
     plus a `credentials.csv` to `App_backend/install/predetermined-users-output/`, instead of one `.xlsx`.
5. Distribute the output — see "Distributing to testers" below.
6. **CLI path only**: once distribution is done, delete `App_backend/install/predetermined-users-output/` from
   the server. It's already gitignored, but the files themselves are real plaintext credentials sitting on disk
   until removed. (The admin-UI `.xlsx` download has no server-side equivalent to delete — it's generated on
   demand, nothing is written to disk on the server.)

## Recovering credentials for testers that already exist

Passwords are stored as a one-way bcrypt hash, the same as every other account in this app —
there is no way to read an existing tester's original plaintext password back out, from the
database, the admin UI, or anywhere else. If the `.xlsx`/`credentials.csv`/QR PNGs from when a
batch was generated are gone, those testers' passwords are gone with them.

The only fix is to issue each of those accounts a new password. Setup → Preferences → 🧪 Testers
has a second button for exactly this, below the generate-new-batch one: **"Reset Passwords &
Download Excel"**. It finds every existing `tester###` account, gives each one a fresh random
password, and downloads the same `.xlsx` format (QR code, login link, username, new password,
role, blank Email column) for them. It creates no accounts and deletes none — only each
account's password changes — but it does mean any tester still signed in, or about to sign in
with their old password, needs the new one instead from this point on.

## Opening/closing tester logins

Setup → Preferences → 🧪 Testers has an on/off switch at the top: **"Tester logins: Open /
Closed"**. It flips every existing `tester###` account's `status` between `active` and
`inactive` — the exact same column/check `POST /auth/login` already enforces for every
account in the app, so closing it just makes every tester see "Account is not active" at
login, same as a suspended account would. It only ever matches usernames against the
`tester###` pattern, so no other account — including the administrator using this switch —
is ever affected. No accounts are created, deleted, or have their passwords touched by this.

## The tester roster

Each account is fully independent: its own username (`tester001`, `tester002`, ...), its own randomly-generated 12-character password (an alphabet that excludes visually-ambiguous characters — `0/O`, `1/l/I` — since a password might occasionally need to be typed by hand if a QR scan fails), created directly as `status: 'active'` (no approval step — these are pre-vetted testers, not self-registrants) with `totp_enabled` left at its schema default of `0`.

Every account's QR code encodes nothing but the login link itself, with the username/password embedded in it (`${FRONTEND_URL}/login?username=...&password=...`) — the bare URL and nothing else, deliberately: a phone's built-in camera scanner only offers to open a QR code as a tappable link when the whole payload parses as a URL, so appending any other text (as an earlier version of this did) made most scanners show it as plain text instead of a link, as well as making the code denser and harder to scan. Username/password are also in their own Excel/CSV columns alongside the QR, for anyone who needs to type them in by hand instead of scanning. Opening the link logs the tester straight in, no form to fill in; `LoginPage.js` only ever honors `?username=`/`?password=` when the backend has confirmed `deploymentMode: 'predetermined'`, so a standard deployment's login page never acts on credentials passed in a URL. This is a deliberate trade against normal security hygiene (credentials end up in browser history and any proxy/access logs) — acceptable here only because predetermined-mode accounts are anonymous, low-stakes, and hold no real facility data; it is never available outside this mode.

- **Admin UI**: one `.xlsx` (`POST /api/predetermined-testers/generate`, developer/administrator only), one row per account — QR code image, login link, username, password, role, blank Email column.
- **CLI**: one `<username>.png` QR code per account plus a single `credentials.csv` (username, password, role) as an admin-facing backup record, appended to (not overwritten) on every run. The login link isn't repeated per CSV row since it's the same for every account — it's logged once to the console when the script runs, and only written into the QR codes.

**`FRONTEND_URL` must be set in `.env` before generating any testers — both paths refuse otherwise.** The whole point is a QR code that actually gets a tester to the login page, not just credentials to type somewhere else, so there's no "generate anyway without the link" fallback. It's the same variable every other part of the app already uses to build links in emails (`App_backend/src/core/utils/emailLinks.js`).

Both paths are safe to re-run later to add a further batch — they auto-detect the highest existing `tester###` username and continue numbering from there, so a later run never collides with or overwrites an earlier one.

**Both paths refuse to run at all unless the database they're pointed at is actually recorded as `predetermined`** — this is what stops 120 fake test accounts from ever landing in a real production database by accident, whether triggered from the server's shell or from the admin UI.

## Distributing to testers

Nothing in core emails this out or shows it to anyone but the admin who generates it — getting it to the actual testers is on you either way. There is no mechanism in this app that sends it on your behalf.

**Admin UI path**: the `.xlsx` downloads straight to your own machine (browser download), one row per tester — QR code, login link, username, password, role, and a blank Email column. Fill in each tester's email in that column yourself, then send each row on — split the sheet, copy each tester's row into your own mail merge/mailing tool, or send the whole sheet to whoever is doing the distribution for you. Nothing about this stores or uses the email you type in; the column exists for you to fill in and act on outside the app.

**CLI path**: the output only exists on disk on whatever server you ran `install/generate-predetermined-users.js` on, under `App_backend/install/predetermined-users-output/`.
1. Copy that whole directory off the server to your own machine, e.g. `scp -r user@server:/path/to/App_backend/install/predetermined-users-output ./` (or `sftp`/whatever file transfer you already use to reach that server).
2. Match each tester to their `<username>.png` — `credentials.csv` has the same username/password/role in one file if you'd rather read it as a table than open 120 images.
3. Send each tester their own QR code (and only their own — one PNG per account) by whichever channel you'd use to reach them normally: email it as an attachment, print it and hand it over, message it directly.
4. Once every tester has theirs, delete the directory from the server (step 6 above) — it's unencrypted plaintext passwords sitting on disk until you do.

## Switching modes

There is no runtime toggle, and this is enforced in code, not just documented. `install/setup-database.js` writes the chosen mode into a single-row `core_deployment_mode` table, once. At every backend startup, before the port opens, `index.js` checks `DEPLOYMENT_MODE` in `.env` against that row:

- **Row missing** (every deployment that predates this feature, including current production, the moment `schema.sql`'s `CREATE TABLE IF NOT EXISTS` is picked up) — self-heals: records `'standard'` once, logs that it did, continues. Zero action required, zero behavior change, on every such deployment.
- **Row present but disagrees with `.env`** — fatal error, refuses to start.
- **Row present and agrees** — starts normally.

So actually changing a deployment's mode means wiping its database and reinstalling from scratch — editing `.env` and restarting an existing deployment will only make it refuse to start.
