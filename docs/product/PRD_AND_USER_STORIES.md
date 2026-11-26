# Product Requirements & User Stories

> [!NOTE]
> This document defines the **v1.0 Core Gateway Baseline** (`E1`–`E6`, `US-01`–`US-16`). For the extended **v2.0 Platform Scope** tailored to our primary organizational demand profile (**15+ active users**, **large Hugging Face Paid Team compute + OpenAI Enterprise OAuth**, hosted on **2× NVIDIA DGX Spark**), see [PLATFORM_PRD_V2.md](PLATFORM_PRD_V2.md).

## 1. Problem Statement, Objectives & Non-Goals

Sensitive organizations need developer and analyst productivity from frontier closed models (OpenAI, OpenRouter) alongside private local models (vLLM, Ollama), without risking leakage of classified records, credentials, proprietary source code, internal infrastructure, or PII—and without letting unmonitored users execute unauthorized AI-assisted actions.

### Core Objectives
- Provide a single **OpenAI-compatible gateway** across OpenAI, OpenRouter, and internal vLLM/Ollama models.
- Enforce **100% local deterministic DLP** with tiered controls (`BLOCK`, `REROUTE_LOCAL`, `REDACT`, `PSEUDONYMIZE` + restore) before any external egress.
- Add **opinionated, non-deterministic local guardrails** (heuristics + local LLM-as-a-Judge) that warn users on unauthorized or risky actions.
- Deliver **maintenance telemetry and tamper-evident audit logs** with zero raw secret or PII persistence.

### Explicit Non-Goals
- Not an LLM training, fine-tuning, or prompt-management studio.
- Not a persistent PII vault or long-term conversation memory store.
- Never delegates DLP scanning or guardrail judging to external cloud APIs.
- Not an end-user chat web UI (acts strictly as an API gateway + admin CLI/API).

---

## 2. User Personas

| Persona | Primary Need | Key Pain Point Solved |
| :--- | :--- | :--- |
| **Internal Engineer / Analyst** | Drop-in OpenAI-compatible API access | Uses frontier and local models without memorizing manual redaction rules |
| **Security & Compliance Officer** | Hard blocks on secrets/classified data + action warnings | Prevents egress leaks and creates verifiable, zero-leak audit trails |
| **Platform / SRE Maintainer** | OTel traces, Prometheus metrics, quotas, offline deploy | Isolates upstream outages, tracks TTFT/cost, rotates keys via CLI |
| **Internal AI Agent / Service** | Predictable JSON outputs, `tool_calls`, SSE streaming | Receives safety warnings without breaking structured JSON or tool parsers |

---

## 3. Epics & User Stories

### E1: Unified Multi-Model Gateway

- **US-01 — Unified OpenAI-Compatible Proxy**
  *As an Internal Engineer / Analyst*, I want to call OpenAI, OpenRouter, and local vLLM/Ollama models through `/v1/chat/completions`, `/v1/embeddings`, and `/v1/models`, *so that* existing SDKs and tools work with only a base URL and API key change.
  - **Acceptance Criteria:**
    - Logical model aliases (e.g., `openai/gpt-4o-mini`, `openrouter/anthropic/claude-3.5-sonnet`, `local/llama3.2`) route to configured upstream adapters.
    - Supports both synchronous JSON responses and Server-Sent Events (SSE) streaming with usage reporting.
    - `GET /v1/models` returns OpenAI-formatted model entries filtered to the caller's authorized role.

- **US-02 — External Provider Retention & Metadata Hardening**
  *As a Security & Compliance Officer*, I want external OpenRouter calls to enforce Zero-Data-Retention and strip identifying internal user metadata, *so that* external sub-providers do not retain or train on organizational prompts.
  - **Acceptance Criteria:**
    - Every OpenRouter request injects `"provider": {"data_collection": "deny"}` unless explicitly overridden in policy.
    - Caller-supplied `user` fields (e.g., employee emails) are hashed or scrubbed prior to external dispatch.
    - Outbound connections are verified against an explicit `allowed_egress_hosts` allowlist.

### E2: Deterministic Egress Sanitization & Tiered Routing

- **US-03 — Fail-Closed Hard Block on Secrets & Classification Markings**
  *As a Security & Compliance Officer*, I want requests containing classification banners (`TOP SECRET`, `SECRET`, `CONFIDENTIAL`, `TLP:RED`) or credentials (API keys, PEM keys, JWTs, DB URIs) blocked locally before external egress, *so that* critical assets never leave the boundary.
  - **Acceptance Criteria:**
    - Returns HTTP `403` (`policy_violation`) listing category and policy reason without echoing raw secrets.
    - Zero bytes leave the gateway to OpenAI or OpenRouter when a Tier-1 finding triggers `BLOCK`.
    - If any deterministic detector errors or times out on an external request, the gateway fails closed (`503` or authorized local fallback).

- **US-04 — Auto-Reroute Proprietary Code to Local Models**
  *As an Internal Engineer / Analyst*, I want requests containing proprietary code or internal package headers to automatically reroute to a designated local model (e.g., `local/qwen2.5-coder`), *so that* work continues securely without manual retries.
  - **Acceptance Criteria:**
    - `PROPRIETARY_CODE` findings route the unsanitized prompt to `local_fallback_model` (must have `trust_zone == "local"`).
    - Response includes `X-Gateway-Policy-Action: reroute_local` and `X-Gateway-Routed-Model`.
    - Sending `X-Gateway-No-Fallback: true` disables silent rerouting and returns HTTP `403` instead.

- **US-05 — Reversible PII Pseudonymization & Chunk-Safe SSE Restoration**
  *As an Internal Engineer / Analyst*, I want PII (emails, phones, SSNs, Luhn credit cards, mod-11 NRIC/FIN, mod-97 IBANs) and internal IPs pseudonymized before external egress and seamlessly restored in streaming and non-streaming responses, *so that* frontier models stay useful without exposing real identities.
  - **Acceptance Criteria:**
    - Upstream payloads contain only placeholders (`[EMAIL_1]`, `[NRIC_1]`, `[INTERNAL_IP_1]`).
    - Ephemeral RAM `TokenVault` maps identically within one request and clears in a `finally` block (never persisted).
    - SSE chunk splits across placeholder boundaries (including `[` + `EMAIL` + `_1]` inside `delta.content` or `tool_calls`) restore cleanly with zero lost characters.

- **US-06 — Multimodal Egress Gatekeeping**
  *As a Security & Compliance Officer*, I want uninspectable external modalities (`image_url`, `input_audio`, `file`) blocked by default when targeting closed models, *so that* users cannot bypass text DLP inside screenshots or attachments.
  - **Acceptance Criteria:**
    - Requests to `trust_zone == "external"` containing non-text parts are blocked (`403`) unless the caller's role explicitly grants `allow_multimodal_external: true` (or reroutes locally).
    - Zero-width/invisible character obfuscation (`\u200b`–`\u200f`, `\ufeff`) is normalized and flagged before detection.

### E3: Opinionated Non-Deterministic Guardrails

- **US-07 — Behavioral Warning on Unauthorized Actions**
  *As a Security & Compliance Officer*, I want local heuristics plus a local LLM-as-a-Judge to evaluate prompts against an opinionated organizational rubric, *so that* users are warned when attempting unauthorized actions, policy evasion, jailbreaks, or unapproved official communications.
  - **Acceptance Criteria:**
    - Covers `policy_evasion`, `unauthorized_security_action`, `unauthorized_data_exfiltration`, `unapproved_official_action`, and `prompt_injection_or_jailbreak`.
    - Judge model is strictly enforced to `trust_zone == "local"` (Ollama/vLLM) and wraps untrusted text in per-request randomized nonce delimiters.
    - Defaults to warn (`WARN`) in policy threshold bands and supports optional escalation to `REROUTE_LOCAL` or `BLOCK` above high-confidence cutoffs.

- **US-08 — Client-Safe Warning Delivery**
  *As an Internal AI Agent / Service*, I want guardrail warnings delivered in HTTP headers and as a clean markdown callout for standard chat—while automatically skipping text injection for JSON schemas and pure tool calls, *so that* human users see clear guidance without breaking automated parsers.
  - **Acceptance Criteria:**
    - Every evaluated request emits `X-Gateway-Guardrail-Status`, `X-Gateway-Guardrail-Categories`, `X-Gateway-Guardrail-Confidence`, and `X-Gateway-Guardrail-Warning`.
    - In-band markdown banner prepends standard chat responses (sync and initial SSE delta) by default.
    - Banner text injection is suppressed automatically when `response_format.type` is `json_object`/`json_schema`, response is pure `tool_calls`, or `X-Gateway-Warning-Mode: headers-only` is passed.

- **US-09 — Post-Flight Risky Tool-Call Inspection**
  *As a Security & Compliance Officer*, I want assistant-emitted `tool_calls` checked post-flight for destructive or privileged operations (shell execution, mass deletes, IAM changes, external exfiltration), *so that* agentic workflows flag dangerous actions before client execution.
  - **Acceptance Criteria:**
    - Inspects both synchronous `tool_calls` and accumulated SSE `delta.tool_calls` arguments.
    - Flags matched patterns via `risky_tool_invocation` guardrail category and emits warning headers/events.

- **US-10 — Configurable Judge Degradation Mode**
  *As a Platform / SRE Maintainer*, I want the probabilistic local LLM judge to fail safely according to environment posture (`fail_open_with_warning` vs `fail_closed`), *so that* local GPU judge hiccups do not silently disable safety visibility or unnecessarily halt traffic.
  - **Acceptance Criteria:**
    - Default `fail_open_with_warning` runs heuristic fallback, sets `X-Gateway-Guardrail-Status: degraded`, and increments degradation metrics while deterministic DLP stays strictly fail-closed.
    - Setting `degradation_mode: fail_closed` rejects traffic (`503`) whenever the local judge times out or emits invalid output.

### E4: Identity, RBAC & Usage Governance

- **US-11 — Scoped API Keys & Role-Based Model Allowlists**
  *As a Platform / SRE Maintainer*, I want scoped gateway API keys (`gw_live_...`) stored strictly as HMAC-SHA256 hashes with role-based trust-zone and model allowlists, *so that* compromised or low-trust keys cannot invoke restricted external models.
  - **Acceptance Criteria:**
    - Plaintext keys are shown once at creation and never stored in SQLite (`hmac.compare_digest` verification).
    - Revoked or expired keys are rejected with `401`; unauthorized model/trust-zone access returns `403`.

- **US-12 — Rate Limiting & Daily Spend Quotas**
  *As a Platform / SRE Maintainer*, I want sliding-window RPM rate limits and daily token/USD cost quotas enforced per key, *so that* runaway scripts cannot exhaust budgets or upstream limits.
  - **Acceptance Criteria:**
    - Exceeding RPM returns `429 Too Many Requests` with `Retry-After`.
    - Daily token and estimated USD cost usage persists across requests in SQLite and blocks requests exceeding configured key/role budgets.

### E5: Telemetry & Tamper-Evident Audit Logging

- **US-13 — OpenTelemetry Traces & Prometheus Metrics**
  *As a Platform / SRE Maintainer*, I want per-stage OTel spans and Prometheus `/metrics` tracking TTFT, DLP latency, guardrail triggers, token throughput, estimated USD cost, and upstream errors, *so that* I can maintain and right-size models without inspecting user prompts.
  - **Acceptance Criteria:**
    - `/metrics` exposes `gateway_requests_total`, `gateway_ttft_seconds`, `gateway_sanitization_duration_seconds`, `gateway_dlp_detections_total`, `gateway_guardrail_triggers_total`, `gateway_tokens_total`, and `gateway_estimated_cost_usd_total`.
    - OTel spans (`auth`, `dlp_scan`, `policy_eval`, `guardrail_eval`, `upstream_call`, `stream_restore`, `audit_write`) export over OTLP when configured and run offline cleanly when unset.

- **US-14 — Zero-Leak Hash-Chained Audit Trail**
  *As a Security & Compliance Officer*, I want every request logged to a SHA-256 hash-chained audit ledger storing only counts, HMAC entity digests, prompt hashes, guardrail verdicts, and metadata, *so that* security teams can detect tampering without creating a secondary PII honeypot.
  - **Acceptance Criteria:**
    - Raw secrets, raw PII, and raw prompt text never appear in application logs, OTel attributes, or audit rows.
    - Serialized hash-chain writes compute `entry_hash = SHA256(prev_hash + "|" + canonical_json)`.
    - `GET /admin/audit/verify` and `gateway-cli audit verify` detect any modified, inserted, or deleted row.

### E6: Offline Operations & Maintenance

- **US-15 — Policy Dry-Run & Key Lifecycle Tooling**
  *As a Security & Compliance Officer*, I want a `/admin/policy/dry-run` endpoint and `gateway-cli` commands, *so that* I can test new org terms and guardrail rules without contacting external APIs.
  - **Acceptance Criteria:**
    - `POST /admin/policy/dry-run` and `gateway-cli policy test` return DLP findings, policy decision, sanitized preview, and guardrail verdict without external dispatch.
    - `gateway-cli` supports `key create`, `key list`, `key revoke`, and `audit verify`.

- **US-16 — Offline-First & Air-Gapped Readiness**
  *As a Platform / SRE Maintainer*, I want the gateway to run and pass its test suite 100% offline inside segmented containers, *so that* deployment passes strict internal network accreditation.
  - **Acceptance Criteria:**
    - Built-in detectors and heuristics require zero runtime internet downloads; optional Presidio/spaCy loads strictly from local model paths.
    - Container runs non-root with segmented `internal_net` (`internal: true`) and `egress_net` attached only to the gateway.

---

## 4. Service Level Objectives & Success Metrics

| Metric / SLO | Target | Verification Method |
| :--- | :--- | :--- |
| Deterministic DLP + heuristic latency (4KB prompt) | `< 10 ms` p95 | `tests/test_performance.py` offline benchmark |
| Raw Tier-1 secrets/classification/PII sent externally | `0 bytes` | `tests/test_e2e_proxy.py` inspecting `MockTransport` wire payload |
| Raw secrets/PII written to logs, traces, or audit DB | `0 occurrences` | `tests/test_audit_chain.py` + `scrub_secrets_processor` tests |
| Local LLM judge bounded timeout | `<= 1500 ms` | `tests/test_guardrails.py` timeout & degradation assertions |
| Audit log cryptographic chain verification | `100%` tamper detection | `tests/test_audit_chain.py` SQL mutation detection |
| Offline automated test & static check pass rate | `100%` offline | `pytest -v`, `ruff check`, `mypy src` with zero network access |
