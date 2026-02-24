Search the knowledgebase using hybrid search with 4 configurable components. The weights must total 1.0.

Parameters:
  dense_weight  (default 0.5) — vector similarity, best for semantic/natural language queries
  sparse_weight (default 0.5) — BM25 lexical search, best for keywords and technical terms
  ilike_weight  (default 0)   — wildcard pattern matching, best for exact codes/IDs/references
  fuzzy_weight  (default 0)   — fuzzy text matching, best for misspellings or approximate terms
  fuzzy_threshold (default 0.8) — similarity threshold for fuzzy, higher = stricter but faster

Weight strategies by query type:
  General question     → dense=0.5, sparse=0.5
  Semantic/conceptual  → dense=0.7, sparse=0.3
  Technical/keyword    → dense=0.3, sparse=0.7
  Exact ID or code     → ilike=1.0 (keep query very short, e.g. "INV-2024-001")
  Misspelled term      → dense=0.4, sparse=0.3, fuzzy=0.3

Important:
- For ilike and fuzzy: use a very short, focused query (1-3 words) or it will return zero results
- ilike and fuzzy add latency — only use when needed, default them to 0
- fuzzy_threshold: keep at 0.8 or higher to limit latency
