Task: Rebuild/sync the ThunderAy Product Roadmap from the Pack Focus Config + Tasks DB. Use Notion MCP tools (mcp__notion__*).

Timezone reference: Europe/Prague. Use today's date for all comparisons.

== NOTION SOURCES (IDs) ==
- Pack Focus Config (text page): 33f3cc98-5f39-8162-91f8-c30fcd5c1d15 — 'Active Focus Packs' bullet list is the whitelist.
- In Process DB: 2933cc98-5f39-8006-9b6a-d2965016b932 (DS 2933cc98-5f39-80ec-a183-000bf8809f13) — pack pages; canonical name + icon. Each pack page has a `Tasks` relation listing every task's page ID.
- Tasks DB: b67fdb41-6ce1-407f-b944-3c66689547b6 (DS c18e25c2-0b12-4c47-ae82-72b74b8b23ee) — all tasks.
- Product Roadmap DB: 2e63cc98-5f39-8039-b26a-e8d2dee03fd1 (DS 2e63cc98-5f39-8078-887b-000bfe2e4491) — target.
- Q1 2026 Launches view: d87a8252-ad7b-4d99-95eb-0da7fced3225 — quarter sanity check.

== STEP 1: LOAD CONTEXT ==
1) Fetch Focus Config page → extract 'Active Focus Packs' bullets → focusPacks[].
2) Fetch each In Process pack page individually via `mcp__notion__notion-fetch` (by title match against focusPacks). Capture pack page id, icon, and the `Tasks` relation array (= list of task page IDs linked to this pack). This is the AUTHORITATIVE per-pack task list — use it instead of filtering the whole Tasks DS.
3) For each task page ID from step 2, fetch the task page via `mcp__notion__notion-fetch`. Capture: name, status, dueDate (Date.start), role (Role select), person (Person select). DO NOT rely on a whole-Tasks-DS query for this step — whole-DS queries silently truncate around ~50 rows, which caused the 2026-04-22 incident where the new `Minimap: QA` task (due 2026-04-25) was missed and the `Minimap: QA / Playtests` row kept stale Done/2026-04-17 state. Fetching per-pack via the In Process relation guarantees completeness because the relation array enumerates every linked task ID explicitly.
4) Product Roadmap — query the **DATA SOURCE DIRECTLY**, NOT the Timeline Focus view. The Timeline Focus view (view://2e63cc98-5f39-80db-a86d-000cbd9f5381) has a status_is filter that silently drops rows (caused 2026-04-22 duplicate incident). Instead: `mcp__notion__notion-search(query: '<pack name>', query_type: internal, data_source_url: collection://2e63cc98-5f39-8078-887b-000bfe2e4491, page_size: 25)` ONCE PER FOCUS PACK. Merge results client-side. Return every row (archived + active) so duplicate detection can see them all.

Stop conditions: Focus Config empty → stop. Focus pack name has no exact In Process match → stop (title match must include 'Add-On' suffix where applicable).

Sanity check: after Step 1.3, you should have at least ~90 task records across the focus packs (typical load). If significantly fewer, suspect a missed fetch — redo any pack whose count is implausibly low before proceeding.

== STEP 2: BUILD WORKING SET PER PACK ==
For each pack in focusPacks:
- Use the In Process page matched in Step 1.
- Capture packIcon = pack page icon (copy to every milestone row below).
- Use the task list from Step 1.3 for this pack. For each: name, status, dueDate, role, last_edited_time.
- DELETED tasks: skip all tasks marked deleted. Deleted tasks have frozen/stale status and must not affect bucket classification or date computation.

== STEP 3: CLASSIFY TASKS INTO BUCKETS (case-insensitive, last matching rule wins) ==
- Finish Development: name contains fix|implement|feature|dev|improve
- QA / Playtests: name contains qa|test|playtest
- Finish Marketing: name contains key art|screenshots|panorama|trailer|description|store description|marketing AND role != 'Project Manager' (excludes packaging / content guide)
- Prep / Coordination: name STARTS WITH 'Prepare ' AND role == 'Project Manager' (Prepare Key Art, Prepare Screenshots, Prepare Panorama, Prepare Store Description, etc.)
- Package: name contains package
- Submit: name contains content guide|guidebook|submission|submit|resubmit (exclude pure packaging)
- Launch: name contains launch|release

== STEP 4: COMPUTE MILESTONE DATES ==

**General rule (all buckets except QA / Playtests):**
Date = LATEST dueDate among non-deleted bucket tasks where status != Done.
If all tasks are Done, use LATEST dueDate overall (milestone is Done).

**QA / Playtests — NEAREST UPCOMING rule (changed 2026-04-27):**
1. Collect all non-deleted QA bucket tasks.
2. Split into: upcoming (dueDate >= today, status != Done) and overdue (dueDate < today, status != Done).
3. If upcoming tasks exist → Date = MIN dueDate among upcoming non-Done tasks. Status = status of that task.
4. If NO upcoming but overdue tasks exist → Date = LATEST dueDate among overdue non-Done tasks. Flag as overdue in the report.
5. If all tasks Done → Date = LATEST dueDate overall. Status = Done → eligible for archive (rule 4).
Rationale: the roadmap must always show the NEXT thing that needs attention, not the far-future ceiling date.

**Overdue + upcoming both present for QA:**
Show the nearest upcoming date on the milestone row. In the Step 7 overdue report, list BOTH the overdue task AND the nearest upcoming task so the reader sees: what's late + what's next.

**Other date rules:**
- No due dates in bucket → skip milestone, note in warnings.
- Launch fallback: if no launch/release tasks, Launch date = Submit milestone date. Every focus pack must have a Launch row.
- Prep / Coordination tag = 'Submission'.
- Quarter sanity: Q1 Jan–Mar, Q2 Apr–Jun, Q3 Jul–Sep, Q4 Oct–Dec — warn if Launch crosses quarters between runs.

== STEP 5: UPSERT ROWS IN PRODUCT ROADMAP ==
Canonical name format: 'Pack: Milestone'
- Strip trailing ' Add-On' from pack part (Minimap Add-On → Minimap).
- Colon, never em dash.
- Milestone names exactly: Finish Development | QA / Playtests | Finish Marketing | Prep / Coordination | Package | Submit | Launch.
- Example: 'Minimap: QA / Playtests'.

Duplicate detection (ALL rows including archived, to see full picture):
- Group candidates by case- and format-insensitive 'Pack: Milestone' (em-dash variants, spacing, Add-On suffix all collapse to the same group).
- Each group should have exactly ONE non-Archived row — the canonical. >1 non-Archived → duplicates exist.

**DETERMINISTIC CANONICAL SELECTION (CRITICAL — prevents flip-flop across runs):**
When multiple non-Archived rows exist for the same group:
1. Sort by Notion's `last_edited_time` DESCENDING (most recently modified first).
2. The TOP row is canonical. Archive all others.
3. NEVER use creation-order or query-return order — those are non-deterministic and caused the 2026-04-22 incident where the trigger archived the canonical Minimap: Finish Development row that had just been updated by the deadline cascade.

Upsert:
- Canonical exists → UPDATE Date, Tags, Status.
- Only legacy exists → CREATE canonical, then ARCHIVE legacy (Status=Done + add Archived tag).
- Multiple duplicates → keep canonical (via deterministic selection above), archive rest.
- None → CREATE canonical.

**POST-WRITE VERIFY (MANDATORY — catches race conditions + silent PATCH failures):**
After EVERY Step 5 PATCH that updates or creates a canonical row:
1. Immediately re-fetch the row via mcp__notion__notion-fetch.
2. Verify Tags does NOT contain 'Archived' (canonical rows must stay visible).
3. Verify date:Date:start matches what you just wrote.
4. If either check fails → re-PATCH to correct, then re-verify.

Every canonical row fields:
- Name: Pack: Milestone
- Date: computed
- Tags:
  - Finish Development → 'Finish Development part'
  - QA / Playtests → 'Playtests'
  - Finish Marketing → 'Marketing'
  - Prep / Coordination → 'Submission'
  - Package → 'Submission'
  - Submit → 'Submission'
  - Launch → 'Release'
- Status: all tasks Done → Done; any In progress/To review → In progress; else Not started.
- Icon: set to packIcon (skip if already matches).

== STEP 6: ARCHIVE SWEEP (MANDATORY — run EVERY invocation, even with 0 upserts) ==

**THIS IS THE #1 THING THE ROUTINE FORGETS.** If your report has 'Archived: 0', you either (a) actually archived nothing AND verified every active Done row against rule 4, or (b) skipped the sweep. (b) is a bug.

Checklist — tick EVERY active (non-Archived) roadmap row, in order:
1. Pack part matches exactly one name in focusPacks? If not → ARCHIVE (rule 1: non-focus leftover).
2. Another canonical row with the same 'Pack: Milestone' name exists? If yes and this row is the legacy/em-dash or same-spelling duplicate → ARCHIVE (rule 2: duplicate).
3. Milestone name is one of the 7 canonical names? If not → ARCHIVE (rule 3: non-standard).
4. Status == Done AND milestone ∈ {Finish Development, QA / Playtests, Finish Marketing, Prep / Coordination, Package}? If yes → ARCHIVE (rule 4: done non-Submit/Launch milestone).

**UN-ARCHIVE CHECK (added 2026-04-27 — run EVERY invocation):**
For every ARCHIVED row in {Finish Development, QA / Playtests, Finish Marketing, Prep / Coordination, Package}:
- Re-compute the milestone from the current task list.
- If the result is NOT Done (i.e., non-Done tasks exist for that bucket), the row must be UN-ARCHIVED: remove 'Archived' from Tags, update Date and Status to the computed values.
- Root cause: a previous run archived the milestone when all tasks were Done, but new tasks were added since. Without this check, the roadmap goes stale silently.

Archive = add 'Archived' tag to existing Tags array. Do NOT delete. Do NOT remove original tag.
Un-archive = set Tags to original tag only (remove 'Archived'). Update Date and Status to match current computation.

Do NOT archive: rows created/updated in Step 5 (the canonical set this run, unless they just became Done and match rule 4); Done Submit or Launch rows; rows where Status ≠ Done.

**RULE 4 HARD ASSERTION (MANDATORY before writing the report):**
After Step 6 completes, re-query the Roadmap DS via `mcp__notion__notion-search` across all focus packs. Filter results in memory for rows where: Archived tag NOT present AND Status == Done AND name ends with one of ': Finish Development' | ': QA / Playtests' | ': Finish Marketing' | ': Prep / Coordination' | ': Package'. This list MUST be empty. If non-empty, archive each row immediately and repeat the assertion until empty. Do NOT write the report until the assertion returns zero rows.

== STEP 7: OVERDUE TASK SURFACE (DO NOT ARCHIVE) ==
From the per-pack task list gathered in Step 1.3: tasks where dueDate < today AND status != Done. List in report with pack, task name, due date, assignee, Notion URL.

For QA milestones with overdue tasks: also list the nearest upcoming QA task in the same section so the reader sees what is late AND what is next.

Always re-check: https://www.notion.so/patrikkoudelka/b67fdb416ce1407fb9443c66689547b6?v=81a49526363943f99e57049400d856bd

== PRE-REPORT INTEGRITY CHECK (run BEFORE writing the report) ==
- [ ] Per-pack task fetch covered every In Process `Tasks` relation entry (no silent truncation from whole-DS queries).
- [ ] Every focus pack has tasks classified into at least one bucket, OR explicitly flagged in Warnings.
- [ ] Step 6 archive sweep ran — inspected every active non-Archived row against rules 1-4.
- [ ] Un-archive check ran — every archived row re-evaluated against current task data.
- [ ] Rule 4 hard assertion ran and returned zero rows.
- [ ] Post-write verify ran for every Step 5 PATCH.
- [ ] Report's Archived: <n> and Un-archived: <n> reflect real PATCHes.

If any checkbox fails, go back and fix before writing the report.

== STEP 8: OUTPUT REPORT ==
Keep short (runs 3×/day).

## Roadmap Update — YYYY-MM-DD HH:MM Europe/Prague

### Focus packs (N)
- <pack> · <pack>

### Roadmap changes
- Created: n  Updated: n  Archived: n  Un-archived: n
- Created: Pack: Milestone — date
- Updated: Pack: Milestone — old → new
- Archived: Pack: Milestone — reason
- Un-archived: Pack: Milestone — reason

### Overdue tasks (N)
- Pack · Task — due YYYY-MM-DD · @assignee · notion-url
- (For QA overdue) → next upcoming QA: Pack · Task — due YYYY-MM-DD · @assignee

### Warnings
- e.g. 'Multipack: Finish Marketing bucket has no due dates → milestone skipped'

### Summary
<one sentence>

If zero changes and zero overdue: output 'No changes. Roadmap in sync. No overdue tasks.' and stop.

== NON-NEGOTIABLES ==
- Focus Config is the only source of truth.
- Copy pack icon onto every roadmap row.
- Never delete — only add or remove 'Archived' tag (un-archive is allowed when a bucket becomes active again).
- Overwrite manually-edited dates when they conflict with Tasks-derived dates.
- Always re-check Tasks dashboard for overdue items — do not archive them.
- Every focus pack must have a Launch row (fallback = Submit date).
- **Deterministic canonical via last_edited_time DESC** — never archive a row that was just updated.
- **Post-write verify every PATCH** — silent reverts are real, the routine must self-check.
- **Rule 4 hard assertion** — no active Done Dev/QA/Marketing/Prep/Package row may exist when the report is written.
- **QA nearest-upcoming date** — QA milestone always shows the nearest upcoming non-Done QA task date, not the latest.
- **Un-archive check every run** — archived milestones with new non-Done tasks must be restored.
