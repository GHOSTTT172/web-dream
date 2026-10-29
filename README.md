# Brookhaven RP — Staff Applications

Next.js (App Router) + TypeScript + Tailwind + Supabase (Postgres, Auth, RLS) + Zod + React Hook Form.

## Run locally
1. Install Node 20+ and VS Code. `npm install`
2. `cp .env.example .env.local` and fill in values. `npm run dev` → http://localhost:3000
3. Checks: `npm run typecheck`, `npm test`

## Supabase setup
1. Create a project at supabase.com. Copy URL, anon key and service-role key into `.env.local` (the service-role key is server-only).
2. SQL editor: run `supabase/migrations/0001_init.sql`. It enables RLS on every table and creates the policies, sequence, and functions.
3. Seed the two admins (lowercase, exactly two; a trigger blocks a third):
   `insert into public.admin_allowlist (email) values ('admin1@gmail.com'), ('admin2@gmail.com');`
4. Set the same two addresses in `ADMIN_EMAIL_1` / `ADMIN_EMAIL_2`. If either is missing/invalid/duplicated the dashboard is closed.

## Google OAuth
1. Google Cloud Console → Credentials → OAuth client (Web). Authorized redirect URI: `https://<project>.supabase.co/auth/v1/callback`.
2. Supabase → Authentication → Providers → Google: paste client ID/secret, enable.
3. Supabase → Authentication → URL Configuration: Site URL = your domain; add `http://localhost:3000/auth/callback` and `https://YOUR-DOMAIN/auth/callback` to Redirect URLs.
4. Disable email/password sign-ups in Supabase Auth so Google is the only way in.

## How access is enforced (three layers)
- App: every admin page and server action calls `requireAdmin()` (validated Auth session + env allowlist).
- Database: RLS policies call `is_admin()`, which checks the signed JWT `email` claim against `admin_allowlist`. Anonymous/other users get no rows.
- Public submission goes only through `/api/applications` using the service-role key and the `submit_application` function (not callable by anon/authenticated).

## Identifiers
`submit_application` generates `BRP-YYYY-000127` (Postgres sequence, never reused) and `BRP-XXXX-XXXX` (pgcrypto random, 32-char alphabet) atomically with the insert, retrying on collisions. A per-form `submission_key` makes double clicks/retries return the same record. Discord fields are self-reported and displayed as unverified.

## Email (optional)
Create a Resend account and verify a domain; set `RESEND_API_KEY`, `EMAIL_FROM`, optionally `TEAM_NOTIFY_EMAIL` and `APPLICANT_DECISION_EMAILS=true`. Without them nothing is sent; the confirmation page only says an email was sent when the provider accepted it.

## Test unauthorized access
Sign in with a third Google account → you are signed out and sent to `/login?denied=1`. Visiting `/admin` anonymously redirects to `/login`. In the SQL editor, `set role anon; select * from applications;` must return a permission error.

## Deploy (Vercel)
Import the repo, add every variable from `.env.example` (set `NEXT_PUBLIC_SITE_URL`), deploy, then add the Vercel URL to Supabase redirect URLs.

## Change / revoke an administrator
`delete from public.admin_allowlist where email = 'old@gmail.com';` (revokes database access immediately), update the `ADMIN_EMAIL_*` env var, redeploy. Also remove the user in Supabase → Authentication → Users to end sessions.

## Troubleshooting
- Redirect loop to `/login`: env admin emails don't match the Google account or the allowlist table.
- `redirect_uri_mismatch`: Google redirect URI must be the Supabase callback URL.
- Submission returns 500: migration not applied or `SUPABASE_SERVICE_ROLE_KEY` missing.

## Known limitations
Rate limiting is in-memory per instance (use Upstash for production). Tests cover validation and the allowlist logic only; RLS/API integration tests need a live Supabase project. Retention is a policy note, not automated.
