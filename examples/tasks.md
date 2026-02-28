---

description: "Task list template for feature implementation"
---

# Tasks: TaskToBeads Demo

**Input**: `.specify/specs/001-task-to-beads/`
**Prerequisites**: plan.md, spec.md

**Organization**: Tasks grouped by user story for independent implementation.
Each `[US\d+]` group becomes an **epic** Bead in `bd` and a parent issue on GitHub.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (US1, US2, US3)

---

## Phase 1: Setup (Shared Infrastructure)

Tasks with no `[US\d+]` marker are standalone — no epic is created for them.

- [ ] T001 Create project structure and install dependencies
- [ ] T002 [P] Configure linting and CI pipeline

---

## Phase 2: Foundational (Blocking Prerequisites)

- [ ] T003 Initialise bd database with bd init
- [ ] T004 [P] Configure gh CLI authentication
- [ ] T005 [P] Set up .beads/ directory and .gitignore entries

**Checkpoint**: Foundation ready — user story work can now begin

---

## Phase 3: User Story 1 - Import spec-kit tasks (Priority: P1)

`bdim create` reads the nearest `## Phase` header to name the US1 epic:
**"User Story 1: Import spec-kit tasks"**

- [ ] T006 [P] [US1] Implement tasks.md parser in bdim
- [ ] T007 [US1] Map task fields (ID, [P], [US#], status) to Bead fields in bdim
- [ ] T008 [US1] Implement bd create/update idempotency logic in bdim
- [ ] T009 [US1] Add auto-discovery of .specify/specs/*/tasks.md in bdim

**Checkpoint**: bdim create imports tasks from tasks.md into bd

---

## Phase 4: User Story 2 - Sync Beads to GitHub Issues (Priority: P2)

Epic title extracted: **"User Story 2: Sync Beads to GitHub Issues"**

- [ ] T010 [P] [US2] Implement format_gh_body for task Beads in bdim
- [ ] T011 [US2] Implement _sync_one_bead create path in bdim
- [ ] T012 [US2] Implement _sync_one_bead update/skip path in bdim
- [ ] T013 [US2] Write back github_issue_number/url to Bead metadata in bdim

**Checkpoint**: bdim sync pushes Beads to GitHub Issues

---

## Phase 5: User Story 3 - Status and team workflow (Priority: P3)

Epic title extracted: **"User Story 3: Status and team workflow"**

- [ ] T014 [P] [US3] Implement cmd_status table output in bdim
- [ ] T015 [US3] Implement bdim latest (git pull + bd import)
- [ ] T016 [US3] Implement bdim update (bd export + git commit + push)
- [ ] T017 [US3] Document team workflow in README.md

**Checkpoint**: Full pipeline works end-to-end

---

## Phase 6: Polish & Cross-Cutting Concerns

Standalone tasks — no epic.

- [ ] T018 [P] Write end-to-end integration test against test repo
- [ ] T019 Update README with new tasks.md workflow

---

## Dependencies & Execution Order

- **Setup (Phase 1)**: No dependencies
- **Foundational (Phase 2)**: Depends on Setup
- **US1 (Phase 3)**: Depends on Foundational — no other story dependencies
- **US2 (Phase 4)**: Depends on Foundational — integrates with US1
- **US3 (Phase 5)**: Depends on Foundational — integrates with US1 + US2
- **Polish (Phase 6)**: Depends on all stories complete

## What bdim create produces from this file

```
Epic beads (issue_type: epic):
  bd-xxxx1  "User Story 1: Import spec-kit tasks"    [speckit, US1, epic]  P1
  bd-xxxx2  "User Story 2: Sync Beads to GitHub Issues" [speckit, US2, epic]  P2
  bd-xxxx3  "User Story 3: Status and team workflow"  [speckit, US3, epic]  P3

Task beads (issue_type: task):
  bd-yyyy   "Create project structure and install dependencies"  [speckit]  P2
  bd-yyyy   "Configure linting and CI pipeline"                 [speckit, parallel]  P2
  ... (T003–T005, T018–T019 — standalone, no parent)
  bd-yyyy   "Implement tasks.md parser in bdim"  [speckit, US1, parallel]  P1
  ... (T007–T009 linked to US1 epic)
  ... (T010–T013 linked to US2 epic)
  ... (T014–T017 linked to US3 epic)
```
