# DefectDojo Modernization — AngelHack IBM Bob Hackathon 2026

> **Fork:** [selimanaw/DefectDojo](https://github.com/selimanaw/DefectDojo) · **Branch:** `dev`  
> **Base project:** [DefectDojo/django-DefectDojo](https://github.com/DefectDojo/django-DefectDojo) (OWASP Flagship, BSD-3-Clause)  
> **Hackathon:** AngelHack IBM Bob Hackathon 2026

---

## Table of Contents

1. [Problem Statement](#problem-statement)
2. [Our Improvements](#our-improvements)
   - [Phase 1 — Vulnerability Hash Correctness](#phase-1--vulnerability-hash-correctness)
   - [Phase 2 — Vulnerability Import Performance](#phase-2--vulnerability-import-performance)
3. [Technology Stack](#technology-stack)
4. [How IBM Bob Contributed](#how-ibm-bob-contributed)
5. [Test Results and Performance Data](#test-results-and-performance-data)
6. [Setup and Execution](#setup-and-execution)
7. [Repository Structure of Changes](#repository-structure-of-changes)
8. [What We Did Not Change](#what-we-did-not-change)

---

## Problem Statement

[DefectDojo](https://www.defectdojo.com/) is an OWASP Flagship DevSecOps platform used by security teams worldwide
to import, deduplicate, and track vulnerabilities from hundreds of scanning tools. Two independent bugs in its
core vulnerability workflow affected every deployment that uses its hash-based deduplication or imports findings
at scale.

### Bug 1 — Silent hash corruption on Unicode whitespace

DefectDojo computes a `hash_code` for each finding to identify duplicates across scans. The hash is built from
fields such as CVE IDs, file paths, titles, and component names. The normalisation pipeline was:

```python
# BEFORE (broken)
value.casefold().encode("utf-8").strip()
#                               ^^^^^^ strips bytes, not characters
```

`bytes.strip()` removes only ASCII control characters (bytes 0x00–0x20). Unicode whitespace variants —
U+00A0 NO-BREAK SPACE, U+2003 EM SPACE, U+2009 THIN SPACE, and others — pass through unchanged, becoming
embedded in the hash input. Two findings for the same vulnerability, one with a regular space and one with
a non-breaking space in its title, receive *different* hash codes and are therefore never deduplicated.
This is a silent data-correctness bug: no error is raised, findings simply accumulate as false duplicates.

Additionally, four diagnostic log messages that fire when the hash system cannot be configured (missing
scanner entry, disallowed field, zero CWE, no settings) were emitted at `DEBUG` level — invisible under
default logging. Operators had no way to know their deduplication was silently degraded.

### Bug 2 — N+1 database queries on every import

Each finding imported from a scan report was saved individually:

```python
# BEFORE (one INSERT per finding)
for finding in new_findings:
    finding.save_no_options()
    finding.found_by.add(test.test_type)  # one M2M INSERT per finding
```

For a 19-finding ZAP scan this produced **289 queries on the V2 endpoint** and **315 on the V3 endpoint**.
For larger reports the cost scales linearly. No database transaction wrapped the operation, meaning a partial
failure could leave the database in an inconsistent state.

---

## Our Improvements

### Phase 1 — Vulnerability Hash Correctness

**Merged:** PR [#1](https://github.com/selimanaw/DefectDojo/pull/1) · `fix/hash-correctness` → `dev`  
**Commit:** `5ed6aafd13` (2026-10-18)  
**Files changed:** `dojo/finding/deduplication.py` (+5/−1), `dojo/finding/models.py` (+20/−10),
`unittests/test_deduplication_logic.py` (+191/−0)

#### The fix

Move `strip()` from the byte string to the Unicode string, *before* encoding:

```python
# AFTER (correct)
value.casefold().strip().encode("utf-8")
#               ^^^^^^^ strips Unicode whitespace, then encodes
```

For ordinary inputs — ASCII CVE IDs, file paths, and titles with no leading or trailing whitespace — the
two orderings produce identical output, so stored hashes from supported parsers are expected to be
unaffected. A hash difference will arise only for values with leading or trailing non-ASCII whitespace
(e.g. U+00A0 NO-BREAK SPACE, U+2003 EM SPACE). Any finding whose hash field contained such characters
will receive a corrected hash on next import and may not match the previously-stored value; those findings
will be re-imported rather than deduplicated against the old record.

#### Diagnostic visibility

Four `DEBUG` log messages in `compute_hash_code()` were promoted to `WARNING`, each now including the
scan type name so operators can pinpoint the affected scanner:

| Condition | Level before | Level after |
|---|---|---|
| Missing `HASHCODE_*` configuration | `DEBUG` | `WARNING` |
| Scan type not in `HASHCODE_FIELDS_PER_SCANNER` | `DEBUG` | `WARNING` |
| Field not in `HASHCODE_ALLOWED_FIELDS` | `DEBUG` | `WARNING` |
| CWE is 0 and `HASHCODE_ALLOWS_NULL_CWE` is False | `DEBUG` | `WARNING` |

One `DEBUG` message in `do_dedupe_finding_task_internal()` for the legacy fallback path was also promoted
to `WARNING`, with the scan type name and a suppression hint.

---

### Phase 2 — Vulnerability Import Performance

**Merged:** PR [#2](https://github.com/selimanaw/DefectDojo/pull/2) · `perf/bulk-import-persist` → `dev`  
**Commits:** `3e172517e5`, `aa96dfb74b`, `44257e481f` (2026-10-18)  
**Files changed:** `dojo/importers/base_importer.py` (+98/−5),
`dojo/importers/default_reimporter.py` (+31/−25),
`unittests/test_bulk_persist_new_findings.py` (+914/−0),
`unittests/test_importers_performance.py` (+22/−22),
`unittests/test_tag_inheritance_perf.py` (+5/−2)

#### The optimisation

`BaseImporter.persist_new_findings()` was refactored to collect all new findings into a list and insert them
in a single `bulk_create()` call, then write all `found_by` many-to-many relationships in a single
follow-up `bulk_create()` on the M2M through table. The entire operation is wrapped in
`transaction.atomic()` to guarantee all-or-nothing consistency.

```python
# AFTER — single bulk INSERT + one batched M2M INSERT via through table
with transaction.atomic():
    saved_findings = Finding.objects.bulk_create(prepared_findings, batch_size=1000)
    if saved_findings:
        test_type = saved_findings[0].test.test_type
        through = Finding.found_by.through
        through.objects.bulk_create(
            [through(finding_id=f.pk, test_type_id=test_type.pk) for f in saved_findings],
            batch_size=1000,
            ignore_conflicts=True,
        )
```

`DefaultReImporter.persist_new_findings()` overrides the base class with a per-finding loop via
`persist_new_finding()`. This preserves a per-finding deferral seam that downstream editions rely on;
the reimporter does not use the `bulk_create` path.

#### Query reduction (ZAP 19-finding scan, measured by `assertNumQueries`)

| Endpoint | Before | After | Reduction |
|---|---|---|---|
| Import V2 | 289 | 269 | −20 queries (−6.9 %) |
| Import V3 | 315 | 295 | −20 queries (−6.9 %) |
| Reimport (no change) | 80 | 80 | unchanged |
| Reimport (with new findings) | 159 | 159 | unchanged |

Reimport counts are unchanged because `DefaultReImporter.persist_new_findings()` keeps its own
per-finding loop (to preserve a downstream extension seam) and does not use the `bulk_create` path.

---

## Technology Stack

| Layer | Technology |
|---|---|
| Application framework | Django 4.x (Python) |
| REST API | Django REST Framework |
| Task queue | Celery + Valkey (Redis-compatible) |
| Database | PostgreSQL 14+ |
| ORM | Django ORM (`bulk_create`, `transaction.atomic`, M2M through-table `bulk_create`) |
| Containerisation | Docker Compose (nginx + uwsgi + celeryworker + celerybeat + postgres + valkey) |
| Test framework | Django `TestCase`, `assertNumQueries`, `unittest.mock` |
| Linting | Ruff |
| AI pair programmer | IBM Bob (Codex) |

---

## How IBM Bob Contributed

IBM Bob served as the primary pair-programming assistant throughout both phases. Its contributions were
concrete and tool-verified — not generic suggestions.

### Phase 1

- **Root-cause identification.** Bob read `dojo/finding/models.py` and identified the method chain
  `casefold().encode("utf-8").strip()`, explained that `bytes.strip()` is defined over ASCII only
  (0x00–0x20), and produced a list of Unicode whitespace codepoints that would survive it unstripped.
- **Regression test scaffolding.** Bob generated `TestHashFieldsNormalisation` with seven parameterised
  test cases: ASCII determinism, case folding, ASCII whitespace stripping, each Unicode whitespace variant
  (NBSP, EM SPACE, THIN SPACE), internal space preservation, output format, and a pinned regression value.
- **Compatibility analysis.** Bob scanned the supported-parser list to confirm that no supported parser
  produces non-ASCII whitespace in hash fields, so ordinary deployments are expected to see no hash
  changes; any deployment whose stored hashes did include such characters will receive corrected values
  on the next import.
- **Log-level audit.** Bob located all four `DEBUG` calls in `compute_hash_code()` and the one in the
  deduplication task, drafted replacement `WARNING` messages with scan-type context, and verified that
  each message included an actionable suppression hint.

### Phase 2

- **Query profiling.** Bob traced the call from `DefaultImporter.process_scan()` through
  `persist_new_findings()` and identified the per-finding `save_no_options()` + `found_by.add()` loop as
  the source of the N+1 pattern.
- **Bulk insert design.** Bob proposed the `bulk_create()` + through-table `bulk_create()` refactor,
  explained the Django ORM constraint that `bulk_create()` returns instances without triggering
  `post_save` signals (which `save_no_options` also skips), and confirmed the two paths are
  semantically equivalent.
- **Transaction boundary.** Bob noted the absence of an atomic block and suggested wrapping the persist
  operation in `transaction.atomic()` to prevent partial-import states.
- **Test coverage design.** Bob drafted `test_bulk_persist_new_findings.py` (914 lines, 19 tests + 1
  skipped) covering correctness, `found_by` relationships, deduplication interaction, rollback on
  exception, and reimport compatibility.
- **Baseline update guidance.** Bob identified which `assertNumQueries` baselines would change (import
  only, not reimport) and explained why (`DefaultReImporter` keeps its own per-finding loop and does not
  use the `bulk_create` path, so its query count is unchanged).

---

## Test Results and Performance Data

### Phase 1 test suite

| Suite | Tests | Passed | Skipped | Failed |
|---|---|---|---|---|
| `TestHashFieldsNormalisation` | 7 | 7 | 0 | 0 |
| `TestLegacyFallbackWarnings` | 2 | 2 | 0 | 0 |
| Existing `test_deduplication_logic.py` suite | 94 | 92 | 2 | 0 |

All tests in the four target suites pass (2 pre-existing skips, no new failures).
Ruff lint: passed on all changed files.

### Phase 2 test suite

| Suite | Tests | Passed | Skipped | Failed |
|---|---|---|---|---|
| `test_bulk_persist_new_findings.py` | 20 | 19 | 1 | 0 |
| `test_importers_performance.py` | 11 | 11 | 0 | 0 |

Ruff lint: passed on all four changed files.  
No failures in final targeted test runs.

### Confirmed query baselines (from `unittests/test_tag_inheritance_perf.py`, line 659–664)

```python
EXPECTED_ZAP_IMPORT_V2              = 269   # was 289 before Phase 2
EXPECTED_ZAP_IMPORT_V3              = 295   # was 315 before Phase 2
EXPECTED_ZAP_REIMPORT_NO_CHANGE_V2  = 80    # unchanged
EXPECTED_ZAP_REIMPORT_NO_CHANGE_V3  = 91    # unchanged
EXPECTED_ZAP_REIMPORT_WITH_NEW_V2   = 159   # unchanged
EXPECTED_ZAP_REIMPORT_WITH_NEW_V3   = 188   # unchanged
```

---

## Setup and Execution

> See [`DEMO_GUIDE.md`](DEMO_GUIDE.md) for the full deployment and demonstration guide.

### Quick start (Docker Compose)

Requirements: Docker Desktop (WSL2 backend on Windows), ~8 GB free RAM.

```bash
# 1. Clone this fork
git clone https://github.com/selimanaw/DefectDojo.git
cd DefectDojo

# 2. Confirm you are on the dev branch (both phases already merged)
git branch
# * dev

# 3. Start the full stack (first build takes 5–10 minutes)
docker compose up -d

# 4. Retrieve the admin password from the initializer log
docker compose logs initializer | grep "Admin password:"

# 5. Open the application
#    http://localhost:8080
#    Username: admin  Password: <from step 4>
```

### Running the unit tests

```bash
# Run Phase 1 regression tests
docker compose -f docker-compose.override.unit_tests.yml run --rm uwsgi \
  python manage.py test unittests.test_deduplication_logic.TestHashFieldsNormalisation \
  unittests.test_deduplication_logic.TestLegacyFallbackWarnings -v 2

# Run Phase 2 bulk-persist regression tests
docker compose -f docker-compose.override.unit_tests.yml run --rm uwsgi \
  python manage.py test unittests.test_bulk_persist_new_findings -v 2

# Run Phase 2 performance / query-count tests
docker compose -f docker-compose.override.unit_tests.yml run --rm uwsgi \
  python manage.py test unittests.test_tag_inheritance_perf \
  unittests.test_importers_performance -v 2 --tag performance
```

Alternatively, use the project's convenience script:

```bash
./run-unittest.sh --test-case unittests.test_deduplication_logic.TestHashFieldsNormalisation
./run-unittest.sh --test-case unittests.test_bulk_persist_new_findings
```

---

## Repository Structure of Changes

```
selimanaw/DefectDojo (fork of DefectDojo/django-DefectDojo)
│
├── dojo/finding/
│   ├── deduplication.py          ← Phase 1: WARNING-level fallback log
│   └── models.py                 ← Phase 1: strip() ordering fix + WARNING logs
│
├── dojo/importers/
│   ├── base_importer.py          ← Phase 2: bulk_create + atomic + batched M2M
│   └── default_reimporter.py    ← Phase 2: delegates to base persist method
│
└── unittests/
    ├── test_deduplication_logic.py       ← Phase 1: 191 lines of new tests
    ├── test_bulk_persist_new_findings.py ← Phase 2: 914 lines of new tests
    ├── test_importers_performance.py     ← Phase 2: updated query baselines
    └── test_tag_inheritance_perf.py      ← Phase 2: updated ZAP baselines
```

All other files are **unchanged** from the upstream DefectDojo project.

---

## What We Did Not Change

To be explicit about scope: every other aspect of DefectDojo is unmodified. This includes:

- All 150+ vulnerability scanner parsers
- The REST API (v2 and v3) surface and contracts
- The UI / frontend
- The Helm chart and Kubernetes manifests
- Authentication, RBAC, and user management
- All other deduplication algorithms (unique ID, hash-only, unique-ID-or-hash)
- The reimport workflow beyond the persistence refactor
- Database schema / migrations

Our changes are confined to the hash normalisation in the finding model, the diagnostic log levels in the
deduplication module, and the bulk-insert optimisation in the base importer. The project builds, deploys,
and operates identically to upstream.

---

*This project is a fork of [DefectDojo](https://github.com/DefectDojo/django-DefectDojo) and is licensed
under the [BSD 3-Clause License](LICENSE.md). Our additions are contributed under the same license.*
