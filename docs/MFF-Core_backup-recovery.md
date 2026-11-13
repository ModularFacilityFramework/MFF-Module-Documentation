# Backup and Recovery

This document explains how to back up and restore the database and uploaded files.

## Overview
The application stores state in two places:
- MariaDB/MySQL database (users, roles, tokens, preferences, metadata)
- Uploads directory (binary files uploaded via the API)

Backups must cover both to enable full disaster recovery.

## Scripts
The repository includes scripts that implement simple, production‑friendly backups with rotation and integrity checks:

- `scripts/db_backup.sh` — exports the database using `mysqldump` and compresses the result.
- `scripts/db_restore.sh` — restores a dump created by `db_backup.sh`.
- `scripts/files_backup.sh` — archives the uploads directory as a tar.gz.
- `scripts/backup.env.example` — template for configuration.

Backups are written under `backups/` in the repo by default. You can override locations in `scripts/backup.env`.

## Configuration
Copy the example and fill in values:

```bash
cp scripts/backup.env.example scripts/backup.env
# Edit scripts/backup.env
```

Required variables:
- `DB_HOST`, `DB_PORT`, `DB_NAME`, `DB_USER`, `DB_PASSWORD` — database connection.
- `RETENTION_DAYS` — how many days to keep old backups (default 7).
- `BACKUP_DIR` — directory for DB dumps (optional; default `./backups/db`).
- `FILES_BACKUP_DIR` — directory for file archives (optional; default `./backups/files`).
- `UPLOAD_DIR_NAME` — uploads subdir name inside `App_backend` (if not set, the script reads it from `App_backend/.env`).

## Usage

- Database backup:
  ```bash
  scripts/db_backup.sh
  ```
  Produces files like `backups/db/<DB_NAME>_YYYYMMDD_HHMMSS.sql.gz` and a `.sha256` checksum.

- Files backup:
  ```bash
  scripts/files_backup.sh
  ```
  Produces files like `backups/files/uploads_YYYYMMDD_HHMMSS.tar.gz` and a `.sha256` checksum.

- Database restore:
  ```bash
  scripts/db_restore.sh backups/db/yourdb_YYYYMMDD_HHMMSS.sql.gz
  ```
  The script verifies checksum when available and prompts for confirmation.

## Scheduling
Use cron or systemd timers on your server.

Example cron (daily at 02:30):
```
30 2 * * * cd /opt/your-app && ./scripts/db_backup.sh >> /var/log/db_backup.log 2>&1
45 2 * * * cd /opt/your-app && ./scripts/files_backup.sh >> /var/log/files_backup.log 2>&1
```

For multiple instances or off‑host storage, sync the `backups/` directory to remote storage (rsync/S3).

## Recovery Procedures
1. Stop the app services to ensure consistency.
2. Restore the database:
   ```bash
   scripts/db_restore.sh /path/to/backup.sql.gz
   ```
3. Restore files archive to backend uploads directory if needed:
   ```bash
   tar -C App_backend -xzf /path/to/uploads_YYYYMMDD_HHMMSS.tar.gz
   ```
4. Start services and verify:
   - Backend health: `GET /api/health`
   - Login flows and file access

## Verification & Drills
- Periodically test restore in a staging environment using the latest backups.
- Monitor backup logs and ensure new artifacts are produced daily.
- Consider off‑site storage and encryption for compliance requirements.
