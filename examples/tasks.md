---

description: "Task list template for feature implementation"
---

# Tasks: TaskToBeads Demo

**Input**: `.specify/specs/001-task-to-beads/`
**Prerequisites**: plan.md, spec.md

**Organization**: Tasks grouped by user story for independent implementation.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (US1, US2, US3)

---

## Phase 1: Setup (Shared Infrastructure)

- [ ] T001 Create project structure and install dependencies
- [ ] T002 [P] Configure linting and CI pipeline

---

## Phase 2: Foundational (Blocking Prerequisites)

- [ ] T003 Initialise bd database with bd init
- [ ] T004 [P] Configure gh CLI authentication
- [ ] T005 [P] Set up .beads/ directory and .gitignore entries

**Checkpoint**: Foundation ready — user story work can now begin

---

## Phase 3: User Story 1 - Import spec-kit tasks (Priority: P1) 🎯 MVP

**Goal**: Read tasks.md from .specify/specs/ and create Beads in bd

**Independent Test**: Run bdim speckit-to-beads --dry-run and verify tasks are parsed correctly

- [ ] T006 [P] [US1] Implement tasks.md parser in bdim
- [ ] T007 [US1] Map task fields (ID, [P], [US#], status) to Bead fields in bdim
- [ ] T008 [US1] Implement bd create/update idempotency logic in bdim
- [ ] T009 [US1] Add auto-discovery of .specify/specs/*/tasks.md in bdim

**Checkpoint**: bdim speckit-to-beads creates Beads from tasks.md

---

## Phase 4: User Story 2 - Sync Beads to GitHub Issues (Priority: P2)

**Goal**: Push Beads to GitHub Issues with idempotent sync

**Independent Test**: Run bdim sync --repo owner/repo --dry-run and verify issues would be created

- [ ] T010 [P] [US2] Implement format_gh_body for task Beads in bdim
- [ ] T011 [US2] Implement _sync_one_bead create path in bdim
- [ ] T012 [US2] Implement _sync_one_bead update/skip path in bdim
- [ ] T013 [US2] Write back github_issue_number/url to Bead metadata in bdim

**Checkpoint**: bdim sync pushes Beads to GitHub Issues

---

## Phase 5: User Story 3 - Status and team workflow (Priority: P3)

**Goal**: Show sync state table; support team git workflow via JSONL export

**Independent Test**: Run bdim status and verify table output

- [ ] T014 [P] [US3] Implement cmd_status table output in bdim
- [ ] T015 [US3] Implement bd_export_jsonl for team sharing in bdim
- [ ] T016 [US3] Document team workflow in README.md

**Checkpoint**: Full pipeline works end-to-end

---

## Phase 6: Polish & Cross-Cutting Concerns

- [ ] T017 [P] Write end-to-end integration test against test repo
- [ ] T018 Update README with new tasks.md workflow

---

## Dependencies & Execution Order

- **Setup (Phase 1)**: No dependencies
- **Foundational (Phase 2)**: Depends on Setup
- **US1 (Phase 3)**: Depends on Foundational — no other story dependencies
- **US2 (Phase 4)**: Depends on Foundational — integrates with US1
- **US3 (Phase 5)**: Depends on Foundational — integrates with US1 + US2
- **Polish (Phase 6)**: Depends on all stories complete
