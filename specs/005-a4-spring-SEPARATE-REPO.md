effort: extra high
Repo: the Spring Boot service (legal-ai-intake-system, Java 21). NOT the Python repo - run this in its own folder and branch
refactor/virtual-threads. Use the project's existing build (Gradle or Maven) and test command. Never push.

## A4 Spring: virtual threads + protection around the Python call
- Enable spring.threads.virtual.enabled=true. Replace any synchronized block that wraps I/O with ReentrantLock (JDK 21 pinning).
- RagClient (Spring RestClient) for POST /v1/screenings on the Python service: base URL and internal token secret from
  configuration/environment (never in the repo); connect timeout 2 s, read timeout 30 s; Idempotency-Key = job id.
- Bulkhead: a Semaphore (default 32 permits, configurable) around RagClient calls; when no permit is free within 200 ms,
  return the fallback immediately.
- Circuit breaker (Resilience4j if already allowed by the build, else a small hand-written breaker with tests): open after
  5 consecutive failures for 30 s; fallback = job stays QUEUED with "AI review queued".
- Retry once on timeout/5xx with the SAME Idempotency-Key.
- 202 + job: POST consultation saves it, creates a job (QUEUED), returns 202 with jobId; a virtual-thread executor runs the
  screening; GET /jobs/{id} returns status and the report when DONE; SSE endpoint pushes "done" events.
- Tests with a stub Python server (WireMock or MockWebServer): success, slow (timeout -> retry -> fallback), 500 (breaker opens),
  bulkhead full (fast fallback), duplicate job id sends the same key, SSE emits done. No real network.
