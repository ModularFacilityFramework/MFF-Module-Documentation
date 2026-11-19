# API Documentation

This document describes the API exposed by the backend. All endpoints are prefixed by the API base path configured via `API_BASE_PATH` (default: `/api`); paths below are shown relative to that prefix.

Authentication
- Protected endpoints require `Authorization: Bearer <accessToken>`.
- Obtain tokens via `POST /auth/login`. If the account has MFA enabled, login returns a challenge instead of tokens — complete one of the `/auth/mfa/verify-*` endpoints to finish.
- When an access token expires, refresh via `POST /auth/refresh` (rotates the refresh token).
- File downloads (`GET /files/:filename`) also accept the access token as a `?token=` query parameter, since `<img>`/`<a>` tags can't send an `Authorization` header — do not rely on this for any other endpoint.

## Auth (`/auth`)

POST /auth/register-smtp
- Access: Public
- Body: `{ username, password, first_name, last_name, email, university?, company?, faculty?, institute?, building?, room?, phone_number?, profile_link?, photo_link?, linkedin_link?, group_website? }`
- Creates a standard account with `status: 'inactive'` and `registration_pending = 1` (pending admin approval) and sends a confirmation email via SMTP. "Inactive" no longer means "pending approval": only `registration_pending` does.
- Response: `{ success: true, message, mfaSetupToken }` — `mfaSetupToken` can be used with the registration-time MFA endpoints below for optional post-registration enrollment.

POST /auth/register-mail
- Access: Public
- Same body/behavior as `register-smtp`, but notifies via the local `mail` command instead of SMTP.
- Response: `{ success: true, message, mfaSetupToken }`

POST /auth/login
- Access: Public
- Body: `{ username, password }`
- If the account has TOTP enabled: `{ success: true, mfa_required: true, mfa_session_token }` — no tokens issued yet; call one of `/auth/mfa/verify-totp`, `/auth/mfa/verify-recovery`, or `/auth/mfa/verify-email` (after `/auth/mfa/request-email`) with this token to finish logging in.
- Otherwise: `{ success: true, accessToken, refreshToken, user: { id, username } }`
- Only a standard, active account can log in. A no-login account (`account_type = 'no_login'`) never logs in.

POST /auth/refresh
- Access: Public (requires a valid, unrevoked refresh token)
- Body: `{ refreshToken }`
- Rotates the refresh token (old one is revoked). Response: `{ success: true, accessToken, refreshToken }`
- Standard accounts only.

POST /auth/logout
- Access: Public
- Body: `{ refreshToken }` — revokes it.
- Response: `{ success: true, message }`

GET /auth/me
- Access: Protected
- Response: `{ success: true, user: { id, username, first_name, last_name, email, status }, roles: [{ role }] }`

Password reset (standard accounts only)
- `POST /auth/reset-password-request` — Body: `{ username }`. Always responds with a generic success message (no user enumeration), regardless of whether the account exists.
- `GET /auth/verify-reset-token/:token` — Response: `{ valid: boolean }`
- `POST /auth/reset-password` — Body: `{ token, password }`. On success, revokes all of the user's refresh tokens (forces re-login everywhere).

## MFA (`/auth/mfa/*`)

Login-time verification (public — requires the `mfa_session_token` issued by `POST /auth/login`):
- `POST /auth/mfa/verify-totp` — Body: `{ mfa_session_token, code }`
- `POST /auth/mfa/verify-recovery` — Body: `{ mfa_session_token, code }` (one-time use recovery code)
- `POST /auth/mfa/request-email` — Body: `{ mfa_session_token }` — sends a 6-digit email OTP, valid 10 minutes
- `POST /auth/mfa/verify-email` — Body: `{ mfa_session_token, code }`

Each verification endpoint responds `{ success: true, accessToken, refreshToken, user: { id, username } }` on success.

TOTP enrollment for an already-logged-in user (protected):
- `POST /auth/mfa/setup/init` — generates a TOTP secret, returns `{ success, secret, otpauth, qr }`
- `POST /auth/mfa/setup/confirm` — Body: `{ code }` — verifies the code, enables TOTP, returns `{ success, message, recoveryCodes: [10 codes] }`
- `GET /auth/mfa/setup/status` — Response: `{ success, totp_enabled, recovery_codes_remaining }`
- `POST /auth/mfa/setup/regenerate-recovery` — invalidates old recovery codes, returns 10 new ones
- `POST /auth/mfa/disable` — disables TOTP and clears recovery codes for the current user

Registration-time enrollment (public — requires the `mfaSetupToken` returned by `register-smtp`/`register-mail`):
- `POST /auth/mfa/setup/register-init` — Body: `{ mfa_setup_token }` — same response shape as `setup/init`
- `POST /auth/mfa/setup/register-confirm` — Body: `{ mfa_setup_token, code }` — same response shape as `setup/confirm`

## Email (`/email`)

POST /email/send-local-mail
- Access: Protected
- Body: `{ to, subject, body }`
- Response: `{ success: true, message }`

## Users (`/users`)

GET /users
- Access: Protected, `administrator` only
- Query (optional): `page`, `limit`, `sortBy`, `sortDir`, `search`
- Response (paginated): `{ success, items, total, page, pages, limit }` — each item includes `roles: [{ role }]`
- Each user also includes `job_position_id` and `job_position`, and `account_type` (`standard` or `no_login`) and `registration_pending`.
- Response (unpaginated/legacy): `{ success, users }`

GET /users/:id
- Access: Protected — self or `administrator`
- Response: `{ success, user }` with `user.roles` as a plain string array; `user` includes `account_type` and `registration_pending`.

PUT /users/:id
- Access: Protected, `developer` or `administrator`
- Body: any subset of the user's profile fields (`first_name`, `last_name`, `email`, `university`, `company`, `faculty`, `institute`, `building`, `room`, `phone_number`, `profile_link`, `photo_link`, `linkedin_link`, `group_website`, `group_id`, `status`, `job_position_id`, `account_type`, `password`)
- `job_position_id`: a valid job position id, or `''`/`null` to clear it. An unknown id is a `400` with "Unknown job position".
- `status` (`active`/`inactive`) applies to every account, standard or no-login.
- `account_type`: `standard` or `no_login`. Changing a no-login account to standard needs a `password` (8+ characters, with an upper-case letter, a lower-case letter and a digit). Changing a standard account to no-login removes its password and MFA and signs the user out everywhere. A no-login account has the unusable password `!no_login`, no MFA, and can still be a group member and a PI.

GET /users/job-positions
- Access: Protected (any logged-in user)
- Response: `{ success, jobPositions: [{ id, name }] }`

POST /users/:id/approve
- Access: Protected, `developer` or `administrator`
- Only for a pending registration (`registration_pending = 1`). Clears the flag.

POST /users/:id/reject
- Access: Protected, `developer` or `administrator`
- Only for a pending registration. Clears the flag.

POST /users/:id/activate
- Access: Protected, `developer` or `administrator`
- No body.
- Sets `status` to `active` for any account (standard or no-login), clears `registration_pending`, and gives the user the default role if none is stored. The user is emailed only for standard accounts.
- Response: `{ success, message: "User status updated to active" }`. `404` for an unknown id.

PUT /users/:id/status
- Access: Protected, `developer` or `administrator`
- Body: `{ status: 'active' | 'inactive' }` — also emails the user notifying them of the change.

DELETE /users/:id
- Access: Protected, `developer` or `administrator`
- Cascades to the user's roles and other owned rows.

## Groups (`/groups`)

- `GET /groups` — Protected (any authenticated user). Each group has `member_count` and `pis: [{ user_id, is_primary, username, first_name, last_name, name, email }]`, primary PI first.
- `GET /groups/pi-candidates` — Protected, `developer`/`administrator`/`facility_manager`/`facility_staff`
- `POST /groups` — Protected, `developer`/`administrator`/`facility_manager` — Body: `{ group_name, pis: [{ user_id, is_primary }] }`. The name alone is valid. A `400` is returned for two primaries, a duplicate user, or an unknown user.
- `PUT /groups/:id` — same roles as create. Same body; `pis: []` clears the PIs, and omitting `pis` leaves them unchanged. `pi_name`/`pi_email` are still accepted in the body, but only as legacy free text for a group with no linked PI (ignored once PIs are linked). `{ group_name, pis }` is the normal body.
- A group can have several PIs, exactly one of them primary (kept by the app). `pi_name`/`pi_email` on the group are a derived copy of the primary PI; a group with no linked PI keeps free text there. No-login accounts can be PIs.
- `PATCH /groups/:id/archive` / `PATCH /groups/:id/restore` — same roles as create
- `DELETE /groups/:id` — Protected, `developer`/`facility_manager`

## Preferences (mounted at the API root — no `/preferences` segment)

GET /user-preferences
- Access: Protected
- Returns the current user's own preferences, including their own `user_image_path`. Also still returns joined `faculty_logo_path`/`favicon_path`/`login_logo_path` for whatever happens to be on this user's own row, but those three are not the facility's active branding — see GET /app-branding and GET /login-logo below. Returns in-memory defaults if the user has no row yet.

PUT /user-preferences
- Access: Protected
- Body: `{ user_image_file_id?, theme?, language?, notifications?, auto_save?, default_view?, ticket_notification_user_ids?, project_info_user_ids?, user_management_user_ids? }`
- Personal preferences only. `faculty_logo_file_id`/`favicon_file_id`/`login_logo_file_id` are not accepted here (silently ignored if present) — see PUT /app-branding.
- Response: the updated preferences row, with joined branding-asset paths.

GET /app-branding
- Access: Public (no auth — the favicon and header logo must render for every user, logged in or not)
- Response: `{ faviconPath, facultyLogoPath }` — each independently resolved to whichever `setup_user_preferences` row most recently set that specific field (facility-wide, not per-user; an admin sets one or both via PUT /app-branding and it applies for everyone).

PUT /app-branding
- Access: Protected, `developer`/`administrator` only (or `SUPERUSER_ROLE`)
- Body: `{ faculty_logo_file_id?, favicon_file_id?, login_logo_file_id? }`
- Sets the facility-wide app logo, browser-tab favicon, and login-page logo. Any file id given is copied into the public, unauthenticated `public-uploads/` directory (served at `<API_BASE_PATH>/public-files/...`), since these assets render before login or via tags that can't send an auth header. Restricted to admin-capable roles because every user reads these back globally — an unrestricted write would let any logged-in user overwrite the whole facility's branding, not just their own.
- Response: `{ success, faviconPath, facultyLogoPath, loginLogoPath }` — the resulting global values.

GET /login-logo
- Access: Public (no auth — needed for the pre-login screen)
- Response: `{ loginLogoUrl, loginLogoPath }` or `{ loginLogoUrl: null }` — same "most recently set wins" global resolution as GET /app-branding, kept as its own route since it's needed before login.

## Page access (`/page-access`)

- `GET /page-access` — Protected — full list of configured pages
- `GET /page-access/my-pages` — Protected — pages the current user can see, filtered by role (`developer` sees everything)
- `GET /page-access/check/:pageKey` — Protected — `{ success, canAccess }`
- `PATCH /page-access/:id` — Protected, `developer`/`administrator`/`facility_manager` — Body: `{ allowed_roles?, is_enabled? }`

## Announcements (`/announcements`)

- `GET /announcements` — Public — active announcements only
- `GET /announcements/all` — Protected, `administrator` only — includes suspended announcements
- `POST /announcements` — Protected, `administrator` only — Body: `{ title, content, image_file_id?, weight?, status? }`
- `PUT /announcements/:id` — Protected, `administrator` only
- `DELETE /announcements/:id` — Protected, `administrator` only

## Notifications (`/notifications`)

- `GET /notifications` — Protected — latest 100 for the current user, unread first
- `POST /notifications/:id/read` — Protected — marks one as read

## Comments (`/comments`)

- `GET /comments?entity=&entityId=` — Protected — comments on a given entity
- `POST /comments` — Protected — Body: `{ entity, entityId, body }`
- `PUT /comments/:id` — Protected, owner or admin — Body: `{ body }`
- `DELETE /comments/:id` — Protected, owner or admin — soft delete (`is_deleted = 1`)

## Tickets (`/tickets`)

Ticket "staff" roles (`developer`, `administrator`, `facility_manager`, `facility_staff`, `facility_superuser`) see and manage everyone's tickets; the base `facility_user` role only sees and can respond to their own.

- `GET /tickets` — Protected — full list for staff, own tickets only otherwise
- `POST /tickets` — Protected (any authenticated user) — Body: `{ title, category?, priority?, description }`
- `PUT /tickets/:id` — Protected, staff only — Body: `{ status }`
- `POST /tickets/:id/response` — Protected — staff or the ticket's own owner — Body: `{ response }`
- `GET /tickets/notification-users` — Protected, staff only — users configured to receive ticket notification emails
- `PUT /tickets/:id/assign` — Protected, staff only — Body: `{ assigned_to_id }`
- `DELETE /tickets/:id` — Protected, staff only

## Files (`/files`)

Private uploads are encrypted at rest (AES-256-GCM) and require both authentication and authorization to download — see `GET /files/:filename` below. This is a deliberate hardening from an earlier design where the download route had no auth check at all.

POST /files/upload
- Access: Protected
- Form data: `file` (multipart)
- Auto-assigns the uploader's own roles as the file's initial `core_file_roles`.
- Response: `{ success, fileId, file: { id, filename, originalName, mimetype, size, url } }`

GET /files
- Access: Protected
- Query (optional): `page`, `limit`, `sortBy`, `sortDir`, `search`
- Response (paginated): `{ success, items, total, page, pages, limit }`, each item including `roles: []`
- Response (unpaginated/legacy): `Array<File>` with `url`/`roles` fields

GET /files/available-roles
- Access: Protected
- Response: `{ success, roles: [role names] }` — must be registered before `/:filename` in the router so it isn't swallowed by that param route.

GET /files/:filename
- Access: **Protected and authorized** — accepts the token via `Authorization` header or `?token=` query param.
- Access is granted to: the file's owner, an `administrator`/`developer`, or a user sharing one of the file's assigned `core_file_roles`. If a file has no `core_file_uploads` record at all (legacy/orphaned), it's served to any authenticated user, since there's no ownership/role data to check.
- Response: the decrypted file content with `Content-Type`/`Content-Disposition` headers (inline for images, attachment otherwise).

PATCH /files/:id
- Access: Protected, owner or admin/developer
- Body: `{ file_category?, description? }`

DELETE /files/:id
- Access: Protected, owner or admin/developer

GET /files/:id/roles
- Access: Protected, owner or admin/developer
- Response: `{ success, roles: [role names] }`

POST /files/:id/roles
- Access: Protected, owner or admin/developer
- Body: `{ roles: [role names] }`

DELETE /files/:id/roles/:role
- Access: Protected, owner or admin/developer

## Shares (`/shares`)

- `GET /shares?entity=&entityId=` — Protected — owner-only view (recipients aren't shown their incoming shares here yet)
- `POST /shares` — Protected — Body: `{ entity, entityId, targetUserId?, targetRole?, canEdit?, expiresAt? }`
- `DELETE /shares/:id` — Protected, owner only

## Reports (`/reports`)

- `GET /reports/users-summary` — Protected, `administrator` only — counts by status and by role
- `GET /reports/files-summary` — Protected, `administrator` only — counts/total size by file type, and uploads per day (last 30 days)
- `GET /reports/groups-distribution` — Protected, `facility_staff`, `facility_manager`, `administrator` or `developer` — chart shape (see `docs/MFF-Core_module-core-api.md`, "Charts: the shared drill-down pie"): members per group. Users without a group are "No group".
- `GET /reports/job-roles-distribution?groupId=` — same roles — chart shape: **active** users per job position, "Not set" for none. `groupId` omitted = everyone; `groupId=0` = users with no group; a non-integer or negative value is a `400`.
- `GET /reports/:name.csv` — Protected, `administrator` only — CSV export; `name` must be `users` or `files`

## Search (`/search`)

- `GET /search?q=<term>` — Protected — aggregated results across users and files (top 10 each)
- `GET /search/health` — Public — `{ ok: true, service: 'search' }`

## Health, Config, Metrics

- `GET /health` — Public — `{ status: 'ok', timestamp }`
- `GET /config` — Public — runtime config the frontend fetches at startup (app name, banner title, API base path, upload path defaults). There is no frontend `.env`; this endpoint is the only source of frontend runtime config.
- `GET /modules` — Public — `[{ id, label, version }, ...]`, the active module registry (see `App_backend/src/core/services/moduleRegistry.js`). Returns `[]` when no modules are installed; the frontend uses this to decide which module-provided dashboard tabs to render.
- `GET /metrics` — Public when `ENABLE_METRICS=true` — Prometheus metrics

## Error Shape

Errors generally follow `{ success: false, message, error? }`, though a few endpoints (groups, tickets, search) instead return `{ error: <string> }` directly.

## Notes

- API base path is configurable via `API_BASE_PATH`; examples above assume `/api`.
- Some list endpoints (`/users`, `/files`) support both a paginated response and an older unpaginated/array response, kept for backward compatibility with existing frontend code.
