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
  - *Trigger signal:* the first real branch in the prompt (e.g. a genuine output-format or
    detail-level knob that changes instructions). Re-evaluate then, not before.
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

---

## Phase 4 — Production-hardening the service (provider robustness, cache, streaming, observability)

Once the vertical slice answers correctly, these are the layers that turn a working endpoint into
a service that survives contact with real use. None of them changes *what* the answer is — they
change availability, cost, perceived latency, and your ability to see what happened. They all sit
on the provider seam (Phase 1) and the output contract (Phase 2).

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
- **The cache decides whether there's a stream at all.** A response cache sitting above the
  provider layer means a hit never calls the model — so there is nothing to stream. Look up cache
  first; on hit, return whole; only on miss open the stream. Cache and streaming don't interact
  beyond this: the cache decides *if* the request even enters the streaming path.
- **Truncation is an integrity failure, not a cosmetic notice — when traceability is mandatory.**
  `finish_reason == "length"` may have cut the answer before its citation, yielding a normative
  claim with no anchor. Detect it and mark the response unreliable; do **not** try to prevent it by
  instructing the model to be brief (that competes with completeness — it may drop applicable
  requirements to fit a word budget). Generous `max_tokens` + detection, not prevention by prompt.
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

- **Response caching — exact-match.**
  - *Maturity:* MVP-cheap once inputs repeat.
  - *Applicability condition:* a **deterministic** task where repetition is desirable (not creative
    generation). For a **normative/traceability corpus this is stronger than cost savings**: an
    identical query returning an identical, identically-cited answer is an *auditability* property,
    not just a latency win.
  - *Trigger signal:* observed repeated inputs (hit rate above ~20% justifies the infra).
  - *Key design (the core decision):* build the key from **explicit slots** —
    `question + system_prompt + context + corpus_version` — and keep `context` and `corpus_version`
    as **separate slots even when static today**. This is why you'd hand-roll the cache instead of
    using a library's built-in: when retrieval later injects dynamic context, populating the
    `context` slot is a fill, not a redesign. Caching on `question + system_prompt` alone, once
    context is dynamic, would serve a stale answer built on chunks retrieval would no longer
    return — a **phantom citation served from cache**, the exact failure a traceability system
    exists to prevent.
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

- **Cache invalidation strategy.**
  - *Applicability condition:* for a **versioned corpus**, the correct answer changes when the
    *corpus* changes (a new standard revision), not on a time schedule. Prefer **invalidation by
    corpus version** (a `corpus_version` slot in the key) over pure TTL; TTL is the safety net, not
    the primary mechanism. Event-based invalidation, which generic material treats as the exception,
    is here the main mechanism.

- **Semantic cache — match on *meaning* instead of string equality.** *(a real win for the right
  corpus; recorded as a full fork, since whether it ships is corpus-dependent)*
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
  schema change. Real multi-turn (sending prior turns as context) is a later, separate concern; don't
  smuggle it in via the UI. Because the history is presentational, migrating the frontend stays cheap —
  that cheapness is a property of the *scope* (single-turn) as much as of the HTTP decoupling.
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
  so it rides the query response (opt-in behind a flag — it's *content*, not metadata, and can be
  large), never a global endpoint that RAG would invalidate.

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
    rely on alone); fine as one cheap layer. Applicability is domain-shaped: near-irrelevant for a
    normative technical corpus, essential for open user-generated input.
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
  - *Bridge pattern until structured output — a natural marker phrase.* To act on a refusal (e.g.
    suppress the candidate citation on it) before you have a typed refusal field, have the model
    **begin the refusal with a fixed natural phrase** (`"Out of scope:"` / `"Not covered by the
    provided material:"`) and detect it by prefix at the shared assembly point. Choose a *natural
    phrase over a technical marker* deliberately: a technical token (`OUT_OF_MATERIAL:`) would have to
    be stripped from the text, and on a streaming endpoint that means **buffering the stream start** —
    streaming-only logic the "streaming is a deletable delivery layer" rule (Phase 4) forbids. A
    natural phrase *is* the readable message: it streams as-is, detection happens on the assembled
    text, not on bytes already sent to the client. **The streaming principle dictates the guardrail
    mechanism.** It's a bridge — prefix detection is frail if the model doesn't emit the phrase
    verbatim — that retires when structured output gives a typed refusal field. *(A related domain
    trap: a candidate citation derived from the injected material and attached unconditionally becomes
    a phantom citation on every refusal — and on a single-fragment CAG slice, refusals are the common
    case, so the better the scope guardrail works, the more phantom citations it produces. Cut the
    citation on detected refusal.)*

---
