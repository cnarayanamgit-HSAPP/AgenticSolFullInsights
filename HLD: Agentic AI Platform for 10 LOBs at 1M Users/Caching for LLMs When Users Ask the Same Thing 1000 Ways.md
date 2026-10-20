# Caching for LLMs When Users Ask the Same Thing 1000 Ways

The core problem: exact-match caching fails because "reset my password", "how do I change my pwd?", and "forgot login, help" all hash differently. The solution is a **layered cache**, from cheapest/safest to most flexible.

## Layered architecture

```
Request
  │
  ├─ L0: Normalize (lowercase, trim, strip punctuation, expand abbreviations)
  │
  ├─ L1: Exact-match cache (hash of normalized prompt + params) ── hit → return
  │
  ├─ L2: Semantic cache (embedding similarity) ───────────────── hit → return
  │
  ├─ L3: Provider prompt caching (shared prefix: system prompt, docs)
  │
  └─ L4: LLM call → write back to L1/L2
```

## 1. Exact cache (L1)
- Key = `hash(model + system_prompt_version + normalized_query + temperature + tools)`
- Store in Redis with TTL.
- Cheap, zero risk of wrong answers. Catches repeated and copy-pasted queries.

## 2. Semantic cache (L2), where the 1000 variations get collapsed
1. Embed the incoming query (small model, e.g. 384-768 dims).
2. Nearest-neighbor search in a vector store (Redis Vector, pgvector, Qdrant, FAISS).
3. If cosine similarity ≥ threshold (start ~0.92-0.95), return the cached answer.
4. On miss, call the LLM, then store `(embedding, query, response, metadata)`.

**Improvements that matter in practice:**
- **Intent canonicalization:** use a cheap LLM or classifier to rewrite the query into a canonical form ("user wants to reset password"), then embed and cache that. This is much more robust than raw-text embeddings.
- **Cluster instead of storing every variant:** keep one entry per intent cluster, with variants pointing to it, so the index doesn't bloat.
- **Two-stage verification:** vector search returns top-k, then a cheap reranker or small LLM confirms "do these two questions have the same intent?" for borderline scores (0.85-0.95).
- **Tune the threshold per domain.** Too low gives wrong answers served confidently. Too high gives a poor hit rate. Evaluate on labeled paraphrase pairs.

## 3. Provider-side prompt caching (L3)
Doesn't help with varying questions directly, but reduces cost of the shared part: put the static system prompt, instructions, and retrieved docs **first**, and the variable user query **last**, so the prefix gets cached by the provider.

## 4. What must be part of the cache key (avoid wrong hits)
- Model and version
- System prompt / prompt template version
- Tenant/user scope if answers are personalized or permission-dependent
- Language/locale
- Retrieved-document version (for RAG)
- Tool availability

**Never share cache across users when the answer depends on private data.** Either skip caching those queries or scope the key per user/tenant.

## 5. What to cache vs. skip
| Cache | Don't cache (or scope tightly) |
|---|---|
| FAQ-type, factual, policy answers | Personalized or account-specific answers |
| Deterministic tasks (classification, extraction) | Time-sensitive info ("today's price") |
| Stable RAG answers | Creative requests wanting variety |
| | Multi-turn context-dependent queries |

For multi-turn chats, cache on the **standalone rewritten question** (after resolving "what about that one?"), not the raw last message.

## 6. Freshness and invalidation
- TTL by content type (hours for volatile, days/weeks for stable).
- Event-based invalidation: when a source doc or policy changes, purge entries tagged with that doc ID.
- Version tags on prompts and models so deploys don't serve stale-format answers.
- Optional **stale-while-revalidate**: serve the cached answer, refresh in the background.

## 7. Safety and quality
- Don't cache responses that were flagged, refused, errored, or low-confidence.
- Add user feedback (thumbs down) to evict bad entries.
- Run shadow evaluation: periodically compare cached answers to fresh LLM answers for sampled hits to measure drift and false-hit rate.
- Log hit type (exact, semantic, miss), similarity score, and latency.

## 8. Metrics to track
- Hit rate (exact vs. semantic)
- **False-positive rate** (the critical one for semantic caches)
- Cost and latency saved
- Cache size and eviction rate
- Answer quality delta on cached vs. fresh

## 9. Recommended starting stack
- **Store:** Redis (exact + vector) or Postgres + pgvector
- **Embeddings:** a small sentence-embedding model, self-hosted for low latency
- **Libraries:** GPTCache, LangChain/LlamaIndex caches, or a custom thin layer
- **Rollout:** start with exact + normalization, add semantic cache in shadow mode (log would-be hits without serving), tune the threshold, then enable.

## Quick pseudo-code

```python
def answer(query, ctx):
    q = normalize(query)
    key = make_key(q, ctx)            # model, prompt ver, tenant, locale
    if hit := redis.get(key):
        return hit

    emb = embed(canonicalize(q))
    cands = vector_search(emb, filter=ctx.scope, k=3)
    if cands and cands[0].score >= HIGH:
        return cands[0].response
    if cands and cands[0].score >= LOW and same_intent(q, cands[0].query):
        return cands[0].response

    resp = llm(q, ctx)
    if cacheable(resp, ctx):
        redis.set(key, resp, ttl=TTL)
        vector_store.add(emb, q, resp, tags=ctx.doc_ids, ttl=TTL)
    return resp
```

**Bottom line:** normalize, then exact match, then semantic match with a verification step, all scoped correctly, with solid invalidation. The semantic layer gives the big wins, but the false-positive rate is what you must measure and control.

If you share your use case (support bot, RAG, internal tool), traffic volume, and whether answers are personalized, I can tailor thresholds and the architecture.
