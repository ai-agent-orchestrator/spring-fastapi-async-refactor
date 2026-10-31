effort: extra high
Repo: legal-rag-service (FastAPI). Work on branch refactor/async, created from night/rag AFTER the tag v1-sync exists.
Test command: C:\work\legal-rag-service\.venv\Scripts\python.exe -m pytest -q -p no:cacheprovider
One commit per task "refactor: A<n> <title>". Never push. No network in tests. Never delete or weaken existing tests.
The v1 sync behaviour must keep working until A5 says otherwise. Read ../README.md (design) before starting.

## A2 Python async request path
- GuardedLLM.ainvoke_json(...): same contract as invoke_json, but await chat_model.ainvoke inside
  asyncio.timeout(config llm_timeout_s, default 30); reserve before, settle on success, release in finally on ANY other
  outcome including asyncio.CancelledError (re-raise it after releasing). Per-provider asyncio.Semaphore (config, default 8).
- ScreeningService.arun(...): same steps and order as run(); CPU-heavy or sync-only steps go through asyncio.to_thread
  (memory search via memvid, repository calls until A3); guard and masking may stay on the loop (fast). Keep run() working.
- routes: POST /v1/screenings becomes async def and calls arun. Other routes stay as they are in this task.
- Idempotency-Key header (optional): per tenant, store key -> screening id + response; a repeat returns the stored response
  with the same status and does NOT reserve or charge again; a concurrent duplicate waits for the first (lock per key).
- Blocking detector test: run the async route with a fake model and assert no call to time.sleep, requests, or a sync DB
  method happens on the event loop thread (patch and record the calling thread).
- Tests: cancellation releases the reservation; timeout releases and returns the documented error; semaphore caps concurrent
  provider calls; idempotent retry charges once; all A1 contract tests pass (remove the xfail).
