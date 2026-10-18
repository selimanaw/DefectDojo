# Hackathon Submission Checklist

**Event:** AngelHack IBM Bob Hackathon 2026  
**Project:** DefectDojo Modernization  
**Fork:** [selimanaw/DefectDojo](https://github.com/selimanaw/DefectDojo) · branch `dev`  
**Status as of:** 2026-10-18

---

## Items Completed ✅

| # | Item | Evidence |
|---|---|---|
| 1 | Phase 1 — hash correctness fix | Merged PR #1, commit `5ed6aafd13` |
| 2 | Phase 1 — WARNING-level diagnostic logs | Same PR |
| 3 | Phase 1 — regression tests (9 tests passing) | `unittests/test_deduplication_logic.py` +191 lines |
| 4 | Phase 2 — bulk_create() optimisation | Merged PR #2, commit `3e172517e5` |
| 5 | Phase 2 — transaction.atomic() boundary | Same PR |
| 6 | Phase 2 — batched M2M relationships | Same PR |
| 7 | Phase 2 — regression tests (19 passing, 1 skipped) | `unittests/test_bulk_persist_new_findings.py` +914 lines |
| 8 | Phase 2 — query-count baselines updated | `test_tag_inheritance_perf.py`: 269 / 295 confirmed |
| 9 | Ruff lint passing on all changed files | Per PR #2 body |
| 10 | Both PRs merged to dev | `git log` confirmed |
| 11 | HACKATHON.md written | `HACKATHON.md` in this repo |
| 12 | Deployment guide written | `DEMO_GUIDE.md` in this repo |
| 13 | Video script written | `VIDEO_SCRIPT.md` in this repo |

---

## Items Still Required ⚠️

These items cannot be confirmed without access to the AngelHack submission portal.
Check each one against the official submission form.

### 1. Submission form on the AngelHack portal

**Status:** Unknown — must be verified  
**Action:** Log into the AngelHack portal and confirm:
- Has a project been created for this submission?
- Are all required fields filled in (project name, description, category, team members)?
- Is the GitHub repository URL (`https://github.com/selimanaw/DefectDojo`) entered?
- Is the branch (`dev`) specified if the form asks for it?

### 2. Video demonstration uploaded

**Status:** Not yet recorded  
**Action:** Record the video using `VIDEO_SCRIPT.md`, upload to YouTube (unlisted) or Vimeo,
and paste the URL into the submission form.  
**Time required:** ~30 minutes to record, render, and upload.

### 3. Live demo URL (if required by the submission form)

**Status:** Not yet deployed publicly  
**Action:** Either:
- Run locally and expose via ngrok (see `DEMO_GUIDE.md` Option B), or
- Deploy to Railway (see `DEMO_GUIDE.md` Option C)

**Note:** Check whether the submission form requires a live URL. Some hackathons accept a video-only submission.

### 4. IBM Bob usage documentation

**Status:** Described in `HACKATHON.md` — verify the portal's specific requirement  
**Action:** The submission form may require:
- A specific "How did you use IBM Bob?" field
- A chat log or screenshot export from the Bob session
- A minimum number of Bob interactions documented

If a chat log is required, export it from the IBM Bob session history before closing the session.

### 5. Team member information

**Status:** Unknown — must be verified  
**Action:** Confirm all team members are listed on the submission form with their names, emails,
and GitHub handles.

### 6. Project category / track selection

**Status:** Unknown  
**Action:** AngelHack IBM Bob Hackathon 2026 likely has specific tracks (e.g., "Security", "DevOps",
"AI-Assisted Development"). Select the most relevant track on the submission form.  
This project fits: **Security tooling** and/or **AI-assisted open source contribution**.

### 7. Submission deadline confirmation

**Status:** Unknown — verify on the portal  
**Action:** Confirm the exact submission deadline time and timezone. Do not assume the deadline
is end-of-day in your local timezone.

---

## Recommended Completion Order (fastest path to done)

1. **Open the AngelHack portal now** — create or complete the project entry, paste the GitHub URL.
   This takes 5 minutes and locks in your submission slot.

2. **Record the video** — use `VIDEO_SCRIPT.md`. The 90-second alternate version is sufficient if time
   is tight. Upload to YouTube (unlisted) immediately; paste the URL into the form.

3. **Deploy a public URL if required** — run `docker compose up -d` and then `ngrok http 8080`.
   Takes 15 minutes. The ngrok URL is valid for the rest of the session.

4. **Add the three documentation files to the GitHub repository** — commit `HACKATHON.md`,
   `DEMO_GUIDE.md`, and `VIDEO_SCRIPT.md` to the `dev` branch and push. This takes 2 minutes
   and makes the submission self-contained for judges who browse the repository directly.

5. **Final check** — re-read the submission form fields against the completed items above,
   submit, and keep the confirmation email.

---

## Files Added by This Preparation

| File | Purpose |
|---|---|
| `HACKATHON.md` | Primary project documentation for judges |
| `DEMO_GUIDE.md` | Deployment and demonstration instructions |
| `VIDEO_SCRIPT.md` | Word-for-word narration script for the video |
| `SUBMISSION_CHECKLIST.md` | This file — tracks remaining requirements |
