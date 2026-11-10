# Token budget

Token efficiency matters for cost and latency. Caps live in config and are
enforced in Lambda **before** Bedrock, not as a polite request in the system prompt.

---

## Caps (v1 defaults)

| Bucket | Cap (tokens) | Enforced how |
| --- | --- | --- |
| System prompt template | 450 | Template lint in CI; fail deploy if over |
| Retrieved CONTEXT | 1,200 | Pack top-k; drop lowest score when over |
| User message | 400 | Truncate to cap **or** refuse `shorten` if still huge after sanitize |
| Optional prior turns | 200 | Include at most 2 short turns; drop oldest first |
| Model response | 350 | `max_tokens` / equivalent on `InvokeModel` |
| **Per-turn ceiling** | **~2,400** | Sum of inputs + max response; metric if exceeded |

Tokenizer: use the model family’s counter where available; for budget gates a
shared approximate (`chars/4`) is acceptable if we add a 10% safety margin.
Document which one we use in kickoff.

---

## Assembly order

```
system (fixed)
+ context chunks (highest score first, stop at maxContextTokens)
+ optional history (budgeted remnant)
+ user (redacted, capped)
```

Never: dump entire FAQ JSON into the prompt. That is the anti-pattern this design
exists to avoid.

---

## Pseudo-code

```
function buildPrompt(userText, chunks, history):
  system = SYSTEM_TEMPLATE  # assert tokenCount(system) <= maxSystemTokens

  ctx = []
  used = 0
  for chunk in sortByScoreDesc(chunks):
    t = formatChunk(chunk)
    n = tokenCount(t)
    if used + n > maxContextTokens: continue
    ctx.append(t)
    used += n

  if ctx is empty: return FallbackOrRefuseUnknown

  user = redact(userText)
  if tokenCount(user) > maxUserTokens:
    return RefuseShorten

  hist = packHistory(history, maxHistoryTokens)

  return Prompt(system, ctx, hist, user)
```

```
function invoke(prompt):
  return bedrock.invoke(
    modelId,
    prompt,
    maxTokens = maxResponseTokens
  )
```

---

## When we skip Bedrock entirely

- User over budget after policy → `shorten`
- Clear PII / off-topic rule hit → `refuse_*` with `bedrockInvoked: false`
- Feature flag off → JSON path in API (no Lambda AI path)

Skipping the model is the cheapest correct answer.

---

## Observability

Emit per turn (CloudWatch metrics / structured log):

- `prompt_tokens_est`, `context_tokens`, `user_tokens`, `response_tokens`
- `chunks_considered`, `chunks_used`, `min_score`
- `bedrock_invoked` (bool)
- `fallback_json` (bool)
- `refuse_code` (nullable)

Alarm ideas: p95 `prompt_tokens_est` creeping toward ceiling; spike in
`bedrock_invoked=false` refusals; unexpected jump in response length.

---

## Tuning knobs

If evals show incomplete answers: raise `maxContextTokens` slightly **or** improve
chunking, not the response cap first.

If cost rises: lower `topK`, raise `minScore`, shorten system boilerplate, prefer
Haiku-class / smaller model for FAQ.

Workers remain free; budgets protect **our** bill and latency, not a user wallet.
