# Troubleshooting Guide

This guide lists common issues and resolutions.

## Authentication and Authorization

Problem: 401 Unauthorized on API calls
- Cause: Access token expired or missing Authorization header
- Fix: Log in again. Ensure the frontend uses the centralized API client which attaches `Authorization: Bearer <accessToken>` automatically.

Problem: 403 Forbidden on /users
- Cause: Account lacks the `administrator` role (or `SUPERUSER_ROLE` isn't set to a role you hold)
- Fix: Ask an administrator to grant the proper role

## CORS

Problem: CORS error in browser console
- Cause: Frontend origin not allowed
- Fix: Set `CORS_ORIGINS` in backend `.env` to include the frontend URL (comma-separated). Restart the backend.

## File Uploads

Problem: Upload fails with unsupported file type
- Cause: MIME type not in whitelist
- Fix: Upload allowed types: PNG, JPEG, GIF, WEBP, SVG, PDF, TXT

Problem: Large files fail to upload
- Cause: File exceeds size limit (50MB)
- Fix: Reduce file size or increase limit in `fileService.js` (limits.fileSize)

## Database

Problem: Cannot connect to DB
- Cause: Incorrect credentials or DB not running
- Fix: Verify `DB_HOST`, `DB_USER`, `DB_PASSWORD`, `DB_NAME`. Ensure MariaDB is running and accessible.

Problem: Preferences endpoint returns defaults only
- Cause: No row exists in `setup_user_preferences` for the user
- Fix: Save preferences once via PUT to create the row

## Email and Password Reset

Problem: Not receiving reset email
- Cause: SMTP not configured or local mail not functioning
- Fix: Set SMTP env vars for SMTP flow or verify local mail service. Check backend logs for errors.

## Environment and Config

Problem: API paths 404
- Cause: Wrong `REACT_APP_API_BASE_PATH` on frontend or `API_BASE_PATH` on backend
- Fix: Ensure they match (default is `/api`) and rebuild/restart.

## Tests

Problem: Tests fail with connection or path errors
- Cause: Tests expect `API_BASE_PATH` and `BACKEND_PORT` to be set from the environment
- Fix: Ensure `.env` is configured before running tests; there is no CI pipeline defined in this repo yet.
