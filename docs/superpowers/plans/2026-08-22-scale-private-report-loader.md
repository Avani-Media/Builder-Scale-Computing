# Scale Private Report Loader Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace Scale's browser-side multi-megabyte AES decrypt wrapper with a lightweight private shell plus a Scale-only server-side loader, while preserving current private URLs and report contents.

**Architecture:** `publish-scale-html` will keep encrypting report HTML but will write ciphertext to `private-payload.json` and a small loader shell to `index.html`. A new `scale-private-report-loader` Edge Function validates the existing `?access=` token, fetches and decrypts the payload server-side, and returns report HTML. A one-time migration converts the current live/history inline wrappers into shell + payload files without changing client URLs.

**Tech Stack:** Supabase Edge Functions (Deno/TypeScript), GitHub Contents API, GitHub Pages, Web Crypto AES-GCM/HMAC-SHA256.

**Spec:** `docs/superpowers/specs/2026-08-22-scale-private-report-loader-design.md`

## Global Constraints

- Scale Studio only.
- Do not modify Syndication Studio, its repositories, its private links, or its runtime.
- Do not modify Scale Preview rendering, Paid Media parsing, PDF fallback, PDF export, or workspace state.
- Do not change the client-facing report layout or report contents.
- Preserve Reporting History and current private-link URL shape.
- Preserve `Open private live report` and `Copy private link` behavior in Scale Studio.
- Never expose access tokens or GitHub secrets in logs, responses, or user-facing output.

---

### Task 1: Persist approved design and plan

**Files:**
- Create: `docs/superpowers/specs/2026-08-22-scale-private-report-loader-design.md`
- Create: `docs/superpowers/plans/2026-08-22-scale-private-report-loader.md`

**Interfaces:**
- Consumes: approved design from the current task.
- Produces: reviewable architecture and execution checklist on branch `scale-private-report-loader-20260822`.

- [ ] **Step 1: Write both documents to the feature branch.**
- [ ] **Step 2: Verify the files exist and contain the scope constraints and acceptance criteria.**
- [ ] **Step 3: Confirm no `index.html` or Syndication files are changed by this documentation commit.**

### Task 2: Add Scale-only server-side private loader

**Files:**
- Create/Deploy: Supabase Edge Function `scale-private-report-loader/index.ts`

**Interfaces:**
- Consumes: `scope=live|history`, optional `date=YYYY-MM-DD`, and `access=<token>`.
- Produces: decrypted report HTML for valid access; 403/404/500 response for invalid requests.

- [ ] **Step 1: Write a failing probe against the current system showing there is no `scale-private-report-loader` endpoint and that the current live wrapper contains inline ciphertext.**
- [ ] **Step 2: Deploy `scale-private-report-loader` with exact HMAC derivation compatibility with `publish-scale-html`.**
- [ ] **Step 3: Implement constant-time access comparison, payload path selection, GitHub raw payload fetch, AES-GCM decrypt, `Cache-Control: no-store`, and restricted CORS.**
- [ ] **Step 4: Verify a valid derived access token decrypts the current extracted payload fixture while missing/invalid access returns 403.**

### Task 3: Update Scale publisher to shell + payload output

**Files:**
- Modify/Deploy: Supabase Edge Function `publish-scale-html/index.ts`

**Interfaces:**
- Consumes: existing `target=report` HTML publish request.
- Produces: unchanged response shape (`secureUrl`, `pagesUrl`, archive URL, history metadata), but GitHub Pages stores loader shell + encrypted payload separately.

- [ ] **Step 1: Write a failing source-level test asserting the current publisher still embeds ciphertext directly in `index.html`.**
- [ ] **Step 2: Add `encryptPayload(html,access)` returning `{schemaVersion,iv,ciphertext}`.**
- [ ] **Step 3: Add `privateLoaderShell(scope,date?)` with access forwarding, abort timeout, document replacement, and visible failure messages.**
- [ ] **Step 4: Change live and archive publishing to write `private-payload.json` plus shell `index.html`.**
- [ ] **Step 5: Keep builder target behavior, secure URL generation, history index generation, and private access derivation unchanged.**
- [ ] **Step 6: Deploy and verify function syntax plus response-shape parity.**

### Task 4: Migrate current live report and history wrappers

**Files:**
- Modify/Create only under `Report-Scale-Computing`: live/history shell and payload files.

- [ ] **Step 1: Read current live/history wrappers and extract only IV/ciphertext from wrappers carrying `data-avani-scale-private-report-v1="1"`.**
- [ ] **Step 2: Write payload JSON alongside each migrated wrapper.**
- [ ] **Step 3: Replace wrapper HTML with shell configured for live/history scope.**
- [ ] **Step 4: Leave `history/history.json` content unchanged during migration.**

### Task 5: End-to-end verification and cleanup

- [ ] **Step 1: Verify current private Scale URL resolves through the new loader.**
- [ ] **Step 2: Verify no-access and invalid-access cases do not reveal report content.**
- [ ] **Step 3: Verify at least one Reporting History snapshot opens with the same access value.**
- [ ] **Step 4: Verify live `index.html` is small and contains no inline ciphertext; payload remains encrypted.**
- [ ] **Step 5: Verify publisher still returns private URLs and builder target is unchanged.**
- [ ] **Step 6: Verify Scale builder Preview/PDF/parser/private-link markers are unchanged.**
- [ ] **Step 7: Verify no Syndication Studio repository/function was modified.**
- [ ] **Step 8: Neutralize one-time probe/migration functions and remove temporary database extensions.**
