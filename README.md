# Starter — Circuit Breaker Proof Slice (PromptPilot)

This is the starter repository for **Option B (provided scenario)** of the assignment.

Repository: `https://github.com/kalviumcommunity/Production-Hardening-Proof-Slice-and-Review-Assignment` (public, owned by
`kalviumcommunity`). Fork it, implement the breaker, and open a pull request back to your fork.

> The control here is a **circuit breaker**, covered earlier in this module. You are applying a
> concept you have already learned, not a new one.

## Scenario

PromptPilot drafts support replies by calling an external model provider. When the provider is slow
or failing, every request still tries it, so doomed calls pile up and the failure spreads. You will
add a **circuit breaker** so sustained failures short-circuit fast to a safe fallback, then recover
through a probe.

## Your task

Implement the circuit breaker in [`src/circuitBreaker.js`](./src/circuitBreaker.js). The file has a
skeleton and a `TODO`. The behaviour is fully specified by the supplied tests.

Prerequisite: Node.js 18 or newer (the tests use Node's built-in test runner). Then run:

```bash
node --test
```

Make **all tests pass**. Do **not** modify the tests. The reply path in `src/replyService.js` is
already wired to your breaker.

The breaker must:

- start `CLOSED` and count calls and failures;
- move to `OPEN` when `failures / calls >= failureThreshold` **and** `calls >= minimumRequests`;
- while `OPEN` and within `openMillis`, short-circuit (throw `CircuitOpenError` without calling the
  dependency) and increment `metrics.short_circuited_total`;
- after `openMillis` elapses (using the injected `now()`), allow one `HALF_OPEN` probe — success
  closes it and resets counts, failure re-opens it;
- increment `metrics.breaker_open_total` on every trip and call `onStateChange(state)` on every
  transition.

---

## Production-Hardening Proof Slice Documentation

### 1. Risk addressed
Prevents cascading system failure, latency pile-ups, and thread exhaustion caused by continuing to send doomed requests to an unavailable or degrading external model provider.

### 2. Chosen control
A stateful circuit breaker (`CLOSED`, `OPEN`, `HALF_OPEN`) implemented in [`src/circuitBreaker.js`](./src/circuitBreaker.js) that intercepts calls in [`src/replyService.js`](./src/replyService.js) and fails fast to a human queue fallback.

### 3. How to run / test
Run `node --test` (requires Node.js 18+). Expected result: all 8 test cases pass (`# pass 8 # fail 0`).

### 4. Evidence
`test/circuitBreaker.test.js` verifies threshold tripping, short-circuiting without invoking the provider, and half-open probing; `test/replyService.test.js` verifies that requests degrade gracefully to `circuit_open` fallback once tripped.

### 5. Observability note
Exposes `metrics.breaker_open_total` (increments on every transition to `OPEN`), `metrics.short_circuited_total` (increments on every short-circuited call), and triggers `onStateChange(state)` (`OPEN`, `HALF_OPEN`, `CLOSED`) for real-time alerting and structured logging.

### 6. Trade-off
Favors availability and fast failure over eventual success; sudden intermittent error spikes may trip the breaker and temporarily serve fallback replies even if upstream has partially recovered.

### 7. Remaining risk
Does not enforce individual request timeouts if the external provider hangs indefinitely without throwing, nor does it isolate failures on a per-tenant or per-model basis.

### 8. How this production control protects my Modules 1–3 design
Protects core SLA commitments and architecture boundaries by decoupling user ticket ingestion from third-party model availability, preserving system responsiveness even under upstream dependency outage.
