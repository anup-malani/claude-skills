# Setup Guide: Four-Layer Memory Architecture

How to wire up the four-layer memory architecture in a new Claude Code project. Estimated time: 20 minutes for the directory structure and files; 30 additional minutes if you are also setting up the SessionEnd hook and QMD server.

## Prerequisites

- Claude Code installed and running
- A project directory, referred to below as `PROJECT_ROOT`
- QMD MCP server registered for Layer 4. See the QMD server documentation for installation.
- Claude Code hooks documentation: https://docs.claude.com/en/docs/claude-code/hooks

---

## Step 1: Directory Structure

Create the following structure under `PROJECT_ROOT`:

```
PROJECT_ROOT/
├── CLAUDE.md                  # Project instructions, if not already present
├── MEMORY.md                  # Layer 1 project index, or symlink (see Step 4)
├── memory/                    # Layer 1 project topic files
│   └── .gitkeep
├── decisions/                 # Layer 2 decision files
│   └── .gitkeep
├── session-log.md             # Layer 3 ledger
└── .claude/
    └── settings.json          # Claude Code project settings
```

```bash
mkdir -p PROJECT_ROOT/memory
mkdir -p PROJECT_ROOT/decisions
touch PROJECT_ROOT/memory/.gitkeep
touch PROJECT_ROOT/decisions/.gitkeep
```

The global Layer 1 file, `~/.claude/CLAUDE.md`, already exists if you use Claude Code. Edit it in place to add global rules.

---

## Step 2: Initial File Templates

### `~/.claude/CLAUDE.md` (global Layer 1)

Your global file likely already exists. Add sections as needed. The template below shows the shape of a minimal global file; the specific rules inside are illustrative examples, not required content. Substitute your own.

```markdown
# User Profile

For stable facts about the user, read `~/.claude/bio.md` on demand. Do not read it every session.

# Email sending
# (illustrative example of an always-on safety rule. Keep it, replace it, or remove it.)

Never send an email on the user's behalf without showing the draft in the conversation
and receiving explicit approval in the conversation. Show the full text, wait for
approval or edits, then call the send tool. Do not create a draft in the mail app and
tell the user to press send themselves; send it from within the session after approval.

# [Additional global sections as needed]
```

Keep this file lean. When a section grows past a dozen lines, offload to an on-demand file (`~/.claude/bio.md`, `~/.claude/style.md`, etc.) and reference it with an on-demand instruction.

### `MEMORY.md` (project Layer 1)

```markdown
# [Project Name] Memory

_Index file. Keep entries to one line (~200 chars max). Move detail into memory/*.md._
_Exceeds 200 lines? Prune by moving detail to topic files._

<!-- Add index entries below as they accumulate. -->
```

### `session-log.md`

```markdown
# Session Log

_Append-only. One entry per session. Most recent at bottom, or top, pick one and keep it._

---

## YYYY-MM-DD HH:MM: [Session title]

**Summary:** What happened in this session.

**Inputs:** Files read, data pulled, papers reviewed.

**Outputs:** Files created or modified, actions taken.

**Decisions and Rationale:** Any decisions made. If promoted to decisions/, note the file.

**Open Items:**
- [ ] Placeholder
```

### `decisions/.gitkeep`

The `decisions/` directory starts empty. Create decision files as needed. See the template in README.md.

---

## Step 3: Wire Project `MEMORY.md` into Every Session

Claude Code loads files listed in `CLAUDE.md` or referenced in `.claude/settings.json`. The simplest approach is to add an explicit instruction to your project `CLAUDE.md`:

```markdown
## Session Memory Protocol

**On session start:** Read `MEMORY.md` and `session-log.md` (last 3 to 5 entries) before beginning work.

**On session end:** Run `/mexit`, or `/log` for a mid-session checkpoint.
Do not exit without logging if the session involved substantive work.
```

If you want to enforce auto-loading via settings rather than instruction, add the file to your project's `.claude/settings.json`. Check the Claude Code docs for the current syntax. Include-field names change between versions.

---

## Step 4: Cross-Machine Setup (Dropbox or Shared Sync)

If the project directory is shared between two machines via Dropbox or similar sync:

On each machine, create a per-machine memory file and symlink:

```bash
# On MacBookPro:
touch PROJECT_ROOT/memory-macbook.md
ln -s PROJECT_ROOT/memory-macbook.md PROJECT_ROOT/MEMORY.md

# On MacMini, or whatever the second machine is called:
touch PROJECT_ROOT/memory-mini.md
ln -s PROJECT_ROOT/memory-mini.md PROJECT_ROOT/MEMORY.md
```

Add to `CLAUDE.md`:

```markdown
## Cross-Machine Memory

MEMORY.md is symlinked to this machine's file. On session start, also read the other
machine's memory file from the project directory if it exists (memory-macbook.md or
memory-mini.md, whichever is NOT this machine's). On session end, write only to
MEMORY.md (the symlink target). Never write to the other machine's file.
```

Why: Dropbox sync conflicts occur when two machines write the same file concurrently. Per-machine files with a symlink give each machine exclusive write ownership while preserving read access to both.

---

## Step 5: SessionEnd Hook (Layer 4)

The SessionEnd hook exports each session's transcript to markdown and re-indexes QMD automatically. Wire it up in `.claude/settings.json`:

```json
{
  "hooks": {
    "SessionEnd": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "/path/to/your/session-export-script.sh"
          }
        ]
      }
    ]
  }
}
```

A minimal export script (`session-export-script.sh`):

```bash
#!/bin/bash
# Export the session transcript to a dated markdown file and re-index QMD.
# Claude Code passes the session transcript path as an environment variable.
# Check the Claude Code hooks docs for the exact variable name in your version:
# https://docs.claude.com/en/docs/claude-code/hooks

set -e

TRANSCRIPT_PATH="${CLAUDE_SESSION_TRANSCRIPT:-}"
EXPORT_DIR="${HOME}/.claude/session-exports"
DATE=$(date +%Y-%m-%d-%H%M)

if [ -z "$TRANSCRIPT_PATH" ]; then
  echo "No transcript path available. Skipping export." >&2
  exit 0
fi

mkdir -p "$EXPORT_DIR"
EXPORT_FILE="$EXPORT_DIR/session-${DATE}.md"

cp "$TRANSCRIPT_PATH" "$EXPORT_FILE"

# Re-index QMD if the CLI is available.
# Replace with your actual QMD re-index command.
if command -v qmd &>/dev/null; then
  qmd update
fi
```

Make it executable:

```bash
chmod +x /path/to/your/session-export-script.sh
```

**Note on variable names:** The exact environment variable name for the transcript path (`CLAUDE_SESSION_TRANSCRIPT` above) may differ in your Claude Code version. Consult the hooks documentation at https://docs.claude.com/en/docs/claude-code/hooks for the current variable names and the hook invocation contract.

### Periodic embedding for semantic search

`qmd update` re-indexes new transcripts for BM25 keyword search. Semantic and vector search requires a separate embedding pass. Run `qmd embed` periodically to keep the vector index current. Options:

- **Manual:** Run `qmd embed` yourself weekly, or whenever you notice recent sessions are findable only by keyword.
- **Scheduled:** Add a cron or launchd entry to run `qmd embed` nightly, or whenever the machine is idle. Example launchd plist (`~/Library/LaunchAgents/com.user.qmd-embed.plist`):

  ```xml
  <?xml version="1.0" encoding="UTF-8"?>
  <!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
  <plist version="1.0">
  <dict>
      <key>Label</key>
      <string>com.user.qmd-embed</string>
      <key>ProgramArguments</key>
      <array>
          <string>/path/to/qmd</string>
          <string>embed</string>
      </array>
      <key>StartCalendarInterval</key>
      <dict>
          <key>Hour</key>
          <integer>3</integer>
          <key>Minute</key>
          <integer>0</integer>
      </dict>
  </dict>
  </plist>
  ```

  Then: `launchctl load ~/Library/LaunchAgents/com.user.qmd-embed.plist`

Without `qmd embed`, newly exported sessions are findable by keyword immediately but will not surface in semantic search until the next embed run.

---

## Step 6: QMD Prompt Instruction (Layer 4 Discoverability)

Without an explicit instruction, Claude will not search QMD even when it would help. Add to your `CLAUDE.md`, either globally or per-project:

```markdown
## QMD Session Search

When the user says "we did this before," "how did we set up X," or references past work
you do not have context for: search QMD (collection `sessions-md`) before saying the
information is unavailable.

BM25 keyword search (`mcp__qmd__search`) is fast. Semantic search (`mcp__qmd__query`)
finds conceptually related content. Use keyword search first.
```

---

## Step 7: Skill References

The architecture relies on three skills for the human-in-the-loop memory layer:

| Skill | When to run | What it does |
|---|---|---|
| `/log` | Mid-session checkpoint | Appends a structured entry to `session-log.md` |
| `/memory` | Session end | Curates `MEMORY.md`: prunes, promotes, updates index |
| `/mexit` | Session end (preferred) | Runs `/log` then `/memory`, then signs off |

These skills need to be defined in `.claude/` as skill files. If you are using the `claude-skills` repository, copy or reference the relevant skill files. If you are writing them from scratch, each skill is a markdown file in `.claude/skills/` that instructs Claude on the procedure.

Minimal skill stub for `/log` (`.claude/skills/log.md`):

```markdown
# /log

Append a structured session entry to `session-log.md`.

Entry format:
## YYYY-MM-DD HH:MM: [one-line title]

**Summary:** [2 to 3 sentences]
**Inputs:** [files read, data pulled]
**Outputs:** [files created or changed, actions taken]
**Decisions and Rationale:** [decisions made; note decisions/ file if promoted]
**Open Items:**
- [ ] [pending task]

Use today's date and current time. Infer the session title from the main task completed.
Append the entry to the bottom of session-log.md.
```

Adapt similarly for `/memory` (prune and update `MEMORY.md`) and `/mexit` (run both in sequence).

---

## Step 8: First Decision File

When you make the first real architectural or policy decision for the project, create:

```
decisions/YYYY-MM-DD-[topic-slug].md
```

Use the template from README.md. Add a one-line pointer to MEMORY.md:

```
## [Topic] (Added YYYY-MM-DD)
See [decisions/YYYY-MM-DD-topic-slug.md](decisions/YYYY-MM-DD-topic-slug.md). [One-line summary of the decision.]
```

---

## Verification Checklist

After setup, verify:

- [ ] `~/.claude/CLAUDE.md` edits are loaded in a new session (test by asking Claude to state a global rule you added)
- [ ] Project `MEMORY.md` is read at session start (manually confirm in first session)
- [ ] `session-log.md` gets an entry after the first working session (run `/log`)
- [ ] At least one `decisions/` file exists after the first real decision
- [ ] SessionEnd hook fires at session end (check export directory for transcript files)
- [ ] QMD keyword search returns results from a past session
- [ ] QMD semantic search returns results after `qmd embed` has been run
- [ ] Cross-machine: both machine memory files are readable, only local file is written

---

## License

MIT. Part of the [claude-skills](https://github.com/anup-malani/github/anup-malani/claude-skills) repository.
