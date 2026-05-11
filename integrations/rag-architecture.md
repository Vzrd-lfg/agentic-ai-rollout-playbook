# RAG Architecture Patterns

Retrieval-augmented generation is the primary mechanism for grounding agent responses in enterprise knowledge. The choice of RAG pattern is the largest single determinant of agent quality on knowledge tasks — larger than the choice of frontier model — because a model cannot reason well from context it never received. The playbook documents ten RAG patterns, from the baseline to the most sophisticated, with guidance on when each is appropriate. The principle throughout is to use the lowest-complexity pattern that achieves the required faithfulness score; complexity adds latency, cost, and operational burden at every step up the stack.

## Pattern Comparison Table

| Pattern | Best For | Retrieval Type | Complexity | Relative Cost |
|---|---|---|---|---|
| Naive RAG | Proof of concept, stable corpora | Dense vector | Low | Low |
| Hybrid RAG | Production standard, keyword-important domains | Dense + sparse (BM25) | Medium | Medium |
| Contextual RAG | Long documents, chunk coherence | Dense with context prefix | Medium | Medium |
| HyDE | Queries with no lexical overlap to corpus | Dense (hypothetical doc) | Medium | Medium |
| Graph RAG | Relationship-rich corpora, entity-centric queries | Graph traversal + dense | High | High |
| Adaptive RAG | Mixed query complexity (simple + complex) | Routed (direct/RAG/agentic) | High | Variable |
| Self-RAG | Quality-critical responses, hallucination-sensitive | Dense + self-critique | High | High |
| Modular RAG | Large teams, evolving corpora, experimentation | Composable modules | High | Variable |
| Agentic RAG | Multi-source, multi-step research tasks | Multi-tool retrieval | Very High | Very High |
| Agentic Graph RAG | Strategic relationship exploration | Graph + multi-step agent | Very High | Very High |

## Pattern Descriptions

### Naive RAG

The foundational pattern: embed the query, retrieve the top-k chunks by cosine similarity, and pass them to the model as context. It deploys in hours and materially reduces hallucination on small, well-structured corpora with consistent vocabulary. It fails when queries use terminology that differs from the corpus — the semantic gap between question and document is too wide for a single dense embedding to bridge — and when documents span multiple topics such that a retrieved chunk loses meaning without its surrounding paragraphs.

### Hybrid RAG

Hybrid RAG combines dense vector retrieval with BM25 keyword search; results from both systems are merged via reciprocal rank fusion or a learned cross-encoder re-ranker. This is the production standard for most enterprise corpora. Dense retrieval handles natural language paraphrase and semantic similarity; sparse retrieval handles exact matches on product codes, proper nouns, regulatory identifiers, and any other token the corpus uses in a specific, non-paraphraseable way. Together they close the majority of the failure modes that either system exhibits alone.

### Contextual RAG

Before indexing, a context summary — capturing the document's topic, section heading, and positional metadata — is prepended to each chunk. The enriched chunk is then embedded, so the chunk's vector representation carries document-level context alongside the chunk's own content. This eliminates the "orphaned chunk" failure mode: a passage that refers to "the approach described above" or "this regulation" retrieves coherently even when the surrounding paragraphs are not in the same chunk. The cost is a more expensive indexing pipeline and additional storage per chunk.

### HyDE (Hypothetical Document Embeddings)

Rather than embedding the user's query directly, the model first generates a hypothetical document that would answer the query — hallucinations in this draft are acceptable and expected — and that hypothetical document is used as the retrieval query. This moves the search from "question space" into "answer space," dramatically reducing the lexical gap between the query and the documents that contain the answer. Every retrieval incurs one additional model call; the latency and token cost must be justified by a measurable improvement in retrieval hit rate on eval data before HyDE is deployed to production.

### Graph RAG

At indexing time, entities, relationships, and factual claims are extracted from the corpus to construct a knowledge graph; community detection algorithms produce multi-level summaries of related concept clusters. At retrieval time, queries traverse entity relationships as well as similarity, enabling answers to questions about connections, implications, and chains of causation that chunk-level search cannot construct. Graph RAG excels at entity-centric and multi-hop queries. The operational overhead is significant: graph construction is expensive, incremental updates require careful design, and the graph must be maintained as the corpus evolves.

### Adaptive RAG

A lightweight query classifier assigns each incoming query to one of three retrieval tiers: a direct answer (no retrieval needed), standard RAG, or multi-step agentic retrieval. Simple factual lookups resolve in milliseconds without touching the index; complex analytical queries receive the full agentic treatment. This eliminates the waste of running expensive retrieval on queries that do not need it, without sacrificing quality on the queries that do. The classifier itself must be maintained, and misclassification — routing a complex query to a simple path — produces confident but incomplete answers that are harder to detect than outright retrieval failures.

### Self-RAG

The model uses learned reflection tokens to decide, at generation time, whether to retrieve additional context, whether retrieved passages are relevant to the current generation step, and whether the generated response is supported by the retrieved evidence before it is delivered. This produces active, iterative self-correction rather than a single retrieve-then-generate pass. The result is the highest faithfulness at the highest cost: the pattern requires a model fine-tuned with reflection token supervision and accepts multi-pass latency in exchange for calibrated, evidence-grounded responses. It is the appropriate choice in domains — medical, legal, financial — where the cost of a confident hallucination outweighs the cost of latency and inference.

### Modular RAG

The RAG pipeline is decomposed into interchangeable modules — retriever, re-ranker, generator, evaluator — each with a defined interface that can be swapped independently of the others. An embedding model upgrade, a new re-ranker, or a switch of the generation model is a module replacement, not an architectural rewrite. This structure enables A/B testing of individual components on production traffic, team-level ownership of each module, and incremental improvement without full pipeline regression. Modular RAG is the appropriate chassis for a platform team building a shared RAG layer that multiple agents consume with different retrieval requirements.

### Agentic RAG

The agent determines which retrieval tools to call, in what order, and whether to reformulate the query based on intermediate results. There is no predefined retrieval path: the agent plans a research strategy, executes retrieval steps, evaluates intermediate context, and decides whether to continue, pivot, or synthesise. This supports multi-source, multi-hop tasks — cross-referencing a policy document against a database against a ticket history — that no fixed retrieval pipeline can handle. The cost is unpredictable latency and token spend; Agentic RAG is appropriate for research-style tasks where the quality of the answer justifies the cost, not for high-volume, latency-sensitive queries.

### Agentic Graph RAG

The most capable and the most expensive pattern. An agent combines the strategic planning of Agentic RAG with knowledge graph traversal: it formulates an exploration strategy, decides which graph paths to follow, retrieves associated documents, evaluates intermediate findings, and backtracks or branches as needed. It can build a comprehensive answer by following relationship chains that no static query would have anticipated at the time of indexing. This pattern is appropriate for complex investigative tasks — financial fraud analysis, sanctions tracing, strategic relationship mapping — where the cost of missing a connection is greater than the cost of exhaustive exploration.

## Decision Tree

```
Is the corpus well-structured and small (<100k docs)?
  └─ Yes → Naive RAG (for PoC); Hybrid RAG (for production)
  └─ No
      Are relationship queries important ("who, what connects to what")?
        └─ Yes → Graph RAG or Agentic Graph RAG
        └─ No
            Is query vocabulary mismatched to corpus vocabulary?
              └─ Yes → HyDE or Contextual RAG
              └─ No
                  Is hallucination risk unacceptable?
                    └─ Yes → Self-RAG
                    └─ No
                        Is the query complexity mixed (simple + research)?
                          └─ Yes → Adaptive RAG
                          └─ No → Hybrid RAG (default)
```

## Moving Up the Stack

Each step up the complexity stack adds latency, cost, and operational burden. HyDE adds one model call per retrieval. Graph RAG adds an indexing pipeline and a graph store. Agentic RAG adds unbounded planning steps. The principle is to start at the lowest-complexity pattern that achieves the required faithfulness score on the eval harness, and move up only when eval data shows a measurable quality gap that the current pattern cannot close — not because a more sophisticated pattern is available, or because the corpus is large, or because the use case sounds complex. Complexity is a cost that must be justified by measured quality improvement.
