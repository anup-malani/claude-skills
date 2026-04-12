# Building your own voice-critic edit corpus

The critic is only as good as your corpus. A thin or mixed corpus produces vague feedback. Fifteen minutes of deliberate work pays off for months of drafts.

## What you need

Five to fifteen pairs of your own before-and-after edits. A pair is:

- A draft sentence or paragraph that you rewrote
- Your rewritten version
- A one-line note naming the pattern the rewrite reveals

More pairs help a little, but the returns diminish fast. Five deliberately chosen pairs beat thirty random ones.

## How to find pairs (about 15 minutes)

### Option 1: Git history (fastest if applicable)

If any of your writing lives in a git repository, run:

```bash
git log -p --all -- "**/*.md"
```

Scan for commits where you revised your own prose. Each `-` line paired with the next `+` line is a candidate pair. Skip formatting-only changes; you want substantive rewrites.

### Option 2: Track Changes on a recent document

Open the last document you heavily edited in Word or Google Docs. Turn on Track Changes or open Version History. Each revision block that replaces draft prose with revised prose is a candidate pair.

### Option 3: Synthetic pairs from a published piece

Find a paragraph in something you already published that you are proud of. For each sentence, ask: "What would a less skilled writer have written here, and why did I change it?" Write the weaker version as the "draft" and your published sentence as the "rewrite." This is synthetic but valuable — it teaches the critic what you specifically avoid.

## What makes a good pair

**Good:**
- Shows a pattern the critic can generalize (hedging, nominalization, weak verbs, bad rhythm, filler openings)
- Has a one-line note that names the pattern in the abstract, not just the specific fix
- The rewrite is clearly better, not just different

**Bad:**
- Two versions that differ only in word choice with no pattern behind the change
- Edits that fix facts or add information — that is a different agent's job
- Polishing that only the author can justify ("I just liked it better")

## Format

Follow the structure in `example-corpus.md` exactly. Each entry is:

```
## Entry N

**Draft:**
> The original sentence or paragraph, quoted as a blockquote.

**Rewrite:**
> Your revised version, also quoted.

**Pattern:** One line. Name the pattern in the abstract — what rule did the rewrite enforce?
```

## Where to save it

Save the file as `corpus.md` in this directory (`~/.claude/skills/voice-critic/corpus.md`). The critic reads from this default path.

If you want to keep your corpus somewhere else — a private repo, a Dropbox folder, a synced note — set the environment variable `VOICE_CRITIC_CORPUS_PATH` to the absolute path and the critic will read from there instead.

## When to refresh it

Whenever you catch yourself consistently editing out a pattern the critic failed to flag, add that pattern as a new entry. The corpus is a living document; it should grow as your voice sharpens.

One healthy habit: after each drafting session where you did substantial revision, copy one new pair into the corpus. That alone gives you a corpus of 50+ pairs in two months, all drawn from real work.
