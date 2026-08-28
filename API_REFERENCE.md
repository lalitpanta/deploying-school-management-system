# SchoolMIS API Reference

Production API: `https://api.benchmarkassociates.com.np`

All application routes are prefixed with `/v1`.

## Quick Test

```bash
curl https://api.benchmarkassociates.com.np/v1/health
```

The current live response observed on 2026-08-28 is:

```text
It works!
```

If the deployed application is running the repository version, the expected JSON response is:

```json
{"status":"ok"}
```

If you continue receiving `It works!`, confirm the cPanel Node.js App startup file is `src/server/server.js`, restart the application, and check its logs. This response is not produced by the current repository health handler.

## Authentication

Public endpoints:

- `GET /v1/health`
- `POST /v1/auth/login`
- `POST /v1/auth/admin/login`
- `POST /v1/auth/tenant/login`
- `POST /v1/auth/staff/login`
- `GET /v1/results/public`

For protected endpoints, send:

```http
Authorization: Bearer YOUR_JWT_TOKEN
X-Tenant-ID: TENANT_ID
Content-Type: application/json
```

The frontend stores the access token as `mis_auth_token`. A user token normally carries tenant information, but sending `X-Tenant-ID` explicitly is recommended for admin-selected tenants.

The API uses these user types: `admin`, `tenant`, `staff`, and `super_admin`.

## Login Examples

Unified login:

```http
POST /v1/auth/login
Content-Type: application/json
```

```json
{
  "email": "user@example.com",
  "password": "your-password",
  "tenantSlug": "school-slug"
}
```

The `tenantSlug` field is optional when the account is not tenant-ambiguous. Role-specific login endpoints use the same credential style unless the deployed controller specifies additional fields.

Example with curl:

```bash
curl -X POST https://api.benchmarkassociates.com.np/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"user@example.com","password":"your-password","tenantSlug":"school-slug"}'
```

Save the token from the login response and use it as `Authorization: Bearer <token>`.

## Response and Error Format

Successful responses commonly return JSON objects or arrays from the controller for that module. File export endpoints return a file/blob instead of JSON.

Errors use this shape:

```json
{
  "success": false,
  "message": "Description of the problem"
}
```

Common status codes:

- `200` request succeeded
- `201` resource created
- `400` invalid request or validation error
- `401` missing or invalid JWT
- `403` insufficient role, module, or tenant permission
- `404` resource or route not found
- `409` duplicate/conflicting data
- `500` server or database error

## Endpoint Catalog

`Auth` is required unless marked **Public**. Tenant/module requirements are shown after each group.

### Health and Authentication

| Method | Path | Access |
|---|---|---|
| GET | `/health` | Public |
| POST | `/auth/login` | Public |
| POST | `/auth/admin/login` | Public |
| POST | `/auth/tenant/login` | Public |
| POST | `/auth/staff/login` | Public |
| POST | `/auth/tenant/change-password` | Authenticated |
| POST | `/auth/tenant/change-email` | Authenticated |
| POST | `/auth/staff/change-password` | Authenticated |
| POST | `/auth/staff/change-email` | Authenticated |
| POST | `/auth/tenant/create` | Admin |
| GET | `/auth/tenant` | Admin |
| GET | `/auth/tenant/:id` | Authenticated |
| PATCH | `/auth/tenant/:id/status` | Admin |
| PATCH | `/auth/tenant/:id` | Admin |
| DELETE | `/auth/tenant/:id` | Admin |
| DELETE | `/auth/tenant/:id/permanent` | Admin |

### Dashboard

Requires authentication and tenant context.

| Method | Path |
|---|---|
| GET | `/dashboard/stats` |

### Calendar

Requires authentication, tenant context, and the `calendar` module.

| Method | Path |
|---|---|
| GET | `/year/getyear` |
| POST | `/year/uploadyear` |
| PATCH | `/year/:id` |
| DELETE | `/year/:id` |
| GET | `/month/getmonth` |
| POST | `/month/uploadmonth` |
| PATCH | `/month/:id` |
| DELETE | `/month/:id` |
| GET | `/day/getday` |
| POST | `/day/uploadday` |
| PATCH | `/day/:id` |
| DELETE | `/day/:id` |
| GET | `/day-category` |
| POST | `/day-category` |
| PATCH | `/day-category/:id` |
| DELETE | `/day-category/:id` |
| GET | `/calendar/month` |
| GET | `/calendar/month/weeks` |
| GET | `/calendar/bs/month` |
| GET | `/calendar/bs/month/weeks` |
| GET | `/calendar/bs/weeks` |
| GET | `/calendar-days/month` |
| POST | `/calendar-days/generate` |
| PATCH | `/calendar-days/:id/assign-type` |

Calendar query examples:

```text
/calendar/month?year=2025&month=5&date_format=AD
/calendar/month/weeks?year=2025&month=5&date_format=AD
/calendar/bs/month?year=2081&month=2
/calendar-days/month?month_id=MONTH_UUID&date_format=BS
/calendar-days/generate?month_id=MONTH_UUID
```

Use `date_format=AD` or `date_format=BS` when supported. UUIDs are returned by the API and must be reused as IDs.

### Students

Requires authentication and tenant context. Student create/update routes accept `multipart/form-data` when files are included.

| Method | Path |
|---|---|
| GET | `/students` |
| GET | `/students/:id` |
| POST | `/students` |
| PATCH | `/students/:id` |
| DELETE | `/students/:id` |
| GET | `/students/export` |
| POST | `/students/bulk/import` |
| POST | `/students/import` |
| DELETE | `/students/:id/documents/:docId` |

Upload field names used by the backend include `photograph` and `documents`. Export responses are downloadable files; use a blob/file response type in Axios or fetch.

### Teachers

Requires authentication and tenant context.

| Method | Path |
|---|---|
| GET | `/teachers` |
| GET | `/teachers/options` |
| GET | `/teachers/:id` |
| POST | `/teachers` |
| PATCH | `/teachers/:id` |
| DELETE | `/teachers/:id` |
| POST | `/teachers/import` |
| GET | `/teachers/export` |
| GET | `/teachers/:id/download/:filename` |

Teacher create/update/import requests may use `multipart/form-data`. The import upload field is `file`.

### Employees

Requires authentication and tenant context.

| Method | Path |
|---|---|
| GET | `/employees` |
| GET | `/employees/options` |
| GET | `/employees/:id` |
| POST | `/employees` |
| PATCH | `/employees/:id` |
| DELETE | `/employees/:id` |
| GET | `/employees/:id/download/:filename` |

Employee create/update requests may use `multipart/form-data` with employee photograph/document fields.

### Classes, Sections, Rooms, Courses, and Departments

These routes require authentication and tenant context.

| Method | Path |
|---|---|
| GET/POST | `/classes` |
| GET/PUT/DELETE | `/classes/:id` |
| GET/POST | `/sections` |
| GET/PUT/DELETE | `/sections/:id` |
| GET | `/sections/by-room/:roomId` |
| GET | `/sections/by-class/:classId` |
| GET/POST | `/rooms` |
| GET/PUT/DELETE | `/rooms/:id` |
| GET/POST | `/departments` |
| GET/PUT/DELETE | `/departments/:id` |
| GET/POST | `/settings/classrooms` |
| PUT/DELETE | `/settings/classrooms/:id` |
| POST | `/settings/classrooms/:id/sections` |
| GET | `/settings/classrooms/:id/sections` |
| PUT | `/settings/classrooms/sections/:sectionId` |
| DELETE | `/settings/classrooms/sections/:sectionId` |
| GET | `/settings/courses` |
| GET | `/settings/courses/:id` |
| POST | `/settings/courses` |
| PUT | `/settings/courses/:id` |
| DELETE | `/settings/courses/:id` |

### Attendance

Requires authentication and tenant context.

| Method | Path |
|---|---|
| GET | `/attendance` |
| GET | `/attendance/summary` |
| GET | `/attendance/history` |
| POST | `/attendance` |
| GET | `/attendance/report` |

Typical filters include `classId` and `date`. Bulk attendance requests use a `records` array. Reports are downloadable files.

### Results and Exams

Requires authentication, tenant context, and the `results` module, except the public lookup.

| Method | Path |
|---|---|
| GET | `/results/public` | Public |
| GET | `/results/classroom/:classroomId` |
| GET | `/results/teacher/classrooms` |
| POST | `/results` |
| GET | `/results/:resultId` |
| PATCH | `/results/:resultId` |
| DELETE | `/results/:resultId` |
| POST | `/results/bulk` |
| POST | `/results/exam-formats` |
| GET | `/results/exam-formats` |
| GET | `/results/exam-formats/:id` |
| PATCH | `/results/exam-formats/:id` |
| PATCH | `/results/exam-formats/:id/publish` |
| DELETE | `/results/exam-formats/:id` |
| POST | `/results/exam-subjects` |
| GET | `/results/exam-subjects/:exam_format_id` |
| PATCH | `/results/exam-subjects/:id` |
| DELETE | `/results/exam-subjects/:id` |
| POST | `/results/student-marks` |
| GET | `/results/student-marks` |
| GET | `/results/student-marks/:id` |
| DELETE | `/results/student-marks/:id` |
| GET | `/results/class-students` |
| GET | `/results/class-courses/:class_id` |

Public result lookup requires the query/body values implemented by the result controller, commonly tenant slug, roll number, and date of birth. Do not expose administrative result endpoints publicly.

### Fees

Requires authentication and tenant context.

| Method | Path |
|---|---|
| GET | `/fees/categories` |
| POST | `/fees/categories` |
| GET | `/fees/structures` |
| POST | `/fees/structures` |
| GET | `/fees/student-fees` |
| POST | `/fees/student-fees` |
| POST | `/fees/student-fees/bulk-generate` |
| GET | `/fees/receipts` |
| POST | `/fees/pay` |
| GET | `/fees/dashboard-stats` |

Payment requests should be treated as financial operations. Validate the student, amount, and payment reference on the server and retain the returned receipt ID.

### Leave Management

Requires authentication, tenant context, and the `leave_management` module.

| Method | Path |
|---|---|
| GET | `/leave` |
| GET | `/leave/my` |
| POST | `/leave` |
| PUT | `/leave/:id/status` |

### Devices

Requires authentication and tenant context.

| Method | Path |
|---|---|
| POST | `/devices` |
| GET | `/devices` |
| GET | `/devices/:deviceId` |
| PATCH | `/devices/:deviceId` |
| DELETE | `/devices/:deviceId` |
| POST | `/devices/:deviceId/test-connection` |
| POST | `/devices/:deviceId/sync-now` |
| GET | `/devices/:deviceId/sync-logs` |
| POST | `/devices/:deviceId/enroll-teachers` |
| GET | `/devices/:deviceId/enrollments` |
| GET | `/devices/:deviceId/unmatched-ids` |
| GET | `/devices/:deviceId/attendance-records` |
| GET | `/devices/:deviceId/attendance-summary` |
| PATCH | `/devices/attendance/:recordId/override` |

### Daily Reports

Requires authentication and tenant context.

| Method | Path |
|---|---|
| GET | `/daily-reports/templates` |
| POST | `/daily-reports/templates` |
| PATCH | `/daily-reports/templates/:id` |
| DELETE | `/daily-reports/templates/:id` |
| GET | `/daily-reports` |
| POST | `/daily-reports` |
| POST | `/daily-reports/bulk-send` |
| DELETE | `/daily-reports/:id` |

### Settings and Notices

Requires authentication, tenant context, and settings access.

| Method | Path |
|---|---|
| GET/PATCH | `/settings` |
| GET/PUT | `/settings/school` |
| POST | `/settings/test-email` |
| GET/PUT | `/settings/notifications` |
| GET/POST | `/settings/rooms` |
| GET/PUT/DELETE | `/settings/rooms/:id` |
| GET/POST | `/settings/notices` |
| PUT/DELETE | `/settings/notices/:id` |
| POST | `/settings/notices/:id/read` |
| POST | `/settings/notices/:id/pin` |
| POST | `/settings/notices/:id/archive` |
| POST | `/settings/notices/:id/send-email` |
| GET | `/settings/sms` |
| GET/PUT | `/settings/sms/config` |
| GET/POST | `/settings/sms/templates` |
| PUT/DELETE | `/settings/sms/templates/:id` |
| POST | `/settings/sms/send` |
| GET | `/settings/sms/logs` |
| GET/PUT | `/settings/theme` |
| GET | `/settings/audit-logs` |
| GET | `/settings/audit-stats` |

`/settings/sms`, `/settings/sms-config`, and `/settings/sms/config` are configuration read aliases currently registered by the backend.

### Users, Roles, and Permissions

Requires authentication, tenant context, and settings access unless otherwise stated.

| Method | Path |
|---|---|
| GET/POST | `/users` |
| GET | `/users/me` |
| POST | `/users/me/change-password` |
| POST | `/users/me/change-email` |
| GET/PATCH/DELETE | `/users/:id` |
| PUT | `/users/:id/toggle-active` |
| POST | `/users/:id/change-password` |
| POST | `/users/:id/reset-password` |
| POST | `/users/:id/roles` |
| GET | `/roles` |
| POST | `/roles` |
| GET/PUT/DELETE | `/roles/:id` |
| POST | `/roles/:id/permissions` |
| GET/POST | `/permissions` |
| GET/PUT/DELETE | `/permissions/:id` |
| GET | `/permissions/by-resource` |
| GET | `/users-with-roles` |
| GET | `/users-with-roles/:userId` |
| POST | `/users-with-roles/:userId/roles` |
| DELETE | `/users-with-roles/:userId/roles/:roleId` |
| GET | `/users-with-roles/:userId/permissions` |
| POST | `/users-with-roles/:userId/check-permission` |

### Super Admin

Requires authentication and admin privileges.

| Method | Path |
|---|---|
| GET | `/super-admin/overview` |
| GET | `/super-admin/packages` |
| POST | `/super-admin/packages` |
| PUT | `/super-admin/packages/:id` |
| DELETE | `/super-admin/packages/:id` |

## Static Uploads

Uploaded files are served under:

```text
https://api.benchmarkassociates.com.np/uploads/<folder>/<filename>
```

Folders commonly include `students`, `teachers`, and `employees`. Treat these URLs as protected application data even though they are served as static files. Do not expose filenames or upload access to unauthenticated users without confirming the intended privacy policy.

## Frontend Configuration

Set the frontend build variable to:

```text
VITE_API_BASE_URL=https://api.benchmarkassociates.com.np
```

Do not include a trailing `/v1`; the Axios client appends route paths according to the existing frontend API modules.

## Postman Environment

Create an environment with:

```text
api_url=https://api.benchmarkassociates.com.np/v1
token=PASTE_LOGIN_TOKEN
tenant_id=PASTE_TENANT_ID
```

Use these collection headers:

```text
Authorization: Bearer {{token}}
X-Tenant-ID: {{tenant_id}}
Content-Type: application/json
```

Example request URL:

```text
{{api_url}}/health
```

## Security and Operations

- Never commit `.env` files, database passwords, or JWT secrets.
- Use HTTPS for both the frontend and API domains.
- Rotate `JWT_SECRET` if it is exposed; all existing tokens will become invalid.
- Keep `ENABLE_AUTO_MIGRATE=false` in production and run migrations manually after a database backup.
- Use a restricted PostgreSQL user with only the permissions required by the application.
- Back up PostgreSQL and the `backend/uploads` directory before upgrades.
- Check cPanel Node.js logs after deployment and after every migration.

## Source of Truth

This reference is generated from the route registrations under `backend/src/routing` and the mount table in `backend/src/routing/index.js`. Request fields and response fields are validated in the controller and validation files; when exact payload details matter, inspect the corresponding controller before integrating a new client.
