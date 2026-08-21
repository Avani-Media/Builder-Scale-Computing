# Scale Private Report Loader Redesign

## Problem

Scale Studio currently publishes the entire client report as a large AES-GCM encrypted payload embedded directly inside the public GitHub Pages `index.html`. The client link is correctly generated as `https://avani-media.github.io/Report-Scale-Computing/?access=...`, but Chrome must download the multi-megabyte wrapper, derive a key from the access value, decrypt the full report locally, and rewrite the document before the client report can appear. The current live report is stalling on `Verifying your secure report link...` during that browser-side decrypt/render step.

## Goal

Keep the same private client URL behavior while moving decryption and access verification off the browser. GitHub Pages should load a small private shell, the shell should send the access value to a Scale-only Edge Function, and the Edge Function should validate the access token, decrypt the protected report payload server-side, and return the report HTML.

## Architecture

### Published files

For the live report, `Report-Scale-Computing` will contain:

- `index.html` — a lightweight private loader shell only.
- `private-payload.json` — AES-GCM encrypted report payload containing only `iv`, `ciphertext`, and a schema/version marker.

For each Reporting History snapshot:

- `history/YYYY-MM-DD/index.html` — the same lightweight shell configured for that history date.
- `history/YYYY-MM-DD/private-payload.json` — the encrypted historical report payload.
- `history/history.json` — remains the reporting history index and keeps access placeholders as today.

### Scale-only loader endpoint

Add a dedicated Supabase Edge Function named `scale-private-report-loader`.

The loader will:

1. Accept only `GET` requests for `scope=live` or `scope=history&date=YYYY-MM-DD`.
2. Read the `access` value supplied by the GitHub Pages shell.
3. Derive the expected Scale access token with the same existing HMAC scheme used by `publish-scale-html`, preserving current private URLs.
4. Reject missing or incorrect access values with HTTP 403.
5. Fetch the matching encrypted payload from `Report-Scale-Computing`.
6. Derive the AES-GCM decryption key from the validated access value.
7. Decrypt the payload server-side.
8. Return the plain report HTML with `Cache-Control: no-store` and a restrictive CORS policy allowing the Avani GitHub Pages origin.

The loader will never expose the GitHub publishing token or private access secret to the browser.

### Publisher changes

Update only the Scale `publish-scale-html` Edge Function for `target=report`.

Instead of embedding the ciphertext into `index.html`, publishing will:

1. Encrypt the generated report exactly once with AES-GCM.
2. Write the ciphertext and IV to `private-payload.json`.
3. Write the lightweight loader shell to `index.html`.
4. Do the same for the archive path.
5. Keep `history/history.json`, secure URLs, archive URLs, reporting dates, and returned `secureUrl/pagesUrl` behavior unchanged.

The builder publishing target remains unchanged.

### Legacy migration

The current live report and existing protected history snapshots use the older inline-encrypted wrapper. During rollout, a one-time migration will:

1. Read each existing inline protected wrapper.
2. Extract its IV and ciphertext without decrypting or altering the report contents.
3. Save those values into the new `private-payload.json` format.
4. Replace only the wrapper `index.html` with the lightweight shell.

This preserves existing client URLs and historical report contents while removing browser-side decryption immediately.

## Security

- The private URL format remains `?access=...`.
- The expected access token is validated server-side before any report HTML is returned.
- Payload files on GitHub Pages remain encrypted at rest/public transport.
- Invalid or plain links continue to display a private-report-required message.
- Responses from the loader use `Cache-Control: no-store`.
- No private token is written into report HTML, source summaries, logs, or chat output.

## Scope Constraints

- Scale Studio only.
- Do not modify Syndication Studio, its repositories, its private links, or its runtime.
- Do not modify Scale Preview rendering, Paid Media parsing, PDF fallback, PDF export, or workspace state.
- Do not change the client-facing report layout or report contents.
- Preserve Reporting History and current private-link URL shape.
- Preserve `Open private live report` and `Copy private link` behavior in Scale Studio.

## Failure Behavior

- Missing/invalid access: loader returns 403; shell displays `This report link is missing or invalid. Please use the private link sent by AVANI Media.`
- Missing payload/history date: loader returns 404; shell displays `Report unavailable.`
- Decryption/server failure: loader returns 500; shell displays a temporary report-unavailable message rather than hanging indefinitely.
- The shell must always leave `Verifying...` within a bounded timeout and show a visible error if the request cannot complete.

## Acceptance Criteria

1. The current Scale private live URL opens the client report without remaining on `Verifying your secure report link...`.
2. Plain `Report-Scale-Computing/` without `access` does not reveal report content.
3. An invalid access value does not reveal report content.
4. Existing current private URL continues to work after migration.
5. Reporting History entries open with the same access value.
6. A newly published report writes a small loader shell and a separate encrypted payload.
7. Scale Studio still receives and stores the returned private `secureUrl/pagesUrl`.
8. Preview, PDF fallback, Paid Media parser, PDF export, and client report layout remain unchanged.
9. No Syndication Studio repository, Edge Function, or live report is modified.
