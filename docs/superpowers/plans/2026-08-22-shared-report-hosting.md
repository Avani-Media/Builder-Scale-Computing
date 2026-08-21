# Shared AVANI Report Hosting Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox syntax for tracking.

**Goal:** Host Syndication and Scale client-report shells from the same GitHub Pages repository and custom domain without changing the existing Syndication URL or coupling their publishing logic.

**Architecture:** Keep the existing Syndication shell at the root of `avani-media/Syndication-Report`, preserving `reports.avanimedia.com`. Add Scale under `/scale/` in the same repository, with its own lightweight private shell, encrypted payload, and reporting history. Update only the Scale publisher and Scale private loader to target the shared repository subfolder. Keep `Report-Scale-Computing` unchanged as rollback.

**Tech Stack:** GitHub Pages, GitHub Contents API, Supabase Edge Functions, AES-GCM, custom domain via GitHub Pages CNAME.

**Spec:** Approved in chat on 2026-08-22: one repository, separate shells, separate publisher/data logic, preserve Syndication and rollback paths.

## Global Constraints

- Do not move, rename, or modify the existing Syndication root `index.html`.
- Preserve the existing root `CNAME` value `reports.avanimedia.com`.
- Scale files live only below `scale/`.
- Scale Reporting History lives only below `scale/history/`.
- Scale publisher writes only Scale paths in `Syndication-Report`.
- Syndication publishing/runtime logic is not changed.
- Keep `Report-Scale-Computing` intact as rollback.
- Preserve Scale private access behavior and access-token derivation.
- New Scale live URL is `https://reports.avanimedia.com/scale/?access=...`.
- Existing Scale builder, Preview, PDF parsing, PDF export, and Paid Media logic are not changed.

---

### Task 1: Add Scale hosting subtree without touching Syndication root

**Files:**
- Create in `avani-media/Syndication-Report`: `scale/index.html`
- Create in `avani-media/Syndication-Report`: `scale/private-payload.json`
- Create/copy in `avani-media/Syndication-Report`: `scale/history/history.json`
- Create/copy historical `scale/history/YYYY-MM-DD/index.html` and `private-payload.json` files.

**Interfaces:**
- Consumes current Scale shell/payload/history from `Report-Scale-Computing`.
- Produces a byte-equivalent Scale private hosting subtree under the shared repo.

- [ ] Step 1: Record SHA/content checks for root `CNAME` and root `index.html` before migration.
- [ ] Step 2: Run a RED check proving `https://reports.avanimedia.com/scale/` is absent before migration.
- [ ] Step 3: Copy the current Scale live shell and encrypted payload into `scale/` using GitHub API writes.
- [ ] Step 4: Copy the current Scale history index and all current history snapshot shell/payload pairs under `scale/history/`.
- [ ] Step 5: Verify the Syndication root `CNAME` and root `index.html` SHA/content are unchanged.

### Task 2: Retarget Scale private loader to the shared hosting repo

**Files:**
- Modify Supabase Edge Function `scale-private-report-loader`.

**Interfaces:**
- Consumes `scope=live` or `scope=history&date=YYYY-MM-DD`, plus `access`.
- Reads encrypted payloads from `avani-media/Syndication-Report/scale/...`.
- Returns decrypted report HTML only after existing access validation passes.

- [ ] Step 1: Run a RED test against a staging loader configuration proving the old repo is still the source.
- [ ] Step 2: Change only repository/path constants to `Syndication-Report` + `scale/` prefixes.
- [ ] Step 3: Verify valid access returns the full live Scale report from the shared repo.
- [ ] Step 4: Verify invalid and missing access remain HTTP 403.
- [ ] Step 5: Verify at least the latest history date returns the full Scale historical report.

### Task 3: Retarget future Scale publishing

**Files:**
- Modify Supabase Edge Function `publish-scale-html`.

**Interfaces:**
- Builder target remains `Builder-Scale-Computing/index.html`.
- Report target becomes `Syndication-Report/scale/index.html` with Pages base `https://reports.avanimedia.com/scale/`.
- Archive base becomes `scale/history/YYYY-MM-DD/`.
- History index becomes `scale/history/history.json`.

- [ ] Step 1: Run a source-level RED test showing report target still points to `Report-Scale-Computing`.
- [ ] Step 2: Update report target repository, path, history-index path, and custom-domain Pages URL only.
- [ ] Step 3: Verify publisher status reports the new Scale custom-domain target and private-loader-v2 remains enabled.
- [ ] Step 4: Verify builder publishing target is unchanged.
- [ ] Step 5: Verify publisher still returns `secureUrl/pagesUrl` with an `access` query parameter.

### Task 4: End-to-end custom-domain verification

**Files:** None beyond Tasks 1-3.

**Interfaces:** Client-visible behavior.

- [ ] Step 1: Verify `https://reports.avanimedia.com/` still serves the original Syndication shell byte-for-byte.
- [ ] Step 2: Verify `https://reports.avanimedia.com/scale/` without access does not reveal Scale report content.
- [ ] Step 3: Verify the current valid Scale access value opens the full Scale report at the new custom-domain path.
- [ ] Step 4: Verify latest Scale Reporting History opens under `/scale/history/...` with the same access value.
- [ ] Step 5: Verify `Report-Scale-Computing` remains unchanged and functional as rollback.
- [ ] Step 6: Verify no Syndication Edge Function or runtime was modified.

### Task 5: Cleanup temporary migration/probe helpers

**Files:** Temporary Edge Functions/extensions only.

- [ ] Step 1: Neutralize one-time migration/probe Edge Functions so they cannot rewrite repositories later.
- [ ] Step 2: Remove temporary database HTTP extensions if they are no longer required by production workflows.
- [ ] Step 3: Run one final live verification of Syndication root and Scale custom-domain path.
