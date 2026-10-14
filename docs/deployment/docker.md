# Docker deployment

This production-oriented Compose file runs the frontend and API as built images with MySQL, Redis and MinIO on a private Docker network. Put the stack behind a TLS reverse proxy on the host.

## Configure

1. Copy `apps/backend/.env.example` to `apps/backend/.env` and `apps/frontend/.env.example` to `apps/frontend/.env`.
2. Copy `.env.docker.example` to `.env.docker`. Replace every placeholder with actual values. Use different randomly generated values for database/root/MinIO passwords and the internal API secret.
3. Complete `apps/backend/.env` according to `apps/backend/src/core/config/env.ts`, including required backend/public URLs, DB credentials, Redis, S3, Clerk, Stripe, Sentry, Umami, Cloudflare, analytics encryption key, SMTP and any other enabled integrations.
4. Set `DB_HOST=db`, `DB_USER=qrcodly`, `DB_PASSWORD` to `MYSQL_PASSWORD`, `DB_NAME=qrcodly`, `REDIS_URL=redis://redis:6379`, and `S3_ENDPOINT=http://minio:9000` in the backend environment. Set `BACKEND_URL` to the public API origin without `/api/v1` (for server-side API calls) and `NEXT_PUBLIC_API_URL` to the public API URL including `/api/v1`.
5. Complete the frontend variables using `apps/frontend/src/env.js`, including a real `NEXT_PUBLIC_SENTRY_DSN` (required by the current schema). The frontend container loads `apps/frontend/.env` for server-only values such as Axiom and Clerk secrets. Public `NEXT_PUBLIC_*` values are embedded at image build time, so changes to them require rebuilding the frontend image.
6. The MinIO setup job creates the `qrcodly` bucket, creates a dedicated least-privilege application user, and enables anonymous download for that bucket to match the application’s `S3_PUBLIC_URL` behavior. **Objects placed in this bucket are publicly readable**; do not store private uploads or secrets there. Keep the MinIO admin console/API unexposed, and use a separate bucket or private policy if confidential objects are added.

## Build and start

```sh
docker compose --env-file .env.docker -f compose.production.yaml config
docker compose --env-file .env.docker -f compose.production.yaml build
docker compose --env-file .env.docker -f compose.production.yaml up -d db redis minio
docker compose --env-file .env.docker -f compose.production.yaml run --rm -e DB_MIGRATING=true backend node build/src/core/db/migrate.js
docker compose --env-file .env.docker -f compose.production.yaml up -d
docker compose --env-file .env.docker -f compose.production.yaml ps
docker compose --env-file .env.docker -f compose.production.yaml logs -f backend frontend
```

Run migrations only after backing up the database and waiting for MySQL health. Do not send production traffic if a migration fails. The migration command uses the compiled migrations directory copied into the backend image.

## Reverse proxy

Route the app hostname to `127.0.0.1:3000` and API hostname to `127.0.0.1:5001` from the host reverse proxy. Restrict both ports at the host firewall. Set `FRONTEND_URL` and Clerk allowed origins to match the real public app URL.

## Operations and limitations

- Back up MySQL and MinIO regularly and test restoring the backups.
- Never commit `.env`, `.env.docker`, passwords, private keys or API secrets.
- Update image tags via reviewed changes; do not use mutable `latest` tags for production.
- Do not run more than one migration process at once.
- Confirm every required variable against the Zod schemas before launch.
- This is a deployment baseline; it has not been built or smoke-tested on a Linux Docker host in this environment.
- The backend health check assumes `GET /api/v1` returns a non-5xx response. Adjust it if the route contract changes.
