# Evaluation: mem0

**Repo:** [mem0ai/mem0](https://github.com/mem0ai/mem0)
**Stars:** 62,549 | **Last updated:** 2026-08-05 | **License:** Apache-2.0
**Last verified:** 2026-10-16
**Last triaged:** 2026-10-16  <!-- triaged: bulk -->
**Dev loop stage:** Reflect
**Layer:** Infrastructure

---

## What it does

Universal memory layer for AI agents. Stores User, Session, and Agent state with entity linking and temporal reasoning. Retrieval fuses semantic search, BM25 keyword search, and entity matching scored in parallel. Available as a Python/Node SDK, self-hosted Docker server, managed cloud platform, or Claude Code plugin (v0.2.10 via `.claude-plugin/marketplace.json` in the repo).

The core differentiator is a research-backed algorithm with published benchmarks: 91.6 LoCoMo, 94.8 LongMemEval, 64.1 BEAM (1M tokens). No other memory tool in the catalog has reproducible retrieval benchmarks. Entity linking connects people, code, and concepts across memories, and temporal reasoning handles "what was X last month?" vs "what is X now?" — critical for long-running projects.

MCP server integration is community-maintained: `coleam00/mcp-mem0` (678 stars) is the most popular, with self-hosted options (`elvismdev/mem0-mcp-selfhosted`) bundling Qdrant + Neo4j + Ollama.

## How we tested it

**Evidence:** MEASURED

**Hands-on, measured.** Installed it with `pip install mem0ai` (v2.2.1) in a clean venv, wired it to a local embedder (`fastembed`, `BAAI/bge-small-en-v1.5`) and an on-disk local Qdrant store, loaded a 30-memory fixture with `add(..., infer=False)`, and scored 24 natural-language queries against a hand-labelled gold memory, against a keyword-overlap baseline. Scope is the **storage and vector-retrieval path only**; the LLM extraction pipeline and the published benchmark algorithm are outside this run (see "What didn't work").

```
$ python run.py     # /tmp/eval-mem0, mem0ai 2.2.1, fastembed bge-small, qdrant local
init_s 0.88
add_total_s 0.57 for 30 (19 ms each)
all n=24 mem0 hit@1 19 hit@3 23 | keyword hit@1 16 hit@3 19
lex n=7 mem0 hit@1 6 hit@3 7 | keyword hit@1 5 hit@3 6
para n=17 mem0 hit@1 13 hit@3 16 | keyword hit@1 11 hit@3 13
search latency ms median 6 max 16
other-user results: 0
dup add: before 30 after 31
after update top: The payments service is written in Go and deployed on Kubernetes in the eu-west-1 region.
after delete count 30
infer=True add FAIL: LLMError LLM extraction failed: Error code: 401 - {'error': {'message': 'Incorrect API key provided: placehol************lled. You can find your API key at htt
```

Second full run reproduced the same hit counts (19/23 vs 16/19); latency median 6 ms, max 9 ms.

## Test design

- **Task/corpus:** 30 short project-knowledge memories (stack choices, policies, owners, SLOs) and 24 queries with one gold memory each: 7 share key words with the gold memory ("lex"), 17 are paraphrases with little word overlap ("para"). Fixture and queries are in the run script, built under `/tmp/eval-mem0/` (not committed).
- **Baseline:** keyword retrieval, Jaccard overlap of lowercased, stop-word-filtered tokens between query and memory (the "without a semantic memory layer" arm, i.e. what `grep`-style recall gives).
- **Metric:** hit@1 and hit@3 (gold memory is the top result / in the top three), search latency per query, plus mechanical behaviour checks (user isolation, delete, duplicate handling).
- **Reproduce:** `python -m venv v && v/bin/pip install mem0ai fastembed`, then build `Memory.from_config` with `embedder=fastembed`, `vector_store=qdrant (path=...)`, an `llm` block holding a throwaway placeholder key (constructor requires one; it is never called with `infer=False`), `add(text, user_id="u1", infer=False)` for each memory, and `search(q, filters={"user_id": "u1"}, top_k=3)` per query.

## What worked

- **Retrieval beat the keyword baseline on this fixture:** hit@1 19/24 vs 16/24, hit@3 23/24 vs 19/24. The gap is in paraphrase queries (hit@3 16/17 vs 13/17); on key-word-sharing queries both do well (7/7 vs 6/7 at hit@3).
- **Fast and fully local at this scale:** about 15-19 ms per `add` and a 6 ms median search with a local embedder and local Qdrant; no server or network once the embedding model is cached.
- **Per-user scoping is real:** a search under another `user_id` returned 0 results; `delete` reduced the stored count 31 to 30 as expected.
- **Config surface is wide:** the embedder/vector-store provider layer accepted a non-default local embedder and local vector store through plain config.

## What didn't work or surprised us

- **The headline pipeline needs an LLM.** `Memory()` with no config fails at construction (`OpenAIError: Missing credentials`), and constructing it with any config still builds an LLM client, so a placeholder key was needed even for `infer=False`. `add()` with default inference failed with `LLMError ... 401 Incorrect API key`. Entity linking, temporal reasoning and dedup are all behind that path, so none of them were exercised here.
- **The published numbers (91.6 LoCoMo, 94.8 LongMemEval) are unverified by this run.** Our result reflects bge-small embeddings plus a vector store, so it says little about mem0's own fusion (semantic + BM25 + entity) algorithm.
- **No dedup without the LLM:** adding the same memory twice with `infer=False` stored both (30 to 31).
- **Update check was confounded** by that duplicate: after `update()` on one copy, the other copy still ranked first, so the update result is inconclusive rather than a bug.
- **Small fixture:** 24 queries over 30 memories is a smoke-scale measurement. The delta is directional, and a different embedder under the baseline harness would likely show a similar gain, so it is weak evidence for mem0 specifically.
- **Argument shape changed:** `search`/`get_all` reject top-level `user_id` in 2.2.1 and require `filters={...}`; older examples (including many community MCP wrappers) pass the old form.

## Quality signals affected

| Signal | Impact | Evidence |
|--------|--------|----------|
| Correctness | + | Measured: hit@1 19/24 and hit@3 23/24 vs keyword baseline 16/24 and 19/24 on a disclosed 24-query fixture (vector path only); published LoCoMo/LongMemEval numbers unverified here |
| Speed | + | Measured: 6 ms median search, 15-19 ms per add with local embedder and local Qdrant |
| Maintainability | neutral | Memory is infrastructure, not code quality; no direct impact |
| Safety | neutral | Apache-2.0, self-hostable; cloud path adds vendor dependency |
| Cost Efficiency | - | Local embedder + local Qdrant ran at zero marginal cost, but the LLM-inference path needs a paid key; cloud platform costs beyond free tier |
| Verifiability | neutral | Retrieval hits are checkable against stored memory text (every result returns the verbatim memory); extracted memories from `infer=True` were not observed, so their faithfulness is unchecked |

## Verdict

**discovery-log — tentative read** — the local retrieval path works and beat keyword recall (hit@3 23/24 vs 19/24, 24-query fixture), but mem0's differentiators (LLM extraction, entity linking, temporal reasoning, dedup, published benchmarks) need an LLM key this run could not use, so the claim that matters is untested. For Claude Code-only workflows claude-mem (ADOPT) stays the better choice; mem0's case is cross-editor memory. A real read needs a keyed run of `infer=True` against a fixed conversation set.

## Triage note

Cites claude-mem (STACK ADOPT) in Overlaps with, but the eval's own REVIEW already draws the line: mem0's cross-platform reach (Codex/Cursor, not just Claude Code) and published retrieval benchmarks are a differentiated job from claude-mem's Claude Code-only auto-capture, not a redundant re-implementation. ★62.5K, Apache-2.0, actively pushed. Left for a real hands-on eval rather than a mechanical SKIP.

## Catalog entry

| Name | Type | One-liner | Problem it solves | Overlaps with |
|------|------|-----------|-------------------|---------------|
| [mem0](https://github.com/mem0ai/mem0) | MCP server | AI memory layer with entity linking, temporal reasoning, and published retrieval benchmarks (58.8K stars) | Need cross-platform memory with relationship-aware retrieval that scales past thousands of entries | claude-mem, OMEGA, SimpleMem |
