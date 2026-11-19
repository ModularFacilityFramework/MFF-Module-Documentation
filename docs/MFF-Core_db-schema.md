# Database Schema and Relationships

This document describes the core tables defined in `App_backend/install/schema.sql`, the single source of truth for the schema (no migrations — see `CLAUDE.md`). Every core table is prefixed `core_` or `setup_`, matching the same domain split used for the route/page file structure. A module prefixes its own tables with its module ID (e.g. `chem_`, `slurm_`) and ships its own setup script; module tables are not covered here.

## Setup domain

### setup_roles
- id BIGINT PK
- role VARCHAR(50) NOT NULL UNIQUE
- description VARCHAR(255) NULL
- created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP

Seeded roles: `developer` (break-glass superuser, see `SUPERUSER_ROLE` in `.env`), `administrator`, `facility_manager`, `facility_staff`, `facility_superuser`, `facility_user`.

### setup_groups
- id BIGINT PK
- group_name VARCHAR(255) NOT NULL UNIQUE
- pi_name VARCHAR(255) NULL
- pi_email VARCHAR(255) NULL
- created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
- archived_at TIMESTAMP NULL

`pi_name`/`pi_email` are a derived copy of the group's primary PI (see `setup_group_pis`). A group with no linked PI keeps free text there.

### setup_group_pis
- group_id BIGINT NOT NULL REFERENCES setup_groups(id) ON DELETE CASCADE
- user_id BIGINT NOT NULL REFERENCES setup_users(id) ON DELETE CASCADE
- is_primary TINYINT(1) NOT NULL DEFAULT 0
- created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
- PRIMARY KEY (group_id, user_id)

Several PIs per group, exactly one primary per group (kept by the app, not by the database). Created automatically at startup if missing; at startup, groups whose `pi_email` matches a user are linked to that user once, as the primary PI.

### setup_job_positions
- id BIGINT AUTO_INCREMENT PK
- name VARCHAR(100) NOT NULL UNIQUE
- sort_order INT NOT NULL DEFAULT 0
- created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP

Seeded with: PhD student, Master student, PI, Staff, Postdoc, Senior Postdoc, RA, SRA, Technician, SO, SSO, Other. Created and seeded automatically at startup on an existing deployment.

### setup_users
- id BIGINT PK
- username VARCHAR(50) NOT NULL UNIQUE
- first_name VARCHAR(100) NOT NULL
- last_name VARCHAR(100) NOT NULL
- email VARCHAR(255) NOT NULL UNIQUE
- password_hash VARCHAR(255) NOT NULL
- university VARCHAR(255) NULL
- company VARCHAR(255) NULL
- faculty VARCHAR(255) NULL
- institute VARCHAR(255) NULL
- building VARCHAR(255) NULL
- room VARCHAR(50) NULL
- phone_number VARCHAR(50) NULL
- profile_link VARCHAR(255) NULL
- photo_link VARCHAR(255) NULL
- linkedin_link VARCHAR(255) NULL
- group_website VARCHAR(255) NULL
- status ENUM('active','inactive') DEFAULT 'inactive' (applies to every account)
- account_type ENUM('standard','no_login') NOT NULL DEFAULT 'standard'
- registration_pending TINYINT(1) NOT NULL DEFAULT 0
- notes TEXT NULL
- group_id BIGINT NULL REFERENCES setup_groups(id) ON DELETE SET NULL
- job_position_id BIGINT NULL REFERENCES setup_job_positions(id) ON DELETE SET NULL (added automatically at startup on an existing deployment)
- totp_secret VARCHAR(255) NULL
- totp_enabled TINYINT(1) NOT NULL DEFAULT 0
- last_login TIMESTAMP NULL
- created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP

Account types: a `standard` account can log in (login, refresh and password reset work only for standard accounts, and login only when also `active`). A `no_login` account is a contact/record: its `password_hash` is the unusable value `!no_login`, it has no MFA, it never logs in, it can still be `active`/`inactive`, and it can be a group member and a PI. A self-registration is created `inactive` with `registration_pending = 1`; approving or rejecting it (`POST /users/:id/approve`, `/reject`) clears the flag.

### setup_user_roles
- id BIGINT PK
- user_id BIGINT NOT NULL REFERENCES setup_users(id) ON DELETE CASCADE
- role_id BIGINT NOT NULL REFERENCES setup_roles(id) ON DELETE CASCADE
- assigned_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
- UNIQUE (user_id, role_id)

Normalized many-to-many between users and roles — `role_id` is a foreign key, never a free-text role string. This is deliberate: a plain-string design previously let `'admin'` and `'administrator'` silently diverge across different files, breaking permission checks inconsistently. The FK makes assigning a nonexistent role impossible at the database level. `middleware/auth.js` is the single place this table is queried (joined against `setup_roles`) to populate `req.user.roles`; nothing else should re-query it.

### setup_user_preferences
- id BIGINT PK
- user_id BIGINT UNIQUE NOT NULL REFERENCES setup_users(id) ON DELETE CASCADE
- faculty_logo_file_id, user_image_file_id, favicon_file_id, login_logo_file_id BIGINT NULL (reference `core_file_uploads(id)` informally — not FK-constrained, since favicon/login_logo/faculty_logo may point at files copied into the public, non-database-tracked `public-uploads/` directory). Despite living on this per-user table, faculty_logo/favicon/login_logo are facility-wide settings, not personal preferences — they're only ever written via `PUT /app-branding` (admin-only) and read back globally via `GET /app-branding`/`GET /login-logo` (whichever row most recently set that field wins), never via this table's own per-user routes. Only `user_image_file_id` is genuinely per-user.
- theme VARCHAR(32) DEFAULT 'light'
- language VARCHAR(8) DEFAULT 'en'
- notifications TINYINT(1) DEFAULT 1
- auto_save TINYINT(1) DEFAULT 1
- default_view VARCHAR(64) DEFAULT 'dashboard'
- ticket_notification_user_ids TEXT NULL (comma-separated user IDs)
- project_info_user_ids TEXT NULL (comma-separated user IDs)
- user_management_user_ids TEXT NULL (comma-separated user IDs)
- created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
- updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP

### setup_page_access
- id BIGINT PK
- page_key VARCHAR(100) NOT NULL UNIQUE
- page_name VARCHAR(255) NOT NULL
- parent_key VARCHAR(100) NULL (for page hierarchy)
- page_order INT DEFAULT 0
- allowed_roles TEXT NULL (comma-separated role names; NULL = all roles)
- is_enabled TINYINT(1) DEFAULT 1
- created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
- updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP

### setup_module_access
- module_id VARCHAR(64) NOT NULL (module manifest id; not a FK)
- subject_type ENUM('user','group') NOT NULL
- subject_id BIGINT NOT NULL (setup_users.id or setup_groups.id depending on subject_type)
- created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
- PRIMARY KEY (module_id, subject_type, subject_id)
- A module with no grants (here or in setup_module_access_roles) is open to everyone. See MFF-Core_module-access.md.

### setup_module_access_roles
- module_id VARCHAR(64) NOT NULL (module manifest id; not a FK)
- role_id BIGINT NOT NULL REFERENCES setup_roles(id) ON DELETE CASCADE
- created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
- PRIMARY KEY (module_id, role_id)

### setup_password_resets
- id BIGINT PK
- user_id BIGINT NOT NULL REFERENCES setup_users(id) ON DELETE CASCADE
- token_hash VARCHAR(128) NOT NULL (SHA-256 of the reset token — the raw token is never stored)
- expires_at DATETIME NOT NULL
- used TINYINT(1) DEFAULT 0
- created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP

### setup_mfa_setup_pending
- id BIGINT PK
- user_id BIGINT NOT NULL UNIQUE REFERENCES setup_users(id) ON DELETE CASCADE
- secret_enc TEXT NOT NULL (encrypted TOTP secret, awaiting confirmation)
- expires_at DATETIME NOT NULL
- created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP

### setup_mfa_recovery_codes
- id BIGINT PK
- user_id BIGINT NOT NULL REFERENCES setup_users(id) ON DELETE CASCADE
- code_hash VARCHAR(64) NOT NULL
- used TINYINT(1) NOT NULL DEFAULT 0
- created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP

### setup_mfa_email_otp
- id BIGINT PK
- user_id BIGINT NOT NULL REFERENCES setup_users(id) ON DELETE CASCADE
- code_hash VARCHAR(64) NOT NULL
- expires_at DATETIME NOT NULL
- used TINYINT(1) NOT NULL DEFAULT 0
- created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP

## Core domain

### core_deployment_mode
- id TINYINT PK, fixed at 1 (single row only)
- mode ENUM('standard','predetermined') NOT NULL
- set_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP

Written once by `install/setup-database.js`, never updated afterward. See `docs/MFF-Core_predetermined-mode.md`.

### core_file_uploads
- id BIGINT PK
- file_name VARCHAR(255) NOT NULL (stored filename on disk)
- original_name VARCHAR(255) NOT NULL
- file_path VARCHAR(500) NOT NULL
- file_type VARCHAR(100) NOT NULL (MIME type)
- file_size BIGINT NOT NULL
- file_category VARCHAR(50) NOT NULL DEFAULT 'document'
- description TEXT NULL
- uploaded_by BIGINT NOT NULL REFERENCES setup_users(id) ON DELETE CASCADE
- upload_timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP

Private uploads are encrypted at rest (AES-256-GCM, see `cryptoService.js`) and served only through the authenticated/authorized `GET /:filename` route. The three public branding assets (favicon, login logo, faculty logo) are copied out to a separate unauthenticated `public-uploads/` directory instead — see `setup_user_preferences` above.

### core_file_roles
- id BIGINT PK
- file_id BIGINT NOT NULL REFERENCES core_file_uploads(id) ON DELETE CASCADE
- role VARCHAR(50) NOT NULL
- created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
- UNIQUE (file_id, role)

Roles allowed to access a given file, in addition to its owner and admins/developers. Auto-populated with the uploader's own roles at upload time; adjustable via `POST`/`DELETE /files/:id/roles`.

### core_announcements
- id BIGINT PK
- title VARCHAR(255) NOT NULL
- content TEXT NOT NULL
- image_file_id BIGINT NULL REFERENCES core_file_uploads(id) ON DELETE SET NULL
- weight INT DEFAULT 5
- status ENUM('active','suspended') DEFAULT 'active'
- created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
- updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP

### core_tickets
- id BIGINT PK
- user_id BIGINT NOT NULL REFERENCES setup_users(id) ON DELETE CASCADE
- title VARCHAR(255) NOT NULL
- category VARCHAR(50) NOT NULL DEFAULT 'technical'
- priority ENUM('low','medium','high') NOT NULL DEFAULT 'medium'
- description TEXT NOT NULL
- status ENUM('open','in_progress','resolved') NOT NULL DEFAULT 'open'
- assigned_to_id BIGINT NULL REFERENCES setup_users(id) ON DELETE SET NULL
- created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
- updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP

### core_ticket_responses
- id BIGINT PK
- ticket_id BIGINT NOT NULL REFERENCES core_tickets(id) ON DELETE CASCADE
- user_id BIGINT NOT NULL REFERENCES setup_users(id) ON DELETE CASCADE
- response TEXT NOT NULL
- created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP

### core_notifications
- id BIGINT PK
- user_id BIGINT NOT NULL REFERENCES setup_users(id) ON DELETE CASCADE
- type VARCHAR(64) NOT NULL
- payload_json TEXT NOT NULL
- read_at DATETIME NULL
- created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP

### core_comments
- id BIGINT PK
- entity_type VARCHAR(32) NOT NULL (e.g. `tickets`, `files` — whatever the comment is attached to)
- entity_id BIGINT NOT NULL
- user_id BIGINT NOT NULL REFERENCES setup_users(id) ON DELETE CASCADE
- body TEXT NOT NULL
- is_edited TINYINT(1) DEFAULT 0
- is_deleted TINYINT(1) DEFAULT 0 (soft delete)
- created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
- updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP

### core_shares
- id BIGINT PK
- entity_type VARCHAR(32) NOT NULL
- entity_id BIGINT NOT NULL
- owner_id BIGINT NOT NULL REFERENCES setup_users(id) ON DELETE CASCADE
- target_user_id BIGINT NULL REFERENCES setup_users(id) ON DELETE CASCADE
- target_role VARCHAR(32) NULL
- link_token VARCHAR(64) NULL
- can_edit TINYINT(1) DEFAULT 0
- expires_at DATETIME NULL
- created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP

### core_refresh_tokens
- id BIGINT PK
- user_id BIGINT NOT NULL REFERENCES setup_users(id) ON DELETE CASCADE
- token VARCHAR(512) NOT NULL (JWT refresh token)
- expires_at DATETIME NOT NULL
- revoked TINYINT(1) DEFAULT 0
- created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP

Indexes: `idx_user_id (user_id)`, `idx_token (token)`.

### core_module_background_audit
One row per `coreApi.callModuleAsUser` call, allowed or refused (`docs/MFF-Core_module-core-api.md`, "Background identity"). Kept 180 days. Created at backend startup on existing deployments, and also in `schema.sql`.
- module_id: the calling module
- target_id: the module called
- user_id: the user acted as
- method
- path: without the query string
- outcome: `ok`, `failed` or `refused`
- reason: why a call was refused
- status: the HTTP status

## ERD (conceptual)

```
setup_users 1─* setup_user_roles *─1 setup_roles
setup_users *─1 setup_groups
setup_groups 1─* setup_group_pis *─1 setup_users
setup_users *─1 setup_job_positions
setup_users 1─1 setup_user_preferences
setup_users 1─* setup_password_resets
setup_users 1─1 setup_mfa_setup_pending
setup_users 1─* setup_mfa_recovery_codes
setup_users 1─* setup_mfa_email_otp
setup_users 1─* core_refresh_tokens
setup_users 1─* core_file_uploads
setup_users 1─* core_tickets (as reporter, and separately as assignee)
setup_users 1─* core_ticket_responses
setup_users 1─* core_notifications
setup_users 1─* core_comments
setup_users 1─* core_shares (as owner, and separately as target)
core_file_uploads 1─* core_file_roles
core_file_uploads 1─* core_announcements (image_file_id, optional)
core_tickets 1─* core_ticket_responses
```

This schema supports JWT auth with refresh-token rotation, normalized RBAC, TOTP/recovery-code/email-OTP MFA, encrypted-at-rest file uploads with per-file role-based access, announcements, tickets, notifications, comments, and sharing. A module installs its own tables (prefixed with its module ID) via its own one-time setup script — this document does not cover module schemas.
