# API Documentation

This document describes the API exposed by the backend. All endpoints are prefixed by the API base path configured via `API_BASE_PATH` (default: `/api`); paths below are shown relative to that prefix.

Authentication
- Protected endpoints require `Authorization: Bearer <accessToken>`.
- Obtain tokens via `POST /auth/login`. If the account has MFA enabled, login returns a challenge instead of tokens — complete one of the `/auth/mfa/verify-*` endpoints to finish.
- When an access token expires, refresh via `POST /auth/refresh` (rotates the refresh token).
- File downloads (`GET /files/:filename`) also accept the access token as a `?token=` query parameter, since `<img>`/`<a>` tags can't send an `Authorization` header. Besides this, only module routes listed in their manifest's `queryTokenRoutes` accept it; no other core endpoint should be relied on for it. Core's request log shows such a token as `token=[redacted]`.

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

Registering with the username or email of a no-login record (both register routes, core v0.1.222):
- The username or email of a **no-login** record is accepted. The registration is a new pending row linked to that record; the record is not changed.
- For whatever collides, the pending row holds a placeholder: `~reg-<hex>` (username) and/or `~reg-<hex>@pending.invalid` (email).
- Refused with `400`:
  - "Username already exists" (a login account has it);
  - "An account with this email already exists" (a login account has it);
  - "A registration with this username or email is already waiting for approval. Please contact an administrator.";
  - usernames starting with `~`.
- The admin notification mail names the matched record.
- Login with a username that has a waiting registration answers "Account is not active. Please contact an administrator."

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
- Query (optional): `page`, `limit`, `sortBy`, `sortDir`, `search`. The default order is by name (first name, last name, username), not by id. `sortBy` accepts `name` (the new default), `id`, `username`, `email`, `status`, `created_at`, `last_login`, `first_name` and `last_name`.
- Response (paginated): `{ success, items, total, page, pages, limit }` — each item includes `roles: [{ role }]`
- Each user also includes `job_position_id` and `job_position`, and `account_type` (`standard` or `no_login`) and `registration_pending`.
- Each user also includes `requested_username`, `requested_email`, `merge_target_id` and `merge_target_username` (core v0.1.222; see "Merging a registration into a no-login record" below).
- Response (unpaginated/legacy): `{ success, users }`

GET /users/:id
- Access: Protected — self or `administrator`
- Response: `{ success, user }` with `user.roles` as a plain string array; `user` includes `account_type` and `registration_pending`.

POST /users/no-login
- Access: Protected, `developer` or `administrator`
- Creates a no-login account (a record-only person who never signs in; no email is sent).
- Body: `username`, `first_name`, `last_name`, `email` (required); `group_id`, `job_position_id`, `university`, `company`, `faculty`, `institute`, `building`, `room`, `phone_number`, `profile_link`, `linkedin_link`, `group_website`, `status` (optional; `status` defaults to `active`).
- Response: `201 { success, id }`. `400` for bad or missing input, or an unknown group or job position. `409` for a duplicate username or email (compared case-insensitively). `403` for other roles.
- The account is stored with `account_type` `no_login`, `password_hash` `!no_login`, `registration_pending` 0, no MFA, and one role row `facility_user` (the same as the migrated users).

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
- May answer `409 { mergeRequired: true }` (see "Merging a registration into a no-login record" below).

POST /users/:id/reject
- Access: Protected, `developer` or `administrator`
- Only for a pending registration. Clears the flag. For a registration that matches a no-login record, it removes only the request (see "Merging a registration into a no-login record" below).

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

### Merging a registration into a no-login record
Core v0.1.222. All four routes are for `developer` or `administrator` only. Errors are `{ success: false, message, blockers?, stale? }`.

- `GET /users/merge-log` — Response: `{ success, entries: [{ id, created_at, action: 'merge'|'reject', performed_by, performed_by_username, source_user_id, source_username, target_user_id, target_username }] }`
- `GET /users/:id/merge-candidates` — Response: `{ success, source, problem, linked_target_id, suggestions, records }`. Each of `suggestions` is a record plus `score` and `reasons[]`; `records` is all no-login records.
- `POST /users/:id/merge-preview` — Body: `{ target_id, choices: { <field>: 'record'|'registration' } }`. Response: `{ success, plan: { source, target, result_username, result_email, result_roles, fields[], changes[], kept[], moved[], dropped[], warnings[], blockers[], fingerprint } }`
- `POST /users/:id/merge` — Body: `{ target_id, choices, fingerprint }`. Response: `{ success, id, username, emailed, message }`. `409` with `stale: true` if anything changed since the preview.

Pending registrations that match a record:
- Activating or approving a registration whose requested username or email a record still holds returns `409 { mergeRequired: true }`. This also applies to `PUT /users/:id` setting it to `active` or to `no_login`. If both the username and the email are free again, they are restored and the request goes ahead.
- Rejecting (or deactivating) such a pending registration removes only the request: `{ success, removed: true, message: "Registration rejected. The existing record is unchanged." }`, and it is logged.

### Projects at registration approval
Core v0.1.227. Administrators and developers.

- `GET /users/:id/project-options` — `{ success, available, projects: [{ id, number, title, owner_label, target_access }] }`.
- `POST /users/:id/project-assignment` — body `{ create: { title, description } | null, grants: [{ project_id, access: 'read'|'write' }] }`; returns `{ success, created, granted }`. `409` until the account is approved; `404` when the provider cannot do it.

## Groups (`/groups`)

- `GET /groups` — Protected (any authenticated user). Each group has `member_count` and `pis: [{ user_id, is_primary, username, first_name, last_name, name, email }]`, primary PI first, then by first and last name (not by user id).
- `GET /groups/pi-candidates` — Protected, `developer`/`administrator`/`facility_manager`/`facility_staff` — per user: `id`, `username`, `email`, `account_type`, `status`, `name`, `first_name`, `last_name`, `job_position`, `is_pi`; sorted by first name, then last name
- `POST /groups` — Protected, `developer`/`administrator`/`facility_manager` — Body: `{ group_name, pis: [{ user_id, is_primary }] }`. The name alone is valid. A `400` is returned for two primaries, a duplicate user, or an unknown user.
- `PUT /groups/:id` — same roles as create. Same body; `pis: []` clears the PIs, and omitting `pis` leaves them unchanged. `pi_name`/`pi_email` are still accepted in the body, but only as legacy free text for a group with no linked PI (ignored once PIs are linked). `{ group_name, pis }` is the normal body.
- `POST /groups` (201) and `PUT /groups/:id` (200) also return `pis`: the PIs actually stored, primary first and then by name (the same shape as the `pis` of `GET /groups`). A `500` is returned if the stored count differs from the number sent.
- A group can have several PIs, exactly one of them primary (kept by the app). `pi_name`/`pi_email` on the group are a derived copy of the primary PI; a group with no linked PI keeps free text there. No-login accounts can be PIs.
- `PATCH /groups/:id/archive` / `PATCH /groups/:id/restore` — same roles as create
- `DELETE /groups/:id` — Protected, `developer`/`facility_manager`

## Preferences (mounted at the API root — no `/preferences` segment)

GET /user-preferences
- Access: Protected
- Returns the current user's own preferences, including their own `user_image_path`. The facility branding is not part of the personal preferences: see GET /app-branding and GET /login-logo below. Returns in-memory defaults if the user has no row yet.

PUT /user-preferences
- Access: Protected
- Body: `{ user_image_file_id?, theme?, language?, notifications?, auto_save?, default_view?, ticket_notification_user_ids?, project_info_user_ids?, user_management_user_ids? }`
- Personal preferences only. The branding keys (`faculty_logo_file_id`, `favicon_file_id`, `login_logo_file_id`) are ignored if present — see PUT /app-branding. The profile image (`user_image_file_id`) stays per user.
- Response: the updated preferences row.

GET /app-branding
- Access: Public (no auth — the favicon and header logo must render for every user, logged in or not)
- Response: `{ faviconPath, facultyLogoPath, loginLogoPath, favicon_file_id, faculty_logo_file_id, login_logo_file_id }`: the facility-wide branding, one slot each, read from `core_app_branding`. A file id is null when its slot is empty. The three paths are file names only (what `/public-files/<name>` serves), never the server's absolute path. The answer has `Cache-Control: no-cache`.

PUT /app-branding
- Access: Protected, `developer`/`administrator` only (or `SUPERUSER_ROLE`)
- Body: `{ faculty_logo_file_id?, favicon_file_id?, login_logo_file_id? }`. The update is **partial**: only the keys present are written. `null`, `''` or `0` clears just that slot; an absent key leaves its slot untouched. A non-integer or nonexistent file id is a `400` and nothing is saved.
- Sets the facility-wide app logo, browser-tab favicon, and login-page logo. Any file id given is copied into the public, unauthenticated `public-uploads/` directory (served at `<API_BASE_PATH>/public-files/...`), since these assets render before login or via tags that can't send an auth header. Restricted to admin-capable roles because every user reads these back globally.
- Response: `{ success, copyErrors, faviconPath, facultyLogoPath, loginLogoPath, favicon_file_id, faculty_logo_file_id, login_logo_file_id }`.

GET /login-logo
- Access: Public (no auth — needed for the pre-login screen)
- Response: `{ loginLogoUrl, loginLogoPath }` or `{ loginLogoUrl: null }` — reads the login slot only, kept as its own route since it's needed before login. `loginLogoUrl` and `loginLogoPath` are file names only, never the server's absolute path, and the answer has `Cache-Control: no-cache`.

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
- `GET /tickets/notification-users` — Protected, staff only — users configured to receive ticket notification emails; sorted by first name, then last name
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

## Page state (`/page-state`)

Remembers each user's own page state (see `docs/MFF-Core_module-core-api.md`, "`usePageState`"). All routes are authenticated, touch only the caller's own state, and answer with `Cache-Control: no-store`.

- `GET /page-state` — all the caller's own keys, or only those starting with `?prefix=`
- `PUT /page-state/:key` — Body: `{ value }`
- `DELETE /page-state/:key`

Errors: `400` for a bad key or value; `413` for a value over 64 KB or more than 1000 keys.

## Modules management (`/modules-management`)

"Update all" on the Modules page (`docs/MFF-Core_module-install-finalize.md`, "Update all"). `developer`/`administrator` only.

- `GET /modules-management/update-all/preview` — checks every installed module with `git fetch`. The page does not use this route any more. Response: `{ updates: [{ id, label, from, to, behind }], unchecked: [{ id, reason }] }`. From core v0.1.217 an entry may also carry `retry: true` and `reasons: [text]` (an unfinished update, see `docs/MFF-Core_module-install-finalize.md`, "Retrying an unfinished update"); the same applies to `update-all` and to the job's `modules`
- `POST /modules-management/update-all` — starts the job. Body: `{ modules?: [ids] }` (only the named modules that still have an update). Response: `{ job, updates, unchecked, message }`
- `GET /modules-management/health` — installed modules whose last update is unfinished (core v0.1.217). Response: `{ success, modules: [{ id, label, registered, reasons: [{ code: 'failed' | 'deps' | 'build' | 'loaded', message }] }] }`; only modules with reasons are listed.
- `GET /modules-management/available` — the modules that can be installed. Core no longer filters repos by name: every repo of the `ModularFacilityFramework` organisation (all pages of the GitHub listing, not just the first 100) whose default branch has a valid `module.manifest.json` with an `id` is a candidate, whatever the repo is called. Each entry has `id`, `label`, `repo`, `duplicateId`, `archived`, `alreadyCloned`, `pendingRestart`, `requires`, `requirementsError` and `missingRequirements`. The response also has `skippedRepos: [{ repo, reason }]` (repos without a usable manifest) and `reposSeen: [names]`. The result is cached for 10 minutes; an Install refreshes it.
- `POST /modules-management/:id/install` — installs a module. Body: `{ repo }` (optional) installs that repo. Without `repo`, an id declared by exactly one repo installs as before; an id declared by several repos answers `409` listing the repos; a `repo` that does not declare the id answers `404`.
- `GET /modules-management/job` — the latest job, with `modules: [{ id, label, from, to, behind, status, step, message }]` (`status` is `pending`, `running`, `done` or `failed`) and `failedModules`.

## Projects (`/projects`)

Core v0.1.218. Served from the projects provider registered by the module that owns projects (`docs/MFF-Core_module-core-api.md`, "Projects"). All routes need a login (`401`), **not** access to that module; project membership, decided by the provider, is the control.

- `GET /projects/provider` — `{ available, module_id, can_create }`.
- `GET /projects?scope=mine|shared|all&search=&archived=1` — `Project[]`. Archived projects are left out unless `archived=1`. `scope=all` is for administrators and developers only, else `403`; any other scope is `400`; no provider is `404`.
- `GET /projects/:id` — `Project`; `404` if absent or not readable.
- `POST /projects` — body `{ title, description }`, `201` with the `Project`. `400` with no title, `403` if the user cannot create, `404` with no provider; read-only view-as sessions get `403`.

Added in core v0.1.227 (all need a login, not access to the owner module):
- `GET /projects/status` — `{ available, module_id, can_create, has_writable_project (true|false|null), can_request, is_approver, pending_request, request_link, can_open_owner }`.
- `GET /projects/settings` — `{ members_may_create }`. `PUT /projects/settings` — administrators and developers; body `{ members_may_create: true|false }`, else `400`/`403`.
- `GET /projects/requests` — your own requests. `POST /projects/requests` — `{ title, description, group_id?, pi_user_id? }`, `201`; `400` with no title. `POST /projects/requests/:id/cancel` — `{ cancelled: true }`. `404` when the provider has no requests.
- `GET /projects/:id/items?module_id=` — items; `404` if the project is not readable or items are not available.
- `POST /projects` now answers `403` to a person with no project (see the rules in `docs/MFF-Core_module-core-api.md`, "Projects").

A provider failure answers `500` "The projects request failed".

## Reports (`/reports`)

- `GET /reports/users-summary` — Protected, `administrator` only — counts by status and by role
- `GET /reports/files-summary` — Protected, `administrator` only — counts/total size by file type, and uploads per day (last 30 days)
- `GET /reports/groups-distribution` — Protected, `facility_staff`, `facility_manager`, `administrator` or `developer` — chart shape (see `docs/MFF-Core_module-core-api.md`, "Charts: the shared drill-down pie"): members per group. Users without a group are "No group".
- `GET /reports/job-roles-distribution?groupId=` — same roles — chart shape: **active** users per job position, "Not set" for none. `groupId` omitted = everyone; `groupId=0` = users with no group; a non-integer or negative value is a `400`.
- `GET /reports/:name.csv` — Protected, `administrator` only — CSV export; `name` must be `users` or `files`. `users.csv` has the columns `first_name,last_name,username,email,status,created_at,id` (it used to be `id,username,email,status,created_at`) and its rows are sorted by name

## Directory (`/directory`)

- `GET /directory?search=&limit=` — Protected (any logged-in user); with no `limit` every matching user and group is returned, A to Z (core v0.1.216; there is no row cap any more); `limit=N` is honoured only when a caller sends it explicitly. Response: `{ users: [{ id, username, label, first_name, last_name, account_type, inactive }], groups: [{ id, label }] }`. By default active standard accounts and non-archived groups only; no email, no roles and no PI details. It supplies the candidates of the shared `ShareDialog` and `UserPicker` (`docs/MFF-Core_module-core-api.md`, "Projects" and "People, group and job position pickers"). Extra query parameters:
  - `includeNoLogin=1` — also returns active no-login contacts.
  - `type=users` — leave out `groups`.
  - `ids=1,2` — look those users up by id (uncapped), whatever their status (a disabled user comes back with `inactive` set); a pending registration is never returned.
- `GET /directory/groups` — Protected (any logged-in user). No row cap (core v0.1.216): every non-archived group is returned. Response: `{ groups: [{ id, label }] }` (non-archived groups).
- `GET /directory/job-positions` — Protected (any logged-in user). Response: `{ jobPositions: [{ id, name }] }`.

## Search (`/search`)

- `GET /search?q=<term>` — Protected — aggregated results across users and files (top 10 each); user results are sorted by name
- `GET /search/health` — Public — `{ ok: true, service: 'search' }`

## Health, Config, Metrics

- `GET /health` — Public — `{ status: 'ok', timestamp }`
- `GET /config` — Public — runtime config the frontend fetches at startup (app name, banner title, API base path, upload path defaults). There is no frontend `.env`; this endpoint is the only source of frontend runtime config.
- `GET /modules` — Public (token optional) — `[{ id, label, version }, ...]`, the active module registry (see `App_backend/src/core/services/moduleRegistry.js`). With a token the list is filtered to what that user may use; with no token it is the full list (without any module or page that restricts itself with `registerAccessRestriction`, core v0.1.234). An expired or refused Bearer token: `401`. Token cannot be checked or list cannot be filtered: `503`. Returns `[]` when no modules are installed; the frontend uses this to decide which module-provided dashboard tabs to render.
- `GET /metrics` — Public when `ENABLE_METRICS=true` — Prometheus metrics

## Error Shape

Errors generally follow `{ success: false, message, error? }`, though a few endpoints (groups, tickets, search) instead return `{ error: <string> }` directly.

## Notes

- API base path is configurable via `API_BASE_PATH`; examples above assume `/api`.
- Some list endpoints (`/users`, `/files`) support both a paginated response and an older unpaginated/array response, kept for backward compatibility with existing frontend code.
