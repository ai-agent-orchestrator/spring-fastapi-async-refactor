effort: high
Repo: legal-rag-service (FastAPI). Work on branch refactor/async, created from night/rag AFTER the tag v1-sync exists.
Test command: C:\work\legal-rag-service\.venv\Scripts\python.exe -m pytest -q -p no:cacheprovider
One commit per task "refactor: A<n> <title>". Never push. No network in tests. Never delete or weaken existing tests.
The v1 sync behaviour must keep working until A5 says otherwise. Read ../README.md (design) before starting.

## A0 Baseline benchmark harness (before any refactor)
- benchmarks/fake_llm.py: a LangChain-compatible chat model with a fixed latency (default 2.0 s, env BENCH_LLM_LATENCY):
  invoke() uses time.sleep, ainvoke() uses asyncio.sleep; returns the fixed screening JSON and usage_metadata.
- benchmarks/serve.py: starts the app with uvicorn on 127.0.0.1:<free port> in a subprocess with the fake model injected via
  deps.reset(...), a fresh SQLite file, credits topped up for a bench tenant, rate limiter disabled for the bench tenant only.
- benchmarks/load.py: async load generator (httpx.AsyncClient) at concurrency levels 10, 50, 200 for N requests each; signs
  tokens with issue_internal_token; records p50, p95, max latency, requests/s, error count, and the server process peak RSS
  and thread count (psutil if installed, else "n/a" - do not add a hard dependency).
- benchmarks/RESULTS.md: table with a row per (version, concurrency). Run it now and fill the v1-sync rows.
- Tests: fake model latency honoured in both paths; load.py on a tiny run (concurrency 2, 4 requests, latency 0.05) produces
  a well-formed results row. Keep the real run out of the unit test suite.
