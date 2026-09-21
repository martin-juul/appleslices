---
name: prose-fix
description: Rewrite and repair technical prose in the aslice project's house voice, modeled on three studied sources — Raymond Chen (The Old New Thing), Donald Knuth (The Art of Computer Programming), and Bjarne Stroustrup (The C++ Programming Language, 4th ed.). Use whenever the user asks to fix, rewrite, polish, humanize, or review prose in project documents (README, MANUAL, DESIGN, runbooks, policies, contributing guides, and similar), when text "reads like LLM output" or has "veered off course", when rating the writing quality of a document, or when continuing the ongoing documentation style program of the appleslices project. Not for fiction or marketing copy.
---

# prose-fix

Repair technical prose so it reads like it was written by a careful human expert. The
voice to imitate is not generic "good writing"; it is a specific blend of three studied
authors, each assigned to the document types he suits.

## The doctrine

1. **Meaning is sacred; every sentence is negotiable.** A rewrite may rebuild whole
   paragraphs, but every fact, number, name, cross-reference, code token, and normative
   word (must/never/cannot/only/required) survives. A rewrite that drifts in meaning has
   failed no matter how well it reads.
2. **Rewrite at the paragraph level, never at the tic level.** Swapping "utilize" for
   "use" and deleting a few em-dashes is not a rewrite. Read the whole document first,
   then rebuild prose sentences and chapters in the assigned register while leaving
   structure (headers, section numbers, code fences, tables) untouched.
3. **Humility over pride.** Judge the result by whether a busy engineer would trust and
   enjoy it, not by how clever it sounds. When in doubt, understate.
4. **Keep the gems.** A line that is already aphoristic and true ("If you can't hash it,
   you can't ship it") is kept deliberately, not rewritten for novelty's sake. Note the
   keep in the changelog or review notes.

## The three registers

Read the matching reference file before rewriting in that register. Each contains the
principles, many verbatim source examples, and per-sentence before/after pairs.

| Register | Reference | Voice in one line | Suitable documents |
|---|---|---|---|
| Chen | [references/chen.md](references/chen.md) | Scenario first, mechanism walked step by step, dry understatement, honest "I don't know" | User guides, manuals, setup/troubleshooting, tutorials, reviews, postmortems |
| Knuth | [references/knuth.md](references/knuth.md) | Formal named procedures, first-person ownership of mistakes, aphoristic morals, scholarly scope-honesty | Runbooks, ceremonies, bootstrap/genesis procedures, inventories, drills |
| Stroustrup | [references/stroustrup.md](references/stroustrup.md) | Declarative topic sentences naming the mechanism, terms defined at first use, dense § cross-refs, advice as rules of thumb | Design documents, specifications, format references, policy documents |

The aslice project's current register→document mapping lives in
[references/project-house-style.md](references/project-house-style.md); follow it unless
the user says otherwise. When a document mixes genres (e.g. a design doc with a tutorial
chapter), pick per chapter and say so.

## The shared spine (all three authors agree on this)

- Concrete before abstract. Name the mechanism, the file, the command.
- Short declarative sentences; one idea per sentence.
- No marketing. No adjectives of enthusiasm ("powerful", "seamless", "robust",
  "cutting-edge"). Let the mechanism impress.
- Honest hedging with specifics, never vagueness: say *what* is uncertain and *why*.
- Humor is dry, structural, and rare — a raised eyebrow, never a joke with a drumroll.
- The reader is an intelligent peer. Never pad, never cheerlead, never apologize for
  difficulty.

## LLM tics to eliminate on sight

- Throat-clearing openings: "In today's fast-paced world", "It's important to note that",
  "When it comes to X", "X plays a crucial role".
- Synonym triads and noun-pile abstractions: "fast, efficient, and reliable".
- "Not only X, but also Y" inversions; "Whether you're A or B, ..." menus.
- Summary paragraphs that repeat what the section just said ("In conclusion",
  "As we've seen").
- Buzzword inventory: delve, landscape, ecosystem (unless literal), leverage (as verb),
  empower, seamless, robust, comprehensive, streamline, utilize, facilitate, plethora.
- False balance hedging: "It could be argued that some might consider...".
- Em-dash deletion is **not** a goal: the house style uses em-dashes deliberately, as do
  all three authors. Removing them to look less "AI" is overcorrection.

## Workflow

1. **Read the entire document first.** No rewriting from a skim. Note structure:
   headers, tables, code fences, § references, status/changelog blocks.
2. **Assign the register** (table above + project-house-style.md).
3. **Rewrite prose lines** in that register. Leave headers, section numbers, code
   fences, table rows, and changelog/history entries untouched. Preserve every inline
   code token exactly.
4. **Audit mechanically.** Run `scripts/audit_rewrite.py OLD NEW` and account for every
   reported delta. Normative-word count changes are the early-warning signal for meaning
   drift; each one must be traceable to a specific line and justified (e.g. a
   contraction expansion "can't"→"cannot" that preserves force) or reverted.
5. **Bump the version and log it.** Add a changelog/history entry in the document's own
   convention: "prose rewrite throughout — <what changed in voice>; no <guidance|
   content|procedural> changes". Never edit historical entries.
6. **Deliver and verify** per the project's delivery rules (project-house-style.md §5):
   for the appleslices repo, push only via the GitHub MCP, verify the returned blob sha
   against a locally computed sha, then fetch back and byte-compare.

For review-only tasks ("rate this text"), report per chapter: register fit, the worst
tics with quoted lines, lines worth keeping, and a proposed register — then wait for the
word to rewrite.

## Files

- [references/chen.md](references/chen.md) — Raymond Chen style sheet: principles,
  verbatim examples, before/after pairs, topic fit.
- [references/knuth.md](references/knuth.md) — Donald Knuth style sheet: formal
  procedures, first-person ownership, aphorisms, scholarship, drills.
- [references/stroustrup.md](references/stroustrup.md) — Bjarne Stroustrup style sheet:
  declarative mechanism prose, term definition, advice sections, analogies.
- [references/project-house-style.md](references/project-house-style.md) — aslice house
  rules: register mapping, charter constraints (no telemetry, no code of conduct, tone),
  mechanical conventions, audit rules, delivery rules.
- [scripts/audit_rewrite.py](scripts/audit_rewrite.py) — fact-preservation audit:
  backtick-token multiset, number multiset, normative-word counts, structure identity.
  Run as `python3 scripts/audit_rewrite.py OLD.md NEW.md`.
