effort: extra high
Repo: legal-rag-service (FastAPI). Work on branch refactor/async, created from night/rag AFTER the tag v1-sync exists.
Test command: C:\work\legal-rag-service\.venv\Scripts\python.exe -m pytest -q -p no:cacheprovider
One commit per task "refactor: A<n> <title>". Never push. No network in tests. Never delete or weaken existing tests.
The v1 sync behaviour must keep working until A5 says otherwise. Read ../README.md (design) before starting.

## A3 Async storage port
- app/storage_async.py: an AsyncDatabase interface mirroring the repository methods the async path uses.
  Adapter 1: SQLite via a single writer thread (asyncio.to_thread with the existing lock) - the default and the test adapter.
  Adapter 2: PostgreSQL via asyncpg (optional dependency, extras group "postgres" in pyproject), same tenant_id-first rules,
  same whitelist checks, transactions for reserve/settle/release.
- Conformance test suite run against SQLite always and against PostgreSQL only when TEST_PG_DSN is set (skip otherwise).
- arun() uses the async port; run() keeps the sync storage. No behaviour change visible in A1 contract tests.
