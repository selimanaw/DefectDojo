# 3-Minute Video Demonstration Script

**Project:** DefectDojo Modernization — AngelHack IBM Bob Hackathon 2026  
**Fork:** [selimanaw/DefectDojo](https://github.com/selimanaw/DefectDojo) · branch `dev`  
**Total runtime:** 3:00 minutes  
**Recommended recording tool:** OBS Studio, Loom, or Windows Game Bar (Win+G)  
**Screen layout:** Code editor (VS Code or similar) + terminal side by side, browser ready in background tab

---

## Before You Hit Record — Pre-flight Checklist

- [ ] `docker compose up -d` is running, app is accessible at `http://localhost:8080`
- [ ] Admin password retrieved from `docker compose logs initializer`
- [ ] VS Code open with files `dojo/finding/models.py` and `dojo/importers/base_importer.py` pre-loaded
- [ ] Terminal open in the `DefectDojo/` directory
- [ ] Browser tab open to `http://localhost:8080` and logged in as admin
- [ ] Browser tab open to `https://github.com/selimanaw/DefectDojo/pulls` (shows both merged PRs)
- [ ] Font size set large enough to be readable in the recording (16–18 pt in terminal, 14 pt in editor)
- [ ] `git log --oneline -5` run once to confirm output is ready to show

---

## Script

### [0:00 – 0:18] Opening — State the problem

**Screen:** Browser, GitHub repository page `selimanaw/DefectDojo`

> "DefectDojo is an OWASP Flagship security platform used by enterprises worldwide to track
> vulnerabilities. We found two bugs in its core workflow — one a silent data-correctness error,
> one a performance bottleneck — and fixed both. Here is what we changed, how we measured it,
> and how IBM Bob helped us do it."

**Action:** Switch to the Pull Requests tab, showing both closed and merged PRs:
- PR #1: `fix(dedup): fix bytes.strip() ordering in hash_fields()`
- PR #2: `perf(importer): optimize new finding persistence with bulk database operations`

---

### [0:18 – 0:48] Phase 1 — The bug in the hash function

**Screen:** VS Code, open `dojo/finding/models.py`, navigate to `compute_hash_code()`

> "DefectDojo assigns every vulnerability a hash code to identify duplicates across scans.
> The hash is computed from fields like CVE IDs and titles.
> The normalisation pipeline called `.casefold().encode('utf-8').strip()`.
> The problem: `bytes.strip()` only removes ASCII bytes 0x00 through 0x20.
> Unicode whitespace — a non-breaking space, an em space — passes through unchanged.
> Two findings for the same vulnerability, one title with a regular space and one with
> a non-breaking space, receive different hash codes and are never deduplicated.
> Silently. No error, no warning. The findings just accumulate."

**Action:** Highlight the line showing `.casefold().strip().encode("utf-8")` — the fixed version. Point to the comment in the code.

> "The fix is a one-line reordering: call `strip()` before encoding, while the value is still
> a Unicode string."

**Action:** Switch to terminal.

```bash
./run-unittest.sh --test-case unittests.test_deduplication_logic.TestHashFieldsNormalisation
```

> "Our regression tests verify this for ASCII whitespace, NBSP, EM SPACE, and THIN SPACE.
> All pass."

**Action:** Let the terminal output scroll to the final line: `OK (tests=7)` or similar.

---

### [0:48 – 1:12] Phase 1 — Diagnostic visibility

**Screen:** VS Code, open `dojo/finding/deduplication.py`, show the WARNING log

> "We also found that four diagnostic messages in the hash system were logging at DEBUG level —
> invisible in any default deployment. If DefectDojo cannot find the scanner's hash configuration,
> it silently falls back to a less accurate algorithm.
> We promoted all five of these to WARNING and added the scan type name to each message,
> so operators know exactly which scanner is degraded and can act on it."

**Action:** Show the warning log line:
```python
deduplicationLogger.warning(
    "No deduplication algorithm configured for scan type '%s'; falling back...",
    scan_type,
)
```

---

### [1:12 – 1:45] Phase 2 — The N+1 query problem

**Screen:** VS Code, show the diff of `dojo/importers/base_importer.py` using:

```bash
git show 3e172517e5 -- dojo/importers/base_importer.py
```

> "The second problem is in the import pipeline. Every time a scan report is imported,
> new findings are saved one by one. One INSERT per finding, then one more query to record
> which tool found it. For a 19-finding ZAP scan, that was 289 database queries on the
> V2 endpoint and 315 on the V3 endpoint.
> No transaction wrapped the operation, so a failure halfway through could leave
> the database in a partially imported state."

**Action:** In the diff, highlight the old loop (red lines) and new `bulk_create` block (green lines).

> "We replaced the per-finding loop with a single `bulk_create()` call — one INSERT for all findings —
> batched the `found_by` many-to-many relationships, and wrapped everything in `transaction.atomic()`."

---

### [1:45 – 2:10] Phase 2 — Measured results

**Screen:** Terminal, run the query-count test

```bash
./run-unittest.sh --test-case unittests.test_tag_inheritance_perf
```

> "These are not estimated numbers. DefectDojo's own test suite uses Django's `assertNumQueries`
> to pin exact database query counts. Before our change, the ZAP import test asserted 289 queries
> on V2 and 315 on V3. After our change, those assertions pass at 269 and 295 respectively.
> That is 20 fewer round-trips to the database per import, confirmed by a test that fails
> if the number changes in either direction."

**Action:** Let the test run to completion. Show the final `OK` line.

---

### [2:10 – 2:35] IBM Bob's role

**Screen:** Split — code on left, IBM Bob chat interface on right (screenshot or live if available)

> "We used IBM Bob throughout both phases as our AI pair programmer. Bob is not just
> autocomplete — it reads actual files in the repository, reasons about the code, and
> produces proposals you can verify.
>
> For Phase 1, Bob read `models.py`, identified the `bytes.strip()` issue, and listed
> the specific Unicode codepoints that would slip through. It then generated the seven
> regression tests for each variant.
>
> For Phase 2, Bob traced the call chain from `process_scan()` down to `save_no_options()`,
> identified the N+1 pattern, proposed the `bulk_create()` refactor, and noted the absence
> of a transaction boundary. It designed the 19-test suite covering correctness, rollback,
> M2M relationships, and reimport compatibility.
>
> Every suggestion Bob made was grounded in the actual codebase — not generic patterns.
> That meant we could move faster and trust the output."

---

### [2:35 – 3:00] Closing — Summary and repository

**Screen:** Browser, GitHub PR list showing both merged PRs, then README

> "Both fixes are merged into the `dev` branch of our fork. Phase 1 adds 191 lines of
> regression tests. Phase 2 adds 914 lines of regression and performance tests.
> All tests pass. Ruff lint passes on every changed file.
>
> The changes are surgical: six files, no schema migrations, no API changes,
> fully backward compatible. Anyone can clone the fork and run the application identically
> to upstream — they just get correct deduplication and fewer database queries.
>
> The repository is at github.com/selimanaw/DefectDojo, branch dev.
> Thank you."

**Action:** End recording with the GitHub repository page visible.

---

## Recording Tips

### Timing check

Run through the script once with a timer before recording. Typical spoken rates:

- This script at a calm, clear pace: approximately 2:50 – 3:05 minutes
- If you exceed 3:00 on a run-through, trim the Phase 1 diagnostic section (0:48–1:12) to one sentence

### Terminal commands to prepare in advance

Open a text file with the commands below so you can copy-paste without typing during the recording:

```bash
./run-unittest.sh --test-case unittests.test_deduplication_logic.TestHashFieldsNormalisation
./run-unittest.sh --test-case unittests.test_tag_inheritance_perf
git show 3e172517e5 -- dojo/importers/base_importer.py
git log --oneline -5
```

### Screen resolution

Record at 1920×1080 or 2560×1440. If sharing a screen with code, use a dark theme (VS Code Dark+ or
Dracula) — it renders clearly in compressed video.

### What NOT to show

- Do not show the admin password on screen. Blur or cut before the password is visible in the logs.
- Do not run `docker compose down -v` during the demo (it deletes all data).
- Do not show the upstream DefectDojo repository — show only the fork at `selimanaw/DefectDojo`.

---

## Alternate Shorter Version (90 seconds, if the event requires)

| Timestamp | Content |
|---|---|
| 0:00 – 0:10 | "Two bugs, two fixes, both merged. Here is what we changed." |
| 0:10 – 0:35 | Phase 1: show the one-line fix in `models.py`, run the 7 hash tests, show OK |
| 0:35 – 1:00 | Phase 2: show the `bulk_create()` block in `base_importer.py`, run ZAP query test, show 269/295 |
| 1:00 – 1:20 | "IBM Bob identified both root causes by reading the actual files, not from patterns." |
| 1:20 – 1:30 | Repository URL, both PRs merged, tests passing. Done. |
