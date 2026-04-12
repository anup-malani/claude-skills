# Example edit corpus (format reference only)

This file shows the format for a voice-critic edit corpus. It is **not** a live corpus — the critic should never read from this file as if it were the user's own voice. It exists so that a first-time user can see what a real corpus should look like before assembling one.

To build your own, see `setup.md` in this directory. Save your real corpus as `corpus.md` in this same directory, or set `VOICE_CRITIC_CORPUS_PATH` to a custom path.

---

Each entry is a draft-rewrite pair with a one-line pattern note. The critic reads across all entries and infers voice from the pattern of transformations. Consistency matters more than quantity — 5 deliberately chosen pairs beat 30 random ones.

---

## Entry 1

**Draft:**
> It is important to note that the policy reduced enrollment significantly across multiple subgroups.

**Rewrite:**
> The policy cut enrollment sharply, and across every subgroup we checked.

**Pattern:** Kill "it is important to note." Kill "significantly." Verbs should be concrete ("cut"). End on the thing the reader should remember.

---

## Entry 2

**Draft:**
> The authors conclude that there may potentially be some evidence for a modest effect.

**Rewrite:**
> The authors find a modest effect. Maybe.

**Pattern:** One hedge is honest; three hedges is cowardice. Collapse stacked qualifications. Let the short second sentence do the uncertainty work.

---

## Entry 3

**Draft:**
> This finding has significant implications for policymakers at the federal, state, and local levels.

**Rewrite:**
> Cut entirely. If the finding has implications, show them. Do not announce them.

**Pattern:** Never write the sentence that says "this is important." Either demonstrate the importance in the next paragraph or delete the claim.

---

## Entry 4

**Draft:**
> Building on the extensive prior literature, our study contributes to the understanding of the mechanism.

**Rewrite:**
> Our paper sharpens one mechanism the prior literature gets wrong.

**Pattern:** "Contributes to" is filler. Say what you sharpen, what you correct, what you add. Specificity beats humility.

---

## Entry 5

**Draft:**
> It was found that recipients were more likely to be employed after the intervention.

**Rewrite:**
> Recipients found jobs faster after the intervention.

**Pattern:** Active voice, concrete verb, no "it was." Subject at the start, outcome at the end. The stress position in English is the last slot.
