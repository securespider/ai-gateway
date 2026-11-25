# Proposed Architecture & Security Design

## 1. System Component Architecture

The gateway enforces a strict internal trust boundary in Python 3.12 (`FastAPI` + `httpx`), isolating local inference and observability inside an internal enclave while gating external API traffic through `EgressGuardTransport`.

```mermaid
flowchart LR
    subgraph InternalClients["Internal Clients"]
        Apps["Internal Apps & AI Agents"]
        Ops["Security / SRE CLI & Admin API"]
    end

    subgraph GW["Secure AI Gateway (Python 3.12 / FastAPI)"]
        Auth["Auth, RBAC & Quota Store"]
        Norm["Input Normalizer & Modality Gate"]
        DLP["Deterministic Local DLP Engine"]
        Pol["Tiered Policy Engine"]
        Guard["Opinionated Guardrails (Heuristics + Judge)"]
        Vault["Ephemeral RAM TokenVault & SSE Restorer"]
        Router["ModelRouter & CircuitBreaker"]
        EgressGuard["EgressGuardTransport (Host Allowlist)"]
    end

    subgraph Enclave["Internal Enclave (No Egress)"]
        LocalModels["Local vLLM / Ollama Models"]
        JudgeModel["Local Guardrail Judge Model"]
    end

    subgraph External["External Providers (Controlled Egress)"]
        OpenAI["OpenAI API"]
        OpenRouter["OpenRouter API (ZDR Enforced)"]
    end

    subgraph TelemetryAudit["Observability & Audit"]
        OTel["OTel Traces & Prometheus /metrics"]
        AuditDB["Hash-Chained SQLite Audit + JSONL SIEM"]
    end

    Apps --> Auth
    Ops --> Auth
    Auth --> Norm
    Norm --> DLP
    DLP --> Pol
    Pol --> Guard
    Guard --> Vault
    Vault --> Router
    Router --> LocalModels
    Guard -->|"trust_zone = local"| JudgeModel
    Router --> EgressGuard
    EgressGuard --> OpenAI
    EgressGuard --> OpenRouter
    GW -.-> OTel
    GW -.-> AuditDB
```

---

## 2. Deployment & Network Segmentation

In containerized deployments, only the hardened non-root `gateway` container bridges `internal_net` (`internal: true`, zero internet routing) and `egress_net`. Internal models, telemetry collectors, and persistent SQLite WAL storage sit strictly inside `internal_net`.

```mermaid
flowchart TB
    subgraph EgressNet["egress_net (Outbound HTTPS Only via Allowlist)"]
        ExtAPI["OpenAI / OpenRouter APIs"]
    end

    subgraph GatewayHost["Non-Root Read-Only Container"]
        GWContainer["secure-ai-gateway (EgressGuardTransport)"]
    end

    subgraph InternalNet["internal_net (internal: true — Air-Gapped)"]
        SQLiteVol["SQLite WAL Volume (Keys, Quotas, Audit Chain)"]
        Ollama["Local vLLM / Ollama Cluster"]
        Judge["Local Judge Model"]
        OTelCol["OpenTelemetry Collector"]
        Prom["Prometheus"]
        Graf["Grafana"]
    end

    GWContainer <-->|"Allowed Hosts Only"| ExtAPI
    GWContainer <--> SQLiteVol
    GWContainer <--> Ollama
    GWContainer <--> Judge
    GWContainer -->|"OTLP HTTP"| OTelCol
    Prom -->|"Scrape /metrics"| GWContainer
    Graf --> Prom
```

---

## 3. Component Responsibility & Latency Budget

| Component | Module Path | Primary Responsibility | Latency Budget |
| :--- | :--- | :--- | :--- |
| **Auth, RBAC & Quotas** | `gateway/auth/` | HMAC-SHA256 key lookup, role/model allowlists, sliding-window RPM & daily USD/token ledger | `1–2 ms` |
| **Normalizer & Modality Gate** | `gateway/dlp/normalizer.py` | NFKC view with offset mapping, zero-width obfuscation flags, external non-text block | `< 1 ms` |
| **Deterministic Local DLP** | `gateway/dlp/` | Classification, Secrets/Entropy, Checksum PII, Infra/Org terms, Proprietary Code (+ optional offline NER) | `< 8 ms` (4KB p95) |
| **Tiered Policy & Vault** | `gateway/sanitization/` | Deterministic `BLOCK`/`REROUTE_LOCAL`/`REDACT`/`PSEUDONYMIZE`, RAM vault, SSE chunk restorer | `< 1 ms` |
| **Opinionated Guardrails** | `gateway/guardrails/` | Fast heuristics (`< 2 ms`) + local LLM-as-a-Judge + post-flight `tool_calls` guard + safe banner injector | `80–500 ms` (bound `<= 1500 ms`) |
| **Provider Router & Egress** | `gateway/providers/` | Model alias resolution, fallbacks, per-provider `CircuitBreaker`, `EgressGuardTransport`, OpenRouter ZDR | Upstream bound (`300–3000 ms` TTFT) |
| **Telemetry & Audit** | `gateway/telemetry/` | Secret-scrubbed `structlog` JSON logs, Prometheus `/metrics`, OTel spans, serialized SHA-256 audit chain | `< 2 ms` async |

---

## 4. Tiered DLP & Opinionated Guardrail Matrices

### Deterministic DLP Policy Matrix (`trust_zone == "external"`)

| Sensitivity Category | Detection Examples | Default Action | Fallback / Notes |
| :--- | :--- | :--- | :--- |
| `CLASSIFICATION` | `TOP SECRET`, `SECRET//...`, `CONFIDENTIAL`, `TLP:RED` | `BLOCK` | Hard `403` block; lower tiers configurable to `REROUTE_LOCAL` |
| `SECRET` | Vendor keys (`sk-`, `AKIA`, `ghp_`), PEM keys, JWTs, DB URIs, high entropy | `BLOCK` | Hard `403` block; zero raw bytes on logs or wire |
| `PROPRIETARY_CODE` | Org copyright headers, internal packages, large fenced code blocks | `REROUTE_LOCAL` | Routes to `local_fallback_model`; degrades to `403` if `X-Gateway-No-Fallback: true` |
| `INTERNAL_INFRA` | RFC1918/ULA IPs, `.internal`/`.corp`/`.local` hosts | `PSEUDONYMIZE` | Replaced with `[INTERNAL_IP_1]`, `[INTERNAL_HOST_1]` |
| `ORG_TERM` | Project codenames & custom dictionary from `org_terms.yaml` | `REDACT` | Irreversibly replaced with `[REDACTED_ORG_TERM]` |
| `PII` | Emails, phones, SSNs, Luhn cards, mod-11 NRIC/FIN, mod-97 IBANs, names | `PSEUDONYMIZE` | Ephemeral RAM restore in sync and streaming responses |

### Opinionated Non-Deterministic Guardrail Matrix (All Trust Zones)

| Guardrail Category | Behavioral Intent Evaluated | Default Policy | Escalation Option |
| :--- | :--- | :--- | :--- |
| `policy_evasion` | Requests to encode/obfuscate sensitive data to bypass DLP | `WARN` | `BLOCK` or `REROUTE_LOCAL` above high threshold |
| `unauthorized_security_action` | Exploitation, credential dumping, internal scanning, phishing | `WARN` / `BLOCK` | `BLOCK` at `>= 0.90` confidence |
| `unauthorized_data_exfiltration` | Instructions to extract/move internal or customer datasets externally | `WARN` / `BLOCK` | `BLOCK` at `>= 0.90` confidence |
| `unapproved_official_action` | Drafting binding legal, HR disciplinary, regulatory, or executive notices | `WARN` | Advises human sign-off via banner & headers |
| `prompt_injection_or_jailbreak` | Overrides in prompts or untrusted `role: "tool"` outputs | `WARN` | `BLOCK` at `>= 0.92` confidence |
| `risky_tool_invocation` | Post-flight destructive shell, IAM, or mass-delete `tool_calls` | `WARN` | Emits warning headers / audit alert |

---

## 5. Threat Model & Security Controls

| Threat Vector | Mitigation / Control | Enforcement Point |
| :--- | :--- | :--- |
| **Raw secret/PII egress to closed LLM** | Fail-closed local DLP + Tier-1 `BLOCK` + RAM-only `PSEUDONYMIZE` | `DetectorEngine` + `PolicyEngine` + `TokenVault` |
| **Multimodal pixel/audio DLP bypass** | External `image_url`, `input_audio`, `file` parts blocked unless role-permitted | `PayloadSanitizer` modality gate |
| **Zero-width / Unicode obfuscation** | Offset-mapped NFKC normalization + invisible character flagging | `TextNormalizer` |
| **Prompt injection against LLM judge** | Randomized per-request `<UNTRUSTED_INPUT_{nonce}>` delimiters + strict JSON parse | `LocalLLMJudge` |
| **ReDoS on Python `re` engine** | Linear-time patterns without nested quantifiers + `max_request_bytes` + timeouts | All `dlp/detectors/` + API middleware |
| **Secrets leaked into logs, spans, or DB** | `scrub_secrets_processor` + safe OTel allowlist + audit stores only counts & HMAC digests | `telemetry/logging.py`, `tracing.py`, `audit.py` |
| **Audit log tampering or row deletion** | Serialized `SHA256(prev_hash + "|" + canonical_json)` chain + `/admin/audit/verify` | `SQLiteAuditStore` (`asyncio.Lock`) |
| **External provider training / retention** | Inject `"provider": {"data_collection": "deny"}` on OpenRouter + scrub `user` ID | `OpenRouterProvider` + `PayloadSanitizer` |
| **SSRF / unintended outbound egress** | `EgressGuardTransport` rejects any outbound host not on `allowed_egress_hosts` | `providers/base.py` |
| **Upstream outage cascading locally** | Per-provider `CircuitBreaker` (`CLOSED`/`OPEN`/`HALF_OPEN`) isolates external failures | `providers/base.py` + `ModelRouter` |

---

## 6. Architecture Decision Records (ADRs)

- **ADR-01 — Python 3.12 (`FastAPI` + `httpx`) over Go or Rust:** Upstream LLM TTFT (`300–3000 ms`) and local LLM judging (`80–500 ms`) dominate latency; Rust/Pydantic v2 and C-backed regexes keep deterministic DLP under `10 ms` (< 0.5% overhead) while enabling native Python NLP (Presidio/spaCy) without a sidecar. If accreditation later demands a zero-runtime static binary at 5,000+ concurrent streams, a Go/Rust edge proxy can wrap this engine cleanly.
- **ADR-02 — Ephemeral Request-Scoped RAM `TokenVault`:** Mappings exist strictly in memory for one HTTP call and are cleared in a `finally` block. Multi-turn conversations re-pseudonymize deterministically each turn, eliminating persistent PII breach risk.
- **ADR-03 — Asymmetric Failure Semantics (DLP vs. Guardrails):** Deterministic DLP is strictly **fail-closed** for external requests (`503` or authorized `REROUTE_LOCAL`). The probabilistic local LLM judge defaults to **`fail_open_with_warning`** (heuristic fallback + `X-Gateway-Guardrail-Status: degraded`) with a **`fail_closed`** toggle for high-assurance enclaves.
- **ADR-04 — Enforced Zero-Data-Retention on OpenRouter:** All OpenRouter calls automatically append `"provider": {"data_collection": "deny"}` so upstream sub-providers cannot retain or train on sanitized traffic.
- **ADR-05 — Hash-Chained Audit Ledger Without Raw Payloads:** Audit records store `policy_version_hash`, prompt `SHA-256`, HMAC-SHA256 value digests, detection counts, guardrail verdicts, and token/cost telemetry—proving compliance and detecting tampering without becoming a secondary sensitive-data store.
