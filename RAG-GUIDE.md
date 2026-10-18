# RAG Project Guide

A distilled, retroactive guide for starting **any** RAG project. It answers, at each fork,
*what applies, when, and under what conditions* — not every technique exists in every project.

## How to use this guide

- **Organized by project phase (maturity), not by technique catalog.** Read top-to-bottom for a
  new project; jump to a phase when you're there.
- **Every phase follows the same shape:** *Technologies* (what & why) · *Patterns & practices*
  (with failure modes where they bite) · *Decision points* (where the path forks).
- **Decision points** carry up to three fields, and only where they earn it:
  - *Maturity* — when in the project's life this earns its place.
  - *Applicability condition* — **static** filter, known up front: what must be true of your
    corpus / queries / requirements for this to make sense at all. This prunes whole branches.
  - *Trigger signal* — **dynamic** symptom, observed at runtime, that says *now it's time*.
  Unconditional practices carry no schema — there the question is *how*, not *if*. Don't force
  the template.
- **Only implemented-and-validated knowledge lives here.** It converges; it does not accumulate
  theory. If something is merely read, it isn't in this guide yet.

---

## Phase 0 — Frame the problem before writing code

Answer three questions first. They are the applicability conditions you'll reference at every
downstream fork, so write them down.

1. **Corpus** — size, stability, structure. *Bounded and stable* → in-context (CAG) is viable.
   *Large, growing, or needing per-query subset selection* → retrieval (RAG).
2. **Queries** — lookup vs. synthesis vs. multi-hop; single-shot vs. conversational.
3. **Traceability** — does every answer need a verifiable source? If yes, this shapes your
   **output contract from day one** (Phase 2). Retrofitting source-tracking after the fact is
   expensive.

> *Example (ECSS normative corpus):* large, hierarchical, cross-referenced, with mandatory
> clause-level traceability → a retrieval project, not CAG. But it can still be **bootstrapped**
> with CAG over a bounded slice (Phase 3).

---

## Phase 1 — Service skeleton & foundations

### Technologies

- **Web framework — FastAPI (ASGI/async).** LLM calls are long (2–30 s) and I/O-bound. An async
  framework serves many concurrent calls per worker while each awaits the model; a
  sync/threaded framework blocks one worker per in-flight call and exhausts its pool under load.
- **Config — Pydantic `BaseSettings`.** Loads environment variables *and* type-validates them at
  startup.
- **Dependencies — a lockfile-based manager** (e.g. `uv`). Reproducible installs.

### Patterns & practices

- **Layered separation.** Transport (`routers`) / business logic (`services`) / data contracts
  (`schemas`) / reference data (`context`). **Thin routers**: receive, delegate, return — no
  prompt building, no SDK calls. Logic lives in services.
- **Isolate the provider behind a single frontier function (a "seam").** Even before you build a
  full provider abstraction, funneling every SDK call through one function keeps business logic
  provider-agnostic and gives the eventual abstraction a clean insertion point. Don't scatter
  `client.create(...)` across the codebase. *(The seam is the insertion point for the
  multi-provider abstraction in Phase 4 — you fill its body, you don't move it.)*
- **Async correctness — a failure mode, not a preference.** Wrapping a *blocking* SDK call in
  `async def` does **not** make it non-blocking; it freezes the event loop for the entire call
  and forfeits the only reason you chose an async framework. Use the SDK's **native async
  client** (`AsyncAnthropic`, `AsyncOpenAI`, …), or offload to a thread (`asyncio.to_thread`)
  when no async client exists. With an aggregation library (Phase 4), this goes further than
  loop-blocking: the **sync and async code paths can carry different fallback logic** (verified
  in LiteLLM's Router source — `router.acompletion()` and `router.completion()` do not behave
  identically). Async isn't only about concurrency there; the sync path is a different code path.
  *(Tutorial/course snippets frequently get this wrong — they call a sync client, or iterate a
  sync stream, inside `async def`. Don't copy them.)*
- **Fail-fast config.** Required secrets have **no default** → the app refuses to start if they're
  missing, instead of surfacing the failure on a user's first request. Keep secrets out of code:
  `.env` (git-ignored), with `.env.example` documenting the required keys. *(Contrast with
  optional/degradable dependencies — Phase 4: not everything should fail-fast.)*

### Decision points

- **Provider / model choice** — deferred cheaply behind the seam. When you do pick:
  - *Practice:* **verify model currency at decision time.** Model lineups and the "cheapest
    current tier" move fast; the obvious default from any written material may already be legacy.
    Confirm the model ID, price, and status before wiring it.

---

## Phase 2 — Output contract first

Define the **response schema before building retrieval**. Lock what an answer looks like — its
fields, and above all how it carries its sources.

> **Governing principle for a traceability system — the citable/context frontier.** Everything that
> reaches the model splits into two epistemic classes: **material the answer is grounded in and may
> cite** (the corpus — authority), and **everything else around the question** (history, user
> attachments, profile knobs, distilled "facts" — context *about* the question, never citable as
> source). A parallel frontier separates **generating** an answer from **verifying** it. These two
> frontiers are the same rule seen twice, and they generate a recurring design test: *almost
> anything the generic playbook solves by letting the LLM infer, adapt, remember, summarize, or
> self-validate, a traceability system must instead solve with an **explicit parameter**, a
> **deterministic invariant**, or a **separation of responsibilities**.* The reason is singular: the
> product cannot let the LLM introduce **unverified text that affects fidelity**. Downstream nodes
> that look unrelated — "history is a cache gate not a slot," "summary is contraindicated," "a knob,
> not extraction," "tier adapts output not citations," "emitting ≠ verifying," and the indexing-phase
> trio "the embed-text is not the cite-text," "the internal index key is not the citation coordinate,"
> "no chunk mixes authority grades" — are all instances of this one frontier. When a new feature blurs
> it (letting generated/echoed/summarized text sit where cited material sits), that blur *is* the bug,
> however well-formatted the output.
>
> **Refinement — citable material has two authority grades, not one.** "Cite verbatim, never
> paraphrase" is too strong as a blanket rule; the grade of the *source* decides both whether the
> citation is **identity** or **provenance**, and how much literal fidelity the source demands.
> **Normative** material (a requirement in a standard): the citation *is* the substance's identity;
> transcribe literally (paraphrasing a norm alters it); it can be the whole answer. **Informative**
> material (a recommendation in a handbook): the citation is the *provenance* of guidance; synthesis
> is allowed (a summarized recommendation is still the recommendation); **it too can be the whole
> answer** — an informative source is autonomous, not always subordinate to a norm. Two consequences
> that are easy to get wrong: (1) synthesis relaxes the fidelity of the *content*, **never of the
> coordinate** — a summarized recommendation needs its "source §Y" as exact as a transcribed
> requirement needs its clause ID, because it sustains an answer on its own; (2) the authority grade
> is a property of the **section**, not the document — an *informative* annex can sit inside a
> normative standard, and a *normative* annex inside a handbook, so resolve grade per section, and
> trust what the document **declares** about a section over what you **infer** from its type.
> Corollary for the refusal contract: **out-of-material becomes conjunctive** — "not covered"
> requires the absence of *both* a requirement and a recommendation, since an informative-only answer
> is valid, not out of scope.

### Patterns & practices

- **If traceability is required (Phase 0), put a citations structure in the schema from v0** —
  even when there's nothing yet to verify against. Every later component (retrieval, generation)
  then targets a fixed output shape; retrofitting a source-tracking contract after those exist is
  costly.
- **Grounding instruction in the system prompt.** Answer *only* from the provided material; if the
  answer isn't there, say so explicitly — do not fabricate. Cheap to add, and in practice it
  behaves correctly from the first plainly-stated version.
- **Citation anchors: prefer stable structural identifiers over layout-derived coordinates.** A
  clause/section ID is semantic and edition-independent; a page number is a render artifact (e.g.
  DOCX has no intrinsic pages — pagination is computed at layout time, so "page" may simply not
  exist in your source). If your source format doesn't carry the coordinate your contract
  promises, treat that as a signal the coordinate is the *wrong anchor*, not that you must
  reconstruct it. Make positional coordinates **optional**; make structural ones the backbone.
  *(For a normative corpus, "ECSS-E-ST-40C §5.2.4.8" is more verifiable to a reviewer than any
  page number.)*
  - *Resolve the anchor from the source's canonical representation, don't recompute it.* The
    structural coordinate may not live in the extractable text at all: in DOCX, section numbering is
    **auto-generated by Word from `numbering.xml`** (`w:numPr`/`ilvl`), so copying a heading yields
    `"In case 1:"` without its `8.6.2` — the number exists only when the list is rendered. A parser
    that **counts** headings to reconstruct the number (its only option without resolving the list)
    desynchronizes and **fabricates identity** — and that failure is mostly *silent*: a counter that
    emits a plausible-but-wrong `E.3.1` where the document shows `E.2.1` trips no invariant. Resolve
    the number from the source's own numbering definition instead; it yields the coordinate the
    reader sees when they open the file — **faithful by construction, verifiable against the render**.
    The general node: *derive citable identity from the source's canonical representation, not from a
    computation of your own that you can't check against it.* And the discipline that makes it safe:
    **any ambiguous path returns "no number," never a guessed one — not-numbered beats mis-numbered**,
    the same rule as declining to invent a page number or a UID that isn't there. (Hierarchy level
    likewise comes from the resolved number, not the paragraph *style* — the style only says "this is
    a heading.")
- **Emitting citations ≠ verifying them.** *Emitting* = transporting the coordinates of the
  material you fed the model (a v0 schema concern). *Verifying* = proving the answer actually used
  what it cites and didn't hallucinate a reference (a later generation-quality concern). The
  schema field's mere presence does not mean verification is done — mark this explicitly so a
  later phase doesn't assume it's solved. *Once retrieval is wired, verification splits into three
  levels* — membership, literalness, support — and only the first two are deterministic (Phase 3.5,
  *Wiring retrieval into generation*).

### Decision points

- **Citation granularity** — document / section / clause / page.
  - *Applicability condition:* driven by what your corpus structure **and source format** actually
    support, and by how precise a citation your users need to verify a claim. Don't promise a
    granularity your data can't back.

- **Structured output — force the model to return a typed shape (JSON Schema), or accept free
  text?** *(the LLM as a typed function, not a text box)*
  - *Maturity:* the **shape** (a Pydantic/JSON-Schema model) belongs in v0 alongside the citation
    structure; **enforcing** it on the model is due when a consumer downstream parses the output.
  - *Applicability condition:* your **product consumes the output as data** (renders components,
    iterates fields, feeds another system) rather than showing prose to a human. If the deliverable
    is prose a person reads, free text is fine and a schema is overhead. The failure it removes: a
    brittle parser (regex / second LLM) between "the model said something" and "the frontend renders
    it" — a piece that breaks silently on any format drift (prompt edit, model update, provider
    swap).
  - *Mechanism (same idea, three packagings):* OpenAI **Structured Outputs** (native param),
    Anthropic **forced tool-use** (a tool whose `input_schema` is the shape, `tool_choice` pinned),
    others via an **aggregator**. A library (Instructor) unifies them and adds validation retries.
    *Build-vs-buy tension:* Instructor is itself a provider-abstraction layer — if you already funnel
    calls through an aggregation Router (Phase 4), verify the two compose rather than fight over the
    seam, and beware **stacking two retry layers** (the library's + the Router's). With two providers
    and an existing seam, forcing the schema as a tool through your own Router can be lighter than a
    second abstraction.
  - *Structural caveat — form is not truth (link to guardrails, Phase 5):* a schema guarantees the
    **shape** of the answer, never its **correctness**. A `Citation{standard, clause}` with a perfect
    shape says nothing about whether that clause exists or the model invented it with impeccable
    formatting. For a traceability corpus this is the same **emitting ≠ verifying** split above:
    structured output cleans up *emission* (typed `{answer, citations[]}` instead of citations buried
    in prose), it does **not** do verification — that stays a later generation-quality phase.
  - *Streaming interaction:* if the *whole* response becomes one structured object, there is nothing
    to stream token-by-token (partial JSON is ugly). This is an argument for a shape where the long
    generated text stays streamable and the structure (citations, metadata) is a single terminal
    event — the same split the streaming node in Phase 4 already wants.

---

## Phase 2.5 — Turn the raw corpus into a citable artifact (the data layer)

The least "RAG-like" phase and the most under-taught: acquiring, parsing, and validating the corpus
**before** any vectorization. It is intrinsically ad-hoc (every corpus is different, which is why
courses skim it), but for a traceability system it is where the citation contract is either earned or
silently broken. Applicability: any project whose answers are grounded in a fixed corpus; the heavier
the fidelity requirement, the more this phase earns.

> **The governing failure mode of this whole phase: correctness degraded in silence.** A clean,
> homogeneous, public corpus suffers the classic data failures (silent version mixing, rotten
> sources, invisible gaps) *as much as* a messy one — and **more dangerously**, because the absence of
> visible dirt removes the suspicion that would trigger a look. Nearly every node below is one
> instance of this.

### The root node — absence is not observable from what's present

*Counting what you have never tells you what's missing.* A verifier globbing `*.docx` sees 14 files
and reports OK — without mentioning the 12 `.doc` it never looked at. A reference graph built over 14
documents doesn't notice 12 are absent. Tables dropped by the parser leave no trace they existed.

*The general node (agnostic):* **a system that only inspects what it holds cannot know its own
absences.** Detecting what's missing requires an **external declaration of what should be there** —
iterate the *manifest of the declared*, not the *directory of the present*. Silent absences are the
dominant failure mode of data pipelines, and they are invisible-by-construction to any check that
doesn't reconcile against a declared list. Everything else in this phase is this node applied to a
different stage.

### Patterns & practices

- **Verify the artifact, not the process that claims to have produced it.** The meta-node of the
  data layer, in five guises: don't trust a server's `Content-Type` → check the file's **magic
  bytes** (a 200 can be an HTML login page); don't trust a converter's **exit 0** → check the output
  file exists and is valid (LibreOffice returns 0 producing nothing — `soffice.com` vs `soffice.exe`
  on Windows); don't trust a **counter** → resolve against the source's own numbering (Phase 2); don't
  trust *"the two implementations agree"* → check against an **external truth** (render, human ground
  truth). *When you delegate to a process whose success report you don't control — a server, an
  external tool, another implementation, an LLM — verify the produced artifact against an independent
  truth, not the return code or the self-report.* This is the node "form is not truth" generalized to
  the whole pipeline.
  - *Corollary — no LLM in ingestion (for a traceability corpus).* An LLM flattening the source into
    structured clauses would transcribe normative text — and can silently change "shall"→"should",
    hallucinate a clause, or "fix" a significant typo, with **no way to detect when it did** (the
    output is plausible). It moves fabrication to the **foundation** (the corpus, not one answer),
    where it is permanent and undetectable. The rule that places the LLM correctly: *it may touch what
    is verified afterward (generation with a citation auditable against the corpus) or what is not a
    source of truth (offline diagnostics a human reviews) — it may not **be** the process that creates
    the source of truth.* Ingestion creates the source of truth → deterministic territory.

- **Coverage is reconciliation against a declared manifest — two artifacts with opposite life
  cycles.** Keep the **declarative manifest** (what *should* be in the corpus: canonical document
  identity *with revision*, tier, type, default authority, `corpus_version` — hand-editable,
  committed, the versioned source of identity) separate from the **generated processing report**
  (what *resulted*: coverage by relevance, identity coverage, table counts by mechanism, graph state
  — regenerated each run, "do-not-edit" header, committed for its **trajectory**, à la lockfile:
  a coverage drop visible in `git log`). Mixing them yields the file nobody knows whether to edit or
  regenerate. The manifest is what turns "count the files" into "reconcile against the declared," and
  it's what lets a missing document be classified **by relevance** — a missing CORE is a failure, a
  permanently-PDF-only ADJACENT is a warning. *(Related: don't declare as a failure what has no
  actionable fix — an alarm that can't be silenced trains everyone to ignore it, and the ones that
  matter with it.)*

- **A version identifier names what it actually distinguishes.** `corpus_version` derived as
  `{date}.{digest over sorted doc-ids}` detects changes in the **declared set** (adding/revising a
  document) but **not** in content (if the publisher reissues under the same ID, it can't tell).
  Naming it "version of the declared set," not "version of the corpus," is the honesty that stops you
  from trusting it beyond what it measures — the "constants without false authority" rule applied to
  an identifier. Any derived artifact (parsed JSON, embeddings, stored answers) should be tie-able to
  the `corpus_version` it was built from.
  - *Corollary — a human label is not an invalidation key; fingerprint the real inputs.* Because
    `corpus_version` names the declared *set*, it does not move when the *processing* changes under a
    fixed set: three later parser changes altered the output while `corpus_version` stayed
    `{same}`, and derived chunks from before and after carried the same label while being different.
    The label is for humans (citation, manifest); **staleness detection is a separate mechanism** — a
    **content fingerprint** stored on each derived artifact, a digest over *what the stage actually
    consumed* (for chunks: the processed content; for embeddings: the embed-texts **plus model and
    params**, since the same text under another model is another vector). A derived artifact is stale
    ⇔ the fingerprint of its current input ≠ the one it recorded. **The fingerprint chains**: each
    stage digests its own input, and since one stage's input is the previous stage's output, a change
    anywhere upstream propagates as a changed input downstream — no manual "bump the version" (that is
    *true by discipline*, the thing this replaces with *true by construction*). And the check is
    honest about ignorance: a three-state `is_stale()` returns **`None`** when it can't know (no
    artifact, or no fingerprint recorded) — a "don't know" must not disguise itself as "it's fine."

- **Immutable-source / derived-staging directory split.** `raw/` (exactly what the server gave,
  immutable, one writer) → `parse_ready/` (a single homogeneous input dir for the parser — native
  `.docx` copied in, legacy `.doc` converted in) → `processed/` (parser output). The parser reads
  **one** directory, never `raw/`. This closes filename-collision and original-vs-derived mixups **by
  construction** (a glob can't confuse them if they live in different dirs), and makes `parse_ready/`
  and `processed/` derivable-and-disposable (rebuild from `raw/`; don't commit).

- **Table extraction routes by type; the test of "flatten-able" is the failure of the flattening,
  not a judgment of the shape.** A merged cell can encode three different things (column-spanning
  header, row-grouping, parent-child nesting); a mechanism that treats them as one will get one right.
  Don't *judge* whether a table is linearizable — **attempt to unfold it and see if a consistent grid
  results**; if resolving the merges leaves ragged rows, there's no faithful linear order → mark it
  **not-extractable** and give the coordinate. That produces a **third result, neither citation nor
  refusal**: "clause X defines this via a structure not reproduced here; see the document" — honest by
  construction (it exists + exact coordinate + not transcribed), where a faithful-looking flattening
  would lie about what the structure says. *(And "discard noise" must name its unit — a junk column
  inside a useful table doesn't make the table junk; drop the column, keep the table.)*

- **Construction-time guarantee over runbook check.** Prefer a structure where the bad case can't
  exist to an invariant someone must remember to check. A label heading modeled as **not-a-Section**
  (so "sections" means "citable clauses," no filter to remember downstream); a table with **no
  identifier of its own** (impossible to cite as if it were a clause); a generated report that
  **overwrites itself** on regeneration (editing it isn't "discouraged," it's undone); verification
  by **conservation** — but conserved *against the source*, not against your own output: a sum of
  tables-per-mechanism checked against the parser's own table list can never fail (it was shipped
  that way and was green for sessions); it only rules out silent discard when the total comes from the
  raw document (its non-nested `w:tbl`), which the parser cannot shape.
  *A precondition someone will skip isn't a precondition, it's a bug with good manners* — enforce it,
  don't document it. *(All-or-nothing belongs here too: a derived artifact consumed as complete is
  replaced only if every declared item produced output — 25 of 26 isn't "almost right," it's a
  smaller corpus occupying a complete one's slot, a silent failure the log dresses up.)*
  - *Corollary — one accessor, or "iterate the declared" (the root node) re-decays into N directory
    scans.* The root node holds by construction only if every consumer reads the declared set through
    **one accessor**; left to each module it re-grows as separate `glob(processed/*.json)` calls with
    divergent ad-hoc exclusion lists — observed as six modules drifting apart, one crashing on a
    derived artifact it mistook for a source document and a coverage report reading **107%**. That one
    accessor also owns **canonical order**: a fingerprint over a *set* must digest it sorted (as
    `corpus_version` already does), or a semantically-null reorder of the hand-edited manifest changes
    the digest and forces a needless re-embed — make order the accessor's property and every downstream
    fingerprint inherits it, instead of each digest site having to remember to sort (one didn't, and
    passed only because the manifest's declaration order *happened* to match sorted). And "sorted" is
    not yet enough: the accessor sorts by *key*, so a digest that sorts by filename (`{key}.json`) is a
    *different* canonicalization, and the two diverge exactly when one key is a prefix of another —
    inert until a corpus has that shape, then a content-neutral addition moves the fingerprint. The
    invariant is *one ordering function* used at the accessor and every digest site, not merely "both
    sorted." The single scan
    that stays legitimate is the **inverse** question — "what is present that the manifest does *not*
    declare?" — which cannot come from the manifest; give it one central registry of derived artifacts,
    never the per-module blacklists that were the bug.

- **A latent defect masked by an accidental precondition activates when the precondition changes.**
  Numbering errors in handbook sections were harmless *only because* another decision (citations hang
  off requirements; handbooks have none) kept them from surfacing — they activate the day chunking
  uses the section as the unit (to reach the handbooks at all), and then contaminate everything built
  on top, retroactively. So repair the latent defect **before** the precondition flips, even while
  it's inert. *(Smaller instance: a `rename`→`replace` bug that slept because the code only ever wrote
  to absent destinations, woken by a re-download feature. When you add a capability that exercises a
  previously-dead path, suspect the APIs whose correctness relied on that path staying dead.)*

### Validating a data-derived extractor — measure before you build on it

- **Precision and recall are not equally grave; the domain sets the weight.** For a reference
  extractor feeding a graph/metadata layer: a false positive is noise a later stage can catch (don't
  assume a reranker will — Phase 3.5 measured one that couldn't separate hard negatives); a false
  negative is *a connection the system doesn't know exists*. In a traceability system the **omission
  (recall) is the expensive failure**, so recall governs the decision to build on the extractor at
  all — a graph over 48%-recall references is debt shaped like a feature. Measure it; don't assume it.

- **Ground truth must be more reliable than what it measures — not its peer.** *"Two implementations
  agree" is not correctness* — they can agree **in the error** if they share hypotheses (a parser and
  a numbering resolver both saying `E.1.1`; an extractor and an LLM both missing the same reference
  because both skim tables). The dangerous false positive is not the disagreement (noisy,
  investigated) but the **agreement-in-error** (silent, filed as success). So: an LLM may **assist**
  ground-truth labeling (propose, the human verifies) but may not **be** the ground truth. And for
  recall specifically, "review the disagreements" **inherits the combined recall of both fallibles**
  and is blind to the case where both miss — the human must be able to **add** what no machine
  proposed, not just filter candidates. *(Corollary: a measurement instrument is software and can
  carry the bug — validate it against external truth before believing its aggregate, and suspect the
  instrument first when the result is large and uniform.)*

### Decision points

- **Freeze vs. live corpus code.** Freezing the ingestion pipeline is legitimate as a **deferral with
  an exit condition** ("frozen until the project reaches the data-handling phase"), not a permanent
  state. When the condition is met, unfreeze deliberately and update every doc that asserted the
  freeze — a freeze without a written exit condition becomes cargo-culted "don't touch."

- **Graph vs. chunk-metadata for cross-references.** *(open, corpus-dependent)* Don't reach for graph
  infrastructure because references *exist* — reach for it because the product needs to *reason over*
  references (multi-hop paths, centrality), not merely *record* which exist. If runtime only ever
  needs 1-hop expansion, an adjacency table (`chunk → [referenced clauses]`) + an inverse index covers
  it without a graph object. *Applicability:* decide this only once chunking (what a chunk is, hence
  what metadata it can carry) and extractor validation (are the references even reliable?) exist —
  deciding earlier pre-empts the fork. *Signal it's the wrong tool:* the graph object is built but
  production only ever consumes the flat adjacency.
  - *Sub-fork resolved (graph object vs. adjacency index):* if runtime is 1-hop, an adjacency index
    built from the reference list (an in-memory dict) beats a graph object (`nx.DiGraph`) — the graph
    object is not a runtime dependency; keep it, if at all, for offline tooling/multi-hop analysis and
    off the hot path. What's built at chunk time to make *any* of these possible is a **ref-ready
    chunk**: it carries its own clause-ID plus its outgoing references resolved to native-granularity
    target-IDs, and stays **agnostic to how retrieval consumes them** — the material is the same for
    adjacency, metadata, or the third option below. (Populating those references is itself gated on
    the extractor's validation — don't fill the slot from a stale/unvalidated extractor.)
  - *Third fork the original framing missed:* the choice is not only "graph vs. metadata" but "graph,
    metadata, or **bake cross-reference awareness into the embedding**" (Contextual-Retrieval-style:
    the embed-text of a chunk names what it references, so a query about the referenced thing retrieves
    it **without traversing** anything). It doesn't let you *walk* references at query time, but it
    dissolves part of the need to. Records the fork; the decision stays open and corpus-dependent.

---

## Phase 2.7 — From citable artifact to retrieval index (chunking & embedding)

The data pipeline's continuation of Phase 2.5: the parsed, citable artifact becomes the vector index
retrieval will query (parse → **chunk → embed** → index). It shares the data layer's governing
failure mode (correctness degraded in silence) with a sharper twist proven here: **content can be
complete and still wrong** — the parts all present, the *relation between them* lost. Placed next to
the data layer because it is that pipeline's continuation and inherits its method; in *time* you
bootstrap the answer loop with CAG (Phase 3) over a bounded slice first and build this full index
only when the slice stops being enough.

### Patterns & practices

- **Dispatch chunking by content anchor, not by document type.** The anchor (an atomic unit like a
  requirement vs. a prose section) is **mixed within one document** — a normative standard holds both
  requirements and section prose; some documents hold only sections. Run each chunker over every
  document and let it emit its own anchor; routing by document *type* drops whatever anchor that type
  "isn't." Authority travels with the **content** (an atomic unit inside an informative annex is
  informative), never deduced from the document type.
- **The chunk carries embed-text and cite-text as separate fields** *(instance of the Phase-2
  frontier)*. `embed_text` = a static contextual header + the verbatim source (enriched so it's
  retrievable in isolation); `cite_text` = the verbatim source **only**, clean. If citation extraction
  ever reads `embed_text`, the synthetic header gets "cited" — the blur *is* the bug.
- **The contextual header is static and deterministic — not LLM-generated.** Built from parent
  fields you already hold. It wins on cost *and* on **reproducibility** (same input → same embed-text →
  same vector), which an LLM contextualizer forfeits by injecting non-determinism into the index.
  Identifiers (clause/standard IDs, version) stay **out of the vector** (embeddings are blind to them)
  and live in filterable metadata.
- **Don't flatten structure at chunk time either.** The parser preserved nesting/indentation because
  a flattened hierarchical list *asserts something different* (six sibling states vs. two with
  sub-states); the chunker must not re-flatten it when composing `embed_text`/`cite_text`. Fixing
  faithfulness upstream and re-breaking it here is the same defect, one layer down.
- **A qualifier bound to a unit travels with that unit, not as a sibling.** A paragraph that qualifies
  an atomic unit (its aim, its expected output) belongs *in that unit's chunk*, under its coordinate —
  treated as free-standing section prose it decouples from what it qualifies and fuses with its
  neighbours. (The binding is a structural association the parser owns; that it's citable is a
  contract decision.)

### Decision points

- **Model choice — take a standard model; don't run a tournament unless measurement demands it.**
  *Verify currency at decision time* (Phase 1). Keep it behind the seam + fingerprint so it stays
  reversible; a full model benchmark before retrieval even exists is over-engineering. What you *do*
  run is the domain probe below — which verifies the model isn't blind to your critical distinction,
  a different thing from picking the best model.
- **Embedding determinism ≠ chunking determinism — don't test the wrong one.** Chunk generation is
  deterministic (test by equality). The embedding API may return small numerical variation between
  calls, so its guarantee is **input-identity** (the fingerprint), not output-equality; a
  vector-equality test across runs is a green-passing test that will fail someday for reasons unrelated
  to the code. Two stages of one pipeline with two classes of guarantee — say which is which out loud,
  so no one assumes the strong one where only the weak holds.
  - *Refinement (measured when adding a chunk field later) — each stage's fingerprint digests its own
    **input**, and a stage with two consumers needs two keys.* Scope the invalidation key to what the
    consumer consumes (the parsed documents for the chunker; the embed-text for the embedder), not to
    the chunk's whole output record. Then adding a non-embedded **identity** field to the chunk moves
    *neither* fingerprint — the vectors stay valid, no re-embed — and "must I re-embed?" stays owned by
    the embedder's own key. A key over the whole record would fire a **spurious re-embed on an
    identity-only edit** — the same identity-vs-content conflation `embed≠cite` warns against, one level
    up.
- **Reference-target addressing — resolve to native granularity by ID-component prefix.** A
  reference's target granularity is set by the *reference* (a clause, a whole section, a table), not
  chosen by you; make the citation ID **prefix-addressable by component** so "all of §5.2" is a
  component-prefix match — never a string prefix (`"5.20"` is not under `"5.2"`). This yields cheap
  parent-child addressing with no physical parent index.
- **Measure the representation before building retrieval on it — the domain probe.** Before indexing,
  embed a handful of **real** domain pairs (built from the corpus, not hand-written), including a
  **trap pair**: two texts whose entire distinction is a bare identifier. Read the *structure* (does
  the trap pair sit below the should-be-similar pairs?), not the absolute number. Catching a collapse
  here reads it correctly — as a property of the source text the dense representation can't capture;
  the identical finding surfacing later in a retrieval evaluation reads as "retrieval is broken" and
  burns weeks tuning k, reranking, and prompts. **Measuring a layer's property before the next layer
  is built separates "the model can't" from "my upper layer is misconfigured."**
- **Dense-only vs. hybrid retrieval — hybrid is a *measured requirement*, not a default, when the
  corpus's critical distinction lives in exact-match identifiers.** *(forward pointer — the hybrid
  mechanism is a later retrieval phase; only the **measurement** graduates now.)* Dense embeddings
  compress *meaning* and discard exact lexical identity; a corpus whose discriminating tokens are
  identifiers or rare jargon (criticality B vs. C, clause/standard IDs) has its distinction exactly
  where dense is blind and lexical search (BM25) is strong — the two are complementary, not ranked.
  - *Applicability condition (static):* the discriminator between confusable clauses is **lexical-exact,
    not semantic**. (A corpus whose distinctions are semantic is served by dense alone.)
  - *Trigger signal (dynamic):* the trap pair's dense similarity does **not** fall below the
    should-be-similar pairs (measured on a real ECSS pair: 0.98 — *above* them, two tokens differing in
    42).
  - *Honest limit (now closed — graduated in Phase 3.5):* the probe measured the **representation**
    collapsing the distinction; that **retrieval** then fails was **confirmed query-side in Phase 3.5**,
    where the hybrid mechanism was built and measured. The forward-looking half held — because the
    corpus condition was known at index time, the dense index design left room for the lexical channel,
    so adding it was a slot-fill, not a redesign. *What the probe could not predict, and Phase 3.5
    measured:* the fix works via lexical **repetition density** (not the shared identifier tokens), and
    it **helps identifiers/tables while hurting generic queries**. The apparent lever — fusion
    weighting — turned out to be a symptom of a broken lexical channel; and identifiers that name a
    whole *document* need routing, not hybrid. See Phase 3.5.
- **Count tokens with the real tokenizer before the window becomes load-bearing.** A ~4 chars/token
  estimate is fine to start, but measured against a real tokenizer it **over**-estimates prose
  (~0.82×) and **under**-estimates tables (up to ×2.3). Under it, 287 chunks exceeded a 512-token
  window — 272 of them tables that were never split — and 204 exceeded even the *estimated* window
  without being flagged. The cost surfaces downstream (silent truncation in any fixed-window model,
  e.g. a reranker). Since changing the counter moves every cut and forces re-chunk + re-embed, batch
  it with the next corpus rebuild rather than patching it alone.

*(Techniques excluded by prior nodes, restated at their point of use: chunking that rewrites the
source — propositional/LLM decomposition, cluster chunking that loses the source location — is out
for **normative** material by the traceability and authority-grade rules of Phase 2; it may be
admissible for **informative** material, on cost/benefit. The embedding seam is the Phase-1 seam
applied to a second provider surface: the frontier of **one request**, with batching left to the
caller as a rate-limit policy; the derived vector artifact is stamped per the fingerprint corollary
in Phase 2.5.)*

---

## Phase 3 — Bounded-context vertical slice (CAG) before retrieval

Get an **end-to-end answer loop** working with reference knowledge injected directly into the
prompt (CAG) **before** adding any retrieval machinery.

**Why:** it validates the entire vertical — schema, grounding, citation contract, provider seam —
against real model behavior with the fewest moving parts. No vector store, no embeddings, no data
pipeline to debug in parallel.

### Patterns & practices

- **Keep injected reference data behind the same seam retrieval will later replace.** The service
  must not know whether reference data came from a static file or a semantic search — it just
  receives *formatted context*. This makes the CAG→RAG transition a **substitution, not a
  rewrite**.
- **Order the prompt to serve two goals at once.** Put everything **static** first (role,
  grounding instruction, citation instruction, output format, reference material) in the system
  prompt; put the only **variable** part (the user's query) last. This maximizes positional
  attention on the query *and* creates a stable static prefix that a future prompt-cache boundary
  can sit behind — without reordering anything later.
- **Expose the context as its own component, even when it lives inside the system prompt.** If a
  cache key or a future retrieval step will need "the context" as a distinct thing, split the
  prompt builder so *static instructions* and *context block* are separately addressable now
  (same final string, separable parts). This is what makes the cache key in Phase 4 — and the
  retrieval swap in the RAG phase — a slot-fill rather than a refactor.
- **Move the static instructions to a file artifact, not a string constant in code.** The prompt
  is a software artifact: living in `app/prompts/…/system.md` (loaded once at module level) makes a
  prompt-only change a **legible diff a non-coder can review in a PR**, editable without opening the
  service. This is separate from — and cheaper than — a template engine (see decision point). The
  *context* stays where the data lives (a static file in CAG, retrieval in RAG); only the
  *instructions* (the contract with the model) move to the artifact. The final composed string is
  unchanged, so it is a pure refactor.
- **Guard a prompt refactor with a characterization snapshot.** Capture the composed prompt as a
  golden snapshot *before* touching anything; after the move, assert byte-identical (the refactor
  changed origin, not content); after a deliberate content change, update the snapshot — its **diff
  in the PR *is* the review of the prompt change**. The snapshot then stays as regression. (This is
  how "the prompt is reviewable like code" becomes real rather than aspirational.)

### Decision points

- **Prompt structure: raw file vs. template engine (Jinja2/ERB/…).**
  - *Applicability condition:* a template engine earns its weight only once the prompt has
    **conditionals or variable interpolation inside the static block** (per-parameter branches,
    `{% include %}` of few-shot sets). While the prompt is a single static text with the only
    variable part (the context/query) composed in code, a file-read-to-string loader is enough — the
    engine is a dependency (and an injection/escaping surface) with no benefit yet.
  - *Trigger signal (sharpened):* not "there's multi-turn" and not "there's a tier" per se, but the
    first of either **partials reused across prompt variants** (`{% include %}` of a shared block
    into two-plus templates) **or a real conditional branch inside the static block**. Multi-turn
    history that is just prepended as messages does *not* branch the static prompt, so it does not
    trip this; a per-tier template that shares a `_context.j2` partial *does*. Re-evaluate at that
    point, not before — and when it trips, a hand-rolled `str.replace` loader stops being defensible
    (partial composition is exactly what the engine buys).
  - *Note — delimiters (XML tags vs Markdown):* Anthropic attends to XML tags, OpenAI to Markdown
    headers; both understand both. If the system prompt must stay **provider-agnostic** (Phase 4),
    the delimiter style is part of that contract — changing it forces re-validation across the wired
    providers, it is not a free cosmetic choice.

- **Start with CAG, or go straight to RAG?** *(marquee decision)*
  - *Maturity:* bootstrap / MVP.
  - *Applicability condition (static):* your reference knowledge **fits in the context window and
    is stable**. If the real corpus is large, growing, or requires selecting the relevant subset
    per query, CAG is not the *destination*.
  - *Trigger signal (dynamic) to leave CAG:* you need more reference material than fits; or you
    need to pick which references are relevant per query; or the corpus updates faster than you
    can re-edit static context.
  - *Note — CAG as a scaffold even when the project is unambiguously RAG:* when the real corpus
    already fails the applicability condition (large + traceability, e.g. ECSS), CAG over a
    **deliberately bounded slice** is still worth doing as an **architectural bridge**. It stands
    up the output contract and the full vertical slice cheaply. You're not "doing CAG to keep it"
    — you're using a bridge whose expiry condition you already know. Choose the slice to be
    representative (e.g. a self-contained requirement that itself contains a cross-reference), so
    the slice exercises the structure retrieval will later have to handle.

- **Enriching context at runtime from external sources — attachments, web search, business DB.**
  *(recorded as a fork; for a traceability corpus most branches are pruned, but the reasoning is the
  node)*
  - *The real axis is not "where the context comes from" but **who decides it enters and whether it's
    deterministic**.* **Pre-injection** (the system decides, before the call — an attachment fills the
    same `context` slot retrieval will) vs **tool-call** (the model decides, inside an agentic loop —
    web search, DB query). The second is an agent-phase change (function calling, M5), not a context
    source you swap in; don't smuggle it into a CAG phase. And determinism gates the cache: an
    attachment is deterministic given the file (fills the `context` slot cleanly); **web search is
    non-deterministic** (a cached answer built on yesterday's search, served today, is a phantom
    citation from cache via a new door); a **tool loop has no context *before* the call at all**, so
    the `guards → cache → LLM` order stops holding as-is. Any dynamic content must also sit **after**
    the static prefix, or it invalidates the provider prompt-cache on every request.
  - *Rule 1 — dynamic context is **input, not program**.* External content is delimited and never
    given to the model as instructions (classic injection). This is where the injection guardrail's
    applicability condition actually bites: **"untrusted content of arbitrary size enters the prompt,"
    not "there is user input."** A short, moderated question is one thing; an arbitrary attachment (or
    a client-owned history) is another category — and delimiting with a tag is necessary but *not*
    sufficient (the attacker closes your tag).
  - *Attachments — multimodal vs local extraction is **not** a free choice for a traceability system.*
    Two conditions close it: (a) **a fallback that can't accept the same input isn't a fallback** — a
    multimodal PDF path crosses the provider seam (verify the aggregation layer even passes document
    blocks rather than flattening them; don't assume), so if the primary takes the PDF natively and
    the backup can't, the rotation doesn't save you. (b) **No parse, no coordinates** — the multimodal
    model "saw" the file but you never parsed it, so you can't cite section/clause. Under mandatory
    traceability, **local extraction is forced**, and the extraction code is the first piece of the
    eventual chunking pipeline. (The generic "either is defensible" holds only for systems with
    neither a fallback nor a citation requirement.)
  - *Web search is **contraindicated** for a normative corpus* — a channel that pulls plausible
    outside text into a system whose premise is "answer *only* from the injected material" is a
    fabrication vector with a confident face (same shape as the semantic-cache contraindication). The
    only web that matters (is there a newer standard revision?) is *offline acquisition*, not runtime
    context.
  - *Business-DB context — the AI service never touches the operational DB directly* (schema coupling,
    over-broad credentials, duplicated business rules); it calls an authenticated HTTP endpoint that
    applies the business's own rules. Keep the distinction sharp: **the vector store is the AI
    service's knowledge base; the operational DB is the business's** — they speak by HTTP contract,
    never a shared DB. For a single-service product with no business backend, this branch is N/A by
    absent infrastructure.
  - *Governing discipline:* start with the minimum context, measure, add dynamic context only on
    evidence the system needs it — "more context can't hurt" is false (more tokens, more latency, more
    injection surface). And **budget/truncation of dynamic context is not neutral**: truncating a
    specification silences requirements (the same integrity failure as `finish_reason == "length"`),
    and *if it doesn't fit, that non-fit is the trigger signal for retrieval*, not a budget knob.

---

## Phase 3.5 — Retrieval: replace the CAG context slot with vector search

The CAG→RAG substitution the seam (Phase 1) was built for: the `context` slot stops being a static
slice and becomes the top-k of a semantic search over the Phase-2.7 index. Build it as a
**standalone, measurable retrieval layer first** — a `/search` that embeds a query and returns cited
chunks — and wire it into generation only afterwards (a standalone retriever keeps the data objective
from tangling with the generation one; the wiring has its own section below). The **starting index is
exact (sequential scan), not approximate** — see the marquee decision below.

### Patterns & practices

- **Add a vector store on two independent axes — and know which one is driving.** What forces a real
  store and what forces a *fancy index* are different questions. **The four properties** —
  persistence, concurrency, joins with relational data, transactions — decide whether you need a store
  at all (an in-process array dies with the process, diverges across replicas, can't join typed
  metadata, can't roll back a partial ingest). **Scale** decides whether you need an *ANN index* on
  top. They don't move together: a small corpus can still need the store (for the joins and the
  transaction), and a stored corpus can still run exact search while it's small. For a traceability
  corpus the decisive driver is the **join + coherence**, not scale.
  - *Store choice under traceability — single-system ACID beats a faster dedicated store when
    coherence is correctness, not just performance.* A dedicated vector DB (Qdrant/Weaviate) has
    superior pre-filtering, but it keeps vectors in one system and citation metadata in another, and
    **their coherence becomes your job**. For a corpus whose value *is* the vector↔citation binding, a
    drift between the two is a phantom citation by the back door — so the single-system atomic query
    (below) is worth more here than the dedicated store's speed. Generic guidance ranks the dedicated
    store higher because it optimizes throughput, not coherence-as-correctness — a different objective.
    *(This is why the store is Postgres+pgvector — vectors and citation metadata in one ACID system —
    rather than a dedicated vector DB beside a relational one.)*
- **Filter inside the top-k, in one atomic statement — not after it.** The typed filters (authority,
  document, chunk type) go in the `WHERE` of the *same* statement as the `ORDER BY <distance> LIMIT k`
  and the join to the document table. Filtering *after* a top-k is a measured failure mode: ask for 5,
  get 1, because the other 4 were filtered out post-hoc — and it gets **silently worse the more
  selective the filter**. This single atomic statement *is* the property that justified the
  single-system store; a vectors-here-metadata-there split cannot do it without coordinating two
  systems. *(In the exact regime the filter just narrows the rows the scan sees; the "a selective
  filter starves an ANN index of candidates" problem — and its `iterative_scan` remedy — is an ANN
  concern that doesn't arise until you add an index.)*
- **The retrieval result carries cite-text, never embed-text** *(instance of the Phase-2 frontier at
  the API boundary)*. The result exposes the verbatim citable text and the citation coordinates; the
  embedded contextual header stays out — and the discipline is to **not even `SELECT` the embed-text
  column**, so no downstream consumer can echo the synthetic header as source. Convention keeps it out
  of the contract; not selecting it keeps it out by construction.
- **Choose the distance operator to match the eventual index's metric — even before the index
  exists.** With no index the operator choice has no measurable effect (every operator is the same
  scan), so it costs nothing to fix now; but an ANN index built for one metric **silently falls back
  to a full scan** for a query written with a different operator — no error, a 100–1000× latency
  regression that surfaces months later in a system that "has an index." Pick the metric once (cosine,
  for text embeddings — and for **robustness**, since cosine doesn't depend on the vectors staying
  normalized where inner-product would), and write every query with its operator from day one.
- **Embed the query through the same seam that embedded the corpus, and guard the dimension at the
  source.** Same model/dims → same space *by construction*, not by discipline. Still assert the query
  vector's dimension before searching: a changed embedding model would put query and corpus in
  different spaces and the distances become **noise with the shape of a result** — five ranked,
  plausible, unrelated hits, with no downstream signal that anything is wrong. Guard where the fault
  is produced, not where it would (never) be noticed.

### Decision points

- **Exact (sequential scan) vs. approximate (ANN) index — the marquee retrieval decision.**
  - *Maturity:* **exact is the MVP** and the correct *starting* state, not a placeholder; ANN is a
    scaling node.
  - *Applicability condition (static):* a **total-recall requirement** (traceability: a neighbor
    dropped by approximation can be a requirement dropped) makes exact the default *while it's
    feasible*. Generic RAG treats 98% recall as ≈100%; a normative corpus does not. Exact is also the
    **ground-truth oracle** any future index is measured against, so you build it regardless.
  - *Trigger signal (dynamic):* the **measured p99 of the database read** crosses the interactive SLO
    (~100–200 ms) — read it off the DB time, **not** the end-to-end. (Measured here: 65/69 ms p50/p99
    in the DB at ~14k vectors, well under SLO; the ~200 ms of end-to-end on top is the query's
    embedding network call, which no index touches — reading the trigger off the e2e number would add
    a vector index to fix a *network* latency. Linear projection to a full ~60–100k corpus nears
    350–450 ms and would cross it — but a projection is not a measurement, and the rule is *measure*.)
  - *When you do leave exact, go to HNSW, not IVFFlat, for a **growing** corpus:* IVFFlat needs
    training and **degrades recall silently as inserts drift from the k-means partition it was built
    on** — exactly the silent recall-loss a traceability corpus can't tolerate; HNSW absorbs inserts
    incrementally. (Build-time `m`/`ef_construction` are informed 1536-dim community defaults, not a
    project decision; query-time `ef_search` is the one you'd tune — and only *after* hybrid and
    reranking exist, since they change the vector channel's recall budget.)
  - *What exact also buys, beyond recall:* it removes three families of operational complexity the ANN
    path carries — index memory sizing, index build parameters, index maintenance (reindex/vacuum) —
    each returning *deliberately* when scale forces the index, not before. And in the exact regime
    `EXPLAIN` showing a **`Seq Scan` is correct** — the acceptance criterion is inverted vs. the
    generic "you must see an index scan."

- **Distance is a ranking device, not a calibrated confidence score.** Measured twice: the
  out-of-domain archetype and the generic-in-domain archetype **overlap** in distance (no threshold
  separates "the corpus doesn't have this" from "it has it, generically"); the *worst*-behaving
  semantic archetype (the identifier trap) had the **lowest** distances of all; and hybrid improved
  that archetype while its **mean distance got worse**. Two consequences. (1) An "I don't know" gate
  **cannot** be a distance threshold — it lives in the generator's grounding/refusal instruction
  (Phase 5), not in a retrieval cutoff. (2) Fuse channels by **rank, not by score** (RRF, below):
  building fusion on the distance scale would rest on a number these measurements show doesn't support
  that reading. *(Reinforces the Phase-2 restraint: don't convert distance into a `1 − d` "similarity"
  and threshold on it.)*

- **Recall-in-k, ranking, and answer-quality are three different failures — measure which one you
  have.** A correct chunk can be *absent from* the top-k (recall), *present but mis-ranked* (ranking),
  or *present and well-ranked but not actually answering* (quality). They need different fixes, and a
  single query set can exhibit more than one: the identifier trap was **both** — for one clause the
  discriminating chunk was in the top-5 but out-ranked by chunks that merely enumerate the terms
  (ranking); for another, the discriminating table sat at rank 6, **outside** the top-5 (recall). Don't
  accept "the vector fails" as a diagnosis — name which failure, because the cheap diagnostic (exact
  regime, recall 1.0 verified against the brute-force oracle) is precisely what lets you tell them
  apart. *And a downstream caveat that resizes the problem:* if generation receives the top-k, a chunk
  mis-ranked but still **inside** k is available to the generator, so the ranking failure may be
  **absorbed** downstream — measure whether the mis-rank is load-bearing for the *generated answer*
  before investing in fixing retrieval ranking for its own sake.

- **Dense-only vs. hybrid retrieval — the Phase-2.7 forward pointer, built and measured here.** The
  index-time probe predicted this (trap pair dense-similar at 0.98); the retrieval phase confirmed the
  failure query-side and built the fix.
  - *Mechanism:* two channels — exact cosine (dense) and lexical `ts_rank` over a full-text index —
    each carrying the **same filters**, fused by **Reciprocal Rank Fusion**. RRF is rank-based: no
    score normalization between two incomparable scales, and (per the node above) it avoids building on
    the uncalibrated distance scale. Two statements + fusion in application code, not one
    full-outer-join SQL — the fusion becomes pure, unit-testable logic without giving up filtering in
    SQL. The lexical index is built over **cite-text, not embed-text** (embed≠cite, third instance:
    indexing the header would fabricate lexical matches from the section path).
  - *Measured result — a surprise that refuted the pre-registered hypothesis.* Hybrid recovered and
    re-ranked the identifier trap where dense alone failed. The predicted reason it *wouldn't* work
    (the distractors share the same tokens `b`/`c`/`criticality`) was wrong: the discriminating content
    lived in **tables**, which repeat those tokens far more than the short definitional distractors,
    and `ts_rank` weights by term frequency. So **lexical repetition density turned out to be a usable
    proxy for "this fragment breaks the topic down by category"** — a structural property the dense
    model missed that had a lexical correlate after all. *(General node: a lexical channel can't tell
    "explains the difference" from "mentions the terms," but it can tell density — and in a tabular
    corpus density proxies for breakdown.)*
  - *The other half of the measurement — hybrid also **hurts**.* On a generic query ("verification
    requirements") the lexical channel matched thousands of chunks, ordered them by density, and —
    because **unweighted RRF gives the lexical channel an equal vote always** — degraded the result.
    The tempting conclusion is "tune the fusion weights." **It was wrong, and that's the node:**
    *"weight the fusion" is usually a symptom, not a cause — fix each channel alone before weighting
    them.* Measured against a golden set, the degradation had two causes inside the lexical channel,
    neither in RRF:
    1. **Asymmetric tokenization.** The application split the query on non-alphanumerics *before*
       the database saw it, so `ECSS-E-ST-40C` reached Postgres as `ecss | e | st | 40c` — four
       common tokens (6–10% document frequency each, in OR) — while the indexed documents kept the
       compound (0.7% df). The discriminating term was never searched. Fix: let the **same parser
       that built the index** tokenize the query (same text-search config), keep the compound,
       drop its hyphen parts. *A tokenizer on one side only is a silent asymmetry: every query still
       returns results.*
    2. **`ts_rank` has no IDF.** It scores by term frequency and proximity inside each document, with
       no corpus statistics — so in a corpus whose documents share vocabulary ("requirements",
       "software"), long chunks that repeat common terms win (the rare discriminating term appeared in
       1 of the top-10 lexical hits). The "ts_rank vs BM25 difference is noise" claim common in
       tutorials holds for heterogeneous corpora, **not** for a normative one. Length normalization
       (`ts_rank(…, 1)`, ÷ 1+log length) or BM25 (a df table computed once per ingestion — no
       extension needed while the corpus is static) fixed it: generic precision recovered, hybrid
       went from 0.64 to 0.67–0.69 p@k, **tying** the dense channel (0.68) by trading strata.
    - *Graduated result:* with each channel working, **unweighted RRF was enough** — a weight tuned
      on a dozen queries would fit them, not the product. Hybrid's remaining value is per-stratum
      (named-standard, cross-standard), not global; it stays an option, dense stays default. The
      earlier "density proxies for breakdown" observation is real but archetype-specific: length
      normalization trades some of it away for generic precision, and that trade was net positive.
    - *What IDF does not fix:* **polysemy.** BM25 promoted an exact lexical match on "memory
      management" from a telemetry-services standard for a flight-software question — right term,
      wrong topic. Lexical precision has a floor that only semantics or routing raises.
    - *What hybrid does not fix:* a query that names the **document**. See the routing node below.
  - *Two failure modes worth keeping.* **OR, not AND, for the lexical query:** `plainto`/`websearch`
    tsquery ANDs terms by default, so a multi-term natural question matches ~nothing, the lexical
    channel returns zero rows, RRF degenerates to the pure vector ranking, and **hybrid silently behaves
    exactly like vector while appearing to run** — a smoke test would pass. (Also discard `and`/`or`/
    `not` appearing *inside* the user's phrase; there they are search words, not operators.) **And a
    second retrieval channel audits the first:** the lexical channel could not index 103 chunks whose
    citable text is a withdrawn-clause marker (`-`, `<<deleted>>`) — chunks the vector channel silently
    *can* return; adding the channel surfaced garbage the first held without complaint. *(An instance of
    "audit what appears to work," Cross-cutting.)* *And a sequel the day those chunks are actually
    filtered out: 103 was the count of the **wrong instrument**. It measured lexical blindness
    (`cite_tsv` empty); the criterion a filter enforces is "the verbatim **is** a withdrawn-clause
    marker," which catches **121**. The 18-chunk gap is not slop — those markers carry lexemes
    (`<<deleted, covered by 5.4.1.2>>`), so the lexical channel **can** retrieve them: the more
    dangerous ones the lexical-blindness count hid. The number you turn into a filter must come from
    the instrument that matches the filter's criterion, not a proxy for it (Cross-cutting: "diagnose
    before you repair").* *And a further sequel once the filter was built: even 121 was
    proxy-complete — the drop predicate missed 5 more markers (a space after `<<`, a `>>deleted>>`
    typo). The completeness guarantee is a separate **independent oracle** over the survivors, broader
    than the drop predicate — provable by construction (predicate off → it flags all 181 markers; on →
    zero survive) — never a count derived from the filter. Calibrating it is its own lesson: key on a
    **distinctive marker word** (deleted, moved, withdrawn), never on mere absence of content, or real
    terse clauses (`See 5.3.5.2`, `A-A-`) trip it.*

- **Identifier-as-content vs. identifier-as-container — route the second, don't search for it.**
  - *Mechanism:* a term in the query can name something the text **talks about** ("Stripe" in a
    budget) or the **document the text belongs to** ("what does ECSS-E-ST-40C require…"). The second
    kind is invisible to both channels: a document rarely names itself in its own clauses, so dense
    retrieval returned **0** clauses of the named standard, and hybrid returned **0** of it plus 3
    clauses *of other documents that cite it*. No fusion weight fixes that — the evidence isn't in the
    text. It's a **metadata filter**.
  - *Applicability condition (static):* the corpus is partitioned into units users refer to **by
    name** (standards, contracts, products, versions) and that name lives in metadata.
  - *Trigger signal (dynamic):* queries that name a unit get results that **mention** it rather than
    results **from** it. Check it directly: count, for named-unit queries, how many retrieved items
    belong to the unit vs. cite it.
  - *Maturity:* **routing ladder, cheapest rung first.** (0) The caller already knows → put the scope
    in the API contract. (1) **Deterministic detection** against the canonical identity table (all the
    forms the corpus uses to name a unit, normalized), returning a *list* (several named → search
    all). (2) An LLM classifier with a **closed enum** of units — only for what rules can't resolve.
    Rung 1 was the largest single gain of the retrieval phase: p@k 0.68 → 0.82 overall, named-unit
    queries 0.41 → 0.74, a follow-up that inherited the unit from history 0.25 → 1.00 — and **faster**
    (the filter scans fewer rows). Queries that name nothing must take the exact pre-routing path
    (pin it with a test); log what was detected and the post-filter cardinality.
  - *Filter vs. quota — measure the cost of the hard filter as its own stratum.* A hard filter cuts
    cross-references: a question naming one unit whose full answer needs another unit **it refers
    to** (measured: half the relevant items came from the referenced unit). In that stratum the filter
    costs precision (0.49 → 0.39) and its candidate recall collapses. A **quota** (most slots from the
    named unit, the rest open) recovers that stratum but loses more elsewhere (named-unit 0.74 → 0.46);
    it does not win on balance. The filter stays; the residual is a known, measured gap whose remedy
    lies in the *references*, not in loosening the filter.
  - *A related volume effect:* one unit held **35%** of all chunks and flooded the top-k even for a
    question naming another unit (6 of 9 retrieved clauses). Routing removes it for named queries;
    for unnamed ones, check the per-unit chunk share before blaming the embedding.

- **Reranker (cross-encoder) — two measurements, not one, decide it.**
  - *Applicability condition (static):* a latency budget that can absorb a model in the critical path
    **before the first token**. Read the reranker's cost as a fraction of **TTFT**, not in absolute ms
    (measured on CPU: +1.5 s for 50 candidates ≈ one extra TTFT; +7 s for 400 ≈ four).
  - *Trigger signal — both halves required:* (1) **margin in the pool:** relevant items sit deeper
    than the top-k (measured: candidate recall 0.46 at 50 chunks → 0.70 at 400, at no retrieval-latency
    cost in the exact regime); **and** (2) **the model converts that margin into precision** — measured,
    not assumed. Here (1) held and (2) didn't: a small English MS-MARCO cross-encoder tied the dense
    baseline (0.70–0.71 vs 0.69 p@k) at every pool depth, trading strata. *Margin is necessary, not
    sufficient.* Rejected for production; reopening means a stronger model, GPU, or a hosted reranker
    run through the **same harness** (swapping the model is a constant).
  - *Feed it the context the embedding got.* Fed only the clean cite-text, the reranker was **worse
    than the vector** (−0.12 to −0.17 p@k), worst on follow-ups; a one-line header (unit · section ·
    title) recovered it (≈ +0.15). This is the embed≠cite frontier with a **third consumer**: anything
    that scores relevance (embedding, reranker) should see the context-enriched side; only the
    generator's quotable source must stay clean.
  - *Silent truncation:* 8% of (query, chunk) pairs exceeded the model's 512-token window and were
    scored on a prefix, without error. Count pairs over the window with the model's **real** tokenizer.
  - *Its score is not a threshold.* The anchor "a pairwise score finally makes per-chunk filtering
    possible" did not survive measurement: AUC relevant-vs-irrelevant 0.50–0.63 globally and **below
    0.5 within most queries** against hard negatives. Before fixing any threshold, measure separation
    within and across queries.
  - *Patterns if it ever ships:* load once (lifecycle singleton; the healthcheck waits), same type in
    and out (the stage toggles by config), keep the score in the trace, run inference off the event
    loop, and keep its heavy dependencies out of the service image until it's decided — an optional
    dependency group the contract suite runs without.

### Measuring retrieval — build the golden set before the first technique

Every retrieval decision above was taken against the same stick; before it existed, a five-query
read-by-eye had produced a confident and wrong diagnosis ("tune the fusion"). Build the stick first;
each later technique becomes a one-line experiment.

- **Small, fixed, stratified, versioned.** A fixed number of real-shaped queries (10–15), stratified
  by the failure each type exposes (names-a-unit, generic, paraphrase, comparative, cross-unit,
  multi-turn, names-a-unit-that-refers-to-another). **Report the table by stratum; the mean hides
  exactly the trade-offs you are deciding** (hybrid tied dense on the mean while swapping strata; the
  quota won one stratum and lost two). With 1–2 queries per stratum the table is indicative: decide
  on large, consistent differences only. Grow it by **versioned extension** (old queries and judgments
  frozen, new ones added where the table had holes), never by silent edits.
- **The unit of relevance is the citable unit, binary, with a written criterion.** Judge clauses, not
  chunks (chunk ids die at the next re-chunk; clauses don't), dedupe by clause before computing
  precision, and write the criterion in one sentence before annotating. A product decision hides in
  it: "useful informative content counts; authority is metadata, not relevance" — then measure the
  authority slice (*normative recall*) from metadata instead of annotating it.
- **Retriever-independent annotation + an incremental mode.** Pooling (annotate only what your
  configurations return) can't see what all of them miss. Annotate relevants by searching the whole
  corpus, then judge — by the same protocol — whatever any later configuration brings into its top-k
  unjudged; a measurement with unjudged items in the top-k is **invalid** until they're judged (else
  the configuration that finds something new is penalized). The annotator misses things too: every
  incremental round found relevants the first pass hadn't (7–23%).
- **An LLM annotator is viable — with three conditions.** (1) **Isolated from the system it judges:**
  opened inside the repo, a coding agent loads the project instructions, which describe the retriever,
  its defaults and its known failures — independence is gone before it starts; run it in a workspace
  holding only the protocol, the document index and the clause export. (2) **Repeatable by protocol,
  not by model version:** a versioned protocol file (criterion, strata, output schema) + model and
  date recorded per judgment. (3) **Verbatim evidence checked by code:** every positive judgment
  carries a literal quote validated against the clause; a non-expert reviewer then only does reading
  work (does the quote exist and answer?) — the code does the existence check. Run on a subscription
  rather than an API, it cost nothing marginal.
- **Two recalls, capped, plus depth.** Precision@k (k = citable units actually delivered) is the robust
  column. Recall needs a **cap** (hits ÷ min(|relevant|, slots)) or it mixes inability with a
  structural ceiling; measure it at the delivered k **and** over the candidate pool, and sweep pool
  depth (50→400) to tell "deeper" from "absent" — that sweep is what decides a reranker.
- **Measure what reaches the model** — through the production context provider (history composition,
  grouping, dedupe), not a raw search endpoint — and refuse to run against a corpus whose fingerprint
  differs from the golden set's. Mark run-to-run instability: query embeddings are not bit-for-bit
  deterministic.
- **Cheap mechanism checks before costly techniques.** Four minute-scale checks — what the channel
  actually returns for a named-unit query, how the query is tokenized on its way to the database,
  whether the ranking has IDF, how chunks are distributed per document — rewrote the next brief
  entirely. Looking at the mechanism is cheaper than measuring a solution to the wrong problem.

### Wiring retrieval into generation

The step where the retriever replaces the static context. It looks like one line (`get_context` now
calls the retriever) and is in fact five decisions: where the call enters, what the unit of context
is, how the model cites, how a conversation reaches the retriever, and what you can verify about the
answer. Most of what was learned here was learned by **correcting** first choices against a UAT, so
the failure modes are named.

#### Patterns & practices

- **Inject a context provider into the preparation point — don't retrieve in the router.** The
  generation service receives a provider (a callable: query → evidence) and calls it **where the
  static context used to be read**, after input guardrails and session resolution. Retrieving in the
  router "to keep the LLM service ignorant of the database" looks cleaner and breaks the invariant
  *guardrails before any effect* (Phase 4): embedding a query and reading the database are effects, so
  a toxic question would trigger retrieval. The provider gives you both properties: the LLM service
  imports nothing from the data layer (enforce it with an import test **in a subprocess** — in the
  suite's process everything is already imported), and the order stays intact. The provider opens its
  own **short** database session per call rather than borrowing a request-scoped one whose lifetime
  against a streaming response is fragile. Return a small typed result (evidence + mode + per-stage
  latencies), not a bare list, so one retrieval log event can be emitted by whoever knows the grouping.
- **No context → fail closed, and verify the seam against the real stack.** A RAG answer without
  retrieval is an ungrounded answer, so an unreachable embedding service *or database* is a 503 /
  typed stream error, never a fallback to "answer anyway" (contrast moderation, which can fail open).
  The mocked test raises the exception you expected; the real driver raised another (`socket.gaierror`
  is an `OSError`, not a `SQLAlchemyError`) — uncaught, it is a 500 and a broken stream. Stop the
  database once and watch.
- **The unit of context is the citable unit, not the chunk.** Group retrieved chunks by their
  citation coordinate (document + clause) and order groups by their best rank. Five chunks of one
  clause take **one** slot, which removes the "clone results" failure without MMR and yields the
  citation granularity the product promises. Keep each chunk's authority grade inside the group — a
  clause can mix normative and informative items, and grouping must not erase that.
- **Recomposition: measure before you expand — and with its own metrics.** Grouping only what was
  retrieved leaves most groups partial (measured: 70% of retrieved clauses incomplete). Expanding
  every hit to its whole parent unit multiplies the context (median ×4, tail ×5 — long annexes).
  Expansion is an **augmentation** stage (after the last selection cut, before the context block): it
  changes *how much* of each delivered unit arrives, never *which* units — so a relevance golden set
  can't see it. Measure completeness (overall and normative), orphan rate and context tokens.
  - *Orphans first, free:* add a unit's normative parts only when it was delivered with **none** of
    them (a note without the requirement it qualifies). Measured: 6 orphans → 0 with 7 chunks across
    14 requests, context tokens unchanged → default.
  - *Bounded completion* (normative first, per-unit and per-request token caps): normative
    completeness 0.59 → 0.95 for **+36% context**; the caps bite on long annexes, as expected. Whether
    that extra context helps or dilutes the answer is a generation-quality question, not a retrieval
    one — keep it as an option until generation is evaluated.
  - *The trace marks what expansion added* (retrieved vs. added): otherwise an audit can't tell what
    the search found from what was dragged along. Added chunks carry no distance — they were never
    ranked against the query.
  - *Completing a unit ≠ following its references.* 4 of 6 orphan notes said "see clause X" — about a
    **different** clause. Expansion brings the note's own requirement; what the reader wanted was the
    referenced clause. That is reference traversal, a separate mechanism with its own validation.
- **Context block: tagged sources with opaque per-request handles.** Each group is a `<source id="n">`
  with its citation coordinates as attributes and its items' verbatim cite-text inside. The model cites
  `[n]`; the server maps `n` back to the coordinate. **Opaque handles over natural IDs when the model
  knows the corpus from pre-training:** with natural IDs it can cite a real clause it was never given;
  with handles, citing what wasn't retrieved is impossible by construction. Escape the **attributes**,
  not the text — the text is what the model must be able to quote verbatim, and an escaped entity
  would end up inside the quote and fail validation for a reason that isn't the model's. Don't expose
  distance or ranks: the model can't use an uncalibrated number and order already carries rank.
- **Citations = what the model marks, not what was injected.** Extract `[n]` from the final text
  (`[1, 3]`, `[1][3]`), keep valid ones, dedupe in order of appearance. Out-of-range handles produce no
  citation and are logged. This is the **fan-out** fix (measured: 1 of 7 injected clauses cited), and
  it changes what "citations" and "evidence" mean: they now differ, so the UI has to show *cited* and
  *retrieved-but-not-cited* separately. *It also retires the "cut citations on refusal" bridge*
  (Phase 5): phantom citations came from attaching injected material; with marker-derived citations
  they can't occur, and the old rule only produced `[3]` in the text with no source underneath.
- **Verbatim quotes give a deterministic support check — on a normative corpus.** Ask the model to
  quote the **minimal** literal span of a requirement next to its marker (`"…" [n]`); paraphrase
  inside a norm alters it anyway. Validate by substring against the cited group after normalizing
  **both** sides identically (Unicode NFKC, collapsed whitespace, re-joined line-break hyphens,
  unified quotes, stripped trailing punctuation). Run it in **observation mode** first — log, don't
  reject — and decide the failure policy with the measured rate. Measured: zero false `not_found`
  across the UAT, so normalization was sufficient.
- **In conversational RAG, the history must reach the retriever, not only the generator.** History
  designed for a CAG generator (resolve "it" against the prior turn) is correct there and broken in
  RAG: the question is *also* the retriever's query, and "And what does Q-ST-80C say about it?"
  retrieves nothing. First tool, no LLM: **the retrieval query = current question + the last N user
  questions**, never assistant answers (long, they drag the vector to a centroid), N≈2 so a chained
  ellipsis that was itself anaphoric still carries the topic. The provider receives a query string and
  stays unaware a conversation exists; moderation still sees only the current question.
- **The stored history is prompt — anything in it is an example the model will copy.** Rewriting
  `[n]` to natural citations in stored turns (so an old `[1]` can't point at a new source) taught the
  model a second citation format; it used it instead of `[n]` (1 in 6) and the answer lost all
  traceability. Store assistant turns **with citation markup stripped**: no stale handle, no format to
  imitate (6/6 valid markers after). What's lost — which clause backed which past claim — is
  re-supplied by the next turn's own retrieval.
- **Claims of absence may only be about what the system checked.** The model sees k chunks, never the
  corpus. "Not covered by the indexed standards" is false whenever retrieval missed — and it was
  measured saying a standard "is not indexed" when it was. Phrase it about **the search** ("I could not
  find this in…"), which is true by construction. Expect the prohibition to be obeyed in the letter
  only (the model moved to "the indexed standards do not contain…"); the structural fix is a typed
  output contract, not more prompt.
- **Reading rules are the CAG inside the RAG.** What belongs fixed in the system prompt of a RAG over a
  normative corpus is not content but the corpus's *grammar*: obligation keywords ("shall" requirement,
  "should" recommendation, "may" permission), informative items are not binding, an enumerated list is
  bounded by the sources found. And write every user-visible phrase in **product voice** — rules that
  talk about "the provided material" get repeated to the user verbatim.
- **Log what was retrieved, not how much.** One event per request with each hit's id, rank and
  distance, the groups after clustering, context-block size, embedding and DB latency separately, and
  the history depth used; one validation event with valid/invalid markers, cited vs injected groups
  and verbatim outcomes. Counts tell you something failed; ids tell you which stage.

#### Decision points

- **Precision vs. recall bias — decided by the asymmetry of errors, possibly per query type.**
  - *Applicability condition:* a compliance-style corpus where **omitting an applicable item** (an
    enumerative question answered with an incomplete list) is worse than citing an irrelevant one.
    There: bias retrieval to **recall** (generous k, no aggressive per-chunk filtering) and recover
    precision in generation (grounding, citation, validation). *Retrieval ensures coverage; generation
    ensures correctness.* A domain where a wrong number is worse than "I don't know" (estimation)
    inverts it. Per-query-type tuning needs query routing (a later phase); start with one setting.
  - *No per-chunk threshold without a calibrated score.* A cosine cutoff applied to fused (RRF) results
    would drop exactly the lexical identifier hits; and distance doesn't separate "absent" from
    "generic" (above). A pairwise reranker score was the next candidate and **also failed** to separate
    (see the reranker node: AUC ≤ 0.63). The abstention decision stays in the generator.

- **Conversational retrieval — concatenation vs. LLM condensation.**
  - *Applicability condition:* a conversational product. Then the history has to reach retrieval
    (above); the fork is *how*.
  - *Maturity:* **concatenate prior user questions** first — zero calls, zero new prompt, no new
    failure mode. Its cost is topic contamination when the user switches subject mid-conversation,
    cheap under a recall bias.
  - *Trigger signal:* contamination **observed in real use** (retrieved sources from the previous
    topic after a self-contained new question) → condense the follow-up into a standalone question with
    a small model. The rewrite only feeds retrieval, never enters the prompt as source, so it doesn't
    cross the citable frontier. *Measured:* none observed with users who chain questions on one topic.

- **Citation validation — three levels, only two of them deterministic.**
  - **Membership** (`[n]` exists in this request) and **literalness** (the quote is in the cited
    source) are cheap and deterministic. **Support** (the source says what the sentence claims) is not:
    measured on a table of near-identical rows ("category: X. Definition: …"), **1–2 answers in 6**
    attributed a row to the wrong category with a quote that was **literal and correctly referenced**.
    Passing the first two levels does not imply the third. The policy on unsupported citations needs a
    faithfulness evaluation, not a substring.

- **Securing the data layer of a RAG — authorization in retrieval, not keys on endpoints.**
  - *Applicability condition:* a corpus with **per-user permissions or multiple tenants**. Then the
    filter goes **inside** the retrieval query — post-filtering the answer is too late, the model
    already read the text. API keys protect the endpoint, not the data.
  - *Maturity:* rate limiting earns its place with **public exposure or multiple consumers**; before
    that, a spend cap at the provider covers the cost risk with zero code (and also catches your own
    runaway loops). Authentication follows the consumer: service-to-service → API key
    (constant-time compare), end user → session/JWT. *(Decided here, not built: public corpus, no
    public URL.)*

---

## Phase 4 — Production-hardening the service (provider robustness, caching, streaming, observability)

Once the vertical slice answers correctly, these are the layers that turn a working endpoint into
a service that survives contact with real use. None of them changes *what* the answer is — they
change availability, cost, perceived latency, and your ability to see what happened. They all sit
on the provider seam (Phase 1) and the output contract (Phase 2).

> **Because none of them changes the answer, every one of them is also *removable*.** That cuts both
> ways: it's what lets you add them cheaply, and it's what obliges you to re-check, at each phase
> change, that each one still earns its keep. The response-caching node below is a worked example of
> the second half — adopted on a valid trigger, retired when the trigger stopped holding. Treat a
> *Trigger signal* as having an exit condition, not just an entry one.

### Patterns & practices

- **A fallback must be observable, not silent.** If a backup provider answers when the primary is
  down, the response object the client receives should reflect *who actually answered*
  (`model_used`/`provider` = the responder, not the configured primary), and the operational log
  should carry a `fallback_used` flag. A silent fallback is nearly as dangerous as none: you learn
  the primary was failing only when something else breaks.
- **Design wrapper return fields for downstream consumers before they exist.** A normalized result
  object (text, model_used, provider, token counts, finish_reason) with a pass-through channel for
  provider-specific params costs nothing to include up front and avoids re-touching the wrapper in
  every later layer (cache needs the token counts, streaming needs finish_reason, observability
  needs model_used). Closing these doors early is expensive to reopen.
- **Degradable dependencies vs. fail-fast.** Correctness-critical secrets fail-fast (Phase 1). An
  *optimization* dependency (a cache, an optional backup provider) should **degrade with an
  explicit warning**, not crash the service: catch the connection error, log it, serve without the
  optimization. Never `except: pass` — a degraded path is logged, not silent. Reflect this in
  orchestration too (e.g. don't gate container startup on a `service_healthy` cache — that
  contradicts "degradable").
  - *The degradability test (which policy does a dependency get?):* a dependency is degradable **iff
    its absence changes the *cost* of the answer, not the *answer*.** Cache down → same answer,
    slower → fail-open. Session/conversation state down → a follow-up gets answered without its prior
    context → the *answer* changes → **fail-loud**. Moderation down → doesn't change a legitimate
    answer, only whether a toxic one is blocked → fail-open **as a consciously documented exception**
    (accepted security debt), not because the test says so. "Correctness degraded in silence" is the
    cardinal sin (same family as the phantom citation); the fail policy is *derived* from whether
    absence changes the answer, not chosen by taste.
  - *The fail policy is a property of the **caller**, not of the datastore.* A store going down is
    just an event; whether you degrade or abort is decided by the accessing code. One physical Redis
    can serve a degradable tenant (cache) and a non-degradable one (session state) with opposite
    policies. The hazard: the non-degradable tenant silently **inheriting the fail-open wrapper** of
    the degradable one. Rule: never share the access wrapper between a degradable and a
    non-degradable tenant, even on the same physical store.
- **Shared-port stores are a cross-project contamination risk.** Any datastore on a default shared
  port (Redis 6379, Postgres 5432, …) can collide with *another project's* instance on the same
  machine. Two guards: (1) tests isolate from the real resource **by default** (opt-in to a real/
  fake client, never opt-out — an autouse fixture that nulls the client), and (2) the project's
  own service doesn't publish the port to the host if only another container consumes it. This is
  learned the hard way: a test suite writing keys into a neighbor project's live store.
- **Stream what improves perceived latency; don't stream what's secondary and cheap to paint at
  once.** Stream the long generated text token-by-token; deliver structured metadata (citations,
  usage, finish_reason) as a single typed terminal event, or in the non-stream response. Nobody
  audits a citation in real time while it's being written. This keeps the stream simple and keeps
  the streaming path *deletable*.
- **The streaming path sits on top of the non-stream path, which is the source of truth.** Both
  share one decision-and-assembly path (cache lookup, grounding, citation building); streaming
  only changes *how* the result is delivered. Consequence: never put business logic that exists
  *only* in the streaming path. If streaming misbehaves, it deletes cleanly — that's the design
  criterion, because streaming is UX, not correctness.
- **Any layer that can answer without calling the model decides whether there's a stream at all.**
  A response cache above the provider layer is the canonical case: a hit never calls the model, so
  there is nothing to stream. Look it up first; on hit, return whole; only on miss open the stream.
  They don't interact beyond this — that layer decides *if* the request even enters the streaming
  path. *(Generalize past the cache: a refusal short-circuit, a stored answer, or a guardrail
  rejection has the same shape. What matters is the ordering — "can this be answered without
  generation?" is resolved **before** the transport decision, never inside the stream.)*
- **Truncation is an integrity failure, not a cosmetic notice — when traceability is mandatory.**
  `finish_reason == "length"` may have cut the answer before its citation, yielding a normative
  claim with no anchor. Detect it and mark the response unreliable; do **not** try to prevent it by
  instructing the model to be brief (that competes with completeness — it may drop applicable
  requirements to fit a word budget). Generous `max_tokens` + detection, not prevention by prompt.
  *And detection is only as honest as the field's **default**.* A wrapper that initializes
  `finish_reason = "stop"` asserts "the model finished normally" before anyone checked; when the
  provider closes a stream with **zero deltas**, the loop never overwrites it and an empty answer
  ships as a success — with its evidence attached behind a text that doesn't exist. A state field
  defaults to **"unknown"**, never to the good case, and a measurement nobody took is **`None`**,
  never a number (`distance = 0.0` reads as "identical to the query"). The empty answer itself is
  then *named*, not delivered: an error event on the stream, a 502 on the request (well-formed
  request, unusable upstream reply), and no assistant turn written to the session (an empty turn
  would be re-injected into the next question). *Residual:* a boolean derived from that field
  (`truncated = finish_reason == "length"`) inherits the problem — under "unknown" it says "not
  truncated" when the truth is "don't know"; it wants to be three-state.
- **Structured logging is unconditional; the decision is *what fields*, not *whether*.** Use a
  structured logger (e.g. structlog) with dual rendering (human console in dev, JSON in prod) and a
  per-request `request_id` bound via middleware so one query's full trace (HTTP → cache → LLM →
  response) is reconstructable by filtering on that id. Minimum LLM-call fields: model_used,
  provider, tokens in/out, cost, latency, cache_hit, fallback_used, finish_reason. For streaming,
  time-to-first-token is the metric that justifies streaming (it measures the UX streaming
  promises), separate from total latency.
- **Log metadata, not content, by default.** Structured logs capture prompts and can capture
  responses. Default to metadata (tokens, model, latency); the full prompt/response text is a
  data-exposure surface. Harmless for a public corpus, but a compliance concern (GDPR) for
  sensitive corpora — gate content logging behind an explicit DEBUG choice.
- **A cache hit is *cost avoided*, not cost zero.** Persist the original call's token usage in the
  cached value so the hit can be logged with the cost it *would* have incurred. "Free" hides the
  savings you're actually achieving.

### Decision points

- **Provider abstraction: aggregation library vs. thin hand-rolled adapters.** *(marquee
  decision)*
  - *Maturity:* MVP for the fallback capability; the library-vs-adapter choice itself is a
    build-vs-buy call independent of phase.
  - *Applicability condition:* a library (e.g. LiteLLM) earns its weight with **many providers**
    and when you want retry/backoff, cost tracking, and unified streaming "for free"; **thin own
    adapters** win with **two providers** and when learning the normalization is itself a goal.
    Either way it goes *behind the Phase 1 seam*, not scattered.
  - *Structural tension to record:* **any unified abstraction can flatten provider-specific
    features.** The library's "one interface for all providers" can obscure a capability that only
    one provider has (e.g. Anthropic prompt-cache breakpoints). For plain text streaming the
    normalization is benign; when the stream or request must carry more than text, verify the
    library exposes it rather than flattening it. Don't assume — verify at the point it matters.
  - *Note — testing a Router's fallback without network:* a "mock fallbacks" flag may still fire a
    real network call to the backup on its own; combine it with per-deployment mock responses to
    exercise the real routing config offline.
  - *Note — resolve provider natively, not by string matching.* After normalization the response's
    model id may lose its provider prefix; derive the provider with the library's own resolver, not
    a `"claude"`/`"gpt"` substring heuristic that breaks on the third provider.

- **Automatic fallback across providers.**
  - *Maturity:* normally production/SLA — but rises to MVP when the artifact is a **high-stakes
    demo**, where a provider outage mid-demo is catastrophic to the *goal* even at low probability.
    Decide on **severity, not frequency**.
  - *Applicability condition:* ≥2 providers you actually wire **and validate** — a fallback to an
    unvalidated provider isn't a fallback. Wiring the Router with a fallback configured ≠ the
    fallback being validated end-to-end; validate it (a real call that forces the rotation) before
    claiming it.
  - *Trigger signal:* observed provider outages, or needing to switch on price/capability.

- **Provider-agnostic system prompt.**
  - *Applicability condition:* the moment a request can be served by more than one provider (i.e.
    once fallback exists), the system prompt must not depend on provider-specific phrasing. Make it
    a design requirement with a regression test that validates correct behavior (e.g. citation
    compliance) across *both* wired providers.
  - *Honest limit:* "agnostic" is **verified against the providers you test**, not guaranteed
    universally — an aggregation layer uniforms the transport, but each model reacts differently to
    the same prompt. Re-validate when a new provider enters.

- **Response caching — exact-match.** *(Closed node — built in the CAG phase of the ECSS project,
  then **removed entirely** at the RAG transition. Kept as recorded knowledge, not as a live
  recommendation. Read the retirement lesson first: it changes when you'd take this branch at all.)*
  - *Retirement lesson (the durable one):* an exact-match response cache is **MVP-cheap to build but
    not cheap to carry**. Its applicability condition (observed hit rate above ~20%) is not a
    one-time gate — it must be **re-evaluated at every phase change**, because the traffic that
    justified it can disappear without the cache disappearing with it. In a **single-user product**
    the only places with a real hit rate are the demo and the evaluation suite, not the traffic; and
    the evaluation suite is precisely where the cache must be *disabled* (see the eval-mode bullet
    below). Building it in the CAG phase was still the right call — **for what it forced, not for the
    hits**: it forced separating static instructions from injected context, it forced the
    guardrail-before-any-effect ordering, it forced the history-as-gate insight, and it forced the
    eval-mode seam. All four outlive it. At the transition to dynamic retrieval its effort/benefit
    ratio inverted (see the key-design caveat below) and it was deleted, not parked. *If a future
    deployment with real users shows a hit rate that justifies it, rebuild it from this node — that
    is what this node is for.*
  - *Maturity:* MVP-cheap once inputs repeat.
  - *Applicability condition:* a **deterministic** task where repetition is desirable (not creative
    generation). For a **normative/traceability corpus this is stronger than cost savings**: an
    identical query returning an identical, identically-cited answer is an *auditability* property,
    not just a latency win. **Second clause, learned the hard way:** enough *distinct users asking
    overlapping questions* for repetition to exist at all. The auditability argument justifies the
    *design* of the key; it does not, by itself, justify carrying the infrastructure.
  - *Trigger signal:* observed repeated inputs (hit rate above ~20% justifies the infra). **Re-check
    this signal at each phase change, not only at adoption** — a decision point with a dynamic
    trigger has a dynamic *exit* too, and nothing else in the system will tell you the trigger
    stopped holding.
  - *Key design (the core decision):* build the key from **explicit slots** —
    `question + system_prompt + context + corpus_version` — and keep `context` and `corpus_version`
    as **separate slots even when static today**. This is why you'd hand-roll the cache instead of
    using a library's built-in: when retrieval later injects dynamic context, populating the
    `context` slot is a fill, not a redesign. Caching on `question + system_prompt` alone, once
    context is dynamic, would serve a stale answer built on chunks retrieval would no longer
    return — a **phantom citation served from cache**, the exact failure a traceability system
    exists to prevent.
  - *Correction — the above generalized past its evidence.* "Populating the `context` slot is a fill,
    not a redesign" was verified true **for getting to the end of the CAG phase**, and was written as
    if it held into the RAG phase. It does not. Once context comes from retrieval, the `context` slot
    is **redundant or harmful**: redundant because retrieval is a deterministic function of the
    question and corpus version (so the slot adds no discrimination the other slots lack), and
    harmful because you must now **run retrieval before you can build the key** — the flow inverts
    from `lookup → retrieve` to `retrieve → lookup`, which throws away the cache's main benefit
    (skipping the expensive work) and keeps only the generation savings. The cheap fill turned out to
    be a re-architecture of the request flow. *This is a worked example of the failure this guide
    exists to police: knowledge converged from one phase's evidence, stated at a confidence that
    covered phases not yet reached. When a node says "this will be a cheap fill later," treat the
    "later" as untested until the later phase actually tests it.*
  - *What is *not* a good key slot:* the **configured** model. It gates on `settings.LLM_MODEL`
    (rarely changed) yet doesn't protect against the real risk — a *fallback*-served response
    getting cached and later served once the primary recovers. That would need the *responder's*
    model (`model_used`) in the key/value, which is heavier; the configured-model slot buys little.
    Accepted limit: after a model change, old entries serve until TTL expiry (mitigate by lowering
    TTL or flushing the namespace, not by re-adding the slot).
  - *A presentation-only flag is not a key slot either.* A request flag that only controls what the
    server *echoes back* (e.g. "include the injected context in the response for diagnostics") must
    **not** enter the key — two requests differing only in it must still hit. Attach the diagnostic
    payload *fresh* when serving, from the value recomputed for this request. *Fidelity caveat that
    surfaces at the RAG phase:* recomputing the echoed context when serving a **hit** is faithful
    only while that recompute is **deterministic** (a static slice). Once context comes from
    non-deterministic retrieval, or a corpus that changed between the cached generation and the hit,
    recomputing would echo something the cached answer was *not* built on — then fidelity requires
    storing the context **inside** the cached entry, not recomputing it.
  - *Order against guardrails (finer than "guardrails before the cache"):* run **input guardrails
    before the lookup** (a hit must not bypass moderation — an attacker who knows a payload is cached
    could trigger the hit to skip it), and run **output guardrails before the `set`** (only validated
    answers enter the cache, so a hit is safe *by construction* and never re-pays the expensive
    check). Decide whether a *filtered/degraded* answer is cacheable: fine if deterministic, a trap
    if the degradation was a transient failure you'd be pinning for the whole TTL.
  - *The input-guardrail-before-lookup order is an **invariant, not an optimization** — and must not
    depend on the cache algorithm.* The defense isn't "the key-collision attack is exploitable today"
    (under exact-match it's practically impossible: a different input yields a different key → miss →
    it *is* moderated). The defense is that **a security invariant must not rest on an accidental
    property of another subsystem** (that the key is the exact text). Add key normalization
    (lowercase/trim) or **semantic caching** and "a different input that hits the same entry" goes
    from impossible to being the cache's *design* — the bypass then opens itself, silently, via an
    unrelated change three phases later. Putting the guard before the lookup costs nothing now and
    closes it by construction. Pin it with a regression test (a would-be-hit input that moderation
    rejects is still rejected), not a comment — the same "works by design, not by today's
    coincidence" discipline as the test-cementing-a-bug lesson above. Structure the input guards as a
    **small ordered pipeline** even with one tenant (toxicity today), so the next layer (injection)
    slots in without reworking the flow; each check declares its failure policy (moderation →
    *exception*). *Sharpened once multi-turn adds state:* the guardrail runs before **any effect** —
    not just before the cache but before reading conversation state, resolving a session, or any
    work at all. A malformed/malicious `session_id` must not trigger work before the *question* is
    moderated. What was "guardrails before the cache" becomes "guardrails before the first side
    effect," and the regression test moves with it (moderation is asserted to run before the store
    is touched, not merely once). **This bullet outlives the cache entirely** — once the cache is
    removed the invariant is unchanged ("before the first side effect" never mentioned the cache),
    and its regression test keeps passing without edits. That is the tell that it was a real
    invariant and not a property of the cache: *the cache was the occasion that surfaced it, not its
    subject.*
  - *Conversation history is a cache **eligibility gate**, not a key slot.* History disambiguates
    the *question*; it does not change the *answer* the corpus holds — so it does not belong in the
    key. Gate on **presence of injected history**, not presence of a session: `cache_eligible =
    (len(history) == 0)`. Turn 1 of a session (empty history) is eligible and **shares the cache
    entry** with the same question asked single-turn (same key → preserves the FAQ value where
    caching actually pays). Turns 2+ (non-empty, per-conversation-unique prefix) **bypass get *and*
    set** — caching a unique prefix never hits and only litters the store. Corollary that must not be
    lost: on a turn-1 **cache hit inside a session, still append the turn** (using the cached answer
    text as the assistant turn), or turn 2 loses the context of turn 1. *(Expiry: once query
    condensation exists — rewriting a follow-up into a standalone question against history — the
    condensed follow-up regains a cacheable identity and re-enters the cache keyed on the condensed
    form. "Cache only the first question" is correct-for-now and temporary.)*
  - *An evaluation mode must be able to **disable** the cache — and the cache is the evaluation
    instrument's adversary.* A response cache exists to *suppress* the variance that quality
    evaluation exists to *measure*; with the cache live, a consistency test gets hits and reports
    variance ≈ 0, measuring the cache, not the model (a silent false positive). So disabling the
    cache is *instrument correctness*, not convenience — and it composes into the same eligibility
    predicate as the history gate (`cache_eligible = len(history) == 0 and not eval_mode`), one
    decision site, not parallel branches. Disable **by environment/mode, not by a request flag**: a
    request flag that turns off the cache is attack surface (force bypass, spike cost) and
    contradicts "the system decides eligibility, not the client." The bypass reuses the *same*
    "serve without cache" seam the degradable path already built (Phase 4 fault-tolerance) — a good
    seam pays twice; you activate it, you don't invent it. *Survives the cache in weakened form:* the
    **mode** (an environment-level switch that puts the system in a state fit to be measured, with a
    fail-fast guard against enabling it in production) is the durable part and stays; "what the mode
    disables" is whatever suppresses variance at the time. With the cache gone the mode has nothing
    to disable **yet** — keep it, because provider-side prompt caching, memoized retrieval, or a
    reranker cache each re-create the same adversary later.
  - *What did **not** survive, and is the honest cost of the retirement:* the exact-match cache was
    also the only mechanism making an identical repeated query return a byte-identical, identically
    cited answer. Removing it means **auditability of repeated queries now rests on model determinism
    alone** (i.e. not at all). If that property is ever required contractually, it comes back as a
    requirement — and it is an argument for *storing answers*, not for caching them to save cost.
    Naming what a deletion costs is part of the deletion.

- **Cache invalidation strategy.** *(Closed with the node above; kept for the corpus-versioning
  insight, which is independent of caching.)*
  - *Applicability condition:* for a **versioned corpus**, the correct answer changes when the
    *corpus* changes (a new standard revision), not on a time schedule. Prefer **invalidation by
    corpus version** (a `corpus_version` slot in the key) over pure TTL; TTL is the safety net, not
    the primary mechanism. Event-based invalidation, which generic material treats as the exception,
    is here the main mechanism. *The durable half:* a normative corpus needs an explicit **corpus
    version identifier** regardless of whether anything caches — it is what lets any derived artifact
    (index, embeddings, stored answers) be tied to the corpus revision it was built from.

- **Semantic cache — match on *meaning* instead of string equality.** *(a real win for the right
  corpus; recorded as a full fork, since whether it ships is corpus-dependent. **Not taken here** —
  contraindicated on clause 2 below, and the exact-match retirement above raises the bar further:
  the traffic that would feed any cache didn't exist.)*
  - *Maturity:* **scaling**, not MVP — it needs an embedding model and a vector store, so it lands
    naturally in/after the embeddings phase. Standing it up earlier pulls that dependency forward.
  - *Applicability condition (two clauses, both required):* (1) users **reformulate the same intent**
    often (N phrasings of one question — the case exact-match misses entirely, since natural-language
    keys are never byte-identical), **and** (2) the domain **tolerates small lexical differences** —
    near-synonym phrasings must map to the same correct answer.
  - *Where it is contraindicated (fails clause 2):* a **normative / legal / medical** corpus where a
    one-token difference changes the correct answer. "criticality B" vs "criticality C" are
    near-identical in embedding space but point to **different clauses** — a semantic hit serves the
    wrong norm with a confident face, and because a hit skips generation, it also skips every
    guardrail on the generation path. The failure is **structural** (the axis the cache groups on —
    surface similarity — is orthogonal to the axis that matters — clause identity), so it is not a
    threshold-calibration problem: raising the threshold until it's safe collapses it back to
    exact-match. *Reopen path:* if the discriminating anchors (standard, criticality) move into
    **structured parameters** (the deterministic bucket below) and only the free-text intent is
    embedded, the risk drops — but that presupposes those knobs exist and retrieval to make them
    mean something.
  - *Key design — composite key:* a **deterministic bucket** (structured params + prompt version)
    that partitions the space, and a **vector part** (the embedding of the free text) searched only
    *within* a bucket. Same principle as the exact-match slots: homogeneous buckets let the threshold
    be more aggressive safely, and putting `prompt_version` in the bucket means promoting the prompt
    orphans old buckets automatically (TTL sweeps them — no manual invalidation).
  - *Threshold is a product decision, not a constant.* Cosine similarity returns a score, not a
    hit/miss; you pick the cut. Aggressive (~0.85) maximizes hits and risks false positives;
    conservative (~0.95) eliminates them and loses hits; legal/medical/financial lean 0.95+. Roll it
    out **log-only first** (compute the lookup, don't use it, log `(input, top-1 neighbor, score)`,
    calibrate on real data), then enable. *(Same log-first discipline as guardrails, Phase 5.)*
  - *Cost is a real ledger:* it adds embedding latency+cost on **every** request (hit or miss) to
    save LLM latency+cost on hits — favorable only above a hit-rate floor (repetitive-query products
    reach 40–60%; per-request-unique ones ~5% and it adds net latency). If unknown, log-only for a
    week and measure.
  - *Implementation — a dependency-appetite choice, not a quality one:* a managed layer (Redis
    `LangCache`), a library abstraction (`redisvl`'s `SemanticCache`, or `langchain`'s
    `RedisSemanticCache`), or hand-rolled over any vector store you already run (pgvector, Qdrant,
    Pinecone). Pick by which dependencies you want to own. *(A blanket "no frameworks" stance is a
    learning choice for a from-scratch project, not a Guide-level verdict — where a framework's
    semantic-cache abstraction genuinely saves work and its coupling is acceptable, it is a valid
    pick.)*

- **Prompt caching (provider-native, e.g. Anthropic `cache_control`).** *(distinct from response
  caching above — it doesn't skip the call, it discounts the repeated-prefix processing)*
  - *Maturity:* deferred until the static prefix is large enough to matter.
  - *Applicability condition / trigger:* the cacheable prefix exceeds the provider minimum (e.g.
    ~1024 tokens). A small CAG prefix may not reach it, so it wouldn't even activate — not a lost
    feature, just not-yet-applicable. Re-evaluate when retrieval inflates the prefix; **that** is
    when to verify the aggregation library passes `cache_control` through rather than flattening it.

- **Streaming transport — SSE vs. StreamingResponse vs. WebSockets.**
  - *Applicability condition:* unidirectional server→client token streaming → **SSE** (structured
    events, auto-reconnect; the de-facto LLM standard). Raw text with no event structure →
    StreamingResponse. Bidirectional (agent asking the user mid-task) → WebSockets — defer until an
    agent phase actually needs it.
  - *Mechanism caveat (framework-specific, learned in implementation):* with FastAPI's `fastapi.sse`,
    SSE mode is a property of the **route** (declared `response_class` + a generator endpoint), not
    decidable **per request**. You cannot return plain JSON on one request and SSE on another from
    the *same* endpoint via this mechanism. Either use two routes, or format SSE by hand with
    `StreamingResponse` — there's no middle path. (Practical consequence: a cache-hit on the stream
    endpoint can just emit a single terminal event rather than switching to JSON.)
  - *Note — two streaming legs, don't conflate:* an aggregation library normalizes the *provider→
    backend* leg (one `async for` over normalized chunks); emitting `text/event-stream` on the
    *backend→client* leg is your endpoint's job. "The library supports SSE" only covers the first
    leg. (A separate `include_usage`-style stream option is typically required for the library to
    include token counts in the stream at all.)

- **Observability tooling — structured logs vs. a visual platform.**
  - *Maturity:* structured logs are MVP and unconditional (above). A **visual platform** (Logfire,
    Langfuse, …) is a **scaling** node.
  - *Trigger signal:* log volume makes terminal debugging untenable, or you need cost dashboards for
    stakeholders. Below that, structured logs to console/JSON suffice.
  - *Applicability note:* pick by stack fit — a Pydantic/FastAPI stack pairs naturally with an
    OpenTelemetry-based tool; open-source/self-hosted needs favor others. Tools whose value is tied
    to a framework you don't use (e.g. LangChain-coupled tracers) are out by construction.
  - *Don't conflate two meanings of "traceability":* **operational** trace (which model, tokens,
    cost, latency, fallback, cache_hit — this phase) is not **citation** trace (which clause grounded
    which claim — a generation-quality phase). Same word, different layers; don't let a logging task
    absorb the citation-verification concern.

- **Multi-turn conversation state — history vs. memory, and who owns it.**
  - *Applicability condition (which one you even need):* distinguish **history** (the raw message
    array, for anaphora/ellipsis resolution — *"and for criticality B?"* only parses against the
    prior turn) from **memory** (distilled domain facts that must survive history truncation, e.g.
    *"the project is called X"*). A **co-authoring** system that negotiates a shared state turn by
    turn needs memory; a **retrieval system whose answers already exist, fixed, in the corpus**
    usually needs only history. For the latter, the corpus *is* the persistent memory — facts are
    re-retrievable by re-asking, so nothing load-bearing is lost on truncation. Building a memory
    layer there is speculative complexity for a problem the product doesn't have.
  - *The memory candidate is usually a **knob**, not extraction.* Where a retrieval product *does*
    have something that should outlive the window (which corpus/discipline is in focus), it's
    typically a user-declared **explicit parameter** — typed, carried per request, surviving
    truncation *by construction* because it never lived in the history — not a fact to distill with
    an LLM extractor. Extraction adds cost, latency, and a **fabrication vector**: a hallucinated
    "fact" persists and poisons every later turn's citations (worse than a one-turn hallucination —
    invisible and propagating). The generic instinct "let the LLM remember" collapses, for a
    traceability product, into "make it an explicit knob."
  - *History ownership: server, decided by the **assistant-turn forgery** vector, not "control."*
    If the client owns history, an attacker injects a fake `assistant` turn (*"§5.1 requires X"*) the
    model then builds on — untrusted content of arbitrary size entering the prompt, exactly the
    injection-guardrail applicability condition. Server ownership closes it (the server generated the
    assistant turns; it knows what was actually said). Cost recorded honestly: the backend stops
    being stateless (each query now needs its state → session affinity or a shared store). An
    in-memory dict is fine for a bootstrap **only with a single worker** — with `--workers N`, turn 2
    can land on a worker that lacks the session; write that as a *condition*, and defer the shared
    store (Redis/PG), TTL, and federation to a deployment phase. It composes with the fail-policy
    test above: session state is **not degradable** (its absence changes the answer) → a
    provided-but-absent `session_id` is a **loud error**, never a silently-single-turn answer.
    *And enforce the condition, don't merely write it:* a startup guard that refuses
    `WEB_CONCURRENCY > 1` while sessions are in-memory turns the comment into a **config-relation
    fail-fast guard** (the shape of the eval-mode guard below). What makes it worth building is that its
    trigger is anchored to the **mechanism**: `WEB_CONCURRENCY` is the variable the server itself reads
    to spawn workers, so declaring N both starts N workers *and* trips the guard — it catches the
    standard deployment path, not just the operator who happens to be honest. *A guard's reach is
    proportional to how tightly its trigger couples to the real failure mechanism*; the gap that stays
    uncoupled (a CLI `--workers N` never passes through settings, because a worker can't count its
    siblings) goes in the guard's own error message, or the operator switches off what they can't see
    the point of.
  - *Window strategy — simple sliding window over cumulative summary, and **why** for a traceability
    corpus.* A plain last-N-turns window is defensible while (a) the source of truth is the corpus,
    not the history, and (b) anaphora is short-range (it points at the prior turn; conversations
    *drift* forward). **Cumulative summary is contraindicated here — not on cost, on traceability:**
    summarizing turns with an LLM is generating **unverified text that re-enters context as if
    faithful**; a mis-compression (criticality B vs C, a clause misattributed to the wrong standard)
    plants a **fabricated normative claim in the history** that contaminates every later turn. Same
    fabrication vector as the memory extractor and the semantic cache, via the summary door. The
    sliding window is the *only* history strategy that introduces no unverified text. Reopen only if
    the product demands **long-range multi-turn synthesis** (a wedge decision — a Q&A product asks
    short-chained questions; an analysis copilot weaves), and even then prefer **anchors of whole
    turns** (hybrid-with-anchors where the anchor is the intact turn) over a generated summary.
  - *Once retrieval is wired, history has two more jobs:* it must reach the **retriever** (the
    question is also the search query), and the **stored** assistant turns must carry no citation
    markup, because stored history is an example the model imitates (Phase 3.5, *Wiring retrieval
    into generation*).
  - *Implementation notes (agnostic, minor):* mutate the history collection **in place**
    (`history[:] = history[-2N:]`) rather than reassigning the attribute, to avoid re-validating the
    whole Pydantic model on the hot path; and beware a constant that encodes an **implicit
    cardinality** (`-2N` assumes exactly two messages per turn) — it's silent debt the day a turn
    gains a third role (a `tool` message), so annotate it with its *trigger condition*, not a vague
    "fix later."

### Frontend / demo UI

A UI belongs to this phase only as a **thin client** — it makes the service *usable and
demonstrable*, it is not a second backend.

- **The frontend is a thin HTTP client, enforced by image, not convention.** It talks to the backend
  only over HTTP (same contract as any external consumer); it must not import the service layer, the
  provider SDKs, the aggregation library, or the cache. Enforce this at the **dependency/image level**
  — the frontend's build installs only its own dependency group (no backend packages), so it *cannot*
  import the backend even by mistake. Payoffs: it exercises the real product contract (the API); it's
  **deletable/replaceable** without touching the backend; and migration to a mature custom frontend
  later is front-end-only work. *(A shared-process UI that calls the model directly — common in course
  material — forfeits all three; it's testing a shortcut, not the product.)*
- **"Conversational" UI ≠ multi-turn.** A visible chat history (session state) that is *not sent back
  to the model* is single-turn with a scrolling log — it maps onto a single-turn endpoint with no
  schema change. Real multi-turn (sending prior turns as context) is a separate, backend concern (see
  *Multi-turn conversation state* above): when it arrives, ownership of the history moves from the
  frontend's presentational `session_state` to the **server**, and the frontend keeps only an opaque
  `session_id`. Don't smuggle real multi-turn in via the UI. Because the *presentational* history is
  cheap to migrate, that cheapness is a property of the single-turn *scope* as much as of the HTTP
  decoupling.
- **Validate a streaming contract by building the real consumer, not just server-side assertions.**
  "Did the server enter the right path?" (`cache_hit: true`, no `chunk` events) does **not** catch a
  delivery failure to the client ("can the client actually read the answer on *every* path?"). Witness:
  a cache-hit over the stream endpoint had no channel to deliver its text (the terminal event didn't
  declare the text field, and a hit sends no chunks) — the bug survived two briefs because every test
  asserted server state, none asserted client-readable output. The test that matters is *can the client
  read the response in all branches*.
- **A test that asserts current behavior without asking whether it's correct is cement, not a safety
  net.** Recurring failure across sessions: a test named for the property it should protect
  (`…does_not_fabricate_citation`) whose body asserted the *fabrication* as expected — it passed
  mechanically and gave false coverage over exactly the broken point. Wrong test *intent* is more
  dangerous than no test: no test is a known gap; a green test encoding the bug is a gap that looks
  covered. When writing a characterization test, assert the behavior you *want*, then make it pass —
  don't enshrine what the code happens to do today.
- **An inherited dependency is not a validated path.** Finding a library already in a sibling project's
  manifest doesn't mean the pattern that uses it was exercised there. Verify the path, not the presence.
- **Don't reconstruct backend state in the frontend to display it.** If a diagnostic panel wants the
  active system prompt or injected context and the backend doesn't expose it, leave it a `TODO` and add
  a backend endpoint — reconstructing it client-side is the coupling-by-reimplementation the thin-client
  rule exists to prevent.
- **The backend counterpart — expose diagnostic state by *wrapping* the real source, never by
  re-deriving it.** When you add that endpoint, it must return the *same* value the model actually
  received: the active-instructions endpoint returns the very constant the prompt builder composes
  (not a re-read of the file), and the injected-context field reuses the *same* context string
  already built for the prompt/cache-key that request (not a second derivation). Factor the builder
  so the diagnostic path and the real path share one source; fidelity is then guaranteed **by
  construction**, not by remembering to keep two copies in sync. Match exposure to naturalness:
  instructions are static (a global `GET` is stable across CAG→RAG); context is per-request in RAG,
  so it rides the query response, never a global endpoint that RAG would invalidate.
  - *Revised once it became evidence — the trace is part of the contract, not an opt-in.* The first
    version gated the context behind a request flag, as a string blob. Once the context *is* the
    retrieved evidence, it travels **always**, typed (a list of evidence atoms), in the response and
    in the stream's terminal event. **An API contract does not depend on a flag or on the
    environment**: a contract that forks by `APP_ENV` makes the tests evidence about a shape that
    isn't the deployed one. Hiding diagnostic noise is the **render's** job, governed by the
    *frontend's* own env variable and compared so that a typo **hides** rather than leaks. And the
    atom is **one shared model**: the search endpoint's hit *extends* it with retrieval-only
    diagnostics (per-channel ranks) instead of a twin DTO with the same fields — two twins are two
    places that drift. Keep *evidence* and *citations* semantically separate: citations are what the
    model **marked** (Phase 3.5), evidence is what was **retrieved**, and the evidence travels on every
    answer including a refusal, because what was retrieved is a fact of the system independent of what
    the model did with it — "the retriever brought these five and the model still couldn't find it" is
    the primary signal for diagnosing retrieval.

#### Decision points

- **UI framework — by maturity and app shape.**
  - *Maturity:* simple Python frameworks for MVP/demo; a custom frontend (React/own) for a mature
    product, if it warrants one.
  - *Applicability condition:* **Streamlit** when you need more than chat — a sidebar, metrics,
    structured presentation of results (e.g. verifiable citations rendered as more than chat text).
    **Chainlit** when the differentiator is inspecting *agent reasoning* step-by-step — irrelevant
    until there are agents to inspect (re-evaluate at an agent phase). **Gradio** for shareable
    multimodal model demos. A framework whose specialization you don't use is cost, not benefit.
  - *Note:* for a product heading toward a real customer, a demo UI is not a product face — the custom
    frontend is a maturity node, deferred, and made cheap to reach by the thin-client decoupling above.

---
## Phase 5 — From working service to product (interface & content guardrails)

A hardened service (Phase 4) answers reliably; it is not yet a *product*. This phase is about the
layers that make it usable and safe for a non-expert end user: taking the prompt off the user, and
validating **content** (not just shape). It sits on the output contract (Phase 2) and the provider
seam (Phase 1). Some of it is MVP-cheap (prompt robustness), some is a scaling node (LLM-as-judge) —
read the maturity field, don't adopt the whole pipeline at once.

### Patterns & practices

- **Where does the prompt live?** — the most useful architectural question when turning a chat
  feature into a product. In a chat, the prompt lives in the user's textarea and product quality
  becomes a function of how well each user prompts — that's a power-user tool, not a product. Move
  the prompt to the **backend** (versioned, testable, route-able for cost): the user supplies
  *parameters and content*, the backend composes the full prompt from a template. Corollary already
  used in Phase 1/3 (thin client, prompt-as-artifact): the frontend sends structured input, not a
  prompt.
- **"Bake the affordance into the interface."** A bare textarea makes every user guess what the
  product can do and how to phrase it; a form with concrete fields, a mode selector, and a verb-y
  button teaches capability *and* guarantees a quality floor. This is the same move as prompt
  robustness below, on the input side.
- **Guardrails are defense-in-depth, ordered cheap→expensive.** No single guardrail covers the
  space. Layer them so each case is rejected by the cheapest layer that catches it: regex/heuristics
  (<1 ms, free) → a moderation classifier (~50–100 ms, free) → prompt robustness (zero runtime cost)
  → a validation retry (a model call) → LLM-as-judge (another model call). Combined, they're cheap;
  any one alone is porous.
- **Every guardrail declares its failure policy — explicitly.** Three canonical policies:
  **exception** (abort with a clear error — severe, non-degradable violations: PII, clear injection,
  explicit toxicity), **fix-with-retry** (show the model its error and re-ask — recoverable: malformed
  JSON, a failed cross-field check), **filter/degrade** (return a safe default — out-of-scope, low
  confidence). *The most common production mistake is not choosing wrong, it's not choosing* — code
  review of any new guardrail should ask "what happens when this fires?" and get one of the three as
  an answer. Don't manufacture guardrails to "have all three": one well-chosen guardrail with one
  declared policy beats three forced ones.
- **Log-only before blocking.** Ship a new guardrail in log-only mode, watch what it fires on for a
  week or two, tune thresholds on real false positives, *then* enable blocking. A moderation/validator
  call returns a score, not a boolean — the threshold is a **product** decision. (Your Phase 4
  structured logging is where log-only lives; a guardrail in log-only is one more structured event.)
- **Content validation is the layer that "form ≠ truth" needs.** A schema-valid response can still be
  a prompt-injection echo, out-of-scope, PII-leaking, or a hallucination with impeccable formatting.
  Syntactic validation (Pydantic: types, ranges, cross-field coherence via a model-validator) is
  cheap and unconditional; *semantic* validation (is it safe, in-scope, grounded, non-fabricated) is
  the harder, often model-priced layer. Keep the split explicit: **emitting a citation is not
  verifying it; a valid shape is not a true content** (the citation-verification layer is a distinct,
  later generation-quality concern — don't let a schema's presence imply it's solved).

### Decision points

- **Where to sit on the interface spectrum** — `chat → chat+params → form/action → generative UI`.
  - *Maturity:* a post-MVP product decision — you have a working single-shot endpoint and are turning
    it into a product surface.
  - *Applicability condition:* driven by how much **legitimate input variability** the task has.
    Genuinely open, exploratory input (brainstorming, "explain X") → chat is correct. Finite,
    consistency-demanding input (an estimate with fixed parameters) → form/action, prompt fully
    abstracted. A **question-answering product over a corpus is a hybrid**: the question itself is
    open natural language (it doesn't reduce to dropdowns), but *knobs around it* are finite (output
    format, detail level, and — once retrieval exists — scope filters by document). Not chat-pure, not
    a form: **open question + structured knobs**.
  - *Note — knobs can be phased:* a knob is only real once the machinery behind it exists. A
    "search only standard X" filter is meaningless until there's retrieval to filter; output-format
    is available from day one. The interface matures with the modules underneath it.

- **Guardrail tooling — by maturity and cost.**
  - *Prompt robustness (Capa 3)* is the cheapest and often most effective, and it's **MVP**: give the
    model explicit permission to say "I don't know," define scope in the system prompt, forbid
    fabrication. For a grounded system this *is* the grounding instruction — it doubles as a guardrail.
    Zero runtime cost.
  - *Moderation API / regex heuristics* — MVP, cheap, for toxicity/injection/PII on user text. Frail
    as a sole defense (a substring list of injection phrases is exactly the brittle heuristic to not
    rely on alone); fine as one cheap layer. *Applicability is decided by the input surface, not the
    corpus domain:* any public endpoint that accepts free user text justifies moderating the input,
    however technical the subject matter — the domain doesn't control what a user types. Two honesties
    to keep: (1) a hosted moderation API is a **network call to a provider**, not a local check — it
    *adds* latency to every request (including cache hits), it does not save calls; its justification
    is input hygiene, never cost. (2) **Toxicity ≠ prompt injection:** a toxicity classifier does not
    catch attempts to subvert the system's instructions; that's a different detector. Adopting
    moderation "to protect the input" covers one threat and leaves the other open — name it the
    *toxicity* guardrail, not *the* input guardrail. Keep the moderation client **separate from the
    completions provider abstraction** (it isn't a completion, has no fallback; routing it through the
    same Router couples "which LLM answers" with "which requests get in").
  - *Pydantic model-validators* — MVP, for internal-coherence rules the schema can't express.
  - *Guardrails AI / LLM-as-judge* — a **scaling** node (a library dependency, or a second model call
    per request). Adopt when Pydantic-plus-prompt no longer covers the semantic cases and the cost is
    justified; don't reach for it by default when validators already in-house suffice.

- **Signalling refusal on a grounded corpus — two refusal types, not one.** *(domain-flavored but
  general to any grounded RAG)*
  - *Applicability condition:* a system that answers **only** from provided material and must not
    fabricate. Distinguish **out-of-domain** (the question isn't about your domain at all) from
    **out-of-material** (a legitimate in-domain question the provided material doesn't cover). A pure
    generator (the estimator) needs only the first; a **traceability** system needs the second — it's
    the one that prevents a phantom citation when the corpus doesn't hold the answer. Both are the
    *filter* policy; instructed in the prompt, not a Python validator (real grounding verification is
    a later phase).
  - *Bridge pattern until structured output — a natural marker phrase.* To act on a refusal (type it
    in the response, log it) before you have a typed refusal field, have the model **begin the refusal
    with a fixed natural phrase** (`"Out of scope:"` / `"I could not find this in the indexed …:"`) and
    detect it by prefix at the shared assembly point. The phrase reaches the user verbatim, so it must
    be in **product voice** and **true by construction**: about the search, never about the corpus
    (Phase 3.5 — an earlier "Not covered by the indexed standards" asserted absences retrieval had
    merely missed). Choose a *natural
    phrase over a technical marker* deliberately: a technical token (`OUT_OF_MATERIAL:`) would have to
    be stripped from the text, and on a streaming endpoint that means **buffering the stream start** —
    streaming-only logic the "streaming is a deletable delivery layer" rule (Phase 4) forbids. A
    natural phrase *is* the readable message: it streams as-is, detection happens on the assembled
    text, not on bytes already sent to the client. **The streaming principle dictates the guardrail
    mechanism.** It's a bridge — prefix detection is frail if the model doesn't emit the phrase
    verbatim — that retires when structured output gives a typed refusal field. *Measured limit:*
    a binary prefix can't express a **partial** answer, and prompting for "answer what you found, name
    the gap at the end" failed with a small model (two rewrites; it still opened with the refusal
    phrase when the named standard wasn't retrieved). The fix is a typed coverage field in the output
    contract, not more prompt. *(Historical bridge, now retired: while citations were derived from the
    injected material, a candidate citation attached unconditionally was a phantom citation on every
    refusal, so citations were cut on detected refusal. Once citations come from the model's markers
    the phantom can't occur, and the cut only hid real citations — a bridge rule outlived its premise.)*

- **Testing & evaluating a probabilistic component — two sovereign suites, not one with exceptions.**
  - *The reframe:* `assert == expected` doesn't apply to LLM output; you test **properties**, not
    equality (else: false negatives when the model says "16h" not "16", or false positives when
    `len > 0` passes while the system returns "I can't help" to everything). But the deeper split is a
    **second axis orthogonal to the hard/soft/judge pyramid: deterministic vs non-deterministic**,
    and it partitions "testing" into two products. **Contract suite** (the code's): does the *code*
    honor the contract — binary verdict, deterministic, **mocked, no network/model**, runs in CI,
    blocks merge. Only the hard family (schema validity, ranges, invariants, assembly order, cache
    gates) lives here, **with the model response mocked**. **Evaluation suite** (the developer's):
    does the *LLM* produce quality — gradient verdict (score, distribution, trend), non-deterministic,
    **real model + cache disabled**, run on demand/periodically, needs statistical analysis, does
    **not** block merge. The soft/judge families live here. A "no network, mocked" rule is the rule of
    the *contract* suite — it is not universal; naming the evaluation suite as sovereign is what keeps
    the contract rule from looking like it was violated.
  - *Evaluating a probabilistic component is a **first-class product function**, not an external
    activity.* In deterministic software the test observes from outside and the system needn't know
    it's under test. With an LLM under test, supporting an evaluation mode is *part of the system* —
    for a traceability RAG, measuring fidelity *is* knowing whether the product works ("works" isn't
    binary). Materialize it as a **single named mode** (`LLM_EVAL_MODE`) that declares *intent* and
    lets the system *derive* the mechanics (cache off; later, content logging, sampling). A handful of
    independent flags turns "being in eval mode" into a memorized ritual whose forgetting yields a
    **silently invalid evaluation** — same failure family as a half-active cache. And the powerful
    mode needs a **fail-fast guard**: refuse to start when the eval mode coincides with production (a
    validator over the *relation between two config fields*, not one field) — the mode turns off
    protections/optimizations, so accidental production activation is catastrophic.
  - *Two domain-specific reframes:* (1) **contract tests must not depend on which provider answered**
    — with a fallback, pinning one provider's output breaks the test when the other answers, which is
    a normal path; test system properties, not model outputs. (2) In a traceability system the
    **consistency metric is citation stability, not text stability** — "do N runs cite the same
    clause?" beats variance of prose; and with a deterministic cache, consistency of *cached*
    questions is an **invariant, not a statistic** (the cache is reproducibility of the cited answer).
  - *What is deferred (don't graduate as done):* the serious evaluation content — golden datasets,
    LLM-as-judge, faithfulness/RAGAS, a library like DeepEval — is a later generation-quality phase;
    it needs retrieval and fidelity to evaluate. Define the mode and the two suites now; populate the
    evaluation suite then. Family-1 contract tests need only pytest + Pydantic, no eval framework.

- **Adapting output to a user profile ("tier") adapts generated output, not cited material.** *(by
  negation — the useful form of tier for a retrieval system)*
  - *Applicability condition:* the generic tier pattern (per-profile template + per-profile output
    schema; adapt *structure*, not just *tone*) presumes the system **generates** the substance it
    reshapes. A **retrieval system with mandatory traceability** *reports* the norm — the substance is
    the source and is **invariant to the profile**: reformulating a clause for an "executive" either
    quotes it verbatim (not adapted) or paraphrases it (unverified text under a normative citation —
    fabrication). So tier here may only touch **the glosa and presentation around the citation**
    (wrapper verbosity, explanation language, presentation format, and — once retrieval exists —
    retrieval breadth), **never the citation itself**. Poorer than the generic pattern, and that
    poverty is the traceability contract doing its job.
  - *And the profile is another **explicit knob**, not inference:* like the focus-corpus knob, it's a
    typed request/session parameter, not tone the model improvises. *The threat vector of a knob is
    proportional to what it unlocks* — a knob that only controls glosa verbosity tolerates
    client-ownership; one that unlocks cost or data does not (contrast history, where client-ownership
    was unsafe because it lets forged turns *into the prompt*). The generic "never trust the client
    with tier" is right for authorization/cost gating, gradient for presentation-only knobs.
  - *Infra caveat:* the article's persistence+propagation layer (a `users.tier` column, JWT/header
    propagation) presumes a business backend that a single-user demo doesn't have — N/A by absent
    infrastructure, same as the business-DB mechanism below. And a tier that dispatches a whole
    *pipeline* (not just a template) is an agent-phase concern; keep the dispatch point at the seam so
    you don't close that door, but don't build it now.

- **Verifying citations (generation-quality phase, deferred) — the canonical shape is
  generation/verification separation.** *(forward pointer, not an implemented node)* The
  emitting≠verifying split (Phase 2) is closed, when its phase arrives, by separating **generation**
  from **verification** in distinct calls (Self-Refine; and the three-role Actor/Critic/Boss, where
  splitting *evaluation* from *decision* breaks the pure-Self-Refine failure modes: infinite
  refinement and early-confirmation bias). In a retrieval system, "criticize the output" *becomes*
  "verify the citation": an Actor generates answer + candidate citations, a Critic checks each claim
  against the clause it cites, a Boss accepts / re-generates / suppresses unverifiable citations. An
  `{answer, citations[]}` contract kept separate from v0 is already compatible with this — nothing to
  undo. Maturity: it earns its place only when the error cost is high, the evaluation criteria are
  clear, and the extra latency is tolerable; it is not an MVP node and does not graduate as
  implemented until that phase.

---

## Cross-cutting method (applies to every phase)

These are not about *what* to build but *how to build it reliably*. They earned their place the hard
way — mostly in the data layer, but they generalize.

- **Diagnose before you repair.** Precede every structural change with a cheap diagnostic that
  characterizes the problem before you touch it. It isn't ceremony: a conversion diagnostic surfaced a
  silent-success tool bug; a numbering diagnostic showed the problem was 500 clauses, not 4; a table
  diagnostic showed "merged" was three problems. Writing the repair without the diagnostic
  mis-sizes it. The diagnostic's output *is* the repair's specification — but only when it measured the
  property the repair will **enforce**, not a proxy for it. A count taken with a mismatched instrument
  becomes a confident, wrong spec: a "lexically-blind chunks" count (103) stood in for a "verbatim is a
  withdrawn-clause marker" criterion (121), and the 18 it missed were the *retrievable* — the dangerous
  — ones (Phase 3.5). Before a diagnostic number becomes an enforced gate, check that its instrument
  and the gate's criterion are the same question. And for a membership predicate, no count can carry
  the guarantee at all: a count is blind to what the predicate *under*-matches, because it is taken
  with that same predicate — verify the complement with an independent, broader oracle instead.

- **A proposed fix names a mechanism; verify the mechanism is the cause before tuning it.** "Weight
  the fusion" was the obvious remedy for a hybrid that hurt generic queries, and it would have been
  tuned on a dozen queries to fit them. Four cheap checks showed the cause sat one layer down (an
  asymmetric tokenizer and a ranking without IDF); fixed there, no weight was needed. Same shape as a
  UAT that can't discriminate: if the expanded unit is never cited, the answer can't change — design
  the probe so it **depends** on what you're testing, or its "no difference" means nothing.

- **Measure and repair are different tasks — keep them separate.** Audit apart from fix, diagnosis
  apart from extraction. A brief that "audits **and** fixes what it finds" is open-scope; the
  diagnostic should hand the repair a concrete list of cases, so the repair works from data rather
  than blind. (Corollary: after measuring a probabilistic component, don't tune it to improve the
  number in the same step — the concrete failures *are* the spec for a separate tuning task.)

- **A specification that inherits invariants must verify each instruction against them, not describe
  the desired result.** When a step depends on invariants an earlier step established, "regenerate the
  output," "iterate the directory," and other result-phrased instructions can each violate one (delete
  the artifact a sibling clause just declared stale; reconstruct identity from a lossy filename the
  prior step made canonical). *The right result reached the wrong way breaks the invariant.* Cross
  each instruction against the standing invariants — the same discipline as a characterization test
  that asserts the *new* behavior, applied to writing the spec itself.

- **A clean-looking corpus hides the silent failure precisely by looking clean.** The single most
  load-bearing process lesson: the whole data-layer overhaul was triggered by an unplanned audit
  ("check the downloader, just in case") that surfaced a 50%-coverage gap nothing else would have
  caught. *Audit what appears to work* — the absence of visible dirt is not evidence of correctness,
  it's the removal of the suspicion that would make you look. It's the course's "vectorize last, not
  first" pushed one step earlier: before you *handle* the data, verify it's the data you think it is.
  - *Generalized across layers — measure each layer's real property before building the next on it,
    against an oracle appropriate to that layer.* Every layer rests on a property of the one below
    that theory assumes and the real world may not honor. **Parsing:** structural fidelity — verify
    against the **render**, because *verifying against the text is not verifying against the meaning*:
    a parser can have every string-equality invariant green and still assert something else, because
    the document means with visual structure (numbering, indentation, nesting, which paragraph
    qualifies which) that plain text doesn't carry. A flattened list and text pulled out of a floating
    box both pass "200/200 chars literal" while saying the wrong thing. **Embedding:** representation
    discrimination — verify with a **domain probe** (the trap pair) that the model separates what the
    system must distinguish. The absence to hunt is never only *missing* content; it's a *relation*
    that the layer silently failed to preserve.

- **A plausible-but-false artifact is the dominant silent failure — and it recurs in one shape.** A
  string-prefix "descendant" test (`"5.20".startswith("5.2")`), a batch response trusted in arrival
  order (each item gets its neighbour's vector), a match key built from repeated content (invents
  correspondences and reports movement where there is none): each produces output that is
  **shape-consistent and content-false**, invisible to any check downstream. Two rules fall out.
  *Where a reliable structural order exists, key on position, not on content* — a content-derived key
  in a repetitive corpus fabricates matches, and in the direction that most alarms (reporting
  re-anchoring that didn't happen). *And compare against the canonical structure by component, not by
  surface.* *And match on the whole value, never a substring of it* — a marker predicate keyed on the
  token "moved" swallowed the real clause "…shall be removed…"; substring containment fabricates the
  same false positives as the prefix test above.

- **Re-examine a bridge rule when the mechanism under it changes.** A protection built for one
  mechanism becomes the bug of the next: "cut citations on refusal" protected against phantom
  citations from injected material, and produced unsourced `[n]` once citations came from markers;
  rewriting stored citations to natural IDs fixed stale handles and created a format the model copied.
  When you replace a mechanism, list the rules that existed *because of* it and re-justify each.

- **One measured prompt iteration, then change the lever.** With small models an exact-format
  instruction outweighs a judgment rule, and prohibitions are obeyed in the letter ("not indexed" →
  "does not contain"). Allow one prompt rewrite with a before/after measurement; if it doesn't move
  the number, the lever is structural (output contract, retrieval), and further prompt edits are
  spend without signal.

- **Verification has its own failure modes — a green check can be a check that never ran.**
  *Resolving a discrepancy is not passing the test:* explaining why an anomaly *looks* wrong (a copy
  error in the source) does not confirm the thing the test asked; a verification that stops when the
  anomaly is "explained" closes a control without executing it. *Some verifications are
  order-sensitive:* a tool that compares on-disk state against a fresh run only means something *before*
  you overwrite the disk — run it after and it reports 0 because you erased the evidence. *A test that
  mocks the frontier does not test the wiring up to the frontier:* injecting the client leaves the
  path from config to seam unexercised, so a seam reading the wrong config source passes every mocked
  test and fails only on first real run (and silently, if the wrong value happens to match). *A small
  count still demands inspection — rare is not nonexistent:* "5 of 1455, negligible" and "0, the regex
  false-matched all 5" are the same decision reached for opposite reasons, and only one leaves a true
  belief behind. *And a concurrency test must count the work done* while *the call is in flight, not in
  total:* a heartbeat tallied after the awaited call returns reaches the same total whether the loop
  stayed free or starved — it just arrives later; count the ticks that occur *during* the outstanding
  call (19 free vs 0 starved), or the green proves nothing. *The general form of several of these: an
  instrument measured with, or against, the thing it verifies cannot detect that thing's omission.* A
  count gate taken with the drop predicate is blind to what the predicate misses ("Diagnose before
  you repair"); a table-conservation check summing the parser's output against the parser's own list
  can't fail (Phase 2.5). Before trusting a green, ask where its reference total comes from — if the
  answer is "the same code under test," it is a check that cannot run red.

---
