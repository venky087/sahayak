# Sahayak – Inclusive Community AI Assistant (base scaffold)
React+TS+Vite+Tailwind · Express+TS · PostgreSQL+Prisma · Gemini via backend proxy.

## Run locally
1. `cp .env.example .env` and set `GEMINI_API_KEY`, `JWT_SECRET`.
2. `docker compose up --build`
3. `docker compose exec api npx prisma migrate dev --name init && docker compose exec api npx tsx prisma/seed.ts`
4. Frontend http://localhost:5173 · API http://localhost:4000/api/health

Demo logins (after seed): `officer@sahayak.test` / `citizen@sahayak.test`, password `Passw0rd!`

## Architecture
Browser → React SPA → REST (JWT, RBAC) → Express → Prisma/Postgres; Express → Gemini (key never leaves backend).

## Done in this base
Auth+RBAC, issue create/list/status/confirm with duplicate detection and ticket IDs, Gemini image analysis (strict JSON, zod-validated), safety-aware chat, nearby services, alerts, dashboard stats, i18n (EN/HI/TE), high-contrast/large-text toggles, report flow UI.

## TODO / roadmap
Heatmap + analytics endpoints, comments, notifications, admin CRUD, Google Maps picker, GCS upload (replace `saveImage`), tests (Vitest + Playwright), full seed, officer dashboard UI, assistant chat UI.
Deploy: `gcloud run deploy sahayak-api --source backend` · `vercel --cwd frontend` (set `VITE_API_URL`).

## Security (see SECURITY section of the brief)
- Auth: 15 min access JWT + 7 day rotating refresh token, both HttpOnly+Secure+SameSite=Strict cookies. Reuse of an old refresh token revokes the whole token family. CSRF: double-submit token (`X-CSRF-Token`).
- Run frontend and API on the **same site** (Vercel rewrite `/api/*` to Cloud Run, see `frontend/vercel.json`) or Strict cookies will not be sent.
- Phone OTP: 6 digits, HMAC-hashed, 5 min expiry, 5 tries, 3 sends/hour. Required before `POST /api/issues`.
- Staff 2FA (TOTP): required for DEPT_OFFICER, SENIOR_OFFICER, SYSTEM_ADMIN. First login forces setup. Staff tokens without 2FA get 403.
- Phone numbers: AES-256-GCM at rest (`DATA_KEY`) plus HMAC lookup hash. Passwords: bcrypt cost 12. Lockout after 5 bad logins.
- Photos: re-encoded with sharp (EXIF/GPS removed), stored in a private bucket, served by 10 min signed URLs.
- Audit log: hash chain (`verifyChain()`), redacted metadata. Make it append-only in Postgres:
  `REVOKE UPDATE, DELETE, TRUNCATE ON "AuditLog" FROM sahayak_app;`
- GCP least privilege: API service account gets only `roles/storage.objectAdmin` on the one bucket (+ `iam.serviceAccountTokenCreator` on itself for signing). Bucket: uniform access, public access prevention enforced. Separate dev and prod projects and keys. Rotate keys every 90 days via Secret Manager.
- Tests: `cd backend && npm test`. Add monitoring alerts on repeated `LOGIN`/`OTP_REQUEST` failures from the audit table.
