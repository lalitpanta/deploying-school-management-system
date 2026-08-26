# SchoolMIS — Mobile-focused System Documentation

## Overview

Purpose: provide a detailed, module-by-module description of the system's core workflows and APIs so you can adapt the web frontend into a mobile application. This document focuses on the behavior, required API calls, headers/payloads, important local storage keys, and mobile migration considerations.

Audience: mobile app developers and architects integrating with the existing backend.

## Quick facts

- Frontend: React + Vite (web). API clients in [frontend/src/api](frontend/src/api).
- Backend: Node + Express, PostgreSQL. API versioning under `/v1/*` (see `backend/src/routing/v1`).
- Auth: JWT tokens; use `Authorization: Bearer <token>`.
- Multi-tenant: tenant context sent via header `X-Tenant-ID` (populated from token or admin selection). Some requests can set `skipTenantHeader` to bypass tenant scoping.

## Local storage / client keys used by web UI (mobile should replicate or map them)

- `mis_auth_token` — JWT access token (stored by `AuthContext`).
- `mis_refresh_token` — refresh token (frontend does not force auto-refresh; consider implementing for mobile).
- `mis_user` — JSON user object (id, name, type, modules, permissions).
- `mis_user_type` — string: `admin`, `tenant`, `staff`, `super_admin`.
- `mis_tenant_id` / `mis_database_name` — selected tenant context for admin consoles.

## Core workflows (module-by-module)

1. Authentication & tenant selection

- Endpoints:
  - POST `/v1/auth/login` — unified login (email, password, tenantSlug optional)
  - POST `/v1/auth/admin/login`, `/v1/auth/tenant/login`, `/v1/auth/staff/login`
- Mobile flow:
  - Collect credentials; call `/v1/auth/login` when tenant slug uncertain.
  - On success store `mis_auth_token`, `mis_user`, `mis_user_type`.
  - Determine tenant ID to use: priority — selectedTenant (admin-selected), token. Send this as `X-Tenant-ID` header in subsequent calls.
  - Implement refresh-token flow if desired (backend supports refresh key but frontend currently stores only token keys). Consider adding `/v1/auth/refresh` if not present.

2. Roles & module permissions

- Clients check `user.modules` and `user.permissions` from `mis_user` to show/hide features.
- Backend helper: `hasModule(moduleKey)` logic in AuthContext. Mobile must implement equivalent checks.

3. Calendar (year/month/day types/calendar days)

- Primary endpoints (examples):
  - GET `/v1/year/getyear` — years list
  - POST `/v1/month/uploadmonth` — create month
  - POST `/v1/calendar-days/generate?month_id=...` — auto-generate calendar days
  - GET `/v1/calendar-days/month?month_id=...&date_format=BS|AD` — flat days for a month
  - GET `/v1/calendar/bs/weeks?month_id=...` — grouped-by-weeks for UI
  - PATCH `/v1/calendar-days/:id/assign-type` — assign a day type to a day
- Mobile considerations:
  - Always pass `date_format` when rendering (BS vs AD). Mobile should support both views and allow server formatting.
  - Calendar generation can be long-running; the create + generate sequence returns created month and generated days count — show progress UI.

4. Attendance

- Primary endpoints:
  - GET `/v1/attendance` — fetch with filters (classId, date)
  - POST `/v1/attendance` — mark attendance (payload: classId, date, records[])
  - GET `/v1/attendance/report` — returns blob for export
  - GET `/v1/attendance/history` — history
- Device-related endpoints in `/v1/devices` include sync and attendance records per device.
- Mobile considerations:
  - Use `responseType: 'blob'` for downloads.
  - For marking bulk attendance, batch records to reduce network calls.
  - Support offline queue and retry when connectivity restored.

5. Results / Exam management / Result portal

- Primary endpoints:
  - POST `/v1/results/exam-formats` — create exam format
  - POST `/v1/results/student-marks` — create/update marks
  - POST `/v1/results/classroom/:classroomId/publish` — publish results
  - GET `/v1/results/public` — public lookup by tenant slug, roll no, DOB
- Mobile considerations:
  - Publishing is an elevated action — ensure proper permission checks and UI confirmation.
  - For result lookup, mobile can use a lightweight public flow without auth.

6. Teachers / Students / Employees

- CRUD endpoints under `/v1/teachers`, `/v1/settings/students`, `/v1/employees`.
- File uploads use multipart/form-data. Backend expects `photograph`, `documents` fields.
- Mobile considerations:
  - Use multipart uploads for images/docs.
  - Use progressive upload or background upload for large files.

7. Devices (biometric readers) — integration points

- Key endpoints:
  - POST `/v1/devices` — create device
  - POST `/v1/devices/:deviceId/test-connection` — test
  - POST `/v1/devices/:deviceId/sync-now` — request immediate sync
  - GET `/v1/devices/:deviceId/attendance-records` — get records
- Mobile considerations:
  - Mobile apps should treat device endpoints as admin-only features.
  - For mobile attendance capture, prefer server-side endpoints rather than connecting to devices directly.

8. Fees

- Endpoints: `/v1/settings/fees`, student fees, receipts, bulk-generate student fees, payment collection.
- Mobile considerations:
  - Payment operations should be idempotent and return receipt IDs.
  - Use small payloads for lists; implement pagination.

9. Notices, daily reports, SMS & email

- Endpoints support creating notices, sending emails, SMS templates and logs.
- Mobile considerations:
  - Provide push-notification integration if desired; backend currently sends SMS/email only.

## Settings module — complete list

This section enumerates every submodule found under the `settings` area of the system, the primary API endpoints used by the frontend, and notes on what a mobile app needs to implement for each submodule. Frontend API wrappers are in [frontend/src/api/settingsApi.js](frontend/src/api/settingsApi.js) and helper hooks in [frontend/src/hooks/useSettings.js](frontend/src/hooks/useSettings.js).

- **General Settings**
  - Purpose: global platform settings (calendar type, platform-wide toggles, system name).
  - Endpoints: GET `/v1/settings`, PATCH `/v1/settings`.
  - Mobile notes: fetch once on login and cache locally; provide synchronized updates.

- **School Profile**
  - Purpose: tenant's school information (name, logo, address, contact details).
  - Endpoints: GET `/v1/settings/school`, PUT `/v1/settings/school`.
  - Mobile notes: used by Navbar/branding and profile screens; upload logo via multipart if supported.

- **Academic Calendar**
  - Purpose: calendar-level settings and academic year preferences.
  - Endpoints: GET `/v1/settings/academic-calendar`, PUT `/v1/settings/academic-calendar`.
  - Mobile notes: read-only for most users; admin can update year settings.

- **Users & Staff**
  - Purpose: tenant users management (create, list, update, delete, reset password).
  - Endpoints: GET `/v1/settings/users`, POST `/v1/settings/users`, PUT `/v1/settings/users/:id`, DELETE `/v1/settings/users/:id`, POST `/v1/settings/users/:id/reset-password`.
  - Mobile notes: admin screens must support role assignment and password reset; implement secure forms.

- **Roles & Permissions**
  - Purpose: define roles and permission resources for tenant scoping.
  - Endpoints (settingsApi): GET `/v1/settings/roles`, POST `/v1/settings/roles`, PUT `/v1/settings/roles/:id`, DELETE `/v1/settings/roles/:id`, GET `/v1/settings/permissions`.
  - Note: frontend also uses dedicated endpoints `/v1/roles` and `/v1/permissions` in other modules — mobile should support both if required by UI flows.

- **Fees**
  - Purpose: fee configuration for the tenant (categories, structures), student fee generation.
  - Endpoints: GET `/v1/settings/fees`, PUT `/v1/settings/fees` plus student-fees endpoints under `/v1/settings` (`/student-fees`, `/student-fees/bulk-generate`) and receipts `/v1/settings/receipts`.
  - Mobile notes: billing screens should call `getFeeSettings`, and payments must be idempotent.

- **Notifications & Notification Settings**
  - Purpose: email/SMS and notification preferences for platform behavior.
  - Endpoints: GET `/v1/settings/notifications`, PUT `/v1/settings/notifications`.
  - Mobile notes: expose toggles for push vs. email vs. SMS; ensure mobile push integration maps to server-side notification channels.

- **Notices (Announcements)**
  - Purpose: create, edit, archive and send notices to recipients; track read status.
  - Endpoints: GET `/v1/settings/notices`, POST `/v1/settings/notices`, PUT `/v1/settings/notices/:id`, DELETE `/v1/settings/notices/:id`, POST `/v1/settings/notices/:id/read`, POST `/v1/settings/notices/:id/pin`, POST `/v1/settings/notices/:id/archive`, POST `/v1/settings/notices/:id/send-email`.
  - Mobile notes: mobile can consume notices feed and implement 'mark as read' and 'archive'; sending email/SMS from mobile requires admin permissions.

- **SMS Config & Templates**
  - Purpose: manage SMS gateway settings and templates used for sending messages.
  - Endpoints: GET `/v1/settings/sms/config`, PUT `/v1/settings/sms/config`, GET `/v1/settings/sms/templates`, POST `/v1/settings/sms/templates`, PUT `/v1/settings/sms/templates/:id`, DELETE `/v1/settings/sms/templates/:id`, POST `/v1/settings/sms/send`, GET `/v1/settings/sms/logs`.
  - Mobile notes: only admins manage gateway; mobile should rely on server-side send endpoints and show logs.

- **Security Settings & Audit Logs**
  - Purpose: security-related configuration (password policies, session timeouts) and audit log retrieval.
  - Endpoints: GET `/v1/settings/security`, PUT `/v1/settings/security`, GET `/v1/settings/audit-logs`, GET `/v1/settings/audit-stats`.
  - Mobile notes: the mobile app should surface audit logs only to super-admins or admin roles and respect retention and pagination.

- **Theme & UI Settings**
  - Purpose: visual theme settings including primary color and dark/light preference.
  - Endpoints: GET `/v1/settings/theme`, PUT `/v1/settings/theme`.
  - Mobile notes: mobile should apply theme locally (dark mode + accent color) and sync changes to server when appropriate.

- **Misc / Platform Settings**
  - Purpose: any additional tenant-level flags stored in `/v1/settings` (e.g., feature toggles, platform name, storage config).
  - Endpoints: included in GET/PATCH `/v1/settings`.

For each settings submodule, mobile should reuse the same authentication (`Authorization` header) and tenant header (`X-Tenant-ID`). When implementing admin-only write operations, ensure the mobile UI checks `mis_user` permissions and `user.type` before showing actions.

## API and headers — practical guide for mobile

- Base URL: `VITE_API_BASE_URL` default `http://localhost:5000` (use production value).
- Required headers on most requests:
  - `Authorization: Bearer <token>`
  - `X-Tenant-ID: <tenantId>` when acting on tenant resources (if not sent, backend may extract from token)
- Axios behavior in frontend:
  - `axiosInstance` attaches token and `X-Tenant-ID` automatically.
  - `skipTenantHeader` request flag bypasses tenant header behavior for endpoints that operate on master DB.

## Recommended mobile API usage patterns

- Authentication: accept both `unifiedLogin` and role-specific login; store tokens in secure storage (Keychain/Keystore). Use refresh tokens.
- Pagination: always request paginated lists where available — backend supports `page`/`limit` on some endpoints (e.g., device logs).
- File downloads: handle `responseType: 'blob'` and use streaming where possible.
- Time/date: server uses both BS and AD; send `date_format` where relevant and normalize timezones on server.

## Mobile-specific considerations & recommended backend additions

- Implement a token refresh endpoint (if missing): `POST /v1/auth/refresh` accepting a refresh token, returning new access token.
- Add compact endpoints for mobile:
  - `/v1/mobile/dashboard` — trimmed dashboard payload for mobile home screen.
  - `/v1/mobile/calendar/summary?month_id=` — smaller calendar summary endpoint.
  - `/v1/mobile/attendance/batch` — accept compressed batches and return per-record status.
- Add push-notification hooks / webhooks so backend can push notices to mobile (APNs / FCM).
- Add rate-limiting and request size limits for mobile file uploads.

## Error handling & UX guidance

- Inspect `error.response?.status` for 401/403/500 and show appropriate screens: login, permission denied, retry later.
- Treat 429 as retry-after with exponential backoff.

## Implementation checklist for mobile porting

- [ ] Implement secure token storage and refresh flow.
- [ ] Implement tenant selection UI (admin mode) and persist `X-Tenant-ID` mapping.
- [ ] Implement a local queue for offline attendance/marks and a background sync worker.
- [ ] Use multipart uploads for photos/documents; provide progress to users.
- [ ] Support BS/AD rendering and allow user to select preferred date format.
- [ ] Add compact/mobile endpoints if performance issues occur.

## Run & testing commands (dev)

Backend:

```bash
cd backend
npm install
npm run dev
```

Frontend (web dev server):

```bash
cd frontend
npm install
npm run dev
```

## Where to look in the repo

- Routes & controllers: [backend/src/routing](backend/src/routing) and [backend/src/controller](backend/src/controller)
- Frontend API clients: [frontend/src/api](frontend/src/api)
- Auth & context logic: [frontend/src/context/AuthContext.jsx](frontend/src/context/AuthContext.jsx)

---

If you'd like, I can:

- generate a mobile-focused API reference listing every v1 endpoint with request/response examples,
- scaffold a lightweight `mobile/` client folder with typed API wrappers and example requests in React Native,
- or create an OpenAPI (Swagger) spec from the routing files to feed to mobile SDK generators.

## All Modules (numbered)

1. Authentication
2. Dashboard
3. Calendar
4. Attendance
5. Results / Exam Management
6. Result Portal (public)
7. Daily Reports
8. Notices / Notifications / SMS & Email
9. Teachers
10. Students
11. Employees
12. Devices (biometric readers)
13. Fees
14. Leave Management
15. Settings
16. Users & Staff Management
17. Roles & Permissions
18. Courses / Classrooms / Sections
19. Rooms
20. Departments
21. Audit & Logs
22. Super-admin / Tenant Management
23. Reports & Exports
