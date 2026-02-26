# beadsync

A macOS CLI tool that bridges **Speckit → Beads → GitHub Issues**.

Convert your Speckit task exports into real [Beads](https://github.com/steveyegge/beads) using the `bd` CLI, then push them to GitHub Issues — with full idempotency, project scoping, and a clear source-of-truth model.

---

## How it works

```
Speckit export (JSON)
        │
        ▼
beadsync speckit-to-beads          ← Phase 1
        │
        ▼
  bd (Bead database)               ← Source of truth after this point
        │
        ▼
beadsync sync                      ← Phase 2
        │
        ▼
  GitHub Issues                    ← Read-only projection
```

**Source of truth rules:**
- Speckit is canonical _until_ Beads are created in `bd`
- After Beads exist in `bd`, **Beads are the source of truth**
- GitHub Issues are downstream projections — edits there do not flow back into Beads (unless `--force` is used)

---

## Requirements

```bash
brew install jq gh
gh auth login

# Install the Beads CLI (bd)
npm i -g @beads/bd
# or: brew install beads
```

- `bash` (any version shipped with macOS works)
- `jq` — JSON processing
- `gh` — GitHub CLI (only needed for `beads-to-gh` / `sync`)
- `bd` — Beads CLI (required for all commands; manages the Bead database)

### First-time project setup

In your project repo, add this to `.gitignore` to exclude the binary Dolt database while keeping the git-trackable JSONL:

```
# bd (Beads CLI) — binary database, never commit
.beads/dolt/
.beads/*.db
```

Then run `bd init` in your project root to initialise the bead database.

---

## Installation

```bash
# Clone the repo
git clone https://github.com/Papanii/beadsync.git
cd beadsync

# Install globally (no sudo needed)
mkdir -p ~/.local/bin
ln -sf "$PWD/beadsync" ~/.local/bin/beadsync

# Add to PATH (add this to your ~/.zshrc or ~/.bashrc)
export PATH="$HOME/.local/bin:$PATH"

# Reload shell
source ~/.zshrc
```

After this, `beadsync` works from any directory on your machine.

---

## Commands

### `speckit-to-beads` — Convert Speckit export to Beads

```bash
beadsync speckit-to-beads <speckit_export.json> [options]
```

Reads each task from the Speckit export and calls `bd create` (or `bd update` if the Bead already exists). All Beads are tagged with the `speckit` label so they can be filtered by downstream commands. Fully idempotent — re-running only touches tasks that have changed.

| Option | Default | Description |
|---|---|---|
| `--project, -p ID` | auto-detected | Project namespace for Bead `external_ref` values |
| `--dry-run` | off | Preview what would be created/updated without calling `bd` |
| `--verbose, -v` | off | Print debug output |

---

### `beads-to-gh` — Push Beads to GitHub Issues

```bash
beadsync beads-to-gh --repo owner/name [options]
```

Reads all Beads tagged `speckit` from the local `bd` database and pushes them to GitHub Issues. Creates new issues for unsynced Beads and updates existing issues when content has changed.

| Option | Default | Description |
|---|---|---|
| `--repo, -r REPO` | _(required)_ | GitHub repo, e.g. `acme/my-project` |
| `--force` | off | Override manual GitHub edits; always push Bead content |
| `--dry-run` | off | Preview without touching GitHub |
| `--verbose, -v` | off | Print debug output |

---

### `sync` — Idempotent sync of changed Beads to GitHub

```bash
beadsync sync --repo owner/name [options]
```

Alias for `beads-to-gh`. Only pushes Beads whose content has changed since the last sync. Same options as `beads-to-gh`.

---

### `status` — Show sync state of all Beads

```bash
beadsync status
```

Prints a table of every Bead (label=`speckit`) in the `bd` database, its status, GitHub issue number, and whether it needs syncing. Read-only — makes no changes.

---

## Usage examples

### Simplest — convert and push in two steps

```bash
beadsync speckit-to-beads export.json
beadsync beads-to-gh --repo acme/my-project
```

### Always preview before writing

```bash
# See what Beads would be created in bd
beadsync speckit-to-beads export.json --dry-run

# See what GitHub issues would be created/updated
beadsync beads-to-gh --repo acme/my-project --dry-run
```

### Multiple projects on the same machine

Each project's Beads are namespaced by their `external_ref` (`speckit-<project>-<id>`), so task IDs never collide across projects. `beadsync` auto-reads the project name from your Speckit export's `project.id` field.

```bash
# Project alpha
cd ~/projects/alpha
beadsync speckit-to-beads export.json
beadsync sync --repo acme/alpha

# Project beta — same task IDs, no collision
cd ~/projects/beta
beadsync speckit-to-beads export.json
beadsync sync --repo acme/beta
```

To set the project name explicitly (useful if your export has no `project.id`):

```bash
beadsync speckit-to-beads export.json --project my-project-name
```

### Check sync state at any time

```bash
beadsync status
```

```
Bead ID              Title                                            Status         GH Issue     Synced?
──────────────────────────────────────────────────────────────────────────────────────────────────────────────
bd-a1b2              Set up monorepo structure                        closed         #12          up to date
bd-c3d4              Implement Speckit JSON parser                    in_progress    #13          needs update
bd-e5f6              Design Bead file schema                          open           #-           not synced

Total: 3 | Synced: 1 | Needs sync: 2
```

### Ongoing sync — run whenever Beads change

```bash
beadsync sync --repo acme/my-project
```

Re-running is fully idempotent: unchanged Beads are skipped, changed Beads are updated, no duplicates are ever created.

### Force-push all Beads (override manual GitHub edits)

```bash
beadsync sync --repo acme/my-project --force
```

### Full example with all options

```bash
beadsync speckit-to-beads export.json \
  --project my-project \
  --dry-run \
  --verbose

beadsync sync \
  --repo acme/my-project \
  --force \
  --dry-run \
  --verbose
```

---

## Speckit export format

`beadsync` reads a JSON file with this structure:

```json
{
  "project": {
    "id": "my-project",
    "name": "My Project"
  },
  "tasks": [
    {
      "id":                   "SPEC-001",
      "title":                "Implement login",
      "description":          "OAuth2 flow with JWT tokens",
      "acceptance_criteria":  "User can log in with Google",
      "notes":                "See ADR-004",
      "priority":             "high",
      "status":               "in_progress",
      "type":                 "feature",
      "assignee":             "alice",
      "labels":               ["auth", "backend"],
      "milestone":            "v1.0",
      "estimated_hours":      6,
      "due_date":             "2026-03-15",
      "parent_id":            null,
      "created_at":           "2026-01-10T09:00:00Z",
      "updated_at":           "2026-01-20T14:00:00Z"
    }
  ]
}
```

**Required fields per task:** `id`, `title`
**All other fields are optional.**

See [`examples/speckit_export.json`](examples/speckit_export.json) for a complete sample with 6 tasks.

---

## Field mapping

### Speckit → Bead (`bd`)

| Speckit field | Bead field | Notes |
|---|---|---|
| `id` | `external_ref` | `"speckit-<project>-<id>"` — stable cross-system identity |
| `title` | `title` | |
| `description` | `description` | |
| `acceptance_criteria` | `acceptance_criteria` | |
| `notes` | `notes` | |
| `priority` | `priority` (int) | `critical`→0, `high`→1, `medium`→2, `low`→3, `none`→4 |
| `status` | `status` | `pending`→`open`, `done`→`closed`, etc. |
| `type` | `issue_type` | `feature`→`feature`, `task`→`task`, etc. |
| `assignee` | `assignee` | |
| `labels[]` | `labels[]` | Always includes `speckit` as a base label |
| `milestone` | `metadata.milestone` | Used for GitHub milestone lookup |
| `estimated_hours` | `estimated_minutes` | Multiplied by 60 |
| `due_date` | `due_at` | `T00:00:00Z` appended if date-only |

### Bead → GitHub Issue

| Bead field | GitHub field | Notes |
|---|---|---|
| `title` | title | |
| `description` + `acceptance_criteria` + `notes` | body | Composed as markdown sections |
| `labels[]` | labels | Labels are created in the repo if missing |
| `assignee` | assignees | Single assignee |
| `metadata.milestone` | milestone | Resolved to GitHub milestone number |
| `status == "closed"` | state = `closed` | Open otherwise |
| — | body (HTML comment) | Hidden idempotency anchor |

---

## Team workflow (multi-machine)

beadsync automatically exports `.beads/issues.jsonl` after every write command (`speckit-to-beads` and `sync`). This JSONL file is one bead per line — plain text, git-diffable, and auto-mergeable when different beads are modified concurrently.

**Never commit `.beads/dolt/`** — it's a binary Dolt database that will produce unresolvable merge conflicts. Only commit `issues.jsonl`.

```
.beads/
  dolt/           ← binary — in .gitignore
  *.db            ← binary — in .gitignore
  issues.jsonl    ← text, one bead per line — commit this
```

### Daily workflow

```bash
# 1. Get the latest beads from the shared repo
git pull
bd import -i .beads/issues.jsonl    # rebuild your local Dolt DB

# 2. Do your work
beadsync speckit-to-beads export.json          # auto-exports issues.jsonl
beadsync sync --repo acme/my-project           # auto-exports issues.jsonl

# 3. Share the updated bead database
git add .beads/issues.jsonl
git commit -m "Update beads from Speckit export"
git push
```

### Why concurrent conflicts are rare

Since Speckit drives all bead creation, two people running `speckit-to-beads` from the same export produce identical output — the second run hits "skipped" on every bead. The only real conflict risk is manual `bd update` edits outside of beadsync, which follow normal git merge resolution on the JSONL.

---

## How idempotency works

Every Bead stores two hashes in its `metadata` (via `bd update --metadata`):

- **`content_hash`** — SHA-256 of the Bead's key fields. Recomputed each time `speckit-to-beads` runs. If the source task hasn't changed, the Bead is not touched.

- **`github_issue_synced_hash`** — the `content_hash` at the time of the last successful GitHub sync. If these two hashes differ, the issue is updated on the next `sync` run.

Every GitHub issue body also contains a hidden HTML comment:
```html
<!-- beadsync:external_id:speckit-my-project-SPEC-001 -->
```

On `sync`, this anchor is used to find the matching issue so it is updated rather than recreated, even if the `bd` database is wiped and rebuilt from scratch.

---

## Exit codes

| Code | Meaning |
|---|---|
| `0` | Success |
| `1` | Runtime error (e.g. a bead failed to sync) |
| `2` | Usage / argument error |

---

## Environment variables

| Variable | Purpose |
|---|---|
| `GITHUB_TOKEN` | Optional — `gh` CLI manages auth by default via `gh auth login` |

---

## Project structure

```
beadsync                  ← the CLI script (single file)
examples/
  speckit_export.json     ← sample Speckit export with 6 tasks
```
