# Task 6: Replace in-app Q&A with a real AI integration on AWS

**Name:** Roshan Joshi  
**Task:** 6 (AI · AWS)  
**Time spent:** ~3 hours

Design / thinking only. No live AWS account, no real Bedrock keys, no deploy against
Jobbser Prod. Workers stay free. This assistant answers product questions; it does
not invent billing for workers.

---

## What I am replacing

Jobbser already ships an in-app Q&A chatbot backed by a simple **JSON** knowledge
form. That is honest and cheap: exact answers for exact-ish questions. It falls
over on paraphrase, follow-ups, and anything outside the file.

I am not proposing “wire ChatGPT to the website.” I want a clear upgrade path:
keep the JSON as a **curated corpus**, put a **guarded retrieval + Bedrock** path
in front of it, enforce **token budgets** and **Jobbser-only scope**, and prove
the new path still matches (or intentionally improves) the old answers.

I do not have the live JSON file in the materials I was given. I treat it as a
flat list of `{id, question, answer, tags?}` entries owned by product, and I
design the migration around that shape. Kickoff question #1 is “send me the
current JSON.”

---

## What I read first

- Intern brief Task 6
- AWS inventory (ECS API/MsgHub, ALB, RDS, S3, SES, Secrets Manager, eu-west-1)
- Skim of the DB schema pack (for what **not** to feed the model: PII in profiles,
  applications, messages)

Inventory shape that matters here:

- Clients already talk HTTPS REST to **Jobbser-Api** on ECS via the ALB
- Runtime config lives in **Secrets Manager**
- **S3** already exists for assets; a second corpus bucket is a natural fit
- Lambdas / Bedrock are called out as growth areas, so I am not forcing AI into
  the always-on Fargate task

---

## Architecture (short)

```
App → ALB → Jobbser-Api (ECS)
               │  POST /v1/assistant/chat  (authn + rate limit + feature flag)
               ▼
         Lambda: jobbser-qa-turn
               │  1) sanitize / classify
               │  2) retrieve top-k chunks from allowed corpus
               │  3) Bedrock InvokeModel (capped)
               │  4) return answer + citations + usage
               ▼
     S3 corpus + embeddings index (OpenSearch Serverless or S3 vector sketch)
     Secrets Manager: model id, budgets, feature flags
```

Why this split:

1. **API stays the front door.** Auth, company/worker identity, rate limits, and
   the existing mobile contract live on Jobbser-Api. The app does not call Bedrock.
2. **Lambda owns the AI turn.** Burst-friendly, no extra always-on Fargate cost
   for a help chatbot. Inventory already warns that always-on Dev hurt.
3. **Bedrock in-region** (`eu-west-1` if the model is available there; otherwise a
   documented cross-region invoke with a privacy note). No third-party ChatGPT
   account hanging off the website.
4. **Retrieval over a closed corpus**, not “search the open web.” Migrated JSON
   Q&A plus a small set of allowed Jobbser product docs.

Longer diagram and sequence: [`docs/architecture.md`](docs/architecture.md).  
Optional endpoint sketch: [`openapi/assistant-chat.yaml`](openapi/assistant-chat.yaml).

---

## Migration plan (JSON → corpus, then cut over)

I would not flip a switch on day one. I would keep the JSON answers as the
**regression oracle** while the AI path earns trust.

| Phase | What happens | App behaviour |
| --- | --- | --- |
| 0 | Export current JSON to versioned S3 (`corpus/qa/*.json`) with stable chunk IDs | JSON bot unchanged |
| 1 | Chunk + embed; build retrieval index; wire Lambda behind a **feature flag** | Flag off → JSON path |
| 2 | Shadow mode: AI answers logged, JSON still shown (or AI shown to internal only) | Compare offline |
| 3 | Canary: AI for a % of sessions when retrieval confidence is high; else JSON fallback | Hybrid |
| 4 | Default AI; JSON remains fallback when retrieval score is low or Bedrock errors | AI primary |

Replace does not mean delete. The JSON becomes the **seed corpus** and the
**golden regression set**. Product still edits Q&A as structured content; a small
ingest job re-embeds on change.

Details: [`docs/retrieval.md`](docs/retrieval.md).

---

## System prompt + retrieval scope

**Allowed knowledge:**

- Migrated in-app Q&A JSON (canonical product answers)
- Curated Jobbser help / product docs we deliberately publish into the corpus
  (how jobs work, company vs worker roles, applications, connections, referrals,
  notifications, pay-per-job listings for companies)
- Short “platform facts” sheet we own (what Jobbser is / is not)

**Not allowed:**

- Open web, competitor sites, general career advice, coding help, medical/legal
- RDS rows (profiles, messages, applications, emails, tokens)
- Secrets, Stripe keys, internal runbooks not marked “assistant-safe”
- Anything that would bill or pressure **workers**

System prompt + refusal rules: [`docs/prompts.md`](docs/prompts.md).

---

## Token budget (hard caps)

Per turn, enforced in Lambda before Bedrock:

| Bucket | Cap (approx tokens) | Notes |
| --- | --- | --- |
| System prompt | 450 | Fixed template |
| Retrieved context | 1,200 | Top-k chunks, truncated |
| User message | 400 | Truncate / reject oversize |
| Model response | 350 | `max_tokens` on invoke |
| **Turn total** | **~2,400** | Hard stop; no multi-doc dump |

If retrieval would blow the context cap, drop lowest-scoring chunks first. If the
user message is over cap after sanitize, refuse with a short “please shorten”
message and **do not** call Bedrock.

Full policy: [`docs/token-budget.md`](docs/token-budget.md).

---

## Samples (in-scope and refused)

Five worked examples in [`docs/samples.md`](docs/samples.md):

- In-scope: what Jobbser is, how a company posts a job, worker apply flow, referrals
- Refused: write my CV, medical advice, competitor pricing, dump another user’s data

Stretch eval set (~10 prompts): [`docs/evals.md`](docs/evals.md).

---

## Extras I added

- Eval set with expected `answer` / `refuse` labels
- Cost per 1k queries sketch ([`docs/cost.md`](docs/cost.md))
- Cite which chunk answered (return `citations[]` with chunk ids)
- PII handling: strip/redact before the model; never inject RDS profile into the prompt

---

## Testing approach

Required. Short version:

1. **Regression vs JSON:** for each golden Q&A, AI answer must cover the same
   facts (embedding or rubric similarity), or explicitly improve with a reviewed
   note. Citations must point at the JSON-derived chunk when that is the source.
2. **Refusal:** off-topic and PII-seeking prompts must return the refusal shape
   with **no** Bedrock call when the classifier is confident, or a refusal
   completion that cites no product chunks.
3. **Token budget:** unit tests on the budgeter; integration fixture that a
   10k-token paste never reaches Bedrock; response length never exceeds cap.
4. **Fallback:** Bedrock timeout / empty retrieval → JSON answer or safe “I only
   know Jobbser help” message.

Longer plan: [`docs/testing-approach.md`](docs/testing-approach.md).

---

## What I would ask in a kickoff

1. Can I have the **current Q&A JSON** (or a sanitized sample) and who owns edits?
2. Is Bedrock already enabled in the Jobbser AWS org / `eu-west-1`, and which
   model family is preferred (Claude Haiku-class vs Titan)?
3. Should the assistant be available to **workers and companies**, or companies only
   at first?
4. Rate limit: per user? per device? daily cap?
5. Do we already have help-centre Markdown, or is JSON the only corpus today?
6. Logging: may we store prompts/answers for eval, and for how long, given GDPR?
7. Feature-flag home: API config secret, or a dedicated flag service?
8. Is OpenSearch Serverless acceptable, or do you want the cheaper S3 + in-memory /
   FAISS-style sketch for v1?

---

## Assumptions (where materials were silent)

- Current bot is client- or API-served JSON Q&A; exact file path unknown.
- Region target remains `eu-west-1`; Bedrock model availability confirmed in kickoff.
- Jobbser-Api can invoke Lambda synchronously (AWS SDK) or via an internal ALB
  route; I sketch SDK invoke from the API task role.
- Corpus is English-first; other languages are out of scope for v1.
- Workers are never billed; the assistant never suggests worker payments.
- Design files only; no real keys in this repo.

---

## Layout of this package

```
README.md                      # this write-up
docs/
  architecture.md              # diagrams + component choices
  prompts.md                   # system prompt + refusal rules
  retrieval.md                 # corpus, chunking, citations
  token-budget.md              # caps and enforcement
  samples.md                   # in-scope + refused Q&A
  evals.md                     # stretch ~10 prompt set
  testing-approach.md          # regression, refusal, budgets
  cost.md                      # stretch cost per 1k queries
openapi/
  assistant-chat.yaml          # language-agnostic chat turn sketch
```

Design only.
