# claude-skills

A personal collection of [Claude Code](https://claude.com/claude-code) skills that I use in my own research, writing, and workflow, published here for anyone who wants to borrow them.

Each skill is a small, self-contained directory under `skills/` with a `SKILL.md` file that Claude Code loads at session start, plus a `README.md` that explains what the skill does for a human reader who lands here from a link or a search result.

## What a "skill" is, briefly

Claude Code skills are markdown files that teach Claude how to perform a specific task the same way every time. They live at `~/.claude/skills/<name>/SKILL.md` and get loaded automatically when Claude Code starts a session. You invoke a skill by typing `/<name>` in any session — Claude reads the skill file and follows its instructions.

Skills are the right primitive when you find yourself giving Claude the same multi-step instructions repeatedly. Rather than re-typing the recipe, you write it once as a skill and reuse it forever. The skills in this repo are things I reuse in my own work and that I think transfer well to other people's workflows.

## How to install a skill from this repo

1. Clone this repo, or copy the specific skill directory you want:
   ```bash
   git clone https://github.com/anup-malani/claude-skills.git
   cp -r claude-skills/skills/<skill-name> ~/.claude/skills/
   ```
2. Restart Claude Code so the new skill loads at session start.
3. Read the skill's own `README.md` for any setup steps (some skills need a corpus, an environment variable, or a one-time configuration pass).
4. Invoke with `/<skill-name>` in any Claude Code session.

## Current skills

| Skill | What it does |
|---|---|
| [voice-critic](skills/voice-critic) | Runs a critic agent conditioned on your own past edits to catch the gap between "technically correct" and "sounds like you." |

(The collection starts small on purpose. I'd rather ship one skill that's been used for months than ten that look clever in a screenshot.)

## Publishing process

Every skill in this repo is a sanitized copy of a private skill that lives in my own `~/.claude/skills/` directory. The sanitization pass strips hardcoded paths, credentials, references to private collaborators, and anything coupled to my specific infrastructure. The goal is that each skill is a drop-in for any Claude Code user — no environment-specific edits required.

The private-to-public pipeline has two parts. First, a sanitization scan walks every file in the private skill and flags hard blockers (hardcoded paths, secrets, ongoing-work references) and soft blockers (references to other unpublished skills, paths that assume pre-existing state). Hard blockers have to be rewritten before anything ships; soft blockers get rewritten in the staged copy, not the source. Second, a diff review shows exactly what changed between private and public before any file is committed.

## What's missing: stress-testing

Skills in this repo are fact-checked by the sanitization pipeline and reviewed by hand, but they are not yet stress-tested in the same structured way I stress-test writing or research. A reasonable next step would be a container-based test harness that loads a fresh Claude Code environment, installs each skill, exercises it against a synthetic test case, and verifies the output shape. The goal would be to ship every skill with a visible "pre-tested, not just copy-pasted" signal, so a drive-by reader can trust that the skill survives contact with a clean environment.

This is on the roadmap, not in place yet. If it ships, skill cards in this README will get a test-status badge.

## Contributions

This is a personal collection, not a community repo, so I'm unlikely to accept general-purpose pull requests. Bug reports, sanitization holes, or install failures are welcome as issues. If you have a skill you think fits the collection, fork it and ship your own `claude-skills` repo — that's the pattern this is meant to encourage.

## License

[MIT](LICENSE). Do what you want with the skills. Credit is appreciated but not required.

## Credits

Maintained by [Anup Malani](https://x.com/anup_malani). Senior advisor to the CMS Chief Economist (day job), professor at the University of Chicago Law School, and an opinionated user of Claude Code.
