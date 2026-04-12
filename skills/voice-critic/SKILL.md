---
name: voice-critic
description: "Run a critic agent conditioned on the user's own past edits to catch voice and tone problems in a draft. Use when user says 'run voice-critic', 'voice check', 'does this sound like me', or wants a pass on a draft that has already been fact-checked and style-checked but still reads wrong. Do NOT use for fact-checking (claims verification is a separate agent) or for style-guide enforcement (also a separate agent). This skill specifically catches the gap between 'correct' and 'sounds like the author'."
argument-hint: "<path to draft file, or inline draft text>"
metadata:
  version: 1.0.0
---

# Voice Critic — The Agent That Reads Your Past Edits

Most AI drafting pipelines run a drafter, a fact-checker, and a style-guide enforcer. All three are blind to the same thing: voice. A draft can pass all three and still read wrong, because "wrong" in that sense means "not how the author would have written it."

This skill runs a fourth agent whose entire job is to catch that gap. It reads a corpus of the author's own past edits and looks for places where the current draft would not survive the author's red pen.

## When this skill fires

User says any of:
- "run voice-critic on <draft>"
- "voice check this"
- "does this sound like me"
- "check my voice"
- "voice-critic <path>"

## Argument

`$ARGUMENTS` is either:
- A path to a draft file (markdown, plain text, or similar), or
- Inline draft text passed directly in the message.

If neither is obvious, ask the user which draft to critique before proceeding.

## Prerequisites

The user must have an edit corpus at one of:

1. The path in `$VOICE_CRITIC_CORPUS_PATH` if that environment variable is set
2. `~/.claude/skills/voice-critic/corpus.md` (default location)

If neither exists, stop and point the user to `setup.md` in this skill directory. It walks through building a minimum-viable corpus (5 pairs) in about 15 minutes. Do not proceed without a real corpus — the `example-corpus.md` file is for reference only and should not be used as a live corpus, because it teaches generic "good writing," not the user's specific voice.

## Steps

### 1. Locate the draft

If the argument is a path, read the file. If inline text, use it directly. If both are supplied, prefer the inline text and note the redundancy.

### 2. Locate the edit corpus

Check in order:
1. `$VOICE_CRITIC_CORPUS_PATH` if set
2. `~/.claude/skills/voice-critic/corpus.md`

If neither exists, stop. Tell the user to read `setup.md` in this skill directory and assemble a corpus first. Do not fall back to `example-corpus.md` silently — that file exists only as a format reference.

### 3. Load the prompt template

Read `prompt-template.md` in this skill directory. It contains a self-contained critic prompt with two slots:
- `{edit_corpus}` — replace with the full contents of the corpus file
- `{draft}` — replace with the full draft text

Do not add instructions outside the template. The template is intentionally self-contained so the critic behavior is reproducible and auditable.

### 4. Run the critic as a subagent

Spawn a general-purpose subagent using a mid-tier model (Sonnet class or equivalent). This task requires judgment about voice and rhythm; a Haiku-class model is too shallow and an Opus-class model is overkill for a single-pass critique. Pass the filled template as the subagent's task with no additional system instructions.

### 5. Return the critic's output verbatim

The critic returns a structured critique with three sections:
1. **Voice violations** — verbatim quotes of phrases that would not survive the author's red pen, each with a one-line pattern note
2. **Suggested rewrites** — a voice-preserving rewrite for each violation (meaning held constant)
3. **Confidence note** — one line on how confident the critic is, given corpus thickness and context familiarity

Surface this full response to the user verbatim. Do not paraphrase, summarize, or filter. Voice critiques lose meaning when paraphrased.

### 6. (Optional) Re-run on revision

If the user revises the draft and asks for another pass, repeat steps 1-5. Each pass is independent — the critic is stateless by design so that a previous pass's conclusions do not grandfather forward.

## Hard NOs

- Do not fact-check claims. That is a separate agent's job.
- Do not enforce a style guide. Style guides are rules; voice is patterns. Different jobs.
- Do not rewrite content beyond what voice requires. The skill adjusts how something is said, not what is said.
- Do not flatter the draft. "Already clean" is fine when true; it is harmful when false.
- Do not proceed without a real corpus. The whole skill depends on the corpus, and a missing corpus should halt the pipeline, not degrade it silently.

## Where this fits in a larger drafting pipeline

Voice critic is designed to run alongside a drafter, a fact-checker, and a style-guide enforcer — not instead of them. A reasonable four-agent pipeline looks like this:

1. **Drafter** writes the first pass from a source (paper, notes, outline).
2. **Fact-checker** verifies every claim against the source.
3. **Style-guide enforcer** checks format, length, banned phrases, required structure.
4. **Voice critic** (this skill) reads the author's past edits and flags everything the author would change.

All four can run in parallel after the drafter. The author then sees a single consolidated critique rather than a raw draft. In practice this tends to change which rewrites get done by the author vs. by an agent, and it tends to shrink the rewrite tax on the author's attention budget — fact-checkers and style checkers catch rule violations, but only the voice critic catches the gap between "correct" and "sounds like you."
