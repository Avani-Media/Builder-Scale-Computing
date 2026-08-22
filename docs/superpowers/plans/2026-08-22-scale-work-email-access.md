# Scale Work-Email Access Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace Scale's normal client bearer-link experience with a clean permanent URL protected by six-digit work-email verification for approved organizations, with server-enforced 90-day trusted-browser access and the current private link retained as an Avani recovery fallback during the pilot.

**Architecture:** Keep GitHub Pages as the public Scale shell and keep the encrypted report payload architecture unchanged. Add a small Supabase-backed authorization layer: a table-driven domain allowlist, a service-role-only session-age check, and a new `scale-report-auth` Edge Function that brokers email OTP request/verify/refresh/logout. Update `scale-private-report-loader` so the primary path accepts a verified Supabase bearer session, re-checks the domain and session age before decryption, and preserves legacy private-link validation as a temporary fallback. Update the generated Scale shell in `publish-scale-html` to present the branded secure access gate, persist/refresh the session, and load live/history content using Authorization headers.

**Tech Stack:** GitHub Pages static HTML/JavaScript, Supabase Auth email OTP, Supabase Edge Functions (Deno/TypeScript), Postgres/RLS, GitHub Contents API, AES-GCM encrypted report payloads.

**Spec:** `docs/superpowers/specs/2026-08-22-scale-work-email-access-design.md`

## Global Constraints

- Pilot scope is Scale only; Syndication access behavior must not change.
- Approved pilot domains are exactly `avanimedia.com` and `scalecomputing.com`.
- Verification uses Supabase native six-digit email OTP; do not build a custom shorter-code system.
- A browser may remain trusted for at most 90 days from the backing auth session's creation time; this must be checked server-side.
- The normal client URL is `https://reports.avanimedia.com/scale/`; the normal client flow must not require a bearer secret in the URL.
- The current private `#access=...` / legacy access path remains available only as a temporary Avani recovery fallback during the pilot.
- Live and Reporting History must use the same authenticated identity and domain policy.
- Encrypted report payload storage/decryption remains AES-GCM and must not be moved into public GitHub HTML.
- Authenticated/report responses must use `Cache-Control: no-store`; report pages retain `noindex,nofollow` and `Referrer-Policy: no-referrer`.
- Never log or commit OTP values, Supabase access tokens, refresh tokens, GitHub publisher tokens, private access tokens, or derived private-link values.
- Production client testing is blocked until custom SMTP can deliver OTP mail to an external `@scalecomputing.com` address.
- Client-facing copy must not claim SOC 2, HIPAA, end-to-end encryption, or other certifications/assurances that have not been established.
- Use TDD for each behavioral change: create a failing probe/test first, verify RED, implement the minimum change, verify GREEN, then commit/deploy.
- Temporary probe/test/write Edge Functions created for this implementation must be neutralized to HTTP 410 after verification; permanent functions must remain active.

---

## File and component map

The current `Builder-Scale-Computing` repository contains the single-file builder at `index.html` plus design/plan docs. The current permanent report infrastructure lives in Supabase Edge Functions rather than in source-controlled files in this repo. During implementation, preserve that deployment model for compatibility and document every permanent function version and database migration applied.

- Modify permanent Edge Function: `scale-private-report-loader/index.ts` — authorize verified work-email sessions before report decryption; retain legacy private-link fallback.
- Modify permanent Edge Function: `publish-scale-html/index.ts` — generate the new secure client shell for live/history and return the clean client URL as the normal share URL.
- Create permanent Edge Function: `scale-report-auth/index.ts` — request/verify/refresh/logout broker with server-side domain policy.
- Create Postgres migration: `report_access_domains_and_scale_session_access` — allowlist table, RLS, seed rows, service-role-only session inspection function.
- Modify builder source: `index.html` — change Scale Studio client-sharing labels/actions so the clean URL is primary and the private link is clearly recovery-only.
- Create temporary TDD/probe functions during execution only: `test_scale_work_email_*_once`, `verify_scale_work_email_*_once` — behavior probes; neutralize after use.

---

### Task 1: Add the server-side Scale domain allowlist and 90-day session check

**Files:**
- Create migration in Supabase: `report_access_domains_and_scale_session_access`
- Test via SQL before/after migration

**Interfaces:**
- Produces table: `public.report_access_domains(report_key text, domain text, enabled boolean, created_at timestamptz, updated_at timestamptz)`
- Produces function: `public.scale_report_session_access(p_session_id uuid, p_email text) returns table(allowed boolean, reason text, domain text, session_created_at timestamptz)`
- Consumers: `scale-report-auth` and `scale-private-report-loader`

- [ ] **Step 1: Write the failing SQL probe**

Run a read-only query that expects the new table/function to exist:

```sql
select to_regclass('public.report_access_domains') as domains_table,
       to_regprocedure('public.scale_report_session_access(uuid,text)') as access_fn;
```

Expected before migration: both values are null.

- [ ] **Step 2: Apply the migration**

Use `apply_migration` with this shape:

```sql
create table public.report_access_domains (
  report_key text not null,
  domain text not null,
  enabled boolean not null default true,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),
  primary key (report_key, domain),
  constraint report_access_domains_report_key_chk check (report_key = lower(trim(report_key))),
  constraint report_access_domains_domain_chk check (domain = lower(trim(domain)) and domain ~ '^[a-z0-9.-]+\.[a-z]{2,}$')
);

alter table public.report_access_domains enable row level security;
revoke all on table public.report_access_domains from anon, authenticated;

insert into public.report_access_domains(report_key, domain, enabled) values
  ('scale','avanimedia.com',true),
  ('scale','scalecomputing.com',true);

create or replace function public.scale_report_session_access(p_session_id uuid, p_email text)
returns table(allowed boolean, reason text, domain text, session_created_at timestamptz)
language plpgsql
security definer
set search_path = public, auth
as $$
declare
  v_domain text := lower(split_part(trim(p_email), '@', 2));
  v_created timestamptz;
begin
  select s.created_at into v_created
  from auth.sessions s
  join auth.users u on u.id = s.user_id
  where s.id = p_session_id
    and lower(coalesce(u.email,'')) = lower(trim(p_email))
    and (s.not_after is null or s.not_after > now());

  if v_created is null then
    return query select false, 'session_invalid', v_domain, null::timestamptz;
    return;
  end if;

  if v_created <= now() - interval '90 days' then
    return query select false, 'session_expired', v_domain, v_created;
    return;
  end if;

  if not exists (
    select 1 from public.report_access_domains d
    where d.report_key='scale' and d.domain=v_domain and d.enabled=true
  ) then
    return query select false, 'domain_not_allowed', v_domain, v_created;
    return;
  end if;

  return query select true, 'ok', v_domain, v_created;
end;
$$;

revoke all on function public.scale_report_session_access(uuid,text) from public, anon, authenticated;
grant execute on function public.scale_report_session_access(uuid,text) to service_role;
```

- [ ] **Step 3: Verify schema and RLS**

Run `select report_key, domain, enabled from public.report_access_domains order by domain;`. Expected exactly two enabled Scale rows: `avanimedia.com` and `scalecomputing.com`.

- [ ] **Step 4: Verify browser roles cannot read the allowlist or execute the session function**

Inspect grants and policies; expected: no anon/authenticated SELECT policy and no EXECUTE grant for anon/authenticated.

- [ ] **Step 5: Run Supabase Security Advisor**

Expected: no new warning that exposes the allowlist or security-definer function to browser roles.

---

### Task 2: Create the `scale-report-auth` OTP broker

**Files:**
- Create permanent Edge Function: `scale-report-auth/index.ts`
- Create temporary probe: `test_scale_work_email_auth_red_once/index.ts`

**Interfaces:**
- Consumes: `report_access_domains`, `scale_report_session_access(uuid,text)`, Supabase Auth REST API.
- Produces actions: `request_code`, `verify_code`, `refresh`, `logout`.
- `verify_code`/`refresh` success: `{ok:true,session:{access_token,refresh_token,expires_in,email}}`.

- [ ] **Step 1: Deploy the RED probe**

Expect endpoint missing or failing these checks before implementation: malformed email -> 400, unrelated domain -> 403, OPTIONS from `https://reports.avanimedia.com` -> 200 exact allowed origin.

- [ ] **Step 2: Implement request validation and exact-origin CORS**

Use `REPORT_KEY='scale'` and allowed origins `https://reports.avanimedia.com`, `https://avani-media.github.io`. Only POST/OPTIONS. All responses use no-store.

- [ ] **Step 3: Implement `request_code`**

Normalize email lowercase, extract exact domain, service-role query the allowlist, reject unapproved domains before Auth. For approved addresses call Supabase Auth email OTP and return only `{ok:true}`.

- [ ] **Step 4: Implement `verify_code`**

Require exactly six ASCII digits. Verify with Supabase Auth, re-check the returned verified email domain, extract the validated session's `session_id`, and call `scale_report_session_access`. Return only the session fields required by the shell. Never log tokens.

- [ ] **Step 5: Implement `refresh` and `logout`**

Refresh via Supabase Auth, then re-run domain/session-age authorization before returning new tokens. Logout is idempotent and never echoes submitted tokens.

- [ ] **Step 6: Re-run the probe and perform a controlled Avani OTP smoke test**

Expected GREEN for validation/CORS and successful real OTP verification for a consenting Avani mailbox.

---

### Task 3: Add authenticated-session authorization to `scale-private-report-loader`

**Files:**
- Modify permanent Edge Function: `scale-private-report-loader/index.ts` (version 4 at plan creation)
- Create temporary probe: `test_scale_authenticated_loader_red_once/index.ts`

**Interfaces:**
- Consumes `Authorization: Bearer <Supabase access token>`.
- Preserves legacy `share=` / `access=` recovery authentication.

- [ ] **Step 1: Prove RED**

A valid Avani bearer session against current v4 should still return 403 because v4 only accepts query credentials. Confirm valid legacy share still returns 200 before change.

- [ ] **Step 2: Allow the Authorization header in CORS**

Preserve exact origin allowlist and no-store behavior.

- [ ] **Step 3: Validate bearer identity with Supabase Auth**

Use the Auth user endpoint to validate the token and obtain verified email. Decode only the already-validated token to retrieve `session_id`.

- [ ] **Step 4: Enforce domain + 90-day session age**

Call `scale_report_session_access(session_id,email)`. Map `session_expired` to 401, `domain_not_allowed` to 403, other invalid auth to 401. Never return HTML on failure.

- [ ] **Step 5: Preserve independent legacy fallback**

Authorization precedence: valid bearer OR valid legacy share/access may proceed. Do not alter current HMAC labels or AES decryption derivation.

- [ ] **Step 6: Run live/history regression matrix**

Expected valid bearer 200 live/history; valid legacy 200 live/history; invalid/missing auth rejected; all report responses no-store/no-referrer.

---

### Task 4: Replace the generated Scale shell with the branded work-email gate

**Files:**
- Modify permanent Edge Function: `publish-scale-html/index.ts` (version 12 at plan creation)
- Create temporary source test: `test_scale_work_email_shell_red_once/index.ts`

**Interfaces:**
- Consumes `scale-report-auth` and `scale-private-report-loader`.
- Stores session under `avani_scale_report_session_v1`.
- Preserves `#access=...` as recovery only.

- [ ] **Step 1: Prove RED from current live shell source**

Assert current shell lacks `Secure Report Access`, `Send verification code`, `Verify & Open Report`, the new storage key, and bearer Authorization loading.

- [ ] **Step 2: Implement the email gate**

Render approved copy: AVANI Media × Scale Computing; Secure Report Access; confidential campaign intelligence restriction; Work email; Send verification code; Protected access · Work-email verification · Trusted browser for 90 days; email-use privacy note.

- [ ] **Step 3: Implement the six-digit verify state**

Use an accessible numeric input with `inputmode=numeric`, `autocomplete=one-time-code`, `maxlength=6`; show masked email, Verify & Open Report, Resend code after 60 seconds, Use a different email.

- [ ] **Step 4: Implement session boot/refresh/load order**

Recovery fragment first; otherwise stored session; refresh when needed; loader with bearer; clear/show gate on 401; clear/show authorization message on domain 403; no session -> email gate.

- [ ] **Step 5: Preserve live/history destination**

History shells use identical auth logic and continue to the requested history date after successful verification.

- [ ] **Step 6: Return clean client URLs from publishing**

Normal client live/history URLs are clean custom-domain paths. Return a distinct recovery private URL for internal Scale Studio use while retaining backward-compatible legacy fields during transition.

- [ ] **Step 7: Re-run source/deployed shell tests**

Expected GREEN for required copy, session flow, Authorization, clean URL, history behavior, and recovery fragment support.

---

### Task 5: Update Scale Studio sharing controls

**Files:**
- Modify `Builder-Scale-Computing/index.html`
- Create temporary source test: `test_scale_clean_client_url_red_once/index.ts`

**Interfaces:**
- Primary client action consumes the clean `https://reports.avanimedia.com/scale/` URL.
- Secondary recovery action consumes the private fragment URL.

- [ ] **Step 1: Prove RED**

Confirm current builder still treats query/hash access URLs as the valid live-report URL and has no separate clean/recovery model.

- [ ] **Step 2: Split URL validation**

Add one normalizer that accepts only the exact clean Scale client URL and a separate recovery normalizer that accepts `?access=` or `#access=`.

- [ ] **Step 3: Update publish response storage and labels**

Primary action: **Copy client report URL** / **Open client report** using the clean URL. Secondary internal action: **Copy private recovery link**. Never print the recovery secret in visible status text.

- [ ] **Step 4: Verify publishing behavior**

Publishing does not create a new client-facing URL; normal client URL remains constant across publishes. Build Preview and other builder behavior remain unchanged.

- [ ] **Step 5: Publish builder and verify GitHub Pages**

Hard-refresh Scale Studio and test both clean primary and private recovery actions.

---

### Task 6: Configure OTP email delivery for the client pilot

**Files:**
- Supabase Auth SMTP and email OTP template configuration

**Interfaces:**
- Produces six-digit OTP mail to approved external recipients.
- Subject: **Your AVANI report verification code**.

- [ ] **Step 1: Inspect only non-secret SMTP readiness**

Determine whether custom SMTP is enabled without printing credentials.

- [ ] **Step 2: If not configured, pause external Scale rollout**

Continue internal Avani testing only until an Avani-controlled SMTP sender is configured through a secure credential path.

- [ ] **Step 3: Configure code-only OTP email template**

Prominently use Supabase `{{ .Token }}`; no magic-link CTA, marketing copy, or unnecessary links.

- [ ] **Step 4: Verify a consenting `@scalecomputing.com` test mailbox**

Confirm receipt and successful code verification without logging/storing the OTP.

---

### Task 7: End-to-end security and lifecycle verification

**Files:**
- Create temporary verifier: `verify_scale_work_email_access_final_once/index.ts`

**Interfaces:**
- Produces booleans/status codes only; never tokens or private URLs.

- [ ] **Step 1: Fresh browser clean URL**

Expected branded Secure Report Access gate and no decrypted HTML.

- [ ] **Step 2: Avani and Scale Computing work-email flows**

Verify real OTP, report open, reload without another code inside trust window.

- [ ] **Step 3: Unapproved domain**

Expected blocked authorization state and no report HTML.

- [ ] **Step 4: Reporting History**

Authenticated browser opens history without re-verification; fresh browser direct history URL verifies once then continues to that snapshot.

- [ ] **Step 5: Server-simulate a >90-day test session**

Use an isolated test session only. Expected loader 401 `session_expired`; never modify a real client session.

- [ ] **Step 6: Test immediate domain revocation**

Temporarily disable the test domain row, expect loader 403, then immediately restore it.

- [ ] **Step 7: Verify legacy recovery path**

Fresh browser with Scale Studio recovery link still opens the report during the pilot.

- [ ] **Step 8: Inspect public source and headers**

No decrypted HTML/auth tokens/private bearer values in GitHub. Auth/report responses no-store; report no-referrer; shell/report noindex,nofollow.

- [ ] **Step 9: Test mobile and keyboard usability**

Verify focus order, Enter submit, visible focus, code input semantics, resend countdown, and readable errors.

---

### Task 8: Cleanup temporary tooling and record rollback points

**Files:**
- Neutralize implementation-specific temporary Edge Functions

**Interfaces:**
- Permanent functions left active: `publish-scale-html`, `scale-private-report-loader`, `scale-report-auth`.

- [ ] **Step 1: Identify only temporary implementation functions**

Include `test_scale_work_email_*_once`, `verify_scale_work_email_*_once`, and one-time planning/probe functions created for this feature.

- [ ] **Step 2: Neutralize each to HTTP 410**

Use `Deno.serve(() => new Response('Gone',{status:410,headers:{'cache-control':'no-store'}}));`.

- [ ] **Step 3: Re-run production smoke tests after cleanup**

Clean URL, OTP, live, history, and recovery link must still work.

- [ ] **Step 4: Record non-secret deployed versions and rollback references**

Capture builder commit plus versions for `publish-scale-html`, `scale-private-report-loader`, `scale-report-auth`, and the migration. Pre-feature publisher v12 and loader v4 are rollback references.

---

## Final acceptance criteria

- Client can bookmark/use `https://reports.avanimedia.com/scale/` without retaining a secret URL.
- Fresh browser cannot decrypt report without approved work-email verification or temporary recovery link.
- Only `avanimedia.com` and `scalecomputing.com` are enabled pilot domains.
- Email verification uses six-digit OTP; external Scale mailbox tested through custom SMTP before rollout.
- Returning browser avoids another code for up to 90 days, while loader rejects sessions older than 90 days server-side.
- Disabling a domain blocks existing sessions on next report load.
- Reporting History uses the same authorization and preserves direct destinations.
- Normal client URL contains no bearer credential.
- Legacy private links remain functional as temporary recovery only.
- Public GitHub files contain no decrypted report HTML or authentication secrets.
- Syndication behavior is unchanged.
- All one-time implementation/test functions are neutralized after verification.
