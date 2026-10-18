# RAG-ECSS — Retrieval golden set: annotation protocol

> Protocol version: **1.0** (S10). Changing anything below the line "Criterion" changes the meaning
> of every measurement taken with the previous version: bump the version and record why.
>
> **Who runs this:** a fresh Claude Code session (Opus), opened on the RAG-ECSS repo, separate from
> any session that implements or tunes retrieval. Paste this whole file as the first message, plus
> the paths to the clause export and the document index that the harness produced.

---

## Your role and the rules you must not break

You are building the reference ("golden") set used to measure a retrieval system over ECSS
standards. Your judgments are the yardstick; a biased yardstick silently corrupts every later
decision. Therefore:

1. **Read only the clause export and the document index.** Do not call the application, its API,
   `/search`, the database, or any script under `app/`, `scripts/` or the harness. Do not open any
   file that contains retrieval results, rankings or previous measurements.
2. **Ground every judgment in the exported text.** You know ECSS from training; that knowledge may
   refer to other revisions or be wrong. A clause is relevant only for what *its exported text*
   says. Never cite a clause number or a quote you did not read in the export.
3. **Quote verbatim.** Every positive judgment carries an `evidence_quote` copied character for
   character from that clause's exported text (≥ 40 characters, a contiguous span). It is verified
   by code; a quote that is not found sends your judgment to human review.
4. **Use the exact identifiers of the export** (`canonical_doc_id`, `section_number`). If a clause
   is flagged as having a repeated `section_number` in its document, say so in `note`.
5. **Do not optimize for any system.** You do not know how the retriever works and should not guess.

---

## Criterion

> **A clause is relevant if an ECSS engineer would use it to answer the question, whether normative
> or informative. Authority is metadata, not relevance.**

- Relevance is **binary**. No "partially relevant": if an engineer would use it, it is relevant.
- A clause that only *mentions* the topic in passing, or only lists it among others without saying
  anything usable, is **not** relevant.
- Notes, handbook explanations and informative annexes count when they help answer the question.
- For a question that names a standard, clauses of *other* standards are relevant only if the
  answer genuinely needs them (e.g. the named standard defers to them).

---

## Mode A — Build the golden set (first time)

### Phase 1 — Queries (do NOT open the clause export yet)

Using only your knowledge of ECSS and the **document index** (IDs and titles, no content), write
**exactly 10** questions an ECSS engineer working on space software would realistically ask, in
English, with this stratification:

| Stratum | Count | What it is |
|---|---|---|
| `names_standard` | 2 | Explicitly names a standard of the index (in any common form: "E-ST-40C", "ECSS-Q-ST-80") |
| `generic` | 2 | Conceptual question, no standard named |
| `paraphrase` | 1 | Asks about a topic using everyday wording rather than ECSS vocabulary |
| `comparative` | 2 | Compares two things (criticality levels, two standards, two processes…) |
| `cross_standard` | 2 | Answer spans several standards ("which standards apply to…") |
| `multi_turn` | 1 | 2–3 turns; the last turn changes topic or is elliptical ("and in Q-ST-80?"). Only the last turn is judged |

Rules: realistic phrasing, varied length, no question written to match a specific clause, only
standards that appear in the index. Output the 10 queries in the Phase 1 format below and **stop**:
the user approves or edits them before Phase 2.

### Phase 2 — Relevant clauses (after the user approves the queries)

For each approved query, search the clause export as thoroughly as you reasonably can (grep for
terms and synonyms, read candidate clauses fully, follow cross-references inside the export) and
list **every** relevant clause you find. Aim for completeness: a relevant clause you miss will be
counted as irrelevant whenever a system retrieves it, until someone judges it. Record what you
searched in `search_notes` so gaps can be understood later.

---

## Mode B — Judge pairs (incremental)

Input: a file of `(query_id, canonical_doc_id, section_number)` pairs produced by the harness.
For each pair, read that clause in the export and decide `relevant: true|false` with the same
criterion. Do not look for other clauses; judge only the pairs given.

---

## Output formats (JSON, UTF-8)

Phase 1:
```json
{
  "protocol_version": "1.0",
  "model": "<model you are running as>",
  "date": "YYYY-MM-DD",
  "queries": [
    {"id": "q01", "stratum": "names_standard", "turns": ["<question>"]},
    {"id": "q10", "stratum": "multi_turn", "turns": ["<turn 1>", "<turn 2>", "<turn 3>"]}
  ]
}
```

Phase 2 (one object per query, appended to the Phase 1 file as `judgments`):
```json
{
  "query_id": "q01",
  "search_notes": "<terms searched, documents scanned, references followed>",
  "relevant": [
    {
      "canonical_doc_id": "<as in export>",
      "section_number": "<as in export>",
      "evidence_quote": "<verbatim span, >= 40 chars>",
      "justification": "<one sentence: why an engineer would use it for this question>",
      "note": "<optional: repeated section_number, doubt, etc.>"
    }
  ]
}
```

Mode B:
```json
{
  "protocol_version": "1.0",
  "model": "<model>",
  "date": "YYYY-MM-DD",
  "pair_judgments": [
    {"query_id": "q03", "canonical_doc_id": "…", "section_number": "…",
     "relevant": true, "evidence_quote": "<required if relevant>", "justification": "…"}
  ]
}
```

---

## Optional drift check

A second fresh session runs Phase 2 for 2 of the approved queries without seeing the first
session's output. The harness reports agreement between both sessions on those queries. Large
disagreement means the criterion is being read differently and must be tightened before trusting
the set.
