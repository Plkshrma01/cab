# AutoSetu — Your Local Ride, Connected

A vanilla HTML/CSS/JavaScript MVP for local auto-rickshaw and taxi connections. It uses Supabase Auth, PostgreSQL, RLS and database RPCs. It is designed for GitHub Pages and does not require Node.js.

## Important status and production limitations

This package contains a working frontend and a database implementation for the core authenticated flows, but **it is not yet a fully audited commercial ride-hailing system**. External configuration and end-to-end testing are required before real customers use it.

- Phone OTP requires Supabase Phone Auth and a configured SMS provider. SMS delivery may cost money depending on provider, destination, and volume.
- GPS requires user permission and HTTPS (GitHub Pages provides HTTPS).
- The current booking UI requires pickup GPS coordinates. Destination coordinates and route drawing are not implemented in this MVP.
- The driver dashboard requires admin approval. Driver acceptance is atomic via a database update.
- Arrival checks the driver's last stored coordinates. The driver location is updated when they go online; continuous GPS tracking is not implemented yet, so the 200m check should not be treated as production-grade proof of presence.
- OTP RPCs demonstrate secure hashed storage and role checks, but the current `request_ride_otp` returns the code to the passenger's authenticated browser. It does **not send SMS**. Before production, replace the return flow with a trusted Edge Function or database-to-provider integration that sends OTP only to the passenger, and never returns it to the driver. Do not advertise OTP/SMS delivery as live until configured and tested.
- Carpool SQL includes atomic seat booking, but a complete carpool UI is not included in this first package.
- Document upload, emergency dispatch integration, account deletion workflow, full admin user management, reviews UI, and a live map/driver-location stream still need implementation before a full launch.
- The UI intentionally does not invent nearby drivers, ratings, distances, or GPS locations.

## 1. Configure Supabase

1. Create a project at https://supabase.com/.
2. In **Project Settings → API** (or API Keys in the current dashboard), copy the Project URL and the **publishable/anon public key**.
3. Open `config.js` and replace:
   - `YOUR_SUPABASE_PROJECT_URL`
   - `YOUR_SUPABASE_PUBLISHABLE_OR_ANON_KEY`
4. Never use the `service_role` / secret key in browser code, GitHub, screenshots, or chat.

## 2. Run SQL files in order

In Supabase, open **SQL Editor → New query**. Run the complete contents of:
1. `database/schema.sql`
2. `database/functions.sql`
3. `database/policies.sql` (storage notes only)

The scripts are designed to be re-run without dropping tables or deleting records. If you already have an older AutoSetu schema, back it up and inspect existing columns/policies before applying changes. SQL may need a migration if your existing enums/tables differ.

After running SQL, go to **Database → Tables** and confirm the tables exist. If Supabase reports a policy/function error, stop and fix it rather than assuming setup succeeded.

## 3. Configure phone OTP

1. Open **Authentication → Sign In / Providers** and enable **Phone**.
2. Configure an SMS provider in **Authentication → SMS Provider** (dashboard wording may vary).
3. Add provider credentials in Supabase dashboard secrets/settings, not in `config.js`.
4. Set SMS templates and rate limits. Use a test number/provider sandbox if offered.
5. Test with a phone you control. Do not disable OTP verification for production.
6. In **Authentication → URL Configuration**, add your GitHub Pages URL to allowed Site URL / Redirect URLs as appropriate, for example `https://YOUR-USERNAME.github.io/autosetu-/`.

Supabase SMS delivery depends on provider configuration and plan/provider pricing. Check current pricing with Supabase and your chosen SMS provider before launch.

## 4. Create the first administrator securely

Public signup always creates a `passenger` role. Never add an admin dropdown to public registration.

1. Sign in once with the intended administrator phone number using the site.
2. In Supabase **Authentication → Users**, copy that user's UUID.
3. In SQL Editor, run this one-time statement after replacing the UUID:
   ```sql
   update public.profiles set role = 'admin' where id = 'PASTE-USER-UUID-HERE';
   ```
4. Restrict SQL Editor access to trusted project owners. Do not expose the Supabase service role key.
5. Sign out and sign in again. The account should show the admin dashboard.

## 5. Storage buckets

In **Storage**, create:
- `profile-photos` — choose public only if you intentionally want profile images public.
- `driver-documents` — **private**. Never use a public bucket for licence/insurance documents.

This MVP does not yet include document upload controls or storage policies for these buckets. Do not upload real identity documents until narrow folder-level policies and admin review access are implemented.

## 6. Enable Realtime

Open **Database → Publications** (or Realtime settings) and add `public.rides` to the Realtime publication. RLS must remain enabled. The app subscribes to ride row changes while a dashboard is open; a browser refresh also reloads current data.

## 7. Deploy to your existing GitHub repository `autosetu-`

1. Download and unzip this project.
2. Open your existing GitHub repository `autosetu-`.
3. Upload/replace the project files and folders in the repository root. Keep `index.html` at the root.
4. Commit to the branch you publish from (often `main`).
5. Open **Settings → Pages**. Choose **Deploy from a branch**, select `main` and `/ (root)`, then Save.
6. Wait for the Pages deployment to finish. Open the Pages URL over HTTPS.
7. If the repo URL includes a trailing dash, use that exact URL in Supabase Authentication URL configuration.
8. Test in a private/incognito window and review browser DevTools Console and Supabase Auth/Logs if a step fails.

No local server or Node.js is required for normal GitHub Pages hosting.

## 8. Test checklist

### Setup and authentication
- [ ] `config.js` has the project URL and public key (never service role).
- [ ] SQL files run without errors.
- [ ] Phone provider is enabled and configured.
- [ ] OTP is received, invalid OTP is rejected, and logout works.
- [ ] First login leads to profile setup.
- [ ] Existing profile leads to the correct role dashboard.

### Passenger
- [ ] GPS permission prompt appears.
- [ ] Denied GPS gives a clear error.
- [ ] A ride request with real GPS coordinates is created.
- [ ] Passenger sees their ride and updated status.
- [ ] Report form accepts only a ride involving the reporter.

### Driver
- [ ] Driver profile exists and remains pending initially.
- [ ] Pending driver cannot go online or accept rides.
- [ ] Admin approves driver after separately verifying documents.
- [ ] Approved driver can go online with GPS.
- [ ] Two drivers cannot accept the same ride.
- [ ] Arrival check fails if stored GPS is more than 200m away.
- [ ] OTP flow is not advertised as SMS-working until provider integration and end-to-end tests pass.

### Admin and security
- [ ] Admin role is assigned only through trusted SQL.
- [ ] Non-admin cannot call admin RPCs.
- [ ] RLS is enabled on all user data tables.
- [ ] Public frontend has no service-role/secret key.
- [ ] Driver documents are private and protected by tested Storage policies.
- [ ] Review, suspension, account deletion, and incident response processes are tested before launch.

## Known implementation scope

This is a secure-leaning MVP foundation, not a claim that every feature in the master specification is finished. The current release implements the landing page, OTP UI, profile/role setup, passenger ride request, basic driver availability and acceptance, arrival proximity check against stored GPS, admin driver approval/statistics, report submission, core SQL schema/RLS, and atomic carpool booking RPC. It does not yet implement all 21 screens, full carpool frontend, document upload, continuous location streaming, external SMS delivery for ride OTP, destination geocoding/routing, full admin user suspension, or account deletion. Those should be completed and security-reviewed before a public launch.
