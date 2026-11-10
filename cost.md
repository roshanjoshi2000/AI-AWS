# Cost per 1k queries (stretch)

Order-of-magnitude only. Prices move; the point is to show the AI path is
**budgeted and measurable**, not an open-ended third-party chat box on the website.

Assumptions (stated so they can be corrected in kickoff):

- Region: `eu-west-1` (or Bedrock region we document)
- Model: small chat model (Haiku-class) + Titan-class embeddings
- Caps from [`token-budget.md`](token-budget.md): ~2,050 input tokens ceiling,
  350 max out; real FAQ turns will average lower
- Average turn used for math: **900 input tokens** (system + context + user),
  **180 output tokens**, **1 embedding of ~50 tokens** for the query
- Corpus embed is a batch cost (amortised), not per query
- Lambda: 512 MB, ~1.5 s billed per turn
- OpenSearch Serverless optional; S3 sketch ≈ negligible request cost at FAQ scale
- 30% of turns refuse **without** Bedrock (rule gate) in steady state

---

## Per 1,000 end-user questions

| Line | Sketch | Notes |
| --- | --- | --- |
| Bedrock chat | 700 invokes × (900 in + 180 out) | 300 refuses skip model |
| Bedrock embeddings | 700 query embeds × ~50 tokens | JSON fallback path may skip |
| Lambda | 1,000 invokes × 1.5 s × 0.5 GB | All turns hit Lambda or API JSON only |
| S3 GET | ~1,000 small GETs if index loaded cold; warmer reuse helps | Tiny vs model |
| OpenSearch Serverless | OCU-hours if chosen | Dominates if left oversized |
| Data transfer / logs | minor | Cap log payload size |

I am **not** pasting a fake dollar total that pretends to be a quote. In a real
kickoff I would plug current Bedrock token rates into a spreadsheet with the
token caps above and publish “$X / 1k at assumed mix.”

### Worked placeholder arithmetic (replace rates at kickoff)

Let:

- `C_in`, `C_out` = $ per 1M input / output tokens for the chat model  
- `C_emb` = $ per 1M embedding tokens  
- `C_lambda` = Lambda GB-s price  

Then:

```
chat_cost_1k   ≈ 700 * (900 * C_in + 180 * C_out) / 1e6
embed_cost_1k  ≈ 700 * 50 * C_emb / 1e6
lambda_cost_1k ≈ 1000 * (0.5 GB * 1.5 s) * C_lambda
total_1k       ≈ chat + embed + lambda + (AOSS monthly / queries_per_month)
```

With tight caps, **chat tokens dominate**. That is why CONTEXT is 1,200 not “whole
site,” and why rule-based refusal skips Bedrock.

---

## Cost controls built into the design

1. Hard token caps (see token-budget doc)
2. `minScore` + topK so we do not stuff weak chunks
3. Pre-model refusal for obvious off-topic / PII
4. Feature flag / canary percent
5. JSON fallback instead of retry storms
6. Small model for FAQ; reserve larger models for later only if evals demand
7. No always-on extra ECS service for chat (inventory cost lesson)

---

## Monthly mental model

Help traffic for an early-stage app is usually modest vs ALB/NAT/Fargate lines in
the inventory. The risk is not “1k FAQ queries”; it is **uncapped prompts** or
**embedding the wrong corpus** (e.g. dumping RDS text). This design refuses both.

Workers are never charged for asking the assistant.
