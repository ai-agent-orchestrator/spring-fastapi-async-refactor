effort: high
Repo: legal-rag-service (FastAPI). Work on branch refactor/async, created from night/rag AFTER the tag v1-sync exists.
Test command: C:\work\legal-rag-service\.venv\Scripts\python.exe -m pytest -q -p no:cacheprovider
One commit per task "refactor: A<n> <title>". Never push. No network in tests. Never delete or weaken existing tests.
The v1 sync behaviour must keep working until A5 says otherwise. Read ../README.md (design) before starting.

## A5 Measure again and write it up
- Run benchmarks/load.py against v1-sync (checkout the tag in a temporary worktree) and against refactor/async with the same
  fake latency and the same concurrency levels; fill RESULTS.md rows for v2-async.
- Add a short analysis section to RESULTS.md: where p95 changed and why (thread pool queueing vs event loop), memory and
  thread counts, any errors, and what the semaphores capped. No marketing language; numbers first.
- Do not change application code in this task.
