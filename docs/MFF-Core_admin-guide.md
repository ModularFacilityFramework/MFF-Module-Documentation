# Administrator Guide

This guide explains how administrators manage users, roles, and system configuration.

## Admin Responsibilities
- Approve/activate newly registered users
- Assign and manage roles (`developer`, `administrator`, `facility_manager`, `facility_staff`, `facility_superuser`, `facility_user`)
- Monitor system health and logs
- Configure environment and security settings

## Managing Users

### Activate or deactivate users
- Navigate to the Users tab (visible if your account has the required permissions)
- Find the user and update their status to Active/Inactive
- The user receives an email notification on status change

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
