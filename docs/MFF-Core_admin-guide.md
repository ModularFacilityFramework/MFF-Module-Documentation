# Administrator Guide

This guide explains how administrators manage users, roles, and system configuration.

## Admin Responsibilities
- Approve/activate newly registered users
- Assign and manage roles (`developer`, `administrator`, `facility_manager`, `facility_staff`, `facility_superuser`, `facility_user`)
- Monitor system health and logs
- Configure environment and security settings

## Managing Users

### Users screen (Setup → Users)
- Each user has a Type badge: **Standard** or **No login**.
- The lists are Pending Approval, Active and Inactive.
- A self-registration appears under Pending Approval (it is inactive with `registration_pending` set). Use **Approve** or **Reject**; both only exist for pending registrations and clear the pending flag. "Inactive" no longer means "pending approval".
- The edit form has **Status** (Active/Inactive, for every account) and **Account type**.
- **Make inactive** replaces "Archive".
- The user receives an email notification on status change.

### User drop-downs and lists of people
Every list of people (Users tables, recipient lists, group PIs, search, exports) is sorted by first name, then last name. Drop-downs and recipient lists show "First Last (username)".

### Account types
- **Standard** accounts can log in (login, refresh and password reset work only for standard accounts, and login only when the account is also active).
- **No login** accounts are contacts/records: unusable password, no MFA, never log in, but still active/inactive, and can be a group member and a PI.
- Changing No login → Standard needs a password (8+ characters, with an upper-case letter, a lower-case letter and a digit).
- Changing Standard → No login removes the user's password and MFA and signs them out everywhere.

### Creating a no-login user
Setup → Users has a green **+ Create No-Login User** button (developer and administrator only). It opens the form "Create No-Login User", for a person who only needs a record: a PhD student, supervisor or PI. The person has no password, cannot sign in or reset a password, and is never sent an email. They can be put in groups, be group PIs, and be picked by modules (for example the finance PI and supervisor pickers).
- **Required:** username, first name, last name, email. Username and email must be unique, case-insensitively. The form checks the loaded list as you type, and the server answers a duplicate with an error.
- **Optional:** group, job position, university/affiliation, company, faculty, institute, building, room, phone, profile link, LinkedIn link, group website. The status defaults to active.
- To let the person sign in later, edit the user, set Account type to Standard and give a password (see above).

### Groups screen
- A group can have **several PIs**, exactly one of them primary. PIs are chosen from users (no-login accounts can be PIs). The PI selector offers only users whose job position is PI, so set a user's Job position to PI in Setup → Users first. PIs already linked to a group stay shown even if their position is not PI. The group's PI name and email are a copy of the primary PI; a group with no linked PI keeps free text.
- In the group form, choosing a PI in the "Add a PI" list adds them immediately; there is no separate Add button. Pick the primary PI with the radio button.
- PIs already linked to a group are kept and stay editable whatever their job position. New picks are limited to users whose job position is PI. A PI removed during the edit can be put back.
- If nobody has the job position PI, the form says so: set a user's Job position to PI in Setup → Users first.
- If a PI cannot be stored, the page shows "the PI ... was not saved" and keeps the form open.
- A group needs only a name; PIs are optional.

### Job positions and statistics
- The Users edit form has a "Job position" select. Positions come from the `setup_job_positions` table (see `docs/MFF-Core_db-schema.md`).
- Users & Groups has a "Statistics" tab: Groups Distribution and Job Roles Distribution. Click a group slice to filter the Job Roles pie; click it again, or press Reset, to show everyone.

### Roles
- Role assignments are stored in `setup_user_roles` (a `role_id` foreign key into `setup_roles` — not a free-text column)
- `administrator` is the default elevated role; `developer` acts as a break-glass superuser, bypassing all `checkRole()` checks (configured via `SUPERUSER_ROLE` in `.env` — empty by default)
- To change a user's role, update `setup_user_roles` via an admin UI (if available) or directly in the database, referencing the existing `setup_roles.id`

## Password Reset and MFA
- Password reset tokens are short-lived and one-time use
- MFA can be enabled per user via the `/auth/mfa/*` endpoints when the UI exposes the feature

## Files and Preferences
- Uploaded files are stored under the uploads directory defined by `UPLOAD_DIR_NAME`
- Preferences allow users to configure logos and UI options; admins may need to assist users with selecting organization assets

## Configuration and Environment
Set variables in the backend `.env` file:
- `BACKEND_PORT` and `API_BASE_PATH`
- Database: `DB_HOST`, `DB_USER`, `DB_PASSWORD`, `DB_NAME`
- Uploads: `UPLOAD_DIR_NAME`, `UPLOADS_BASE_PATH`, `PUBLIC_UPLOAD_DIR_NAME`
- Email: `SMTP_*`, `FROM_EMAIL`, `MAIL_ADMIN_EMAIL`
- Auth: `JWT_ACCESS_SECRET`, `JWT_REFRESH_SECRET`, expiration values
- Security: `CORS_ORIGINS`, `DATA_ENCRYPTION_KEY`, `ENABLE_METRICS`, `SUPERUSER_ROLE`

## Monitoring and Logs
- Structured logs are emitted via pino
- Health endpoint: `GET /api/health`
- Optional metrics: `GET /api/metrics` (when enabled)

## Backups (recommended)
- Schedule database backups (mysqldump/mariadb-dump)
- Periodically back up the uploads directory
- Test restore procedures regularly

## Common Admin Tasks
- Reset a user’s status to active after verification
- Revoke refresh tokens for a compromised account
- Rotate JWT secrets and reissue tokens
- Update allowed CORS origins when frontend URL changes
