# Retrieval: corpus, chunking, citations

The assistant is only as trustworthy as the corpus. This file is how I migrate
the JSON Q&A, what else is allowed in, how chunks are cut, and how we cite them.

---

## Corpus (allowed)

| Source | Origin | Role |
| --- | --- | --- |
| **Migrated Q&A JSON** | Current in-app chatbot file | Seed + golden regression answers |
| **Allowed Jobbser docs** | Curated help Markdown we publish into the bucket | Paraphrase coverage, feature explainers |
| **Platform facts** | Short owned sheet (what Jobbser is / is not) | Grounding for “what is this app?” |

Everything else is out. No web crawl. No RDS. No Slack exports. No random PDFs
from a shared drive unless product marks them `assistant-safe` and strips PII.

### Assumed JSON shape (stand-in)

Interview pack does not include the live file. I assume:

```json
{
  "version": "2026-03-01",
  "entries": [
    {
      "id": "qa-001",
      "question": "Is Jobbser free for workers?",
      "answer": "Yes. Workers use Jobbser free. Companies pay to activate job listings.",
      "tags": ["billing", "workers"]
    }
  ]
}
```

If the real file differs, the ingest mapper changes; the chunk contract stays.

---

## Migration steps

1. **Freeze a copy** of production JSON as `corpus/qa/v1/qa.json` in the corpus S3
   bucket (versioned bucket).
2. **Normalize** each entry into a chunk record (below).
3. **Add** curated help docs as separate objects `corpus/help/*.md` with frontmatter
   ids.
4. **Embed** all chunks; write `index/embeddings.jsonl` (S3 sketch) or upsert into
   OpenSearch Serverless.
5. **Keep JSON path** in the API behind `assistant.fallback = json` and for
   regression tests.
6. **Publish pipeline:** when product edits Q&A, CI or an admin action runs ingest
   (validate → embed → swap index alias / atomically replace embeddings object).

Replace the *runtime* source of answers gradually (see README phases). Do not
delete the JSON; promote it into the corpus.

---

## Chunk record

```json
{
  "chunkId": "qa-001",
  "sourceType": "qa_json",
  "sourcePath": "corpus/qa/v1/qa.json",
  "title": "Is Jobbser free for workers?",
  "text": "Question: Is Jobbser free for workers?\nAnswer: Yes. Workers use Jobbser free. Companies pay to activate job listings.",
  "tags": ["billing", "workers"],
  "updatedAt": "2026-03-01T00:00:00Z"
}
```

For Markdown help docs, chunk by heading (H2/H3) with ~200–400 token targets and
~40 token overlap. Overlap helps when a step list straddles a boundary.

### Chunking rules

- One JSON Q&A entry = **one chunk** (do not split question from answer).
- Help docs: split on headings; merge tiny stubs under 40 tokens into the previous
  chunk.
- Hard max ~500 tokens per chunk so `topK=4` fits the context budget.
- Strip HTML, tracking params, and any accidental emails in source docs at ingest.

---

## Query-time retrieval

1. Sanitize / redact user message.
2. Embed with the configured embedding model.
3. Score:
   - **S3 sketch:** cosine similarity against loaded embeddings; optional keyword
     boost if the query shares tokens with `title`.
   - **OpenSearch:** hybrid k-NN + BM25 on `title`/`text`.
4. Take `topK` (default 4) with `score >= minScore` (default ~0.72; tune on evals).
5. Pack into CONTEXT until `maxContextTokens` is hit (drop lowest score first).
6. If zero chunks pass: JSON keyword fallback **or** `unknown` refusal.

I like a light **keyword fallback to JSON** for exact FAQ hits during canary: if
the AI path is down or retrieval is empty, the user still gets the old answer.

---

## Citations (stretch, but I would ship it)

Response includes:

```json
"citations": [
  { "chunkId": "qa-001", "sourceType": "qa_json", "title": "Is Jobbser free for workers?", "score": 0.91 }
]
```

Rules:

- Only cite chunks that were **actually placed** in CONTEXT for that turn.
- UI may show “Based on Jobbser help” + titles; raw scores optional for internal.
- Eval harness checks: for golden JSON questions, expected `chunkId` appears in
  `citations` when the answer is not a refusal.

This is how we avoid “the model said something; trust us.”

---

## PII handling in retrieval

| Stage | Rule |
| --- | --- |
| Ingest | Reject docs that contain live user emails / phone patterns unless redacted |
| Query | Mask emails, long digit runs, and obvious tokens in the user string |
| Prompt | Never join RDS profile/application rows into CONTEXT |
| Logs | Store redacted prompt text for eval; hash `userId`; retention capped (kickoff) |

The assistant answers “how does apply work?”, not “what did I apply to last week?”
Account-specific state stays in normal API screens.

---

## Corpus ownership

- Product owns Q&A content and help Markdown.
- Engineering owns ingest, index, budgets, and the Lambda.
- A change to JSON without re-embed = stale answers. The publish step is mandatory
  in the runbook (even if the runbook is one Make target).

---

## Minimal allowed topics (v1 content checklist)

I would seed help docs to cover at least:

- What Jobbser is (companies ↔ workers)
- Worker: browse, apply, connections, messages, notifications, referrals
- Company: post job, activate listing (pay-per-job), manage applicants
- What is free vs paid (**workers free**)
- Where to find Help / support in-app

Anything outside that list should refuse until a chunk exists.
