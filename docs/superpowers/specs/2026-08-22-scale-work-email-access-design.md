# Scale Work-Email Access — Design Spec

Date: 2026-08-22
Status: Approved design, pending implementation plan
Scope: Scale only

## Goal

Replace the normal client experience of secret bearer links with a clean permanent Scale URL protected by work-email verification. The client should be able to return to `https://reports.avanimedia.com/scale/` without keeping a private URL. The access experience must look deliberate, secure, and client-ready without making compliance or security claims Avani has not established.

## Pilot access policy

Approved work-email domains for Scale:

- `avanimedia.com`
- `scalecomputing.com`

Only a successfully verified email on an enabled approved domain may establish normal report access. Domain authorization is enforced server-side.

A verified browser remains trusted for 90 days from the authentication session's creation. After 90 days, the user is required to verify the work email again.

## Client experience

The permanent client URL remains:

`https://reports.avanimedia.com/scale/`

A fresh or expired browser sees a branded access gate rather than the current private-link error. The gate should present:

- AVANI Media × Scale Computing branding
- Heading: **Secure Report Access**
- Message: **This report contains confidential campaign intelligence and is restricted to authorized organizations.**
- Work email field
- Primary action: **Send verification code**
- Trust copy: **Protected access · Work-email verification · Trusted browser for 90 days**
- Small privacy copy: **Your email is used only to verify report access.**

The UI must not use account-creation language, passwords, Supabase branding, or technical authentication terminology. It may use a restrained lock/security icon, but must not claim certifications, end-to-end encryption, SOC 2, HIPAA, or similar assurances unless separately established.

After requesting a code, the screen changes to a six-digit verification step:

- Heading: **Verify your identity**
- Message: **A 6-digit verification code was sent to [masked email].**
- One accessible numeric input visually presented as six code positions
- Primary action: **Verify & Open Report**
- **Resend code** becomes available after 60 seconds
- **Use a different email** returns to the email step

A successful verification opens the report immediately at the same clean `/scale/` URL.

## Authentication architecture

Use Supabase Auth email OTP as the identity proof. Native six-digit email OTP is preferred over a custom shorter code.

The public GitHub Pages shell does not talk directly to Supabase Auth. Instead, a Scale-specific Edge Function, `scale-report-auth`, proxies the small set of authentication actions. This keeps the public shell free of auth service configuration details beyond the function URL and centralizes domain checks.

`scale-report-auth` supports these actions:

1. `request_code`
   - Normalize and validate the submitted email.
   - Resolve its domain.
   - Check the current server-side allowlist for report key `scale`.
   - If allowed, request a Supabase email OTP.
   - Return a client-safe response without sensitive details.

2. `verify_code`
   - Verify the six-digit OTP with Supabase Auth.
   - Re-check the verified email domain server-side.
   - Return the resulting Supabase access/refresh session to the browser over HTTPS.

3. `refresh`
   - Exchange a valid refresh token for a new access/refresh pair.
   - Re-check that the authenticated email domain is still enabled before returning the refreshed session.

4. `logout`
   - Revoke the current Supabase session when possible and instruct the browser to remove its local session state.

The browser stores the authenticated session in local storage under a Scale-specific key. The public URL itself contains no bearer secret.

## 90-day trusted-browser enforcement

The 90-day rule must not depend only on a browser timestamp. It is enforced server-side.

Each Supabase access token contains a `session_id`. The private report loader validates the access token with Supabase Auth and then performs a service-role-only database check against the session represented by that `session_id`. Access is allowed only when:

- the session exists and is active;
- the authenticated email is present;
- the email domain is currently enabled for Scale; and
- the session creation time is no more than 90 days old.

A small database function/RPC may encapsulate the `auth.sessions` age check. It must be callable only by the service role, not by anonymous or ordinary authenticated browser clients.

If a session is older than 90 days, the loader returns an authentication-expired response. The shell clears the stale local session and displays the work-email verification gate again.

## Domain allowlist

Use a table-driven allowlist so future client domains can be added without rewriting the authentication logic.

Suggested table: `report_access_domains`

Required fields:

- `report_key` — e.g. `scale`
- `domain` — normalized lowercase domain
- `enabled` — boolean
- created/updated timestamps

For the pilot, seed only:

- `scale / avanimedia.com / enabled`
- `scale / scalecomputing.com / enabled`

Enable RLS and expose no browser policies. Edge Functions use the service role for allowlist reads.

Removing or disabling a domain must take effect immediately on the next loader/auth check, even when a browser still has a previously issued Supabase session.

## Report loader changes

`scale-private-report-loader` remains the only component that returns decrypted report HTML. The encrypted GitHub payload architecture remains unchanged.

Primary access path:

1. Browser sends the current Supabase access token in the `Authorization: Bearer ...` header.
2. Loader validates the token with Supabase Auth.
3. Loader checks the current Scale domain allowlist.
4. Loader checks that the backing auth session is no more than 90 days old.
5. Only then does the loader decrypt and return the report HTML.

The same authorization applies to current live content and Reporting History.

Responses containing report HTML remain `Cache-Control: no-store`. CORS remains restricted to approved report origins.

## Legacy private-link fallback

Keep the existing `#access=...` / legacy private-link mechanism temporarily as an Avani recovery fallback during the pilot. It is not the normal client experience and should not be surfaced as the primary client sharing method.

Scale Studio may retain **Copy private link** / private recovery capability during the pilot so Avani cannot be locked out if email delivery has an incident.

After the work-email pilot is stable, removal of the legacy bearer-link path should be a separate deliberate change.

## Email delivery

Production use with external client addresses requires custom SMTP for Supabase Auth. The default Supabase mail service is suitable for development/testing only and may not deliver to arbitrary external recipients.

The production sender must be an Avani-controlled authenticated mail identity. The exact sender address and SMTP provider are deployment configuration, not application behavior. SPF, DKIM, and DMARC should be configured for the sending domain when available.

The OTP email should be concise and security-focused:

- Subject: **Your AVANI report verification code**
- AVANI branding
- Prominent six-digit code
- Short expiry/security note
- No marketing content
- No unnecessary links

The implementation must not be considered ready for a Scale client pilot until external OTP delivery through custom SMTP is verified.

## Security and privacy controls

- No report HTML is committed publicly in decrypted form.
- No private access key is required in the normal client URL.
- Email-domain checks occur server-side, not only in JavaScript.
- The loader validates authenticated identity before decryption.
- The 90-day limit is checked server-side against the auth session.
- Authentication/report responses use no-store caching.
- The static gate and returned report retain `noindex, nofollow`.
- Report pages retain a no-referrer policy so report URLs are not unnecessarily disclosed through outbound navigation.
- Do not log OTP values, refresh tokens, access tokens, or full authentication payloads.
- Error messages shown to the client are concise and do not expose backend details.
- Native Supabase OTP resend/rate limits remain in force; the UI also prevents immediate resend for 60 seconds.

## Error states

Unapproved domain:
**This work email is not authorized for this report. Please contact your AVANI Media representative if you believe you should have access.**

Incorrect or expired code:
**That verification code is invalid or has expired. Please check the code or request a new one.**

Email delivery failure:
**We couldn't send a verification code right now. Please try again in a moment.**

Expired 90-day trust:
**For your security, please verify your work email again to continue.**

Loader/service incident:
**Secure report access is temporarily unavailable. Please try again shortly.**

Do not fall back to showing decrypted content on any authentication error.

## Scale Studio behavior

Normal client sharing becomes the permanent clean URL:

`https://reports.avanimedia.com/scale/`

Scale Studio should label this as the client report URL once the pilot is enabled. Private bearer-link controls remain available only as the temporary recovery path during the pilot.

Publishing a report must not reset approved domains or invalidate existing authenticated client sessions unless the access policy itself changes.

## Reporting History

Reporting History uses the same authenticated browser session and domain policy. A client who has already verified should be able to open saved history entries without another code during the 90-day trust window.

Direct navigation to a history URL without a valid session must show the same secure work-email access gate and, after successful verification, continue to the originally requested history entry.

## Testing requirements

Before enabling the pilot, verify at minimum:

1. `@avanimedia.com` can request and verify a real OTP.
2. `@scalecomputing.com` can request and verify a real OTP.
3. An unrelated domain cannot request report access.
4. Correct six-digit OTP opens the live report.
5. Incorrect/expired OTP does not open the report.
6. Refresh and browser revisit work without another OTP during the trust window.
7. A server-simulated session older than 90 days is rejected.
8. Disabling an allowed domain immediately blocks a still-signed-in user from decrypting the report.
9. Reporting History respects the same auth policy.
10. Legacy private recovery link still works during the pilot.
11. Missing/invalid auth never returns decrypted HTML.
12. Auth and report responses are not cached.
13. No OTP, access token, refresh token, or private bearer value appears in GitHub source, logs, or user-visible diagnostics.
14. The access screen is usable on desktop and mobile and has keyboard-accessible form controls.

## Rollout

Pilot this only on Scale. Do not change Syndication access behavior as part of this work.

Rollout sequence:

1. Configure and verify custom SMTP for external OTP delivery.
2. Add allowlist storage and 90-day session-age check.
3. Deploy `scale-report-auth`.
4. Update `scale-private-report-loader` to accept authenticated sessions.
5. Update the Scale static shell with the secure access UI and session flow.
6. Preserve legacy bearer access as recovery.
7. Test with an Avani address in a fresh browser.
8. Test with a Scale Computing address in a fresh browser.
9. Only then make the clean `/scale/` URL the normal client-sharing path.

## Out of scope for this pilot

- Google or Microsoft social sign-in
- SAML/enterprise SSO
- Per-user invitations
- Individual user allow/deny lists
- Access analytics dashboards
- Applying this auth model to Syndication
- Removing the legacy private recovery path
