# Render and Neon deployment

The application uses one Neon project for the super-admin database and creates one additional Neon project per new tenant. The super-admin connection string must be stored only in Render environment variables. Never commit a connection string or Neon API key.

## Rotate the exposed credentials

The credentials previously pasted into chat should be considered compromised. Rotate the Neon database password and revoke/regenerate the Neon API key in the Neon console before deploying.

## Backend web service

Create a Render **Web Service** from the repository with:

- Root directory: `backend`
- Build command: `npm ci --omit=dev`
- Start command: `npm start`
- Health check path: `/v1/health`

Set these environment variables in Render:

```text
NODE_ENV=production
PORT=10000
FRONTEND_URL=https://<frontend-service>.onrender.com
DATABASE_URL=<super-admin-neon-connection-string>
DB_POOL_MIN=0
DB_POOL_MAX=10
NEON_API_KEY=<rotated-neon-api-key>
NEON_API_URL=https://console.neon.tech/api/v2
NEON_REGION=aws-us-east-2
NEON_PG_VERSION=16
NEON_ROLE_NAME=neondb_owner
JWT_SECRET=<long-random-secret>
JWT_EXPIRE_ADMIN=1d
JWT_EXPIRE_TENANT=1d
ENABLE_AUTO_MIGRATE=false
```

`DATABASE_URL` is used for the central super-admin database. The legacy `DB_HOST`, `DB_PORT`, `DB_NAME`, `DB_USER`, and `DB_PASSWORD` variables remain available for local PostgreSQL mode, but are not needed on Render when `DATABASE_URL` is set.

Deploy once and verify `GET https://<backend-service>.onrender.com/v1/health`. The backend creates its central tables during startup. Review and run `npm run migrate` manually only when a migration is required.

## Frontend static site

Create a Render **Static Site** from the same repository with:

- Root directory: `frontend`
- Build command: `npm ci && npm run build`
- Publish directory: `dist`

Set the build environment variable:

```text
VITE_API_BASE_URL=https://<backend-service>.onrender.com
VITE_APP_NAME=SchoolMIS
```

Add a rewrite for React Router:

```text
Source: /*
Destination: /index.html
Action: Rewrite
```

After both services deploy, open the frontend, log in as the super admin, and create a test tenant. A successful tenant creation should show a new project in Neon and allow tenant login immediately. Confirm the browser requests use the backend Render URL and that the backend logs contain no database connection errors.

## Important operational notes

- Render services have ephemeral filesystems. Existing uploads under `backend/uploads` will not persist across deploys or restarts; use object storage before relying on production uploads.
- Every tenant creation creates a separate Neon project. Monitor Neon project and compute limits before enabling unrestricted tenant creation.
- The API stores each tenant connection string in the central database so existing tenants can reconnect after a Render restart. Restrict database access and rotate credentials if the central database is exposed.
