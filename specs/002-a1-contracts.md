effort: high
Repo: legal-rag-service (FastAPI). Work on branch refactor/async, created from night/rag AFTER the tag v1-sync exists.
Test command: C:\work\legal-rag-service\.venv\Scripts\python.exe -m pytest -q -p no:cacheprovider
One commit per task "refactor: A<n> <title>". Never push. No network in tests. Never delete or weaken existing tests.
The v1 sync behaviour must keep working until A5 says otherwise. Read ../README.md (design) before starting.

## A1 Contract tests that both versions must pass
- tests/contract/: for every endpoint lock the status codes, JSON keys and types, error codes (RATE_LIMITED,
  CREDIT_INSUFFICIENT, LLM_CALL_FAILED, LLM_INVALID_JSON, SESSION_TENANT_MISMATCH, OPERATOR_ROLE_REQUIRED, ...),
  the domain events emitted (types and order) and the credit ledger rows written (kind, status, credits) for:
  success, guard block, no credits, LLM failure, invalid JSON, cross-tenant access, retry with the same Idempotency-Key (A2).
- Tests must not depend on sync vs async internals (call through HTTP only). Mark the Idempotency-Key test xfail(strict=True)
  until A2 lands.
