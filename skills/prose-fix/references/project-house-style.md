# House style: the aslice project (martin-juul/appleslices)

These rules bind every prose-fix task in the aslice repository. They outrank the
general author registers wherever the two disagree. The three register sheets
(chen.md, knuth.md, stroustrup.md) describe *voice*; this file describes *the project*.

## 1. Register → document mapping

| Register | Documents |
|---|---|
| Chen | README.md, docs/MANUAL.md, docs/SETUP.md, docs/AUTHORING.md, docs/HOMEBREW-REVIEW.md |
| Knuth | docs/GENESIS.md, docs/KEY-RUNBOOK.md |
| Stroustrup | docs/DESIGN.md, docs/PACKAGE-FORMAT.md, docs/REPOSITORIES.md, docs/ORCHARD-POLICY.md, docs/BUILD-INFRA.md |
| (mixed, see doc) | SECURITY.md, CONTRIBUTING.md — policy core in Stroustrup, how-to portions in Chen |

When the user directs a different assignment for a document, the user wins; note the
deviation in the review notes.

## 2. Charter constraints (never negotiate these in prose)

- **No metrics, analytics, or telemetry of any kind — "we are not a product."**
  Farm-side operational metrics about the farm itself are the only permitted
  exception (BUILD-INFRA §10); anything collected from or about users is out.
- **Download statistics are not a value metric.** Obscure libraries may be fetched
  once a month and have immense value, because the project serves deprecated
  operating systems. Never reintroduce popularity-based prioritization or phrasing
  that implies it.
- **No code of conduct document, deliberately.** The expectation is that people are
  decent to each other; this is not a corporate project, and it is acceptable to tell
  someone they are an idiot when they are acting like one. Review tone is direct:
  criticism of work is information about the work. Do not soften this in prose, and
  do not add CoC-style language anywhere.
- **Do not over-engineer — but do not exclude genuinely useful features either.**
  When a rewrite touches a feature description, preserve its existence and rationale;
  trimming scope is a content decision for the user, never a prose decision.

## 3. Mechanical conventions

- **Straight apostrophes** (`'`), straight quotes. House style; do not "fix" them to
  typographic quotes.
- **Em-dashes (—) are deliberate** in this project's prose, as they are in all three
  source authors. Keep them.
- **§ cross-references** are the project's reference unit. Preserve every one; a
  rewrite may add one only when it states an already-true relationship, and each
  addition is noted in the review notes.
- **Status headers / history footers** carry cumulative per-version changelogs.
  Historical entries are never edited, reordered, or summarized — only appended.
- **Charter items and quoted rule texts** stay verbatim even when they read
  awkwardly; flag them to the user instead of rewriting.
- **Section numbers never move.** Headers, code fences, and table rows are structure,
  not prose: leave them byte-identical unless the user asks otherwise.
- **Aphoristic lines may be kept deliberately** (precedent: the README rewrite).
  Keeping a gem is an active choice; note it.

## 4. The rewrite doctrine (learned the hard way)

1. A rewrite is a paragraph-level rebuild in the assigned register. Tic-scanning —
   swapping words, deleting em-dashes, splitting a few sentences — produced the
   versions the user rejected twice. Do not repeat that failure.
2. **Every normative word count is a tripwire.** must / never / cannot / can't /
   always / only / required / shall are counted before and after; every delta must
   trace to a specific line and be either a documented force-preserving change
   (contraction expansion "can't"→"cannot"; "can never X"→"cannot X") or reverted.
   The canonical self-inflicted wound: a rewrite that turns a *descriptive* sentence
   ("a system file aslice does not otherwise touch") into a *normative* one
   ("otherwise never touches") — same words, new rule. Watch for it in your own
   output, not just the source's.
3. **Every number, name, command, path, hash prefix, and inline-code token survives.**
   The audit script enforces this; a clean audit is a precondition for delivery, not
   a nicety.
4. Changelog wording convention for pure prose rewrites:
   "prose rewrite throughout — chapters reworded in the project's technical-writing
   voice; no <guidance|content|procedural> changes". Pick the noun the document's own
   history uses.

## 5. Delivery rules (appleslices repo)

- All writes go through the **GitHub MCP** (`create_or_update_file`) — never local
  git, never API curl for writes. If the MCP tools are unloaded (session break),
  reload them with `select_tools` before pushing. If the MCP server is down, stop
  and say so; do not improvise another write path.
- Before updating, fetch the current remote blob sha; compute the expected new blob
  sha locally (`sha1("blob " + len + "\0" + content)`); push with full inline
  content; **the returned blob sha must equal the computed one** — a mismatch means
  transcription error, fix and retry.
- After pushing, fetch the file back (raw URL or API) and byte-compare. Note that
  raw.githubusercontent.com caches; a stale fetch-back is diagnosed by comparing its
  blob sha to the push response, not by re-pushing.
- Push strictly sequentially: one file, verify, next file.
- Mirror verified files to the local staged tree and the output mirror per the
  session's standing convention.

## 6. Review-only tasks

When asked to *rate* rather than rewrite: for each chapter report the register fit,
quote the worst tic lines verbatim, list lines worth keeping, and propose the register
— then stop and wait. Do not rewrite on a rating request.

## 7. The review-and-fix pass

After a batch of documents, re-read each one end to end as a *reader*, not an editor:
does any sentence sound like it was written to impress? Does any paragraph explain
what the previous paragraph already said? Does any "never" appear where the policy is
merely "does not"? Fix what you find, re-audit, and log the pass as its own changelog
entry if it touched prose.
