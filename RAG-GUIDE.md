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
  `client.create(...)` across the codebase.
- **Async correctness — a failure mode, not a preference.** Wrapping a *blocking* SDK call in
  `async def` does **not** make it non-blocking; it freezes the event loop for the entire call
  and forfeits the only reason you chose an async framework. Use the SDK's **native async
  client** (`AsyncAnthropic`, `AsyncOpenAI`, …), or offload to a thread (`asyncio.to_thread`)
  when no async client exists. *(Tutorial/course snippets frequently get this wrong — they call a
  sync client inside `async def`. Don't copy them.)*
- **Fail-fast config.** Required secrets have **no default** → the app refuses to start if they're
  missing, instead of surfacing the failure on a user's first request. Keep secrets out of code:
  `.env` (git-ignored), with `.env.example` documenting the required keys.

### Decision points

- **Provider / model choice** — deferred cheaply behind the seam. When you do pick:
  - *Practice:* **verify model currency at decision time.** Model lineups and the "cheapest
    current tier" move fast; the obvious default from any written material may already be legacy.
    Confirm the model ID, price, and status before wiring it.

---

## Phase 2 — Output contract first

Define the **response schema before building retrieval**. Lock what an answer looks like — its
fields, and above all how it carries its sources.

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
- **Emitting citations ≠ verifying them.** *Emitting* = transporting the coordinates of the
  material you fed the model (a v0 schema concern). *Verifying* = proving the answer actually used
  what it cites and didn't hallucinate a reference (a later generation-quality concern). The
  schema field's mere presence does not mean verification is done — mark this explicitly so a
  later phase doesn't assume it's solved.

### Decision points

- **Citation granularity** — document / section / clause / page.
  - *Applicability condition:* driven by what your corpus structure **and source format** actually
    support, and by how precise a citation your users need to verify a claim. Don't promise a
    granularity your data can't back.

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

### Decision points

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
