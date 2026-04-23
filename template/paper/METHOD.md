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
distraction (log and move on), or a symptom of drift (revisit the thesis)?
Silent drift into off-thesis content is the failure mode this section
exists to prevent.

The LLM must consult `FOCUS.md` before any compose, lint, or reference
ingest and check that the resulting change stays on-thesis. When in doubt,
it raises the conflict to the human rather than guessing.

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

### 4.1 Link conventions (obsidian-style, non-negotiable)

All internal links across the notebook use **obsidian wikilinks**, not
Markdown `[text](path.md)`. This keeps Obsidian's graph view intact and lets
the LLM rely on basename resolution during refactors.

| link target                                | form                                             | example                                               |
|--------------------------------------------|--------------------------------------------------|-------------------------------------------------------|
| wiki page → another wiki page              | `[[basename]]`                                   | `[[complex_patch]]`                                   |
| any page → a reading note                  | `[[reading/<bibkey>/notes\|<bibkey>]]`           | `[[reading/vinyard2025sparse/notes\|vinyard2025sparse]]` |
| any page → the references index            | `[[references]]`                                 | `[[references]]`                                      |
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

---

## 5. Folder layout

```
paper/
  METHOD.md                # this file (schema)
  FOCUS.md                 # thesis + claims (C1..Cn) + out-of-scope — see §3.1
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

Naming conventions:
- Raw directories: `YYYYMMDD_HHMMSS_<slug>` — timestamp first so `ls` sorts
  chronologically, slug describes the experiment.
- Wiki files: `<topic>.md`, lower_snake. The basename is the `[[link]]` key.
- Draft files: `s<section>_<slug>.md`, e.g. `s4_1_reconstruction.md`.
- BibTeX keys: `<firstauthor><year><slug>`, e.g. `wang2003shazam`.

---

## 6. Operations

Four operations cover almost everything the LLM needs to do. `ingest-experiment`
and `ingest-reference` are **mirrored**: same shape, same fan-out, different
source-of-truth directory (`raw/` vs `reading/`).

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
6. **LOG.md.** Append an entry: `## [YYYY-MM-DD HH:MM] <op> | <target>`
   with a short body of what was touched.
7. **Focus check.** Re-read `FOCUS.md` and confirm the change still maps
   to the declared claims. On "maybe", raise to the human.

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
   LOG, focus check).

Source-integrity rule (applies across all operations): the LLM **never
invents a citation**. If a claim needs a reference and none exists, it
writes `[CITE?]` inline and lists the gap under "### Missing citations" in
`LOG.md`. Better to ship with a visible `[CITE?]` than with a plausible but
wrong cite key. See §8 for the full verification protocol.

### Compose (draft a section)
When a draft section is due:

1. Open `wiki/INDEX.md`, collect the pages grouped under that `§`.
2. Read their headlines, numbers, caveats, and related-links.
3. Write the draft section by quoting wiki tables verbatim and citing
   `references/bibliography.bib` keys.
4. Hard rule: **do not introduce a number that is not already in a wiki
   page.** If the draft wants to say something the wiki has not said yet,
   update the wiki first.

### Lint
Periodically ask the LLM to health-check the wiki:

- verify every `[[link]]` resolves to an existing file (including
  `[[reading/<bibkey>/notes]]` cross-folder links);
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
  (possible thesis drift);
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
Every operation that touches prose (ingest, compose, lint) ends with a
pass against `FOCUS.md`:

- Does the change still support the same `Cn` claims?
- Does it introduce an out-of-scope side-line?
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
- Append-only record of ingests, composes, lint passes, decisions.
- Consistent prefix per entry so `grep`-based tooling stays trivial, e.g.
  ```
  ## [2026-04-23 10:30] ingest-experiment | codec_comparison/20260423_082024_full
  ## [2026-04-23 14:05] ingest-reference  | gao2023codecresponse
  ## [2026-04-23 18:40] compose           | §4.1 Reconstruction
  ## [2026-04-24 09:12] lint              | 3 broken [[links]], 1 orphan page
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

Open these two files at the start of every session, in order:
  1. paper/METHOD.md   — authoritative schema (how the notebook works).
  2. paper/FOCUS.md    — thesis + numbered claims C1..Cn + out-of-scope.
Re-read FOCUS.md whenever you are asked to ingest a reference, compose a
section, or lint — every change must stay on-thesis.

Layers and ownership:
  raw/          — our experiments. Read-only for you.
  reading/      — prior-art sources (PDFs + per-paper notes.md). Read-only
                  for the source.pdf, writable for notes.md and INDEX.md.
  wiki/         — per-topic Markdown notes. You own this layer.
  references/   — BibTeX + annotated index. You own this layer.
  draft/        — paper sections. The human owns framing; you quote wiki.

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
     ## Related          (required, at least two [[links]])
3. Links use obsidian wikilinks, not Markdown [text](path.md):
     - wiki ↔ wiki:                    [[basename]]
     - wiki ↔ reading note:            [[reading/<bibkey>/notes|<bibkey>]]
     - any page → references index:    [[references]]
     - raw artifacts / code paths:     backticks, no link
     - non-note targets (.bib, URLs):  Markdown link allowed
   Every cross-folder [[link]] uses the pipe alias form so displayed text
   stays short while the path keeps basename collisions unambiguous (every
   reading/<bibkey>/ has a notes.md — basename alone is not unique).
4. The draft may only quote numbers that already appear in a wiki page.
   If the draft needs a new number, update the wiki page first.
5. Report losses and caveats, not only wins. Flag contradictions between
   a new source and existing wiki content instead of silently overwriting.
6. Bibliography lives in references/bibliography.bib. Cite by BibTeX key.
   Keep references/INDEX.md in sync with every added or removed entry.
7. wiki/INDEX.md groups pages by draft section. Tick [x] when a page is
   ready to be quoted in the draft.
8. Append an entry to LOG.md for every ingest, compose, or lint pass.
   Entry prefix: `## [YYYY-MM-DD HH:MM] <op> | <target>`.

Source-integrity rules (non-negotiable):
9. Never invent a citation, a quote, or a statistic attributed to another
   work. If a reference is needed and not in hand, write [CITE?] inline
   and log the gap under "### Missing citations" in LOG.md.
10. A citation [@bibkey] is valid only if (a) bibliography.bib has the
    entry AND (b) reading/<bibkey>/notes.md exists AND (c) the claim being
    made is present in notes.md (or confirmed in source.pdf this session).
    No notes.md → no citation.
11. Before attributing a claim to a paper, verify the claim is actually in
    that paper. Do not characterize a paper from memory alone.
12. Tag unverified BibTeX fields with "% TODO verify <field>" and cite the
    entry with an explicit caveat until verified.
13. When confidence is low — on a fact, a number, a name, an attribution —
    say so loudly. Acceptable markers: "⚠ unverified — …" in the wiki,
    "LLM-check: …" in chat, a bullet under "### Open verifications" in
    LOG.md. Silent confident prose is the failure mode to avoid.
14. Every change that touches prose ends with a focus-check against
    FOCUS.md: does it still support the declared Cn claims, introduce
    out-of-scope material, or contradict an earlier wiki page without
    resolution? Raise any "maybe" to the human before merging.

Operations you will be asked to perform. `ingest-experiment` and
`ingest-reference` are mirrored: raw/ and reading/ are handled
symmetrically, and both trigger the same post-ingest fan-out (wiki
updates, obsidian links refreshed, INDEX files updated, LOG entry,
focus check against FOCUS.md).

- "ingest experiment <dir>" — read raw artifacts, create or update the
  matching wiki page (with the Supports line), refresh obsidian [[links]]
  and ## Related, tick wiki/INDEX, append to LOG, focus-check.
- "ingest reference <citation>" — create reading/<bibkey>/ with source.pdf
  and a notes.md following the template, verify metadata, add BibTeX
  entry, update references/INDEX, update reading/INDEX, update wiki pages
  that cite it (attribution only to claims recorded in notes.md; every
  new citation backed by a [[reading/<bibkey>/notes|<bibkey>]] link in
  ## Related), confirm the role maps to a Cn in FOCUS.md, append to LOG.
- "ingest from reading" / "ingest from raw" — shorthand: any new content
  under reading/ or raw/ that has not yet been fanned-out. Run the
  corresponding ingest op on each pending item and report a summary.
- "compose §X.Y" — open the wiki pages grouped under §X.Y in wiki/INDEX.md,
  confirm each supports a Cn, write the draft section by quoting their
  tables and citing by BibTeX key. Never introduce numbers not in the
  wiki. Never introduce citations not in bibliography.bib.
- "lint" — walk the wiki, report broken [[links]] (including
  cross-folder [[reading/<bibkey>/notes]]), any Markdown-style [text](x.md)
  link to an internal note (obsidian-only rule), missing required sections,
  missing Supports lines, orphans, contradictions, stale claims,
  bibliography gaps, [CITE?] markers, unverified BibTeX TODOs, Cn drift,
  BibTeX entries missing a reading/<bibkey>/notes.md, reading directories
  without a BibTeX entry or a notes.md.

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
  produces live tables of what's ready vs. missing.
- **Git.** The whole `paper/` directory is a plain Markdown repo. Commit on
  every meaningful ingest / compose. Branches for alternative framings.
- **PDF ingest.** For prior art, `raw/pdfs/` holds the PDF, `references/`
  holds the BibTeX key, and one or more wiki pages hold the annotated
  summary. The LLM can read PDFs directly; keep the annotation in the wiki
  so the draft can quote without re-reading.
- **Plot provenance.** Every figure in the draft maps 1:1 to a `raw/*.png`
  referenced from a wiki page. If a figure script changes, re-render, update
  the wiki, then the draft.
- **One source of truth per claim.** If a number appears in multiple wiki
  pages, one page owns it and the others `[[link]]` to it. Avoids drift.

---

## 12. Why this works for papers specifically

The tedious part of writing a paper is not the thinking — it is the
bookkeeping. Looking up exact numbers from six different JSON dumps,
remembering which caveat you promised to address in §4.3, re-checking whether
the cited baseline was Encodec-6 or Encodec-12, making sure the claim in §1
survived the final ablation.

LLMs do not get bored, do not forget which run produced which figure, and can
touch fifteen wiki pages in one pass when a new experiment changes the
landscape. The wiki stays maintained because the cost of maintenance is near
zero.

Your job is curation and framing: which experiments to run next, which paper
to read, how to argue the tradeoff, what caveats to own. The LLM's job is
everything else.

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
- collaborative paper: require every ingest to land in a PR so humans review
  LLM updates before merge;
- domain without BibTeX pressure: fold `references/` into `wiki/` and cite
  inline;
- domain with massive raw data: move `raw/` out of the repo, reference by
  absolute path from wiki pages.

The shape that must be preserved across every variant:
- raw is read-only,
- wiki is the single point of truth for claims,
- draft quotes wiki only,
- the schema (this file) is where conventions live.

Everything else is style.
