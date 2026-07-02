# Vaqya Automation Dashboard — Security Hardening Design

**Date:** 2026-07-02
**Status:** Approved (pending spec review)
**Repo:** `vaqya-automation-dashboard` (GitHub Pages, single-file `index.html`)

## Problem

The dashboard is a static site on GitHub Pages with a **client-side password gate**.
The gate does not protect the data: the Supabase **anon key is embedded in the public
page source**, so anyone can call the Supabase REST API (and realtime) directly and read
or write `practice_statuses` and `portal_statuses` without ever seeing the login screen.

Hashing the password (done 2026-07-02, commit `7371a97`) stopped the raw password from
appearing in source, but did not change this fundamental exposure. Real security requires
moving enforcement from the browser to the server.

## Goal

Make data access require a real, server-verified login for a small fixed team, with the
database rejecting all unauthenticated access. No new hosting infrastructure, no new
paid services.

## Non-Goals

- Cloudflare Access / custom domain / edge SSO (explicitly deferred — moderate-effort path chosen).
- Changing the Google Chat webhook automation (see Out of Scope).
- Adding per-user roles/permissions (all team members have equal read/write access).

## Data Sensitivity

Business data only: practice names, per-component automation progress, and payer-portal
**setup status** (Not Started / In Progress / Complete / Roadblock / N/A). No patient data
(PHI), no stored portal credentials. Moderately sensitive internal data.

## Design

### 1. Authentication — Supabase Auth magic-link (passwordless)

- Replace the password overlay (`#loginOverlay`) with an **email-entry form**.
  Submitting calls `supabase.auth.signInWithOtp({ email, options: { emailRedirectTo } })`.
  Supabase emails a one-time link; clicking it returns the user with a valid session.
- **Allowlist by pre-provisioning + closed sign-ups:**
  - Create these emails as users in Supabase Auth:
    - `drmukundan@vaqya.com` (owner)
    - `varun@vaqya.com`
    - `sharath@vaqya.com`
    - `lokesh@vaqya.com`
  - **Disable public sign-ups** in Supabase Auth settings so an email that isn't on the
    list cannot create an account or gain access.
- **Session check on load:** replace the `sessionStorage.getItem('vaqya_auth')` check with
  `supabase.auth.getSession()`. Only a valid signed token boots the dashboard.
- **Logout:** `handleLogout()` calls `supabase.auth.signOut()` and shows the login form.
- **Remove obsolete code:** `TEAM_PASSWORD_HASH`, `sha256Hex` (if unused elsewhere), the
  password comparison in `handleLogin()`, and the `vaqya_auth` sessionStorage flag.

### 2. Database lockdown — Row Level Security (the core fix)

- Enable **RLS** on `practice_statuses` and `portal_statuses`.
- Policies grant `SELECT`, `INSERT`, `UPDATE`, `DELETE` **only to the `authenticated` role**;
  the `anon` role gets no access.
- Effect: the public anon key becomes harmless for data access — REST and realtime return
  nothing without a valid login session. Existing rows are untouched; only access changes.
- Realtime subscriptions continue to work for authenticated users (RLS is enforced on the
  realtime stream too).

### 3. Auth hardening

- Set Supabase **Site URL** and **Redirect URL allowlist** to *only* the GitHub Pages URL
  (`https://shanmukund.github.io/vaqya-automation-dashboard/`) so magic links cannot be
  redirected to an attacker-controlled site.
- Keep OTP/link expiry short (Supabase default ~1 hour is acceptable).
- Confirmed the repo contains only the **anon** key — no `service_role` secret to rotate.

### 4. Verification (mandatory — not skippable)

Because live RLS state could not be tested from the build environment (network egress
blocked), implementation **begins and ends** with explicit checks:

1. **Before:** probe current exposure (anon read + anon write to both tables) and record it.
2. **After RLS + Auth:**
   - Anonymous read/write to both tables returns empty / `401` / `403`.
   - A logged-in allowlisted user loads and edits data successfully.
   - A logged-out / non-allowlisted browser is blocked.
   - Realtime updates still propagate between two logged-in sessions.

## Out of Scope (flagged, unchanged)

The Google Chat webhook URL lives in `localStorage` and the downloaded `SKILL.md`, not in
the committed repo. The Friday automation posts a templated message to Google Chat and does
not query Supabase, so it is unaffected by RLS. Left as-is.

## Known Constraint

Supabase's built-in email has free-tier rate limits (~a few links/hour). Acceptable for a
small team logging in occasionally. If login emails become unreliable, configure custom
SMTP in Supabase Auth settings (future change, not part of this work).

## Affected Surface

- `index.html` — login overlay markup + auth JS (`handleLogin`, `handleLogout`, boot check).
- Supabase project settings — Auth (users, sign-up toggle, redirect allowlist) + RLS policies
  on `practice_statuses` and `portal_statuses`. These are console/SQL changes, not repo files.

## Success Criteria

- No credential or bypass path exists in the public page source.
- All data access requires a Supabase-verified session belonging to an allowlisted user.
- The four allowlisted users can use the dashboard exactly as before.
- Verification checklist in section 4 passes end-to-end.
