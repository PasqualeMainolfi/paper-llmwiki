# Paper-llmwiki — LLM-maintained notebook for a scientific paper

A pattern for using an LLM to incrementally build and maintain the full
knowledge base behind a scientific paper: experiments, prior art, and the
evolving draft itself. An adaptation of [Karpathy's llmwiki](https://karpathy.ai/)
idea, specialized for the scientific-paper workflow.

This file has two jobs:

1. **Schema.** Describe how the paper folder is organized and what conventions
   the LLM should follow when reading from it or writing to it.
2. **Portable prompt.** Be copy-pasteable into a fresh project / fresh LLM
   session to bootstrap the same workflow from scratch.

---

## 1. Core idea

A scientific paper is an artifact produced from three kinds of inputs that
keep evolving until submission:

- **experiments** — your own runs: JSON / CSV / WAV / PNG, captured every time
  a benchmark or ablation executes;
- **prior art** — papers you read, quote, and compare against;
- **decisions** — framing calls, caveats, things tried and abandoned, tradeoffs
  the draft will have to defend.

The naive approach is to keep experiments in one folder, reread them whenever
it is time to write, keep PDFs of prior art scattered across a Zotero library,
and negotiate framing in Slack / head. The result is that the draft is
rediscovered from scratch every time: "what were the exact numbers on that
run?", "which metric were we ranking on?", "why did we reject the 31×13
patch again?".

The pattern here is different. An LLM assistant **incrementally builds and
maintains a wiki** — a structured set of Markdown files — that sits between
the raw data and the draft. Every experiment run is summarized into a wiki
page the first time it matters. Every cited paper gets an annotated entry.
Every framing decision is written down with its reason. When the draft
section is due, the LLM opens the wiki pages the outline points at and quotes
them verbatim. Numbers, caveats, and citations are already there.

Key difference from one-shot RAG over experiment logs: **the wiki is a
persistent, compounding artifact**. Cross-references are pre-computed.
Contradictions between runs have already been flagged. The narrative is
already tracked across pages, not re-derived when you start writing.

You are in charge of running experiments, reading papers, and deciding the
story. The LLM does the bookkeeping: summarizing JSON / logs into readable
tables, adding BibTeX entries, updating `[[links]]` between pages, maintaining
the index, flagging contradictions. The compounding effect is what makes this
pay off: by the time the draft is due, the draft is mostly a matter of
stitching wiki pages together.

The notebook also keeps an explicit record of what is **not yet** in the
paper but is being tracked deliberately: open research questions, planned
ablations, ideas waiting on a maturity gate. This forward-looking layer
lives in `ROADMAP.md` (§3.2) and prevents two failure modes — losing ideas
in chat history, and silently drifting `FOCUS.md` into a wishlist.

In practice: LLM agent on one side, Obsidian on the other. LLM edits files
based on your conversation; you browse the result in real time — follow
`[[links]]`, check the graph, read updated pages. Obsidian is the IDE, the
LLM is the programmer, the wiki is the codebase, and the draft is the build
output.

---

## 2. Layers

There are **five** layers, each with a clear owner and a clear rotation speed:

| layer          | owner    | changes   | authoritative on          |
|----------------|----------|-----------|---------------------------|
| **raw**        | scripts  | every run | our experiments           |
| **reading**    | external | per paper | prior-art primary sources |
| **wiki**       | LLM      | weekly    | what it means             |
| **references** | LLM      | rarely    | citation metadata         |
| **draft**      | human    | monthly   | how we tell the story     |

`raw/` and `reading/` are both read-only inputs and are handled
**symmetrically**. One holds things *we* produced (experiments), the other
holds things *others* produced and we consumed (papers, articles, reports).
Keeping them separate keeps the provenance of every claim legible — did
this number come from one of our runs or from a prior publication?

Symmetry rule: **any addition to `raw/` or `reading/` triggers the same
fan-out** — create or update the wiki page(s) it feeds, update the
corresponding `INDEX.md`, refresh obsidian links, append to `LOG.md`. A
paper dropped into `reading/` is not inert: it is an input on equal footing
with an experiment directory. See §6 for the two mirrored operations
(`ingest-experiment`, `ingest-reference`) and the shared post-ingest
checklist.

Above these five storage layers sit the **orientation files** at the top
level of `paper/`: `METHOD.md` (schema), `FOCUS.md` (thesis, present tense),
and `ROADMAP.md` (planned work, future tense). They are not layers in the
storage sense — they hold no raw artifacts and no claims of their own — but
every operation reads at least the first two of them before touching any
layer. See §3.

### Raw
Your own experiment artifacts. Immutable from the LLM's point of view — it
reads, it does not edit. Timestamped subdirectories under
`raw/<topic>/<YYYYMMDD_HHMMSS_slug>/` make the run history self-sorting.

### Reading
The prior art you actually read. One directory per cited source, keyed by
BibTeX cite key so it lines up 1:1 with `references/bibliography.bib`.
Layout:

```
reading/
  <bibkey>/
    source.pdf              # local copy of the paper (optional but strongly preferred)
    notes.md                # annotated reading notes, claim-by-claim
    extras/                 # quotes, screenshots of figures, etc. (optional)
  INDEX.md                  # catalog of all readings with status (to-read / reading / done)
```

`notes.md` is the T1 record for every fact the paper asserts. It exists so
the LLM can cite `[@bibkey]` against a checked passage rather than against
memory. Minimum shape for `notes.md`:

```markdown
# <bibkey> — <short title>

**Citation.** [@bibkey] → see references/bibliography.bib
**Relevance.** Which FOCUS.md claim(s) this paper touches (support, contrast, background).
**One-line summary.** …

## Key claims
- "quoted or tightly-paraphrased claim" — §N of the source
- …

## Methods / numbers worth quoting
- …

## How it connects to our wiki
- [[our_wiki_page]] — supports / contrasts / extends

## Open questions for re-read
- …
```

The LLM may only attribute a claim to `[@bibkey]` if the claim is present in
`reading/<bibkey>/notes.md` (or confirmable from `reading/<bibkey>/source.pdf`
in the same session). No `notes.md`, no citation — see §8.

### Wiki
Per-topic Markdown notes. One concept per file. Every file uses the same
skeleton (§4) so the LLM can find what it needs without re-reading prose.
Obsidian-style `[[link]]` connects peer pages; `## Related` at the bottom of
each page lists explicit links. The LLM owns this layer entirely: it creates,
updates, refactors, and re-links pages as the project evolves.

### References
Bibliography — BibTeX + annotated index. Cite keys are stable. Every wiki
page that relies on prior art points to cite keys here; every entry here
points back to the wiki pages that use it.

### Draft
The paper itself. Human-authored, because framing is a human job. Rule: the
draft may only introduce numbers that already appear in a wiki page. That one
rule makes the wiki the single point of truth for any claim that will be
peer-reviewed.

---

## 3. Schema (this file)

`paper/METHOD.md` is the equivalent of Karpathy's `CLAUDE.md` / `AGENTS.md` —
the schema document the LLM rereads at the start of a session to understand
the conventions. Keep it authoritative: when a convention changes, change it
here first and let the LLM propagate.

Typical contents of the schema:
- folder layout (§5 below);
- wiki page skeleton (§4 below);
- ingest, compose, lint workflows (§6);
- honesty principles (§7);
- source verification protocol (§8);
- LLM-bootstrap prompt (§10).

Three top-level orientation files anchor every session:

| file         | tense              | owner       | granularity      | mutability                    |
|--------------|--------------------|-------------|------------------|-------------------------------|
| `METHOD.md`  | timeless (rules)   | human       | convention-level | rare, deliberate              |
| `FOCUS.md`   | present            | human       | claim-level (Cn) | rare, deliberate              |
| `ROADMAP.md` | future             | human + LLM | category-level   | frequent, append/reorder      |

The two storage-level temporal records (`wiki/*.md` for verified past,
`LOG.md` for chronological) complete the picture.

### 3.1 Paper focus — `FOCUS.md`

Alongside `METHOD.md` lives `paper/FOCUS.md`. This is the shortest possible
statement of **what this specific paper is about** — thesis, scope, and the
handful of claims the paper is willing to defend. It is the second file the
LLM reads at every session, right after `METHOD.md`.

`FOCUS.md` answers, in one page or less:
- **Thesis.** One sentence. What the paper argues.
- **Claims.** 3–6 bullet points, each numbered `C1..Cn`. The load-bearing
  statements. Every wiki page must map to one or more `Cn`.
- **Out of scope.** What the paper explicitly does *not* argue. Prevents
  scope creep during ingest.
- **Target venue / length.** Disciplines how long the draft can be.

Rule: **every wiki page declares which claim(s) it supports**. Add a line
near the top of the page:

```markdown
**Supports.** C2 (spectral fidelity parity), C4 (edit surface).
```

If a new experiment or a new source does not map to any `Cn`, the LLM must
stop and ask: is this a new claim (update `FOCUS.md`), an out-of-scope
distraction (log and move on), an item for `ROADMAP.md` (deferred but
tracked, see §3.2), or a symptom of drift (revisit the thesis)? Silent
drift into off-thesis content is the failure mode this section exists to
prevent.

The LLM must consult `FOCUS.md` before any compose, lint, or reference
ingest and check that the resulting change stays on-thesis. When in doubt,
it raises the conflict to the human rather than guessing.

### 3.2 Roadmap — `ROADMAP.md`

Alongside `METHOD.md` and `FOCUS.md` lives `paper/ROADMAP.md`. This is the
forward-looking complement to `FOCUS.md`: a structured catalog of work that
is **not yet started but intentionally tracked**, organized by category
(not by timeline). For each entry it records the research question, an
estimated cost, prerequisites (a maturity gate), and the target stage in
the paper.

Where `FOCUS.md` declares what the paper defends *now*, `ROADMAP.md`
declares what it could defend *next*. Promoting a roadmap entry into a
`Cn` is an explicit, human-ratified step (§6 **Promote**), not a side
effect of work happening.

`ROADMAP.md` is the **third file** the LLM opens at every session, after
`METHOD.md` and `FOCUS.md`. Re-read it whenever a new experiment or
reference might satisfy an existing entry, or when the human is deciding
what to work on next.

#### Functions

1. **Scoping buffer.** Ideas parked outside `FOCUS.md` without being lost.
   Prevents `FOCUS.md` from drifting into wishlist territory.
2. **Dependency map.** Explicit maturity gates (e.g. "Cat 4 after
   substitution-matrix calibration") make the ordering non-arbitrary.
3. **Promotion pipeline.** Every entry has a defined path to becoming a
   wiki page (raw artifact or reading note → ingest → wiki page +
   possibly a new `Cn`). `ROADMAP.md` is the source from which `FOCUS.md`
   can grow.
4. **Vocabulary staging.** Informal terms (e.g. "audio-genome") live here
   before they enter `FOCUS.md`. Reduces churn in the formal claim list.
5. **Cost transparency.** Explicit `low / medium / high` cost plus
   maturity gate lets the human decide ordering by effort, not only by
   interest.

#### Distinctions that matter

- **vs. a flat TODO list.** A TODO is flat and meant to be drained;
  `ROADMAP.md` is structured by category and is allowed to grow and
  contract by family. A single entry can sit there for months without
  becoming debt.
- **vs. `draft/OUTLINE.md`.** `OUTLINE.md` is the structure of the
  finished paper (which sections); `ROADMAP.md` is the structure of the
  work that will fill those sections (which experiments and analyses
  feed those sections). `OUTLINE.md` answers "which sections";
  `ROADMAP.md` answers "which raw artifacts and references will support
  them".
- **vs. `FOCUS.md`.** `FOCUS.md` is claim-level and rare-mutation;
  `ROADMAP.md` is category-level and frequent-mutation. The boundary is
  enforced by the **Promote** operation (§6).

#### Skeleton

`ROADMAP.md` is organized **by category**, not by timeline. Each category
groups entries that share a maturity gate or a thematic family. Entries
have stable ids of the form `R<n>` or `R<cat>.<n>` so they can be
referenced from `LOG.md`, wiki pages, and chat without ambiguity.

```markdown
# Roadmap

> Forward-looking work catalog. See METHOD.md §3.2.
> Status legend: `parked` · `in-progress` · `promoted` · `dropped`.

## Cat 1 — <category title>

**Maturity gate.** <what must be true before this category becomes actionable>
**Target stage.** <which Cn this category would feed, or "candidate new Cn+1">

- **R1.1.** <one-line research question>
  - cost: low | medium | high
  - prereqs: [[wiki_page_dep]], R0.2
  - feeds: C2 (or: candidate new C5)
  - status: parked
  - notes: <optional, very short — full content moves to a wiki page on promotion>

- **R1.2.** ...

## Cat 2 — <category title>
...

## Promoted (archive)
- **R0.3.** … → [[spectral_fidelity]] — promoted 2026-04-12

## Dropped
- **R0.7.** … — dropped 2026-03-30: superseded by R1.1
```

Required fields per entry: `cost`, `prereqs`, `feeds`, `status`. The
`notes` line is optional and stays short — anything longer is a sign the
entry is ready to graduate to a wiki page.

#### Lifecycle

```
parked  →  in-progress  →  promoted to [[wiki_page]]
                       ↘
                         dropped (with one-line reason)
```

When an entry reaches `promoted`, the LLM:

1. Confirms a `wiki/<topic>.md` exists for the work and links it from the
   roadmap entry: `status: promoted to [[topic]]` (obsidian wikilink, §4.1).
2. Decides with the human whether the wiki page warrants a **new `Cn`** in
   `FOCUS.md` or simply strengthens an existing one. `FOCUS.md` is the
   only place where claims are declared; the LLM never edits it without
   explicit ratification.
3. Leaves the entry in place (with its `promoted` status) for one review
   cycle so the history of what graduated stays visible. Subsequent lints
   may move it under `## Promoted (archive)`.

When an entry is `dropped`, the LLM records a one-line reason and moves it
under `## Dropped` at the bottom. Dropped entries are kept (not deleted)
so the same idea is not re-litigated silently.

---

## 4. Wiki page skeleton

Every wiki page follows the same shape. The LLM uses the headers to locate
information without re-reading prose; you use them to skim.

```markdown
# <topic> — <one-line framing>

**Headline.** 1–3 sentences. The claim this page supports, in plain language.
Numbers and caveats go here, not buried in later sections.

## Raw
- `raw/<dir>/<file>` — what it contains
- `raw/<dir>/<file>` — ...

## Numbers           # optional — compact table, one row per condition
| condition | metric_a | metric_b |
|-----------|----------|----------|
| ...       | ...      | ...      |

## Figures           # optional
- `raw/<dir>/<file>.png` — what it shows

## Caveats           # or: ## Observations — honest limits, losses, biases
- ...

## Draft target
→ §X.Y (<short description of the subsection this page feeds>)

## Related
- [[peer_page]] — why linked
- [[peer_page]] — why linked
```

Required sections: **Headline**, **Raw**, **Draft target**, **Related**.
Optional: **Numbers**, **Figures**, **Caveats**, plus domain-specific
sections below *Related* (e.g. `## Framing rule`, `## Open work`).

The mandatory structure is what makes the wiki machine-navigable. Do not drop
the required sections "just this once". If a page genuinely does not need one,
write `—` under the heading so the shape is preserved.

If the page was promoted from a `ROADMAP.md` entry, add the back-reference
under `## Related`:

```markdown
- promoted from `ROADMAP.md` R1.2
```

### 4.1 Link conventions (obsidian-style, non-negotiable)

All internal links across the notebook use **obsidian wikilinks**, not
Markdown `[text](path.md)`. This keeps Obsidian's graph view intact and lets
the LLM rely on basename resolution during refactors.

| link target                                | form                                             | example                                               |
|--------------------------------------------|--------------------------------------------------|-------------------------------------------------------|
| wiki page → another wiki page              | `[[basename]]`                                   | `[[complex_patch]]`                                   |
| any page → a reading note                  | `[[reading/<bibkey>/notes\|<bibkey>]]`           | `[[reading/vinyard2025sparse/notes\|vinyard2025sparse]]` |
| any page → the references index            | `[[references]]`                                 | `[[references]]`                                      |
| any page → the roadmap                     | `[[ROADMAP]]` (or `[[ROADMAP#R1.2\|R1.2]]`)      | `[[ROADMAP#R1.2\|R1.2]]`                              |
| reading/INDEX.md → its own notes           | `[[<bibkey>/notes\|notes]]`                      | `[[vinyard2025sparse/notes\|notes]]`                  |
| references/INDEX.md → reading notes        | `[[reading/<bibkey>/notes\|notes]]`              | `[[reading/vinyard2025sparse/notes\|notes]]`          |
| any page → raw artifact (dirs / binaries)  | backticks, plain path                            | `` `raw/codec_comparison/20260423_082024_full/` ``    |
| any page → BibTeX file                     | Markdown link (not an obsidian-resolvable note)  | `` [`bibliography.bib`](bibliography.bib) ``          |

Rules:
- Use the `[[path|alias]]` pipe form for every cross-folder link so the
  displayed text stays short while the path keeps the link unambiguous when
  basenames collide (every reading dir has a `notes.md`; basename alone is
  not unique).
- Never link to `.md` files with Markdown `[text](path.md)`. That form is
  reserved for non-note targets (e.g. `.bib`, external URLs).
- Raw artifacts, code paths, and shell commands stay in backticks. They are
  not notes; obsidian should not try to resolve them.
- Every wiki page's `## Related` section includes at least one `[[link]]`
  to a peer wiki page, one to `[[references]]` when the page cites prior
  art, and one to each relevant `[[reading/<bibkey>/notes|<bibkey>]]`.
- Roadmap entries link to their dependent wiki pages with the `[[basename]]`
  form; wiki pages back-reference the originating roadmap entry with
  `[[ROADMAP#R<n>|R<n>]]` under `## Related`.

---

## 5. Folder layout

```
paper/
  METHOD.md                # this file (schema)
  FOCUS.md                 # thesis + claims (C1..Cn) + out-of-scope — see §3.1
  ROADMAP.md               # forward-looking work catalog (R-entries) — see §3.2
  README.md                # one-paragraph orientation for humans
  INDEX.md                 # catalog — optional mirror of wiki/INDEX.md at top level
  LOG.md                   # chronological log — optional, see §9

  raw/                     # experiment artifacts (our own runs)
    <topic>/<YYYYMMDD_HHMMSS_slug>/
      *.json  *.wav  *.png  *.csv

  reading/                 # prior-art sources we consumed
    INDEX.md               # catalog of readings with status
    <bibkey>/
      source.pdf           # local copy of the paper (preferred)
      notes.md             # annotated reading notes (T1 evidence)
      extras/              # optional: figure clips, quoted excerpts

  wiki/                    # per-topic Markdown notes (the llmwiki)
    INDEX.md               # entry point, grouped by draft section
    <topic>.md             # one page per concept, obsidian [[links]]

  references/
    bibliography.bib       # BibTeX — single source of truth
    INDEX.md               # annotated bibliography + reverse §-index

  draft/                   # paper sections
    OUTLINE.md
    s1_intro.md  s3_method.md  s4_results.md  ...
```

Top-level files line up by tense: `METHOD.md` (timeless rules), `FOCUS.md`
(present claims), `ROADMAP.md` (future work), `LOG.md` (chronological
record). Reading them in this order at the start of a session is the
fastest way for the LLM to recover state.

Naming conventions:
- Raw directories: `YYYYMMDD_HHMMSS_<slug>` — timestamp first so `ls` sorts
  chronologically, slug describes the experiment.
- Wiki files: `<topic>.md`, lower_snake. The basename is the `[[link]]` key.
- Draft files: `s<section>_<slug>.md`, e.g. `s4_1_reconstruction.md`.
- BibTeX keys: `<firstauthor><year><slug>`, e.g. `wang2003shazam`.
- Roadmap entry ids: `R<n>` or `R<cat>.<n>`, stable for the life of the entry
  (do not renumber on reorder; archive on promotion or drop).

---

## 6. Operations

Five operations cover almost everything the LLM needs to do.
`ingest-experiment` and `ingest-reference` are **mirrored**: same shape,
same fan-out, different source-of-truth directory (`raw/` vs `reading/`).
`promote` and `roadmap` maintain the forward-looking layer.

### Shared post-ingest fan-out (applies to both ingest ops)

Whenever `raw/<topic>/<timestamp>_<slug>/` or `reading/<bibkey>/` gains
content, the LLM runs this checklist before closing the operation:

1. **Wiki.** Create or update every wiki page that depends on the new
   material. Refresh `**Headline**`, `## Raw` (for experiments) or add
   `[@bibkey]` citations (for references) backed by `reading/<bibkey>/notes.md`.
2. **Supports line.** Each touched wiki page declares `**Supports.** Cn..`
   tied to `FOCUS.md`. Raise any mismatch.
3. **Obsidian links.** Add / refresh `[[basename]]` to peer wiki pages and
   `[[reading/<bibkey>/notes|<bibkey>]]` to the underlying source notes,
   following the conventions in §4.1.
4. **Related section.** Every touched wiki page's `## Related` block
   includes the new cross-links. No page ends up orphan or one-way-linked.
5. **INDEX files.** Update `wiki/INDEX.md` (tick `[x]` when the page is
   ready to quote) **and** whichever of `raw/` (if scripted) /
   `reading/INDEX.md` / `references/INDEX.md` gained or lost a row.
6. **Roadmap reconciliation.** Walk `ROADMAP.md` and check whether the
   ingested artifact or reference satisfies an existing entry (in part or
   in whole). If yes, update that entry's status (`parked → in-progress`,
   or `in-progress → promoted to [[wiki_page]]`) and add the back-reference
   under the wiki page's `## Related`. Crossing the `promoted` threshold
   is raised to the human for the `Cn` decision (see **Promote** below).
7. **LOG.md.** Append an entry: `## [YYYY-MM-DD HH:MM] <op> | <target>`
   with a short body of what was touched, including any roadmap status
   changes.
8. **Focus check.** Re-read `FOCUS.md` and confirm the change still maps
   to the declared claims. On "maybe", raise to the human; if the work
   is genuinely off-thesis but worth keeping, file it as a new `R<n>` in
   `ROADMAP.md` rather than smuggling it into the wiki.

### Ingest experiment
You finish a benchmark run. Raw artifacts land under
`raw/<topic>/<timestamp>_<slug>/`. You tell the LLM to ingest:

1. Read the new raw artifacts (JSON / CSV / figures).
2. Locate or create the wiki page that covers this topic.
3. Update `**Headline**`, `## Raw`, `## Numbers`, `## Figures`, `## Caveats`.
4. Run the **shared post-ingest fan-out** above.

### Ingest reference
You read a paper you will cite. The LLM:

1. Creates `reading/<bibkey>/` and drops the PDF in as `source.pdf`
   (if available). If only a URL is available, records it under
   `notes.md`'s `**Citation.**` line and flags the missing local copy.
2. Reads the paper. Writes `reading/<bibkey>/notes.md` following the
   template in §2 **Reading**: citation, relevance to `FOCUS.md`, key
   claims with section references, numbers, links to our wiki pages, open
   questions. `notes.md` is the primary-source record — every later
   citation must be traceable to a passage here.
3. Verifies the bibliographic metadata against a trusted source (the PDF
   itself, the publisher page, DOI.org, Semantic Scholar, arXiv). Does not
   guess year, venue, or author list from priors.
4. Adds a BibTeX entry to `references/bibliography.bib`. If any field is
   unverified, tags it with `% TODO verify <field>` and leaves the entry
   marked unconfirmed.
5. Adds the annotated row to `references/INDEX.md` (topic group, short
   gloss, `cited in draft`, wiki anchor, `[[reading/<bibkey>/notes|notes]]`).
6. Adds an entry to `reading/INDEX.md` with the cite key, title, status
   (`to-read` / `reading` / `done`), and the `Cn` claims it touches.
7. Updates any wiki page that gains a citation. Before attributing a claim
   to the paper, quotes or tightly-paraphrases a specific passage recorded
   in `reading/<bibkey>/notes.md` — never characterizes the paper from
   memory alone.
8. Flags contradictions: if the new paper undercuts a claim on an existing
   wiki page, notes it under `## Caveats` rather than silently rewriting.
9. Runs the **shared post-ingest fan-out** above (wiki links, INDEX files,
   roadmap reconciliation, LOG, focus check).

Source-integrity rule (applies across all operations): the LLM **never
invents a citation**. If a claim needs a reference and none exists, it
writes `[CITE?]` inline and lists the gap under "### Missing citations" in
`LOG.md`. Better to ship with a visible `[CITE?]` than with a plausible but
wrong cite key. See §8 for the full verification protocol.

### Promote (roadmap entry → wiki, optionally → `Cn`)
When the work behind a roadmap entry is mature enough to be quoted:

1. Locate the matching `R<n>` in `ROADMAP.md`.
2. Confirm a wiki page exists for the work (creating one via
   `ingest-experiment` or `ingest-reference` first if not). The wiki page
   must satisfy the standard skeleton, including a `**Supports.**` line.
3. Update the roadmap entry: `status: promoted to [[wiki_page]]` with the
   obsidian wikilink. Add the reciprocal back-reference on the wiki page
   under `## Related` (`promoted from ROADMAP R<n>`).
4. Decide with the human whether the wiki page warrants a **new `Cn`** in
   `FOCUS.md` or simply strengthens an existing one. The LLM **never
   edits `FOCUS.md` without explicit human ratification** — it proposes,
   the human approves.
5. Append to `LOG.md`: `## [YYYY-MM-DD HH:MM] promote | R<n> → [[wiki_page]]`.
6. Run a focus-check.

If the work is found to be inconclusive or superseded, demote the entry
back to `in-progress` (with a note) or mark it `dropped` (with reason)
rather than promoting under-supported claims.

### Roadmap maintenance
Bookkeeping operations on `ROADMAP.md` itself:

- **roadmap-add `<category>` `<question>`** — append a new `R<n>` entry
  with `cost`, `prereqs`, `feeds`, `status: parked`. If the category does
  not exist, create it with `**Maturity gate.**` and `**Target stage.**`.
- **roadmap-update `R<n>` `<field>` `<value>`** — change a single field
  (status, cost, prereqs, feeds, notes). For status transitions, follow
  the lifecycle in §3.2.
- **roadmap-drop `R<n>` `<reason>`** — mark as dropped, move under
  `## Dropped` with the one-line reason. Do not delete.
- **roadmap-review** — walk every entry, flag stale ones (no movement in
  N weeks), check that `prereqs` resolve (no dangling `[[wiki_page]]`
  refs or missing `R<n>` ids), report categories with no entries
  (consider removing) or with too many (consider splitting), surface
  entries whose `feeds` no longer matches `FOCUS.md` (claim drift in
  either direction).

Each maintenance op appends an entry to `LOG.md` with prefix `roadmap-…`.

### Compose (draft a section)
When a draft section is due:

1. Open `wiki/INDEX.md`, collect the pages grouped under that `§`.
2. Read their headlines, numbers, caveats, and related-links.
3. Write the draft section by quoting wiki tables verbatim and citing
   `references/bibliography.bib` keys.
4. Hard rule: **do not introduce a number that is not already in a wiki
   page.** If the draft wants to say something the wiki has not said yet,
   update the wiki first.
5. Hard rule: **the draft does not cite roadmap entries.** Future-work
   discussion in the draft (typically §6 Discussion) may paraphrase
   `ROADMAP.md` content but never references `R<n>` ids — those are
   internal bookkeeping, not citable claims.

### Lint
Periodically ask the LLM to health-check the wiki:

- verify every `[[link]]` resolves to an existing file (including
  `[[reading/<bibkey>/notes]]` cross-folder links and `[[ROADMAP#R<n>]]`
  anchors);
- flag any Markdown-style `[text](path.md)` link to an internal note — all
  internal links must be obsidian `[[wikilinks]]` per §4.1;
- ensure every wiki page has all required sections;
- ensure every wiki page declares which `FOCUS.md` claim(s) it supports;
- detect orphan pages (no inbound `[[links]]`);
- detect contradictions between pages (same metric, two different numbers);
- detect stale claims superseded by a newer run;
- suggest topics that should exist but do not;
- suggest `references/bibliography.bib` gaps relative to the draft's claims;
- flag every BibTeX entry tagged `% TODO verify`;
- flag every `[CITE?]` marker in wiki or draft;
- flag every BibTeX entry without a matching `reading/<bibkey>/notes.md`
  (citation without an ingested source);
- flag every `reading/<bibkey>/` missing a `notes.md` or a BibTeX entry
  (source read but not citable yet);
- flag wiki pages whose claims do not map to any `Cn` in `FOCUS.md`
  (possible thesis drift); for genuinely off-thesis pages, suggest
  filing the topic as a new `R<n>` in `ROADMAP.md`;
- flag `ROADMAP.md` entries with `status: promoted` that lack a wiki
  back-reference, or whose target wiki page no longer exists;
- flag `ROADMAP.md` entries whose `prereqs` reference missing wiki pages
  or missing `R<n>` ids;
- flag `ROADMAP.md` entries that effectively duplicate an existing `Cn`
  in `FOCUS.md` (drift in the other direction — reverse promotion);
- flag stale `parked` entries (no movement in N weeks) for human review;
- re-check the draft against `FOCUS.md`: every `Cn` must be supported by at
  least one cited section; every cited section must map to a `Cn`.

---

## 7. Honesty principles

- Report metrics where the system loses, not only where it wins. Rank-based
  framing is fine; cherry-picking is not.
- Record caveats when a metric is noisy, biased, or outside its design
  domain (e.g. speech metric on music).
- Write down what was tried and failed. Negative results save the next
  experimenter from repeating the mistake and are load-bearing in Discussion.
- Numbers in the draft must match numbers in the wiki. If they disagree,
  fix the wiki first, then the draft.
- When the LLM integrates a new source, contradictions with existing wiki
  content are surfaced, not silently overwritten.
- Roadmap entries that fail are marked `dropped` with a reason, not deleted.
  A history of dead ends is part of the project's honesty.

---

## 8. Source verification protocol

Prior-art citations are the easiest place for an LLM to hallucinate: author
names, venues, and whole papers can be synthesized that look right but do not
exist. The rule set below exists to make that failure mode visible instead of
silent.

### 8.1 Zero-fabrication rule
The LLM **never invents a citation, a quote, a statistic, or a claim
attributed to someone else**. If a source is needed and is not in hand, the
LLM writes a visible placeholder (`[CITE?]`, `[STAT?]`, `[QUOTE?]`) and
surfaces the gap in `LOG.md` under "### Missing citations / verifications".

### 8.2 Evidence tiers
Each factual statement the LLM adds to a wiki page or draft section falls
into one of four tiers, and the tier must be legible to a future reader:

| tier | meaning                                         | convention |
|------|-------------------------------------------------|------------|
| T1   | Verified from the primary source in front of us | plain text, cite `[@key]` |
| T2   | Verified from a secondary / aggregated source   | cite `[@key]` + `## Caveats` note |
| T3   | Widely held, not verified by us                 | prefix "commonly reported" or similar, `[CITE?]` allowed |
| T4   | Inference or analogy from our own work          | clearly labeled as authors' reading |

T3 content must not survive into the submitted draft without promotion to
T1 or T2. Lint flags any `[CITE?]` left in `draft/`.

### 8.3 Bibliography-first, reading-backed rule
Before the LLM writes anything that asserts what prior art does, says, or
claims:

1. It opens `references/bibliography.bib` and checks that a matching entry
   exists.
2. It opens `reading/<bibkey>/notes.md` and confirms the claim being made
   is present there (or in the PDF alongside it, read in the same session).
   If `notes.md` does not exist yet, the paper has not been ingested; the
   LLM cannot cite it until ingest-reference runs.
3. If the BibTeX entry exists but is tagged `% TODO verify`, the LLM cites
   it only with a caveat and logs the verification TODO.
4. If no BibTeX entry exists, the LLM **does not invent one**. It writes
   `[CITE?]`, adds an entry to "### Missing citations" in `LOG.md`, and
   asks the human for the source or permission to search.

The chain of custody for every prior-art claim is therefore:

```
draft prose  →  wiki page (Supports Cn)  →  references/INDEX.md row
            →  references/bibliography.bib entry  →  reading/<bibkey>/notes.md
            →  reading/<bibkey>/source.pdf
```

Each arrow must resolve. Breaking the chain is the failure mode to avoid.

### 8.4 Uncertainty is loud, not silent
Whenever the LLM is less than confident about a fact, a number, a name, or
an attribution, it says so explicitly in the output rather than hedging
inside the wiki text. Acceptable forms:

- "⚠ unverified — <what needs checking>" inline in the wiki page;
- "LLM-check: <question>" prefix in the chat response;
- a bullet under `### Open verifications` in `LOG.md`.

Silent confident-sounding prose is the failure mode; visible uncertainty is
the intended behavior. Better to ask the human one extra question than to
ship a hallucinated citation.

### 8.5 Thesis / focus check
Every operation that touches prose (ingest, compose, promote, lint) ends
with a pass against `FOCUS.md`:

- Does the change still support the same `Cn` claims?
- Does it introduce an out-of-scope side-line that should be a roadmap
  entry instead?
- Does it contradict an earlier wiki page without a resolution?

If any answer is "maybe", the LLM raises it in the session rather than
merging silently. The paper's line stays coherent only if this check is
cheap and frequent.

---

## 9. Index and log

Two special files help the LLM (and you) navigate as the wiki grows.

### `wiki/INDEX.md` — content-oriented
- Lists every wiki page with a one-line hook.
- Grouped by **paper section** (§3 Representation, §4.1 Reconstruction,
  §4.3 Ablations, …) — not alphabetically — so the compose operation is
  mechanical.
- Each row: `- [ ] [topic](topic.md) — one-line hook`. Check `[x]` when the
  page is ready to be quoted in the draft.
- Sub-bullets list code / raw paths for quick recall.

### `LOG.md` — chronological
- Append-only record of ingests, composes, lint passes, decisions,
  promotions, roadmap changes.
- Consistent prefix per entry so `grep`-based tooling stays trivial, e.g.
  ```
  ## [2026-04-23 10:30] ingest-experiment | codec_comparison/20260423_082024_full
  ## [2026-04-23 11:15] roadmap-add       | Cat 3 / R3.4 — cross-genre evaluation
  ## [2026-04-23 14:05] ingest-reference  | gao2023codecresponse
  ## [2026-04-23 18:40] compose           | §4.1 Reconstruction
  ## [2026-04-23 19:00] promote           | R2.1 → [[spectral_fidelity]]
  ## [2026-04-24 09:12] lint              | 3 broken [[links]], 1 orphan, 2 stale roadmap entries
  ```
- Gives you a timeline of how the project grew and a cheap way for the LLM
  to recall what happened recently (`tail -n 20 LOG.md`).

---

## 10. LLM Bootstrap Prompt — portable

Paste this into a fresh session / fresh repo to recreate the workflow.
Replace `<PROJECT>` and the draft outline. Adapt the operations to the
domain; the shape stays the same.

````text
You are maintaining a scientific-paper notebook under `paper/` for <PROJECT>.
The notebook follows the llmwiki pattern: raw experiments + per-topic wiki +
bibliography + draft, with obsidian-style [[links]] between wiki pages.

Open these three files at the start of every session, in order:
  1. paper/METHOD.md   — authoritative schema (how the notebook works).
  2. paper/FOCUS.md    — thesis + numbered claims C1..Cn + out-of-scope.
  3. paper/ROADMAP.md  — forward-looking work catalog (R-entries by category).
Re-read FOCUS.md whenever you are asked to ingest a reference, compose a
section, promote a roadmap entry, or lint — every change must stay
on-thesis. Re-read ROADMAP.md whenever a new artifact or reference might
satisfy an existing entry, and during every ingest fan-out.

Layers and ownership:
  raw/          — our experiments. Read-only for you.
  reading/      — prior-art sources (PDFs + per-paper notes.md). Read-only
                  for the source.pdf, writable for notes.md and INDEX.md.
  wiki/         — per-topic Markdown notes. You own this layer.
  references/   — BibTeX + annotated index. You own this layer.
  draft/        — paper sections. The human owns framing; you quote wiki.

Top-level orientation files (by tense):
  METHOD.md   — timeless rules.        Owner: human. You re-read, do not edit.
  FOCUS.md    — present claims (Cn).   Owner: human. You propose; never edit
                                       without explicit ratification.
  ROADMAP.md  — future work (Rn).      Owner: human + you. You append, update
                                       status, archive promoted/dropped
                                       entries; the human approves the
                                       category structure.
  LOG.md      — chronological record.  Owner: you. Append-only.

Rules:
1. Raw is authoritative. Never edit raw data. Wiki summarizes raw.
2. Every wiki page uses this skeleton; required sections must be present:
     # <topic> — <one-line framing>
     **Headline.**       (required, 1–3 sentences with the claim)
     **Supports.**       (required, list of FOCUS.md claim ids, e.g. "C2, C4")
     ## Raw              (required)
     ## Numbers          (optional)
     ## Figures          (optional)
     ## Caveats          (optional)
     ## Draft target     (required, "→ §X.Y …")
     ## Related          (required, at least two [[links]];
                          include "promoted from ROADMAP R<n>" if applicable)
3. Links use obsidian wikilinks, not Markdown [text](path.md):
     - wiki ↔ wiki:                    [[basename]]
     - wiki ↔ reading note:            [[reading/<bibkey>/notes|<bibkey>]]
     - any page → references index:    [[references]]
     - any page → roadmap entry:       [[ROADMAP#R<n>|R<n>]]
     - raw artifacts / code paths:     backticks, no link
     - non-note targets (.bib, URLs):  Markdown link allowed
   Every cross-folder [[link]] uses the pipe alias form so displayed text
   stays short while the path keeps basename collisions unambiguous.
4. The draft may only quote numbers that already appear in a wiki page.
   If the draft needs a new number, update the wiki page first. The draft
   never cites R<n> ids — those are internal bookkeeping.
5. Report losses and caveats, not only wins. Flag contradictions between
   a new source and existing wiki content instead of silently overwriting.
6. Bibliography lives in references/bibliography.bib. Cite by BibTeX key.
   Keep references/INDEX.md in sync with every added or removed entry.
7. wiki/INDEX.md groups pages by draft section. Tick [x] when a page is
   ready to be quoted in the draft.
8. Append an entry to LOG.md for every ingest, compose, promote, roadmap,
   or lint pass. Entry prefix: `## [YYYY-MM-DD HH:MM] <op> | <target>`.

Roadmap rules (ROADMAP.md):
9a. Catalog work that is not yet started but intentionally tracked.
    Organize by category, not by timeline. Each entry is `R<n>` or
    `R<cat>.<n>` with required fields cost / prereqs / feeds / status.
9b. Status lifecycle: parked → in-progress → promoted to [[wiki_page]],
    or → dropped (with one-line reason). Promoted and dropped entries
    are archived under their own headings, never deleted.
9c. Promotion to a new Cn in FOCUS.md is a human-ratified step. You
    propose; you do not edit FOCUS.md without explicit approval.
9d. During every ingest fan-out, walk ROADMAP.md and update entry
    status if the new artifact / reference satisfies it. Crossing the
    `promoted` threshold is raised to the human.
9e. When an experiment or reference is genuinely off-thesis but worth
    keeping, file it as a new R<n> rather than smuggling it into the
    wiki under a stretched Cn.

Source-integrity rules (non-negotiable):
10. Never invent a citation, a quote, or a statistic attributed to another
    work. If a reference is needed and not in hand, write [CITE?] inline
    and log the gap under "### Missing citations" in LOG.md.
11. A citation [@bibkey] is valid only if (a) bibliography.bib has the
    entry AND (b) reading/<bibkey>/notes.md exists AND (c) the claim being
    made is present in notes.md (or confirmed in source.pdf this session).
    No notes.md → no citation.
12. Before attributing a claim to a paper, verify the claim is actually in
    that paper. Do not characterize a paper from memory alone.
13. Tag unverified BibTeX fields with "% TODO verify <field>" and cite the
    entry with an explicit caveat until verified.
14. When confidence is low — on a fact, a number, a name, an attribution —
    say so loudly. Acceptable markers: "⚠ unverified — …" in the wiki,
    "LLM-check: …" in chat, a bullet under "### Open verifications" in
    LOG.md. Silent confident prose is the failure mode to avoid.
15. Every change that touches prose ends with a focus-check against
    FOCUS.md: does it still support the declared Cn claims, introduce
    out-of-scope material that should be a roadmap entry, or contradict
    an earlier wiki page without resolution? Raise any "maybe" to the
    human before merging.

Operations you will be asked to perform. `ingest-experiment` and
`ingest-reference` are mirrored: raw/ and reading/ are handled
symmetrically, and both trigger the same post-ingest fan-out (wiki
updates, obsidian links refreshed, INDEX files updated, roadmap
reconciliation, LOG entry, focus check against FOCUS.md).

- "ingest experiment <dir>" — read raw artifacts, create or update the
  matching wiki page (with the Supports line), refresh obsidian [[links]]
  and ## Related, tick wiki/INDEX, reconcile ROADMAP entries, append to
  LOG, focus-check.
- "ingest reference <citation>" — create reading/<bibkey>/ with source.pdf
  and a notes.md following the template, verify metadata, add BibTeX
  entry, update references/INDEX, update reading/INDEX, update wiki pages
  that cite it (attribution only to claims recorded in notes.md; every
  new citation backed by a [[reading/<bibkey>/notes|<bibkey>]] link in
  ## Related), confirm the role maps to a Cn in FOCUS.md, reconcile
  ROADMAP entries, append to LOG.
- "ingest from reading" / "ingest from raw" — shorthand: any new content
  under reading/ or raw/ that has not yet been fanned-out. Run the
  corresponding ingest op on each pending item and report a summary.
- "promote R<n>" — graduate a roadmap entry to a wiki page. Confirm /
  create the wiki page (skeleton + Supports line), set the entry's
  status to "promoted to [[wiki_page]]", add the back-reference under
  the wiki page's ## Related, raise to the human whether a new Cn is
  warranted, append to LOG.
- "roadmap-add | roadmap-update | roadmap-drop | roadmap-review" —
  maintenance on ROADMAP.md (see METHOD §6). Each call appends to LOG.
- "compose §X.Y" — open the wiki pages grouped under §X.Y in wiki/INDEX.md,
  confirm each supports a Cn, write the draft section by quoting their
  tables and citing by BibTeX key. Never introduce numbers not in the
  wiki. Never introduce citations not in bibliography.bib. Never cite
  R<n> ids in the draft.
- "lint" — walk the wiki, report broken [[links]] (including
  cross-folder [[reading/<bibkey>/notes]] and [[ROADMAP#R<n>]] anchors),
  any Markdown-style [text](x.md) link to an internal note (obsidian-only
  rule), missing required sections, missing Supports lines, orphans,
  contradictions, stale claims, bibliography gaps, [CITE?] markers,
  unverified BibTeX TODOs, Cn drift, BibTeX entries missing a
  reading/<bibkey>/notes.md, reading directories without a BibTeX entry
  or a notes.md, ROADMAP entries promoted-without-wiki, ROADMAP entries
  with dangling prereqs, ROADMAP entries that duplicate an existing Cn,
  stale parked entries.

Outputs:
- Edit files directly. Do not dump page content into chat.
- Keep prose terse and technical. Tables over paragraphs.
- Preserve the existing section order and formatting of touched files.
- Surface all uncertainty explicitly; prefer one extra question over a
  confident-sounding guess.

Draft outline for <PROJECT>:
  §1 Introduction
  §2 Related work
  §3 Method
  §4 Experiments
  §5 <project-specific>
  §6 Discussion
Override with the project's actual draft/OUTLINE.md when available.
````

---

## 11. Tips and tricks

- **Obsidian** is the best front-end: open the `paper/` directory as a
  vault, browse the graph view to see wiki structure, follow `[[links]]`
  while the LLM edits. No index plugin required at small scale.
- **Dataview** (Obsidian plugin) runs queries over YAML frontmatter. If the
  LLM tags wiki pages (e.g. `section: "4.1"`, `status: "draft"`), Dataview
  produces live tables of what's ready vs. missing. The same trick works
  on `ROADMAP.md` if you tag entries with frontmatter — a Dataview query
  can render a live "what's parked / in-progress / promoted" board.
- **Git.** The whole `paper/` directory is a plain Markdown repo. Commit on
  every meaningful ingest / compose / promote. Branches for alternative
  framings; `ROADMAP.md` makes a good branch-merge negotiation surface
  because reordering categories is a small textual diff.
- **PDF ingest.** For prior art, `raw/pdfs/` holds the PDF, `references/`
  holds the BibTeX key, and one or more wiki pages hold the annotated
  summary. The LLM can read PDFs directly; keep the annotation in the wiki
  so the draft can quote without re-reading.
- **Plot provenance.** Every figure in the draft maps 1:1 to a `raw/*.png`
  referenced from a wiki page. If a figure script changes, re-render, update
  the wiki, then the draft.
- **One source of truth per claim.** If a number appears in multiple wiki
  pages, one page owns it and the others `[[link]]` to it. Avoids drift.
- **Roadmap as conversation seed.** When opening a planning session with
  the human, the LLM can `tail` the most recently moved entries in
  `ROADMAP.md` and ask which to advance — cheaper than re-reading the
  whole wiki to find what's next.

---

## 12. Why this works for papers specifically

The tedious part of writing a paper is not the thinking — it is the
bookkeeping. Looking up exact numbers from six different JSON dumps,
remembering which caveat you promised to address in §4.3, re-checking whether
the cited baseline was Encodec-6 or Encodec-12, making sure the claim in §1
survived the final ablation, recalling which idea you parked three weeks ago
because it was waiting on a calibration that has since landed.

LLMs do not get bored, do not forget which run produced which figure, and can
touch fifteen wiki pages in one pass when a new experiment changes the
landscape. The wiki stays maintained because the cost of maintenance is near
zero. The roadmap stays alive because reconciling it during every ingest is a
mechanical step the LLM is happy to repeat.

Your job is curation and framing: which experiments to run next, which paper
to read, how to argue the tradeoff, what caveats to own, whether a roadmap
entry has earned a `Cn`. The LLM's job is everything else.

The pattern is the scientific-paper specialization of Bush's Memex and
Karpathy's llmwiki: a personal, persistent, LLM-maintained knowledge base,
where the human controls intent and the LLM handles compounding the context.

---

## 13. Note (also: when to break the rules)

This document is intentionally opinionated but modular. The exact directory
layout, the skeleton sections, the LOG prefix — all of it can be adapted.
Pick what is useful, drop what is not.

Examples of reasonable deviations:
- small project: drop `LOG.md`, keep a single `INDEX.md`;
- very small project: drop `ROADMAP.md` too — `FOCUS.md` plus a `## Open
  work` section per wiki page is enough until categories of future work
  start to repeat;
- collaborative paper: require every ingest to land in a PR so humans review
  LLM updates before merge; treat `ROADMAP.md` reorderings as PRs too so
  scope changes are reviewable;
- domain without BibTeX pressure: fold `references/` into `wiki/` and cite
  inline;
- domain with massive raw data: move `raw/` out of the repo, reference by
  absolute path from wiki pages.

The shape that must be preserved across every variant:
- raw is read-only,
- wiki is the single point of truth for claims,
- draft quotes wiki only,
- `FOCUS.md` is human-ratified; `ROADMAP.md` (when present) feeds it,
- the schema (this file) is where conventions live.

Everything else is style.
