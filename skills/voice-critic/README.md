# voice-critic

> Run a critic agent conditioned on your own past edits to catch voice and tone problems in a draft — the gap between "correct" and "sounds like you."

## Problem this solves

Most AI drafting pipelines run a drafter, a fact-checker, and a style-guide enforcer. All three are blind to the same thing: voice. A draft can pass every check and still read wrong, because "wrong" in that sense means "not how the author would have written it." This skill runs a fourth agent whose entire job is to catch that gap. It reads a corpus of your own past edits and flags every place where the current draft would not survive your red pen.

## How it works

You assemble a small corpus of your own before-and-after edits — five to fifteen pairs, each showing a draft you rewrote, your rewrite, and a one-line note naming the pattern. The critic reads the whole corpus, studies the transformations you consistently make, and infers your voice from the pattern of rewrites. When you then hand it a new draft, it flags phrases that would not survive your revision pass and proposes voice-preserving rewrites. Meaning is held constant; only voice changes.

The critic returns three sections: voice violations (quoted verbatim with pattern notes), suggested rewrites (meaning-preserving), and a one-line confidence note about how thick the corpus was and whether the draft's context resembles anything it has seen before.

## How to install

1. Clone this repo or copy the `voice-critic/` directory into your `~/.claude/skills/` directory.
2. Restart Claude Code so the skill is loaded at session start.
3. Build your own edit corpus by following `setup.md` in the skill directory. Budget about 15 minutes for the minimum viable version (5 pairs).
4. Save your corpus as `~/.claude/skills/voice-critic/corpus.md`, or set the environment variable `VOICE_CRITIC_CORPUS_PATH` to a custom location.
5. Invoke with `/voice-critic <path-to-draft>` or `/voice-critic <inline-text>` in any session.

## Example use

```
/voice-critic drafts/memo-v2.md
```

The critic reads `drafts/memo-v2.md`, loads your corpus, and returns:

```
### Voice violations

1. "It is important to note that the finding has significant implications..."
   Pattern: you consistently cut "it is important to note" and never use "significant"
   as a standalone intensifier.

2. "The authors conclude that there may potentially be some evidence..."
   Pattern: you collapse stacked hedges. Three qualifiers becomes one.

### Suggested rewrites

1. "The finding matters because..." [continue with the implication directly]
2. "The authors find some evidence. Maybe."

### Confidence note

High confidence. Corpus is thick (12 pairs) and the draft's register matches
your policy-memo voice closely.
```

## Why a corpus, not a style guide?

Style guides are rules ("use active voice," "avoid nominalization"). Voice is patterns — what a specific author cuts, where they place emphasis, which hedges they let stand, what kinds of sentences they refuse to write. A rule-based checker cannot distinguish between two drafts that are both technically correct but one of which sounds like you and the other does not. A corpus-based critic can, because it is comparing the draft to actual evidence of your revision behavior.

The tradeoff is that the critic is only as good as your corpus. A thin or mixed corpus produces vague feedback. See `setup.md` for guidance on building a good one — the short version is that 5 deliberately chosen pairs beat 30 random ones.

## Prerequisites

- Claude Code installed and configured
- An edit corpus at `~/.claude/skills/voice-critic/corpus.md` or at the path set by `$VOICE_CRITIC_CORPUS_PATH`
- A draft file or inline draft text to critique

No MCP servers, no external APIs, no additional dependencies beyond Claude Code itself.

## Files in this skill

- `SKILL.md` — the Claude Code skill definition (invocation rules, orchestration steps, hard NOs)
- `prompt-template.md` — the self-contained critic prompt with `{edit_corpus}` and `{draft}` slots
- `setup.md` — a 15-minute walkthrough for building your own edit corpus
- `example-corpus.md` — a format reference showing what a corpus entry looks like (not a live corpus)
- `README.md` — this file

## Credits

Authored by Anup Malani ([@anup_malani](https://x.com/anup_malani)). Part of the [claude-skills](https://github.com/anup-malani/claude-skills) collection.
