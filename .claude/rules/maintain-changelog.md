# Record changes in CHANGELOG.md

When you change the repo — playbooks, templates, `files/`, `roles/`,
`inventory.example`, docs, or these rules — add a line to `CHANGELOG.md` in the
**same change**, so the log never drifts from the code.

## Skip non-functional changes

Cosmetic changes don't get an entry: play/task renames, comment edits, whitespace,
log/header wording, output formatting — anything that doesn't change what a run does.
The test: would someone running the playbooks get a different result? No → skip.

## Format

Keep a Changelog style: entries under `## [Unreleased]`, grouped
`Added` / `Changed` / `Fixed` / `Removed`. One line per change.

## Factual only — no narration

State WHAT changed. Not why, not the alternatives weighed, not how it was decided,
not the story of the debugging that led there.

- Good: `nginx limit_req flood guard (15r/s, burst 30) + nginx-limit-req fail2ban jail`
- Bad:  `after checking the logs and testing a few rates we chose a generous 15r/s so
         bursty page loads don't false-positive`

Rationale, trade-offs, and narrative belong in the commit body, the PR description, or
a design doc (e.g. `ACL_DESIGN.md`). The CHANGELOG is a factual index, not a story —
the same discipline as the code comments: say what it does, nothing more.
