konduit — small RAG (Retrieval-Augmented Generation) prototype

What this repo contains
- A minimal, easy-to-run RAG pipeline: crawl -> clean -> chunk -> index -> retrieve -> answer.
- Simple, explainable components (pure-Python TF‑IDF retriever, deterministic grounded answers).

Why this design
- Prioritize reproducibility and low dependency burden. No heavy ML packages required to run the default flow.
- Deterministic grounding prevents hallucination: answers are concatenated snippets from retrieved pages.
- Configurable refusal policy for safety (see `config/refusal.json`).

Quick start (Windows PowerShell)
1) Create a venv and install dependencies
```powershell
python -m venv .venv
.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip wheel
pip install -r requirements.txt
```

2) Run demo flow (crawl python.org, index, two asks)
```powershell
.venv\Scripts\python .\src\run_demo.py
```
Outputs: `demo_crawl.json`, `demo_index.json`, `demo_ask1.json`, `demo_ask2.json`.

3) Run the API server
```powershell
uvicorn src.app:app --host 127.0.0.1 --port 8000
```

4) Example API calls (PowerShell)
```powershell
#crawl
$payload = @{ start_url='https://www.python.org'; max_pages=30; max_depth=2; crawl_delay_ms=500 } | ConvertTo-Json
$payload | .venv\Scripts\python .\src\cli.py crawl

#index
$payload = @{ chunk_size=800; chunk_overlap=100 } | ConvertTo-Json
$payload | .venv\Scripts\python .\src\cli.py index

#ask
$payload = @{ question='When was Python first released?'; top_k=5 } | ConvertTo-Json
$payload | .venv\Scripts\python .\src\cli.py ask
```

Files of interest
- `src/crawler.py` — polite crawler: robots.txt, registrable-domain limit, crawl delay, output: `data/pages.json`.
- `src/indexer.py` — chunking and pure-Python TF‑IDF vector construction. Persists `data/meta.json`.
- `src/retriever.py` — cosine-similarity retrieval over sparse TF‑IDF vectors.
- `src/generator.py` — deterministic grounding + configurable refusal (`config/refusal.json`).
- `src/app.py`, `src/cli.py` — API and CLI wrappers.
- `src/run_demo.py` — demo flow used for sample outputs.
- `src/metrics.py` — metrics/recall runner, p50/p95 and recall@5 (supports live HTTP mode and eval files).
- `tests/` — small unit tests for chunking and refusal.

Design notes and tradeoffs (short bullets)
- Chunking: default 800 characters with 100 overlap. Rationale: simple, captures multiple sentences, overlap preserves cross-chunk facts. (Good default for small sites.)
	- Recommendation: default chunk size 500–1000 chars with small overlap (50–150). 800/100 is a good starting point; reduce chunk_size if snippets are noisy, increase if you want longer context per vector.
	- Embeddings: the repo defaults to TF‑IDF for portability. To use higher-quality OSS embeddings, install `sentence-transformers` and update `src/embeddings.py` to return dense vectors. You can then swap the indexer to use those vectors and a vector store (FAISS) for faster nearest-neighbor retrieval.
- Embeddings: default is pure-Python TF‑IDF to avoid heavy dependencies. For higher-quality retrieval, swap in sentence-transformers + FAISS (optional, extra deps).
- Grounding & safety: generator uses only retrieved text; personal/private questions are refused by policy in `config/refusal.json`.
- Observability: timings returned for retrieval and generation; `src/metrics.py` computes p50/p95 and recall@k from demo queries.
- Scope: registry-domain check is a simple heuristic (last two labels). For strict public-suffix correctness, add `publicsuffix2`.

Tooling & prompts
- Crawler: `requests`, `beautifulsoup4`, `robotexclusionrulesparser`.
- Retriever/index: pure-Python TF‑IDF (tokenize with regex, sparse dicts).
- Serving: `fastapi`, `uvicorn`.
- Testing: `pytest`.

Sample request/response (short)
- Crawl (CLI) → returns: `{ "page_count": 12, "skipped_count": 3, "urls": [...] }`
- Index (CLI) → returns: `{ "vector_count": 120, "errors": [] }`
- Ask (CLI) → returns:
```
{
	"answer": "From https://docs.python.org: Python was created in the early 1990s...",
	"sources": [{"url": "https://docs.python.org", "snippet": "Python was created in the early 1990s..."}],
	"timings": { "retrieval_ms": 24, "generation_ms": 0, "total_ms": 24 }
}
```

Refusal example
```
{ "answer": "not found in crawled content", "sources": [ ... ], "timings": { ... } }
```

Quick next steps (if you want more)
- Add sentence-transformers + FAISS for higher-quality embeddings (I can add optional backend and update `requirements.txt`).
- Replace naive registrable-domain detection with `publicsuffix2` for correct scoping across TLDs.
- Add more eval queries and produce recall@1/5/10 breakdowns.

License & attribution
- This code is a small educational prototype. It uses widely-available open-source libraries; see `requirements.txt` for versions.

Contact
- If you want changes, tell me which next step (FAISS/embeddings, publicsuffix, or extended evals) and I will implement it.
