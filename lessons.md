# lessons.md — SIGN + CENVOX Platform
> This file documents every bug, issue, and fix that took significant time to resolve.
> Feed this file to Claude at the start of every session to avoid repeating mistakes.
> Last updated: 2026-04-26

---

## How to Use This File
- Read this file at the start of every Claude session
- Add new lessons after every bug that took more than 30 minutes to fix
- Format: Problem → Root Cause → Fix → How to Avoid

---

## 📋 Table of Contents
1. Clause Extraction — max_tokens too small
2. Clause Extraction — Wrong Arabic clause patterns
3. Clause Extraction — Sub-articles split as separate clauses
4. Clause Extraction — Cover page trimming too aggressive
5. Clause Extraction — Large documents timing out (chunking solution)
6. Clause Extraction — Tiny orphan chunks crashing extraction
7. Clause Extraction — Chunk boundaries splitting articles in half
8. Clause Extraction — TOC entries extracted as real clauses
9. Clause Extraction — Cross-references mistaken as clause headings
10. Clause Extraction — Duplicate clauses from overlapping chunks
11. Clause Extraction — API costs too high (max_tokens fix)
12. Docker — bcrypt binary fails on Windows
13. Docker — CRLF line endings crash entrypoint on Linux
14. Docker — Celery worker code changes not taking effect
15. Docker — Concurrency set to 1 (all documents process sequentially)
16. Docker — Celery tasks killed at 600s (large documents always fail)
17. Word File Extraction — Tables and headers missing
18. Database — Orphaned clauses inflating dashboard count
19. Frontend — currentUser always null after page refresh
20. Frontend — Arabic text displaying left-to-right
21. Frontend — Comment edit/delete icons not showing
22. GitHub — Personal access token expiring
23. GitHub — Mac osxkeychain interfering with token

---

## 1. Clause Extraction — max_tokens Too Small

**Problem:**
- Clause extraction was returning 0 clauses
- No error shown in UI
- JSON responses were being cut off mid-string

**Root Cause:**
- `max_tokens` was hardcoded to `8192` in `clause_extractor.py`
- Arabic contracts with 20+ clauses require 15,000–25,000 output tokens
- Claude was generating correct JSON but hitting the token limit mid-response
- `json.loads()` then threw `JSONDecodeError` on the truncated response

**Fix:**
- Implemented tiered `_calculate_max_tokens()` method:
  - Chunks < 10,000 chars → 16,000 tokens
  - Chunks < 20,000 chars → 24,000 tokens
  - Chunks >= 20,000 chars → 32,000 tokens
- This saves 50-75% vs old hardcoded 64,000

**File:** `ai-backend/app/agents/clause_extractor.py`

**How to Avoid:**
- Never hardcode `max_tokens` for Arabic content
- Arabic JSON output is larger than estimated — always add 50% buffer
- Always check celery logs for JSONDecodeError when clauses = 0

---

## 2. Clause Extraction — Wrong Arabic Clause Patterns

**Problem:**
- Particular Conditions clauses not being recognized
- Extraction returning 0 clauses even with correct text
- The document used `مادة (1)` format but prompt only handled `المادة (1)`

**Root Cause:**
- SYSTEM_PROMPT only listed: `"المادة 1"`, `"المادة (1)"`, `"مادة رقم"`
- Missing the most common Egyptian government contract format: `مادة (1)` (without ال article)
- Also missing: `مادة (١)` Arabic-Indic numerals, `مادة رقم (١)`, `مادة 1` without brackets

**Fix:**
- Added all مادة variations to Guideline 8 in SYSTEM_PROMPT:
  - `مادة (1)` / `مادة (١)` — Western or Arabic-Indic in brackets
  - `مادة رقم (1)` / `مادة رقم (١)` — with رقم
  - `مادة 1` / `مادة ١` — no brackets
  - `المادة 1` / `المادة (1)` / `المادة رقم` — with definite article

**File:** `ai-backend/app/agents/clause_extractor.py`

**How to Avoid:**
- Egyptian government contracts use `البند رقم (N):` format
- Particular Conditions use `مادة (N)` format
- General Conditions use `مادة (N) : title :` format
- Always check actual document text before assuming clause marker format
- Run: `SELECT LEFT(extracted_text, 500) FROM document_uploads WHERE ...`

---

## 3. Clause Extraction — Sub-articles Split as Separate Clauses

**Problem:**
- Particular Conditions: 16 clauses extracted but structure was wrong
- Sub-articles like `9-1`, `9-2`, `9-3` were extracted as separate top-level clauses
- مادة 12 with sub-articles 12-1 through 12-10 created 10 separate clauses instead of 1

**Root Cause:**
- SYSTEM_PROMPT Guideline 3 said "each clause should be atomic" with no exception for sub-articles
- Claude treated `9-2`, `9-3` as "separate" and made them atomic
- No rule existed to explain that N-M and N/M patterns are sub-articles not top-level clauses

**Fix:**
- Updated Guideline 3: Added explicit sub-article exception
- Added Guideline 10: "Everything from مادة (N) to مادة (N+1) = ONE clause"
- Sub-articles numbered N-M (dash) or N/M (slash) must stay inside parent مادة
- `clause_number` should always be the parent number, never the sub-article number

**File:** `ai-backend/app/agents/clause_extractor.py`

**How to Avoid:**
- Always verify sub-article containment after first extraction
- Check DB: `SELECT clause_number, title FROM clauses WHERE source_document_id = '...'`
- If you see clause_number like "9-2" or "4/1" — sub-article rule is broken

---

## 4. Clause Extraction — Cover Page Trimming Too Aggressive

**Problem:**
- Particular Conditions extraction was missing Articles 1-8 completely
- First extracted clause had no section number and started mid-article
- General Conditions started from wrong position

**Root Cause:**
- `trimCoverPages()` in `document-processing.service.ts` matched `تم الاتفاق` pattern
- This phrase appeared inside a sub-article body (not just on the cover page)
- The function cut everything before that match — deleting Articles 1-8

**Fix:**
- For Conditions documents (Particular/General): now searches for FIRST `مادة` marker
- Pattern: `/مادة\s*[\(\s]?[١-٩\d]/`
- For Agreement documents: kept existing behavior (searches for `تم الاتفاق` first)
- Document type detected by label: conditions/شروط/general/particular/spec/مواصفات

**File:** `backend/src/modules/document-processing/document-processing.service.ts`

**How to Avoid:**
- After reprocessing always verify: `SELECT LEFT(extracted_text, 200) FROM document_uploads WHERE ...`
- Text should start at the first real clause marker not mid-sentence
- Never use broad Arabic phrases as cover page markers — they appear in body text too

---

## 5. Clause Extraction — Large Documents Timing Out

**Problem:**
- General Conditions (81,315 chars) always timed out
- Task was killed at exactly 600 seconds every time
- Retry loop: killed → NestJS retried → killed again → infinite loop

**Root Cause:**
- Celery worker `--concurrency=1` — only one document at a time
- Hard time limit was 600s (10 minutes) — not enough for large Arabic documents
- Claude needs 8-10 minutes just for the API call on 70k+ char documents
- The solution was chunking: split document into smaller pieces

**Fix — Chunking Implementation:**
- Documents > 30,000 chars → split into chunks at مادة boundaries
- Chunk size: 15,000 chars maximum
- Each chunk processed by separate Claude API call
- Results merged and deduplicated after all chunks complete
- Methods added: `_extract_chunked()`, `_split_on_article_boundaries()`, `_merge_small_chunks()`

**Fix — Docker Settings:**
- `--concurrency=1` → `--concurrency=3` (3 docs in parallel)
- `--time-limit=2400` (40 minutes)
- `--soft-time-limit=1800` (30 minutes)
- Memory: 2G → 3G

**Files:**
- `ai-backend/app/agents/clause_extractor.py`
- `docker-compose.yml`

**How to Avoid:**
- Any document over 30,000 chars MUST use chunking
- Never send full large Arabic document in one API call
- Check document size: `SELECT length(extracted_text) FROM document_uploads WHERE ...`

---

## 6. Clause Extraction — Tiny Orphan Chunks Crashing Extraction

**Problem:**
- After implementing chunking, extraction kept failing on chunk 3
- Error: `Expecting value: line 1 column 1 (char 0)` — empty JSON response
- A 202-character chunk was being sent to Claude
- Claude returned prose instead of JSON for tiny fragments

**Root Cause:**
- `_split_on_article_boundaries()` was creating tiny leftover fragments
- When splitting large articles, small text pieces (202 chars) were left as separate chunks
- Claude cannot extract clauses from 202 chars and returned empty/prose response
- Empty response caused `json.loads()` to crash the entire task

**Fix:**
- Added `_merge_small_chunks()` method
- Any chunk < 500 chars gets merged into the previous chunk
- Min chunk skip threshold: 500 chars as extra safety
- Graceful error handling: if one chunk fails, log warning and continue with next chunk

**File:** `ai-backend/app/agents/clause_extractor.py`

**How to Avoid:**
- Always run `_merge_small_chunks()` after splitting
- Log chunk sizes when extraction starts — check for any chunk < 500 chars
- If you see "Expecting value: line 1 column 1" — a tiny chunk is the cause

---

## 7. Clause Extraction — Chunk Boundaries Splitting Articles in Half

**Problem:**
- General Conditions extracted 56 clauses instead of 38
- Sub-articles of مادة 12 were extracted as separate top-level clauses
- Some clause content started mid-sentence

**Root Cause:**
- Chunking split at 15,000 char boundaries
- Sometimes a chunk ended in the MIDDLE of a large article (like مادة 12)
- The next chunk started mid-article with no مادة header
- Claude saw the continuation and extracted it as new clauses

**Fix:**
- Added `_add_article_context()` method
- If a chunk doesn't start with a مادة marker → prepend the last مادة heading from previous chunk
- Added explicit instruction to Claude prompt for each chunk:
  "Only extract clauses that START in this chunk. Do NOT extract content that is a continuation of a clause that started before this chunk."

**File:** `ai-backend/app/agents/clause_extractor.py`

**How to Avoid:**
- After chunked extraction always verify total clause count matches expected
- If clause count is higher than expected → chunk boundary issue
- Check: `SELECT clause_number, title FROM clauses WHERE source_document_id = '...' ORDER BY clause_number`
- Look for clause_number like "9-2" or clauses with no section number

---

## 8. Clause Extraction — TOC Entries Extracted as Real Clauses

**Problem:**
- General Conditions document has a Table of Contents appended at the end
- TOC contains entries like: `مادة (1) : تعريفات وتفسيرات ......... 4`
- Claude was extracting these TOC entries as real clauses (creating duplicates)

**Root Cause:**
- TOC entries look identical to real article headings syntactically
- Original Guideline 7 said "skip table of contents" but wasn't specific enough
- Claude couldn't reliably distinguish TOC from real articles

**Fix:**
- Strengthened Guideline 7 with specific TOC identification rules:
  - TOC identified by: مادة (N) followed by dotted lines (.......)
  - Standalone page numbers on their own lines
  - Multiple مادة entries listed sequentially with NO body text between them
  - ALL such entries must be completely skipped

**File:** `ai-backend/app/agents/clause_extractor.py`

**How to Avoid:**
- Word files often append the TOC table at the END of extracted text
- Always verify extracted text end: `SELECT RIGHT(extracted_text, 500) FROM document_uploads WHERE ...`
- If you see dotted lines and page numbers at the end → TOC is appended

---

## 9. Clause Extraction — Cross-References Mistaken as Clause Headings

**Problem:**
- General Conditions body text contains phrases like:
  `طبقا للمادة (22) من هذه الشروط العامة`
- These look identical to real clause headings
- Claude was creating phantom clauses from cross-references

**Root Cause:**
- `مادة (N)` appears both as real headings AND as cross-references in body text
- No rule existed to distinguish them
- Real heading: `مادة (12) :` at start of line followed by title
- Cross-reference: `مادة (22)` mid-sentence after من/طبقا/بموجب

**Fix:**
- Added Guideline 11: Real article headings vs cross-references
- Real heading: مادة (N) at START of line + colon + title + body text follows
- Cross-reference: مادة (N) mid-sentence preceded by من/طبقا للمادة/بموجب مادة/أحكام مادة/وفقاً للمادة
- Cross-references are NEVER clause boundaries

**File:** `ai-backend/app/agents/clause_extractor.py`

**How to Avoid:**
- Any document with frequent article cross-references needs this guideline
- FIDIC-based Arabic contracts are especially prone to this issue
- Check phantom clauses: clauses with no body text or very short content (< 50 chars)

---

## 10. Clause Extraction — Duplicate Clauses from Overlapping Chunks

**Problem:**
- After chunked extraction, duplicate clauses appeared in the database
- Same clause extracted by two adjacent chunks

**Root Cause:**
- When a chunk boundary fell near a مادة heading, both the previous chunk (end) and next chunk (start) extracted the same clause
- No deduplication was happening after merging all chunk results

**Fix:**
- Added deduplication by normalized title after merging all chunks
- `seen_titles` set tracks extracted clause titles
- First occurrence kept, subsequent duplicates silently dropped
- Also added deduplication by clause_number

**File:** `ai-backend/app/agents/clause_extractor.py`

**How to Avoid:**
- Always run deduplication after merging chunk results
- Check for duplicates: `SELECT clause_number, COUNT(*) FROM clauses WHERE source_document_id = '...' GROUP BY clause_number HAVING COUNT(*) > 1`
- If duplicates found: `DELETE FROM clauses WHERE id NOT IN (SELECT MIN(id) FROM clauses WHERE source_document_id = '...' GROUP BY clause_number)`

---

## 11. Clause Extraction — API Costs Too High

**Problem:**
- Anthropic API costs were much higher than expected
- `max_tokens=64000` was hardcoded — paying for 64k tokens even when only 5k used
- With 9 chunks per document × 64,000 max_tokens = massive cost

**Root Cause:**
- Original fix for truncation set max_tokens to 64,000 globally
- Arabic contracts with 15k char chunks only need ~16,000-24,000 output tokens
- Paying for 64,000 but using maybe 10,000 = 6x more expensive than needed

**Fix:**
- Tiered max_tokens based on chunk size:
  - Chunk < 10,000 chars → max_tokens = 16,000 (saves 75%)
  - Chunk < 20,000 chars → max_tokens = 24,000 (saves 62%)
  - Chunk >= 20,000 chars → max_tokens = 32,000 (saves 50%)

**File:** `ai-backend/app/agents/clause_extractor.py`

**How to Avoid:**
- Never hardcode max_tokens to maximum value
- Calculate based on input size: input_chars / 4 * 1.5 safety buffer
- Monitor Anthropic API usage at console.anthropic.com after each extraction

---

## 12. Docker — bcrypt Binary Fails on Windows

**Problem:**
- Backend container crashed on startup after `docker-compose up`
- Error: `Error loading shared library` or `invalid ELF header` for bcrypt
- Only happened on Windows machines

**Root Cause:**
- bcrypt is a native Node.js module compiled for the host OS
- When `node_modules` is volume-mounted from Windows into Linux container
- The Windows-compiled bcrypt binary is incompatible with Linux

**Fix:**
- Added `docker-entrypoint.sh` that runs `npm rebuild bcrypt` at container startup
- This recompiles bcrypt for the Linux container environment
- Added to `backend/Dockerfile`: `sed -i 's/\r//'` to strip CRLF from entrypoint

**Files:**
- `backend/docker-entrypoint.sh`
- `backend/Dockerfile`

**How to Avoid:**
- Any native Node.js module (bcrypt, sharp, canvas) will have this issue on Windows
- Always run `npm rebuild <module>` in the container, not on the host
- Add to entrypoint script, not Dockerfile build step

---

## 13. Docker — CRLF Line Endings Crash Entrypoint on Linux

**Problem:**
- Backend container crashed with: `/usr/bin/env: 'bash\r': No such file or directory`
- Only happened after pulling from Windows-developed code

**Root Cause:**
- Windows uses CRLF (`\r\n`) line endings
- Linux expects LF (`\n`) only
- Git on Windows sometimes commits CRLF even with autocrlf settings
- The `\r` in bash scripts causes Linux to look for `bash\r` instead of `bash`

**Fix:**
- Added to `backend/Dockerfile` build step:
  `sed -i 's/\r//' /app/docker-entrypoint.sh`
- This strips all carriage returns at build time
- Permanent fix — works regardless of how the file was committed

**File:** `backend/Dockerfile`

**How to Avoid:**
- Always add `sed -i 's/\r//'` in Dockerfile for any shell scripts
- Configure Git: `git config --global core.autocrlf input`
- Check for CRLF: `cat -A docker-entrypoint.sh | grep '\^M'`

---

## 14. Docker — Celery Worker Code Changes Not Taking Effect

**Problem:**
- Modified `tasks.py` or `clause_extractor.py` but celery worker kept using old code
- Restarting the container didn't help
- Changes were invisible to the running worker

**Root Cause:**
- The celery-worker container in `docker-compose.yml` only had:
  `volumes: - uploads_data:/app/uploads`
- It did NOT have the source code mounted: `./ai-backend:/app`
- The worker was using code baked into the Docker image at build time
- Only `docker-compose up --build` would pick up changes

**Fix:**
- Added source code volume mount to celery-worker in `docker-compose.yml`:
  `- ./ai-backend:/app`
- Now changes to any Python file are immediately visible to the worker
- Only need `docker restart sign-celery-worker` not full rebuild

**File:** `docker-compose.yml`

**How to Avoid:**
- Always check if the container has a source code volume mount
- Without `./ai-backend:/app` mount → must rebuild image for every code change
- With mount → just restart the container

---

## 15. Docker — Concurrency Set to 1 (Sequential Processing)

**Problem:**
- Uploading 3 documents → total processing time 20+ minutes
- Second document couldn't start until first finished
- Third document couldn't start until second finished

**Root Cause:**
- `docker-compose.yml` had: `command: celery -A app.tasks worker --concurrency=1`
- Only ONE document could be processed at a time
- With 3 documents: 0 + 4min + 4min + 4min = sequential wait

**Fix:**
- Changed to `--concurrency=3`
- All 3 documents now process in parallel
- Total time: max(doc1, doc2, doc3) instead of sum
- Also raised memory limit from 2G to 3G to support 3 parallel workers

**File:** `docker-compose.yml`

**How to Avoid:**
- Always set concurrency to number of expected parallel documents
- Monitor with: `docker exec sign-celery-worker celery inspect stats`
- Check `"max-concurrency"` and `"processes"` in output

---

## 16. Docker — Celery Tasks Killed at 600s

**Problem:**
- Large documents (General Conditions — 81k chars) always failed
- Task killed at exactly 600 seconds every time
- NestJS would auto-retry → killed again → infinite retry loop

**Root Cause:**
- Default Celery hard time limit was 600 seconds (10 minutes)
- Large Arabic documents need 15-20 minutes for Claude API call
- Worker process received SIGKILL at exactly 600s

**Fix:**
- Added to docker-compose command:
  `--time-limit=2400 --soft-time-limit=1800`
- Soft limit (30 min): graceful warning, task can clean up
- Hard limit (40 min): absolute kill — never reached in practice

**File:** `docker-compose.yml`

**How to Avoid:**
- Always check document size before setting time limits
- General Conditions (81k chars) needs ~20 minutes
- Formula: chars / 4000 * 1.5 = estimated minutes needed
- Set hard limit to 2x estimated time

---

## 17. Word File Extraction — Tables and Headers Missing

**Problem:**
- Word files with contract terms in tables were missing those terms
- Extracted text had large gaps compared to actual document content
- Clause extraction missed payment schedules and liability clauses

**Root Cause:**
- `_extract_docx()` in `text_extractor.py` only read `doc.paragraphs`
- Text inside table cells is NOT in `doc.paragraphs`
- Headers and footers are also NOT in `doc.paragraphs`
- Arabic contracts frequently put payment terms and liability clauses in tables

**Fix:**
- Added table cell extraction:
  ```python
  for table in doc.tables:
      for row in table.rows:
          for cell in row.cells:
              if cell.text.strip():
                  paragraphs.append(cell.text.strip())
  ```
- Added header extraction from `doc.sections`

**File:** `ai-backend/app/services/text_extractor.py`

**How to Avoid:**
- Always extract from paragraphs + tables + headers for Word files
- Verify extraction quality: `SELECT length(extracted_text) FROM document_uploads WHERE ...`
- A 3MB Word file should yield at least 50,000+ chars if tables are extracted

---

## 18. Database — Orphaned Clauses Inflating Dashboard Count

**Problem:**
- Dashboard showed 179 clauses but only 85 were in the active project
- Clause count was wrong and confusing

**Root Cause:**
- Multiple failed/killed extraction runs had saved SOME clauses to `clauses` table
- But the pipeline never completed so `contract_clauses` junction table entries were never created
- Dashboard query counted ALL rows in `clauses` table instead of only linked ones
- 94 "ghost clauses" had no `source_document_id` and no `contract_clauses` entry

**Fix:**
- Deleted orphaned clauses:
  ```sql
  DELETE FROM clauses WHERE id NOT IN (
    SELECT DISTINCT clause_id FROM contract_clauses
  );
  ```
- Fixed dashboard query to use `innerJoin` on `contract_clauses` table
- Now only counts clauses properly linked to contracts

**Files:**
- `backend/src/modules/dashboard-analytics/dashboard-analytics.service.ts`

**How to Avoid:**
- After any killed/failed extraction run → check for orphaned clauses
- Check: `SELECT COUNT(*) FROM clauses WHERE id NOT IN (SELECT clause_id FROM contract_clauses)`
- If count > 0 → run the DELETE query above
- Also: if dashboard numbers look wrong → first check for orphaned clauses

---

## 19. Frontend — currentUser Always Null After Page Refresh

**Problem:**
- After page refresh, user appeared logged in (token in localStorage)
- But `currentUser` in Redux was always `null`
- All permission checks failed (comment edit/delete, role checks, admin detection)

**Root Cause:**
- Redux auth slice restores `token` from localStorage on startup
- But `user` object (with id, role, name) is NOT restored from localStorage
- `refreshUserProfile()` was only called in `MfaSetupPage` — never on app startup
- So after refresh: `isAuthenticated = true` but `currentUser = null` permanently

**Fix:**
- Added `useEffect` in `AppLayout.tsx` and `AdminLayout.tsx`:
  ```javascript
  useEffect(() => {
    if (isAuthenticated && !user) {
      refreshUserProfile();
    }
  }, []);
  ```
- This fetches user profile from API on mount if token exists but user is null

**Files:**
- `apps/sign/src/components/layout/AppLayout.tsx`
- `apps/sign/src/components/layout/AdminLayout.tsx`

**How to Avoid:**
- Any feature using `currentUser?.id` or `currentUser?.role` depends on this fix
- Always test permission features after page refresh, not just after fresh login
- If permission checks fail → first check if `currentUser` is null in Redux DevTools

---

## 20. Frontend — Arabic Text Displaying Left-to-Right

**Problem:**
- Extracted Arabic clauses displayed with text aligned to LEFT
- Should be RIGHT aligned (RTL direction)
- Made Arabic content very hard to read

**Root Cause:**
- Clause title and content elements had no RTL direction specified
- Browser defaulted to LTR for all text
- `dir="auto"` was not applied to text containers

**Fix:**
- Added `dir="auto"` and `style={{ unicodeBidi: 'plaintext' }}` to:
  - Clause titles (`<h4>`)
  - Clause content (`<p>`)
  - Edit textarea and input
  - All clause display elements in ContractDetailPage
- `dir="auto"` makes browser detect direction from first strong character
- Arabic text → right aligned automatically
- English text → left aligned automatically

**Files:**
- `apps/sign/src/components/review/ClauseReviewCard.tsx`
- `apps/sign/src/pages/app/ContractDetailPage.tsx`

**How to Avoid:**
- Always add `dir="auto"` to any element that may display Arabic text
- Never assume all text is LTR
- Test with actual Arabic content after any UI change to clause display

---

## 21. Frontend — Comment Edit/Delete Icons Not Showing

**Problem:**
- Pencil and trash icons were in the code but never visible
- No error shown — icons simply didn't appear on hover

**Root Cause (Two Issues):**

**Issue 1:** `currentUser` was null (see lesson #19)
- `isAuthor = comment.user_id === currentUser?.id` → always false when currentUser is null
- `canEdit` and `canDelete` both false → elements not rendered at all
- `opacity-0 group-hover:opacity-100` only helps if elements exist in DOM

**Issue 2:** AdminLayout logout race condition
- `handleLogout` was not async
- `logout()` started (async API call)
- `navigate('/auth/login')` fired immediately before logout completed
- Redux state still had user → LoginPage saw authenticated user → redirect loop

**Fix:**
- Fixed `currentUser` null issue (lesson #19)
- Made `handleLogout` async/await:
  ```javascript
  const handleLogout = async () => {
    await logout();
    navigate('/auth/login', { replace: true });
  };
  ```

**Files:**
- `apps/sign/src/components/layout/AdminLayout.tsx`
- `apps/sign/src/components/layout/AppLayout.tsx`

**How to Avoid:**
- Always make logout handlers async/await
- Test icon visibility after page refresh, not just after fresh login
- If icons not showing → check currentUser in Redux DevTools first

---

## 22. GitHub — Personal Access Token Expiring

**Problem:**
- Push failed with 403 Permission denied
- Colleague couldn't pull latest updates
- Token had expired without notice (well, email was sent but missed)

**Root Cause:**
- GitHub PAT (Personal Access Token) was created with expiry date
- Token expired → all push/pull operations fail with 403

**Fix:**
- Generate new classic PAT at: https://github.com/settings/tokens
- Must use: "Generate new token (classic)" not fine-grained
- Must check: `repo` scope (full control)
- Set expiration to: "No expiration" or 90 days
- Token starts with `ghp_` — fine-grained tokens start with `github_pat_`
- Update remote: `git remote set-url origin https://TOKEN@github.com/USER/REPO.git`

**How to Avoid:**
- Always set token expiration to "No expiration" for development
- Save token immediately in a password manager — GitHub shows it only once
- Never share token in chat, email, or screenshots
- If you see `github_pat_` prefix → you created wrong token type (fine-grained)
- Better long-term solution: use GitHub CLI (`gh auth login`) — no token management needed

---

## 23. GitHub — Mac osxkeychain Interfering with Token

**Problem:**
- Colleague on Mac couldn't push/pull even with correct token
- Error: "Device not configured" or "Invalid username or token"
- Token was correct but Mac keychain was overriding it with old cached credentials

**Root Cause:**
- Mac stores GitHub credentials in osxkeychain
- Old expired credentials were cached in keychain
- New token in remote URL was being ignored in favor of cached credentials
- The `credential.helper = osxkeychain` config intercepts all git operations

**Fix:**
- Clear cached credentials from Mac keychain:
  ```
  git credential-osxkeychain erase
  host=github.com
  protocol=https
  (press Enter twice)
  ```
- Set credential helper to store:
  `git config --global credential.helper store`
- Update remote URL with new token
- Better solution: Use GitHub CLI `gh auth login` (handles credentials automatically)

**How to Avoid:**
- Mac developers should use GitHub CLI instead of token URLs
- `gh auth login` → login via browser → no token management needed
- Never store token in remote URL on Mac (keychain will interfere)
- For Windows: token in remote URL works fine

---

## 📝 Template for New Lessons

```
## N. Category — Short Description

**Problem:**
- What symptom did you see?
- What was the error message?

**Root Cause:**
- Why did it happen?
- What was the underlying issue?

**Fix:**
- What exact change fixed it?
- Code snippet if applicable

**File:** which file was changed

**How to Avoid:**
- How to detect this early
- What to check first next time
```

---

*Last updated: 2026-04-26*
*Feed this file to Claude at the start of every new session*
