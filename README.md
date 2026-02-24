# beadsync

A macOS CLI tool that bridges **Speckit → Beads → GitHub Issues**.

Convert your Speckit task exports into structured [Beads](https://github.com/steveyegge/beads) files, then push them to GitHub Issues — with full idempotency, project scoping, and a clear source-of-truth model.

---

## How it works

```
Speckit export (JSON)
        │
        ▼
beadsync speckit-to-beads          ← Phase 1
        │
        ▼
  Bead files (./beads/*.json)      ← Source of truth after this point
        │
        ▼
beadsync sync                      ← Phase 2
        │
        ▼
  GitHub Issues                    ← Read-only projection
```

**Source of truth rules:**
- Speckit is canonical _until_ Beads are generated
- After Beads exist, **Beads are the source of truth**
- GitHub Issues are downstream projections — edits there do not flow back into Beads (unless `--force` is used)

---

## Requirements

```bash
brew install jq gh
gh auth login
```

- `bash` (any version shipped with macOS works)
- `jq` — JSON processing
- `gh` — GitHub CLI (only needed for `beads-to-gh` / `sync`)

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

### `speckit-to-beads` — Convert Speckit export to Bead files

```bash
beadsync speckit-to-beads <speckit_export.json> [options]
```

| Option | Default | Description |
|---|---|---|
| `--out, -o DIR` | `./beads` | Output directory for Bead files |
| `--project, -p ID` | auto-detected | Project namespace for Bead IDs |
| `--dry-run` | off | Preview what would be created/updated |
| `--verbose, -v` | off | Print debug output |

---

### `beads-to-gh` — Push Bead files to GitHub Issues

```bash
beadsync beads-to-gh <beads_dir> --repo owner/name [options]
```

| Option | Default | Description |
|---|---|---|
| `--repo, -r REPO` | _(required)_ | GitHub repo, e.g. `acme/my-project` |
| `--force` | off | Override manual GitHub edits; always push Bead content |
| `--dry-run` | off | Preview without touching GitHub |
| `--verbose, -v` | off | Print debug output |

---

### `sync` — Idempotent sync of changed Beads to GitHub

```bash
beadsync sync <beads_dir> --repo owner/name [options]
```

Same options as `beads-to-gh`. Only pushes Beads whose content has changed since the last sync.

---

### `status` — Show sync state of all Beads

```bash
beadsync status [beads_dir]
```

Prints a table of every Bead file, its status, GitHub issue number, and whether it needs syncing. Read-only — makes no changes.

---

## Usage examples

### Simplest — convert and push in two steps

```bash
beadsync speckit-to-beads export.json
beadsync beads-to-gh ./beads --repo acme/my-project
```

### Always preview before writing

```bash
# See what Beads would be created
beadsync speckit-to-beads export.json --dry-run

# See what GitHub issues would be created/updated
beadsync beads-to-gh ./beads --repo acme/my-project --dry-run
```

### Custom output directory

```bash
beadsync speckit-to-beads export.json --out ./tasks/beads
beadsync sync ./tasks/beads --repo acme/my-project
```

### Multiple projects on the same machine

Each project gets its own Bead directory. `beadsync` auto-reads the project name
from your Speckit export's `project.id` field, so IDs never collide across projects.

```bash
# Project alpha
cd ~/projects/alpha
beadsync speckit-to-beads export.json --out ./beads
beadsync sync ./beads --repo acme/alpha

# Project beta — same task IDs, no collision
cd ~/projects/beta
beadsync speckit-to-beads export.json --out ./beads
beadsync sync ./beads --repo acme/beta
```

To set the project name explicitly (useful if your export has no `project.id`):

```bash
beadsync speckit-to-beads export.json --project my-project-name
```

### Check sync state at any time

```bash
beadsync status ./beads
```

```
Bead file                                  Status       GH Issue     Synced?
──────────────────────────────────────────────────────────────────────────────
bead-my-project-SPEC-001.json              closed       #12          up to date
bead-my-project-SPEC-002.json              in_progress  #13          needs update
bead-my-project-SPEC-003.json              open         #-           not synced

Total: 3 | Synced: 1 | Needs sync: 2
```

### Ongoing sync — run whenever Beads change

```bash
beadsync sync ./beads --repo acme/my-project
```

Re-running is fully idempotent: unchanged Beads are skipped, changed Beads are updated, no duplicates are ever created.

### Force-push all Beads (override manual GitHub edits)

```bash
beadsync sync ./beads --repo acme/my-project --force
```

### Full example with all options

```bash
beadsync speckit-to-beads export.json \
  --out ./beads \
  --project my-project \
  --dry-run \
  --verbose

beadsync sync ./beads \
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

### Speckit → Bead

| Speckit field | Bead field | Notes |
|---|---|---|
| `id` | `spec_id`, `external_id` | `external_id = "speckit-<project>-<id>"` |
| `title` | `title` | |
| `description` | `description` | |
| `acceptance_criteria` | `acceptance_criteria` | |
| `notes` | `notes` | |
| `priority` | `priority` (int) | `critical`→0, `high`→1, `medium`→2, `low`→3, `none`→4 |
| `status` | `status` | `pending`→`open`, `done`→`closed`, etc. |
| `type` | `issue_type` | `feature`→`feature`, `task`→`task`, etc. |
| `assignee` | `assignee` | |
| `labels[]` | `labels[]` | |
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

## How idempotency works

Every Bead contains two hashes in its `metadata`:

- **`content_hash`** — SHA-256 of the Bead's key fields. Recomputed each time `speckit-to-beads` runs. If the source task hasn't changed, the Bead file is not touched.

- **`github_issue_synced_hash`** — the `content_hash` at the time of the last successful GitHub sync. If these two hashes differ, the issue is updated on the next `sync` run.

Every GitHub issue body contains a hidden HTML comment:
```html
<!-- beadsync:external_id:speckit-my-project-SPEC-001 -->
```

On `sync`, this anchor is used to find the matching issue so it's updated rather than recreated, even if the Bead file has been deleted and re-created from scratch.

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
beadsync                  ← the CLI script (single file, no dependencies beyond jq + gh)
examples/
  speckit_export.json     ← sample Speckit export with 6 tasks
```
