# AGENTS.md — Implementing Agent Operating Manual

This repository currently contains **architecture, product, and implementation specifications**. Use this document as the root instruction set when implementing `secure-ai-gateway` in Python 3.12.

---

## 1. Mandatory Reading Order

Read these files in sequence before writing any code:
1. `AGENTS.md` (this file) — security invariants, toolchain, engineering rules, verification gate.
2. [docs/implementation/IMPLEMENTATION_PLAN.md](docs/implementation/IMPLEMENTATION_PLAN.md) — milestone DAG (`M1`–`M9`), full file tree, per-milestone deliverables.
3. [docs/implementation/CONTRACTS_AND_SCHEMAS.md](docs/implementation/CONTRACTS_AND_SCHEMAS.md) — exact Python protocols, HTTP headers, YAML configs, SQLite DDL, metrics/OTel contracts.
4. [docs/implementation/EDGE_CASES_AND_TEST_SPEC.md](docs/implementation/EDGE_CASES_AND_TEST_SPEC.md) — streaming state machine, checksum algorithms, prompt-injection defense, offline test matrix.
5. [docs/architecture/PROPOSED_ARCHITECTURE.md](docs/architecture/PROPOSED_ARCHITECTURE.md) — trust boundaries, threat model, ADRs.

---

## 2. Target Environment & Toolchain

- **Runtime:** Python `3.12` (available locally at `/opt/homebrew/bin/python3.12`).
- **Package / Environment Manager:** `uv`.
- **Core Stack:** `fastapi`, `uvicorn[standard]`, `pydantic>=2.7`, `pydantic-settings`, `httpx`, `pyyaml`, `aiosqlite`, `structlog`, `prometheus-client`, `opentelemetry-api`, `opentelemetry-sdk`, `opentelemetry-exporter-otlp-proto-http`, `opentelemetry-instrumentation-fastapi`.
- **Optional NER Extra (`[ner]`):** `presidio-analyzer`, `spacy` (disabled by default; strictly loads local model paths with zero runtime downloads).
- **Dev / Verification Stack (`[dev]`):** `pytest`, `pytest-asyncio`, `ruff`, `mypy`.
- **Offline-First Rule:** Zero runtime network calls are allowed for models, rules, schemas, or tests. All unit, security, and end-to-end tests MUST run 100% offline using `httpx.MockTransport`.

---

## 3. Non-Negotiable Security Invariants (Hard Rules)

Every invariant below is mandatory and MUST be explicitly tested:

1. **Zero Secret / PII Persistence:** Never write raw user prompts, raw completions, raw API keys, or raw detected sensitive values to application logs, OpenTelemetry span attributes, error messages, or SQLite/JSONL audit records (unless explicit non-default debug sanitized preview is enabled on non-blocked external payloads; raw blocked/local payloads are NEVER stored).
2. **Fail-Closed Deterministic DLP:** If any `BaseDetector` or `TextNormalizer` fails or exceeds `detector_timeout_ms` while handling a `trust_zone == "external"` request, fail closed (`503` or authorized `REROUTE_LOCAL`). Never forward uninspected data externally.
3. **Multimodal Egress Gate:** Non-text content parts (`image_url`, `input_audio`, `file`) targeting `trust_zone == "external"` MUST be rejected (`403`) by default unless `role.allow_multimodal_external == True` or routed to a local multimodal fallback.
4. **Ephemeral Request-Scoped RAM `TokenVault`:** Pseudonymization maps exist strictly in RAM for one request, execute `vault.clear()` in a `finally` block, and restoration MUST ONLY replace `[ENTITY_N]` tokens minted in that same vault instance.
5. **Local-Only Guardrail Judge:** `guardrails.judge_model` MUST be validated to have `trust_zone == "local"` both at config load time AND at dispatch time. Never dispatch raw prompts to an external judge.
6. **Judge Prompt-Injection Hardening:** Always wrap evaluated user/tool text inside randomized per-request delimiter tags (`<UNTRUSTED_INPUT_{nonce}>...</UNTRUSTED_INPUT_{nonce}>`) before invoking `LocalLLMJudge`.
7. **Client-Safe Guardrail Warning Suppression:** Always emit `X-Gateway-Guardrail-*` headers on warned/degraded calls, and automatically suppress in-band markdown warning text whenever `response_format.type in {"json_object", "json_schema"}`, the response consists solely of `tool_calls` (empty content), the endpoint is `/v1/embeddings`, or the request includes `X-Gateway-Warning-Mode: headers-only`.
8. **Respect `X-Gateway-No-Fallback: true`:** When set, `REROUTE_LOCAL` MUST NOT silently redirect to a local model; return HTTP `403` (`policy_violation`) instead.
9. **HMAC-SHA256 Key Storage:** Gateway keys (`gw_live_<id>_<secret>`) MUST be stored strictly as `HMAC-SHA256(server_pepper, full_key)` and verified using `hmac.compare_digest`.
10. **Strict Egress Allowlist (`EgressGuardTransport`):** Every outbound `httpx` request MUST assert that the target host matches `allowed_egress_hosts` (for external providers) or configured local backend URLs.
11. **Enforced OpenRouter Zero-Data-Retention:** OpenRouter payloads MUST inject `"provider": {"data_collection": "deny"}` unless explicitly overridden in configuration, and caller `user` fields MUST be scrubbed/hashed before external egress.
12. **Serialized Tamper-Evident Audit Chain:** All writes to `SQLiteAuditStore` MUST acquire a single `asyncio.Lock` around reading `prev_hash`, computing `entry_hash = SHA256(f"{prev_hash}|{canonical_json}")`, and inserting the row so the chain cannot fork under concurrency.

---

## 4. Engineering & Style Conventions

- **Clean Boundaries:** Depend on `Protocol` / `ABC` interfaces (`BaseDetector`, `BaseKeyStore`, `BaseQuotaStore`, `BaseProvider`, `BaseAuditStore`) so storage and NLP backends swap cleanly without changing business logic.
- **DRY Provider Transport:** Implement HTTP pooling, retries, timeouts, `EgressGuardTransport`, and `CircuitBreaker` once inside `OpenAICompatProvider` (`src/gateway/providers/openai_compat.py`), keeping `OpenAIProvider`, `OpenRouterProvider`, and `LocalProvider` as thin specializations.
- **Type Safety & Docstrings:** Maintain strict Python 3.12 type annotations passing `mypy src` cleanly. Write concise docstrings explaining **why** a constraint or design choice exists, not merely restating **how** the lines execute.
- **ReDoS Prevention:** Write linear-time regexes without nested ambiguous quantifiers (`(a+)+`), enforce `max_request_bytes`, and run detectors inside per-detector timeouts.

---

## 5. Verification Gate (Definition of Done)

Implementation is complete ONLY when all commands below pass locally with zero internet access:

```bash
uv venv --python /opt/homebrew/bin/python3.12 .venv
uv pip install --python .venv/bin/python -e ".[dev]"
.venv/bin/pytest -v
.venv/bin/ruff check src tests
.venv/bin/mypy src
```

All 8 test suites (`test_detectors.py`, `test_policy_and_vault.py`, `test_guardrails.py`, `test_stream_restorer.py`, `test_auth_and_quotas.py`, `test_audit_chain.py`, `test_e2e_proxy.py`, `test_performance.py`) must pass with `0` failures and p95 DLP + heuristic latency `< 10 ms` on 4KB prompts.
