# Four-Layer Memory Architecture for Claude Code Projects

A pattern for giving Claude long-running context without blowing your context window. Each layer answers a distinct question with a distinct access pattern. The layers are not redundant. They are four different trade-offs between cost and recall latency.

## The Core Insight

**Access pattern is the differentiator, not content type.**

| Layer | File(s) | Question answered | Access pattern | Token cost |
|---|---|---|---|---|
| 1 | `~/.claude/CLAUDE.md` (global) + `MEMORY.md` + `memory/*.md` (project) | What is true right now? | Always in context | Every session |
| 2 | `decisions/YYYY-MM-DD-topic.md` | Why did we choose X over Y? | Cited on demand | Zero default |
| 3 | `session-log.md` | What did we ship? What is still open? | Read at session start | Modest, scoped |
| 4 | QMD `sessions-md` collection | Where did I do X before? | Searched on demand | Zero default |

You can collapse layers to save setup time. The failure modes are predictable and documented below.

---

## Layer 1: Curated Facts (Two Scopes)

**Files:**

- **Global:** `~/.claude/CLAUDE.md`. Rules and preferences that apply across every project and every session.
- **Project:** `MEMORY.md` (index) plus `memory/*.md` (topic files). Facts specific to a single project.

**Question:** What is true right now?

**Access pattern:** Both files are auto-loaded into every Claude Code session. The global file loads in every session regardless of directory. The project `MEMORY.md` loads when Claude Code runs in that project directory (via the auto-memory system, or by explicit reference from the project's own `CLAUDE.md`).

**Writer:** Human-in-the-loop. For global rules, edit `~/.claude/CLAUDE.md` directly. For project facts, use the `/memory` skill at session end, or `/mexit` which bundles `/memory` with `/log`.

**What goes in `~/.claude/CLAUDE.md` (global):**

- Editorial and voice preferences that apply everywhere
- Always-on safety rules (example: "never send an email without showing me the draft in the conversation and waiting for explicit approval")
- Authentication and access protocols that span projects
- References to MCP servers and external integrations you want every session to see

**What goes in project `MEMORY.md`:**

- Current state of project-specific integrations (API keys, endpoint versions confirmed working)
- Settled decisions summarized in one line with a pointer to the full `decisions/` file
- Project-specific constraints (sensitive topics, format rules, editorial standards for this voice)
- Recurring user preferences that matter only in this project

**Prune rule (applies to both):** When a file exceeds roughly 200 lines, context cost grows every session. For project `MEMORY.md`, move detail into `memory/topic-name.md` files and replace the section with a one-liner pointer. Example:

```
## Typefully API Migration (Added 2026-04-20)
See [typefully-migration.md](memory/typefully-migration.md). Summary: v1 deprecated June 15; v2 key + header swap required.
```

For `~/.claude/CLAUDE.md`, the same pattern works. Offload detail to `~/.claude/bio.md`, `~/.claude/style.md`, or topic-specific files referenced from the main file by on-demand instruction (example: "For stable facts about the user, read `~/.claude/bio.md` on demand").

**Cross-machine note:** If two machines share the project directory via Dropbox or similar sync, do not share a single `MEMORY.md` file. Concurrent writes will corrupt it. Keep per-machine files (`memory-macbook.md`, `memory-mini.md`) and symlink `MEMORY.md` to the current machine's file. At session start, read both files for full context. At session end, write only to the symlinked (local) file.

---

## Layer 2: Decision Log

**Files:** `decisions/YYYY-MM-DD-topic-slug.md`

**Question:** Why did we choose X over Y? What alternatives did we rule out?

**Access pattern:** Not auto-loaded. Referenced from `MEMORY.md` index entries or `session-log.md` when the one-line decision summary is not enough.

**Writer:** Human, or assistant under human review, at decision time. Append-only. Never delete a decision file. If a decision is superseded, mark it at the top:

```markdown
> **Superseded by:** [decisions/2026-06-01-new-topic.md](2026-06-01-new-topic.md)
> The v1 approach below is preserved for historical reference.
```

**File structure:**

```markdown
# Decision: [title]

**Date:** YYYY-MM-DD
**Status:** active | superseded | under review

## Context
What situation prompted this decision? What was at stake?

## Options Considered

### Option A: [name]
- Pros: ...
- Cons: ...
- Verdict: rejected / selected / deferred

### Option B: [name]
- Pros: ...
- Cons: ...
- Verdict: ...

## Decision
One clear statement of what was chosen.

## Rationale
Why this option over the others? What tipped the balance?

## Implementation
Concrete changes made or required.

## Risks
What could go wrong? What are the known downsides?

## Review Date
YYYY-MM-DD, or "n/a, permanent policy"
```

**What goes here:** Any decision where someone three months from now would want the reasoning. Architectural choices, vendor selections, format standards, editorial policies, access-control decisions, anything with real alternatives considered.

**Promotion rule (from Layer 3 to Layer 2):** Not every decision recorded in `session-log.md` deserves its own file. Promote to `decisions/` if:

- Someone three months from now would want the reasoning
- The alternatives considered are worth preserving
- The decision sets policy, not a one-off tactic
- It is likely to be re-questioned

Keep in `session-log.md` if:

- Tactical (which slot to schedule, which file to rename)
- Self-evident once the session context is seen
- One-off and not reusable policy

**The failure mode this prevents:** Without a decision log, past decisions get re-litigated. Claude re-proposes alternatives that were already considered and rejected. The team re-argues a settled question because nobody wrote down why they settled it.

---

## Layer 3: Session Ledger

**File:** `session-log.md` (single file, chronological, append-only)

**Question:** What did we ship recently? What is still open?

**Access pattern:** Read at session start (recent N entries, typically the last 3 to 5 sessions). Written at session end.

**Writer:** Assistant, via the `/log` skill (checkpoint during a session) or `/mexit` (end of session, runs both `/log` and `/memory`).

**Entry format:**

```markdown
## YYYY-MM-DD HH:MM: [one-line title]

**Summary:** 2 to 3 sentences on what happened.

**Inputs:** Files read, papers reviewed, data pulled.

**Outputs:** Files created or modified, drafts scheduled, scripts deployed.

**Decisions and Rationale:** Decisions made this session. If major, note the decisions/ file where it was promoted.

**Open Items:**
- [ ] Pending task 1
- [ ] Pending task 2
```

**What goes here:** Every working session. Not just the ones that produced something. A session that diagnosed a bug and found nothing is worth logging, especially the "found nothing" part.

**Read at session start:** The session-log gives the resuming assistant, or human, enough context to pick up work without re-reading the full project. Recent open items are especially important: they are the things most likely to need action.

**The failure mode this prevents:** Without a session ledger, every session starts from zero. The assistant re-asks questions that were resolved. Open items fall through. Work done in one session is invisible to the next.

---

## Layer 4: Raw Transcripts (QMD)

**System:** QMD MCP server, `sessions-md` collection

**Question:** Where exactly did I do X before? What was the exact command, error string, or quote?

**Access pattern:** Searched on demand. Not auto-loaded. Use BM25 keyword search for fast exact-phrase lookup; use semantic or vector search for conceptually related content.

**Writer:** SessionEnd hook (automatic). At the end of each Claude Code session, a hook exports the full conversation transcript to markdown and re-indexes it into QMD. No manual action required.

**When to search Layer 4:**

- User says "we did this before" or "how did we set up X"
- You need the exact error message from a past debugging session
- You need to understand a prior decision that predates the current `decisions/` files
- You are resuming work after a long gap and want the long-tail detail that did not make it into `MEMORY.md`

**When not to search Layer 4:** If the fact is already in `MEMORY.md` or `session-log.md`, those are higher-signal and faster. Layer 4 is for the long tail.

**What goes here:** Everything. The full unstructured transcript of every session. This is the source of last resort, slow to search but complete.

**QMD maintenance note:** Newly exported sessions are indexed for BM25 keyword search immediately but require a separate embedding pass to become searchable via semantic or vector queries. Run `qmd embed` periodically (or on a schedule via launchd or cron) to keep the vector index current. Without this, recent sessions are findable only by keyword until the next embed run.

**The failure mode this prevents:** Long-tail details, exact commands, error strings, specific numbers mentioned once in passing, are irrecoverable without a searchable transcript store. `MEMORY.md` cannot hold everything. Layer 4 is the safety net.

---

## Layer Linkages (Not Duplication)

The four layers are connected by explicit cross-references, not by copying content between them.

**`MEMORY.md` index entry pointing to a decision file:**

```
## API Vendor Choice (Added 2026-01-15)
See [decisions/2026-01-15-api-vendor.md](decisions/2026-01-15-api-vendor.md). Chose Typefully over Buffer. v2 API confirmed.
```

**`session-log.md` entry pointing to a promoted decision:**

```
**Decisions and Rationale:** Chose Typefully reply_to_url architecture over X API direct posting.
Promoted to decisions/2026-04-16-link-reply-architecture.md.
```

**`MEMORY.md` summarizing state documented in recent session-log:**

```
## Migration Status (Updated 2026-04-20)
See session-log 2026-04-20 for full detail. Summary: v2 migration complete, legacy monitor draining, soak target Apr 23.
```

The rule: pointers are free, duplication is debt. When the same fact appears in two places in full, one of them will go stale.

---

## Failure Modes

### 1. Index bloat (Layer 1)

A Layer 1 file grows without pruning. After a few months, it exceeds 200 lines and consumes meaningful context-window space every session. The fix is mechanical: when a file exceeds 200 lines, move detail into a topic file (for project MEMORY.md) or into a referenced on-demand file (for `~/.claude/CLAUDE.md`), and replace the section with a pointer. The `/memory` skill should enforce this automatically if you wire it correctly.

### 2. Decision-writing skipped (Layer 2)

The most common failure. Decisions get made in conversation, nothing gets written down, and three months later nobody knows why the current approach was chosen. The trigger is easy to miss: "we just decided, why write it up?" Build a habit: if you considered alternatives and rejected one, write the file. If you only had one option, skip it.

### 3. Session-log drift (Layer 3)

Free-form notes instead of structured entries. After a few sessions, the log becomes hard to scan. The structure in the entry format above is designed to be skimmable: the heading tells you the date and topic, the Open Items section tells you what needs action. Use `/log` or `/mexit` at session end. Do not freehand the entries.

### 4. QMD underuse (Layer 4)

Claude will not search QMD unless it knows to. Add an instruction to your project's `CLAUDE.md`, or to `~/.claude/CLAUDE.md` if you want it globally:

```
When the user says "we did this before," "how did we set up X," or references past work,
search QMD (`sessions-md` collection) before saying the information is unavailable.
```

Without this, Layer 4 is wired up but never used.

### 5. Cross-machine sync conflicts

If `MEMORY.md` is shared between two machines via Dropbox or similar sync, concurrent writes will produce conflicts. This is a real-world gotcha. The mitigation: per-machine files with a symlink, as described in the Layer 1 section. Each machine writes only its own file. Each machine reads both files at session start.

---

## Why Four Layers (Not Two or Six)

Two layers (a doc file plus transcripts) is the natural starting point. It fails because:

- The doc file becomes a graveyard of verdicts with no rationale (missing Layer 2)
- There is no way to resume work quickly without re-reading everything (missing Layer 3)

Six layers adds overhead without adding recall coverage. The four layers above map onto four genuinely distinct access patterns:

- Always in context: zero discovery cost, token cost every session
- Cited on demand: zero token cost, requires knowing it exists
- Session-start read: modest cost, scoped to recent work
- On-demand search: zero default cost, surfaces the long tail

Each layer is doing something the others cannot do cheaply.

---

## License

MIT. Part of the [claude-skills](https://github.com/anup-malani/claude-skills) repository.
