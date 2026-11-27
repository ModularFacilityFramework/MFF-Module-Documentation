# User Guide

This guide walks end users through common tasks in the application.

## Getting Started

1. Navigate to the login page (/login)
2. Enter your username and password
3. If you have just registered, your account stays inactive and pending until an administrator approves it. If your account is inactive for any other reason, contact an administrator to activate it
   - If you already have a "no login" record (see below), register with your real username and email. An administrator then joins your registration to that record when approving it. If you are told that a registration with your username or email is already waiting for approval, contact an administrator.
   - Some accounts are "no login" accounts: a contact record (for example a group PI) that never logs in. An administrator creates these with Setup → Users → "+ Create No-Login User". Only standard, active accounts can log in.
4. If you forgot your password, use the "Forgot password?" link to request a reset

## Dashboard

After login, you are redirected to the dashboard.
- Use the left panel to access your profile image and logos if available
- Tabs available depend on your role and what's installed: Announcements, Tickets, Files, Collaboration, Search, Reports (staff/admin only), Feedback, and Setup (Preferences, Users/Groups management — admin only)

- Users & Groups has a "Statistics" tab with two pie charts: Groups Distribution and Job Roles Distribution. Click a group's slice to filter the Job Roles pie to that group; click it again, or press Reset, to show everyone.
- The Users edit form has a "Job position" select.

## Lists of people and user drop-downs

Every list of people (Users tables, recipient lists, group PIs, search, exports) is sorted by first name, then last name. Drop-downs and recipient lists show "First Last (username)".

## Projects

Each project module has one tab per project at the top. You need a project before starting project work: one you can write in ("Project template" does not count). If you do not have one, you can request one from the banner. Approvers can create a project for a new person, or give them access to one, while approving their registration.

## Sorting tables

In every table, each column title has ▲ (ascending) and ▼ (descending) buttons. Numbers, dates and names sort naturally, and empty cells are always at the bottom. The table remembers your choice. Click the highlighted arrow again to go back to the usual order.

## Remembered page state

Every page remembers where you were (tabs, open forms, drafts, filters) when you leave it, reload, or log out. A developer's "view as" mode does not touch the viewed user's remembered state. Passwords are never remembered.

## Preferences

- Open Setup > Preferences
- Choose logos and images from your uploaded files (Appearance). The app logo, the favicon, the login logo and your profile image are independent: choosing or removing one never changes another
- Update theme, language, notifications, auto save, and default view
- Changes are saved automatically for image selections; use Save button for other settings

## Files

- Upload files via the Files tab
- Only certain file types are allowed (images, PDF, text)
- After upload, files become available for selection in Preferences

## Password Reset

1. On the login page, click "Forgot password?"
2. Enter your username; you will receive an email if your account exists
3. Open the link and set a new password (valid for 30 minutes)

## MFA (Multi-Factor Authentication)

- Open Setup > Preferences > Security to enable Time-based One-Time Password (TOTP) yourself — no admin action is required to turn it on.
- Scan the QR code in an authenticator app and enter a code to confirm setup. You'll be given 10 one-time recovery codes — save them somewhere safe; they let you log in if you lose access to your authenticator app.
- Once enabled, logging in asks for a code from your authenticator app (or a recovery code, or an emailed one-time code) after your password.
- You can regenerate recovery codes or disable MFA from the same Security section at any time.

## Troubleshooting

- 401 Unauthorized: Your session may have expired; log in again
- File upload fails: Ensure file type and size are allowed
- Preferences not updating: Try saving again or reloading the page
- For more, see Troubleshooting guide (docs/MFF-Core_troubleshooting.md)
