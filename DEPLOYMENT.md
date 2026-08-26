# SchoolMIS deployment on cPanel

This repository contains two applications:

- `frontend`: React/Vite web application. Build it into static files.
- `backend`: Node.js/Express API using PostgreSQL. Run it with cPanel's Node.js App feature.

## 1. Prepare and push to Git

Run these commands from `F:\mis` in PowerShell:

```powershell
git init
git add .
git status
git commit -m "Prepare SchoolMIS for deployment"
git branch -M main
git remote add origin https://github.com/YOUR-USER/YOUR-REPOSITORY.git
git push -u origin main
```

Before committing, confirm that `git status` does not list `.env`, `node_modules`, `frontend/dist`, or files inside `backend/uploads`.

## 2. Create PostgreSQL databases in cPanel

Create the PostgreSQL database and database user in cPanel. Grant the user full access to the database. Record the exact cPanel-prefixed names, for example:

```text
DB_NAME=account_schoolmis
DB_USER=account_schoolmis_user
DB_HOST=127.0.0.1
DB_PORT=5432
```

The application creates central tables during startup and applies migrations when `ENABLE_AUTO_MIGRATE=true`. For production, take a database backup first and normally keep automatic migrations disabled after the first successful migration.

## 3. Clone the repository on cPanel

Use cPanel Terminal or SSH:

```bash
cd ~
git clone https://github.com/YOUR-USER/YOUR-REPOSITORY.git schoolmis
cd schoolmis/backend
npm ci --omit=dev
```

If cPanel's Node version is old, select Node.js 18 or newer in **Setup Node.js App**. The backend uses CommonJS and starts from `src/server/server.js`.

## 4. Configure the backend Node.js App

In cPanel **Setup Node.js App**, create an application with:

- Node.js version: 18 or newer
- Application root: `schoolmis/backend`
- Application URL: your API subdomain, such as `api.example.com`
- Application startup file: `src/server/server.js`
- Application mode: `Production`

Add these environment variables in the Node.js App screen. Do not commit them to Git:

```text
PORT=<the port assigned by cPanel, if shown>
FRONTEND_URL=https://app.example.com
DB_HOST=127.0.0.1
DB_PORT=5432
DB_NAME=account_schoolmis
DB_USER=account_schoolmis_user
DB_PASSWORD=<database password>
DB_POOL_MIN=0
DB_POOL_MAX=10
JWT_SECRET=<long random secret>
JWT_EXPIRE_ADMIN=1d
JWT_EXPIRE_TENANT=1d
NODE_ENV=production
ENABLE_AUTO_MIGRATE=false
```

Run migrations once from the backend directory, after reviewing the migration files:

```bash
npm run migrate
```

Then restart the Node.js App. Test the API host with a route such as `https://api.example.com/v1/...`.

## 5. Preserve uploaded files

The repository intentionally excludes `backend/uploads`. Create the directory on cPanel and make it writable by the Node application:

```bash
mkdir -p ~/schoolmis/backend/uploads
chmod 755 ~/schoolmis/backend/uploads
```

Copy existing uploads separately with SFTP if they are needed. Do not put private uploads in a public Git repository.

## 6. Build and publish the frontend

Set the production API URL before building. The value is embedded into the static build:

```bash
cd ~/schoolmis/frontend
npm ci
```

Create `frontend/.env.production` on the server, or set the value in the shell:

```text
VITE_API_BASE_URL=https://api.example.com
VITE_APP_NAME=SchoolMIS
```

Build:

```bash
npm run build
```

Upload the contents of `frontend/dist` to the document root for `app.example.com` (usually `public_html` or a subdomain document root). Do not upload the `dist` directory as a nested folder unless the document root is configured for it.

## 7. Configure SPA routing

Because this is a React single-page application, direct links need an Apache fallback. Add this `.htaccess` file to the frontend document root:

```apache
RewriteEngine On
RewriteBase /
RewriteCond %{REQUEST_FILENAME} !-f
RewriteCond %{REQUEST_FILENAME} !-d
RewriteRule . /index.html [L]
```

## 8. Verify the deployment

1. Open `https://app.example.com` and confirm the login page loads.
2. In the browser developer tools, verify requests go to `https://api.example.com`, not localhost.
3. Log in and check that authenticated requests include `Authorization` and tenant requests include `X-Tenant-ID`.
4. Test one database read, one write, and one upload.
5. Check the cPanel Node.js App logs for startup or database errors.
6. Confirm HTTPS is enabled for both domains and that `FRONTEND_URL` exactly matches the frontend origin.

## Updates later

```bash
cd ~/schoolmis
git pull origin main
cd backend
npm ci --omit=dev
npm run migrate
```

Build the frontend again with the production API URL, replace the contents of its document root, and restart the backend Node.js App from cPanel.

## Files intentionally excluded from Git

`node_modules`, environment files, frontend build output, backend uploads, temporary scripts, logs, and editor files are local/server artifacts. The source code, migrations, API documentation, and both npm lockfiles remain versioned.
