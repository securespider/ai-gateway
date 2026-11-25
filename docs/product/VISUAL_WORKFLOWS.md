# Visual Workflows

## 1. End-to-End Request Decision Flow

Every incoming `/v1/chat/completions` and `/v1/embeddings` call follows a deterministic security-first pipeline before touching external providers, while running opinionated behavioral guardrails across both local and external traffic.

```mermaid
flowchart TD
    Req["Incoming Request + Bearer Key"] --> Auth["Auth, RBAC, RPM & Daily Quota Check"]
    Auth -->|"Invalid / Quota Exceeded"| Err4xx["401 / 403 / 429 Error"]
    Auth -->|"Authorized"| Zone{"Target Model Trust Zone?"}

    Zone -->|"local"| LocalAudit["Optional Audit Scan (No Sanitization)"]
    LocalAudit --> Guardrail

    Zone -->|"external"| Modality{"Blocked Modality or Normalizer Fault?"}
    Modality -->|"Yes"| FailClose["403 / 503 Fail-Closed"]
    Modality -->|"Text Clean"| DLP["Deterministic Local DLP (5 Detectors)"]
    DLP -->|"Detector Error / Timeout"| FailClose
    DLP --> Policy{"Tiered DLP Policy"}

    Policy -->|"BLOCK (Secrets / Classification)"| Reject["403 Policy Violation"]
    Policy -->|"REROUTE_LOCAL (No-Fallback=false)"| SwapLocal["Switch to Local Fallback Model"]
    Policy -->|"REDACT / PSEUDONYMIZE / ALLOW"| Guardrail

    SwapLocal --> Guardrail["Opinionated Guardrails (Heuristics + Local Judge)"]
    Guardrail -->|"High-Confidence Block"| Reject
    Guardrail -->|"Pass / Warn / Degraded"| Route{"Effective Trust Zone?"}

    Route -->|"local"| CallLocal["Dispatch to Local vLLM / Ollama"]
    Route -->|"external"| Vault["Apply Redaction & RAM TokenVault + ZDR"]
    Vault --> CallExt["Dispatch to OpenAI / OpenRouter"]

    CallLocal --> Post["Tool-Call Guard + Warning Injection"]
    CallExt --> Restore["Sync / SSE Chunk Restorer"]
    Restore --> Post
    Post --> ClearVault["Clear RAM TokenVault in finally"]
    ClearVault --> Audit["Write Hash-Chained Audit & OTel/Metrics"]
    Audit --> Resp["Response + Governance Headers"]
```

---

## 2. Reversible PII Pseudonymization & Chunk-Split SSE Restoration

When external models receive Tier-3 PII or internal host identifiers, mappings exist **only in single-request RAM**. Even when upstream LLM tokenizers split placeholders across SSE frames (e.g., `"[EMAIL"` followed by `"_1]"`), the streaming state machine buffers only candidate prefixes and emits restored text cleanly.

```mermaid
sequenceDiagram
    participant C as Internal Client
    participant GW as Gateway (DLP + Policy)
    participant V as Ephemeral RAM TokenVault
    participant EXT as OpenAI / OpenRouter (ZDR)
    participant AUD as Audit Store & OTel

    C->>GW: POST /v1/chat/completions (stream=true, email + NRIC)
    GW->>GW: Deterministic DLP detects EMAIL & NRIC
    GW->>V: Mint placeholders -> [EMAIL_1], [NRIC_1]
    GW->>EXT: Send sanitized prompt + ZDR flag (zero raw PII on wire)
    EXT-->>GW: SSE chunk 1: "Contact [EMAIL"
    GW->>GW: Restorer holds "[EMAIL" (valid vault prefix)
    GW-->>C: SSE delta: "Contact "
    EXT-->>GW: SSE chunk 2: "_1] for help"
    GW->>V: Lookup "[EMAIL_1]" -> original email
    GW-->>C: SSE delta: "alice@example.com for help"
    GW->>V: vault.clear() in finally block
    GW->>AUD: Log PSEUDONYMIZE + HMAC entity digests + TTFT + tokens
```

---

## 3. Tier-1 Hard Block vs. Local Auto-Reroute

Tier-1 secrets and classification banners fail fast before calling any upstream provider or burning GPU time on the local guardrail judge. Proprietary code transparently reroutes to a local coding model unless the caller explicitly disables fallbacks.

```mermaid
sequenceDiagram
    participant C as Internal Client
    participant GW as Gateway (DLP + Policy)
    participant LOC as Local vLLM / Ollama
    participant AUD as Audit Store

    rect rgb(255, 240, 240)
        Note over C,AUD: Branch A — Tier 1 Secret or Classification Marking
        C->>GW: POST /v1/chat/completions (model=openai/gpt-4o, AWS Key / TOP SECRET)
        GW->>GW: SecretsDetector / ClassificationDetector match
        GW->>AUD: Record BLOCK + HMAC digest (raw secret never stored)
        GW-->>C: HTTP 403 policy_violation (zero external egress)
    end

    rect rgb(240, 248, 255)
        Note over C,AUD: Branch B — Proprietary Code with Local Fallback
        C->>GW: POST /v1/chat/completions (model=openai/gpt-4o, internal package code)
        GW->>GW: ProprietaryCodeDetector -> REROUTE_LOCAL (local/qwen2.5-coder)
        GW->>LOC: Forward unsanitized prompt inside internal enclave
        LOC-->>GW: Completion response
        GW->>AUD: Record REROUTE_LOCAL + target_model
        GW-->>C: HTTP 200 + X-Gateway-Policy-Action: reroute_local
    end
```

---

## 4. Opinionated Non-Deterministic Guardrail Pipeline

Because policy violations are often behavioral (e.g., asking an AI to encode internal data to evade DLP, run unauthorized security scans, or draft unapproved legal/HR notices), local heuristics and a **local-only LLM-as-a-Judge** score intent and emit client-safe warnings without disrupting valid JSON or tool-call automation.

```mermaid
flowchart LR
    Prompt["Normalized User + Tool Messages"] --> Heur["Fast Local Heuristics (under 2ms)"]
    Heur --> Judge["Local LLM Judge (trust_zone = local, Nonce Delimiters)"]
    Judge -->|"Timeout / Invalid JSON"| Deg{"Degradation Mode?"}
    Deg -->|"fail_open_with_warning"| WarnDeg["Status: degraded + Heuristic Fallback"]
    Deg -->|"fail_closed"| Err503["HTTP 503 Guardrail Unavailable"]

    Judge -->|"Valid Score + Category"| Score{"Compare Thresholds"}
    Score -->|"Below warn_threshold"| Pass["Status: pass"]
    Score -->|"Between warn and block"| Warn["Status: warn (Emit Policy Warning)"]
    Score -->|"Severe Category >= block_threshold"| Block["HTTP 403 Guardrail Block"]

    Pass --> OutCheck["Post-Flight Tool-Call Guard"]
    Warn --> OutCheck
    WarnDeg --> OutCheck
    OutCheck --> Inject{"JSON Mode or Tool-Calls Only or headers-only?"}
    Inject -->|"Yes"| HeadersOnly["Set X-Gateway-Guardrail-* Headers Only"]
    Inject -->|"No (Standard Chat)"| Banner["Set Headers + Prepend Markdown Warning Banner"]
```

---

## 5. Security & Maintainer Operational Loops

Operators manage keys, tune DLP/guardrail rules offline via dry-run, monitor real-time telemetry, and verify ledger integrity without accessing raw employee prompts.

```mermaid
flowchart LR
    Sec["Security Officer"] -->|"1. Update YAML rules"| DryRun["POST /admin/policy/dry-run or gateway-cli policy test"]
    DryRun -->|"Inspect sanitized preview & verdict"| Sec
    Sec -->|"2. Periodic audit"| Verify["GET /admin/audit/verify or gateway-cli audit verify"]
    Verify -->|"Validates SHA-256 hash chain"| Sec

    SRE["Platform / SRE"] -->|"Key lifecycle"| CLI["gateway-cli key create / list / revoke"]
    SRE -->|"Monitor TTFT, errors, cost"| Prom["Prometheus /metrics & OTel Collector"]
    Prom -->|"Circuit breaker / judge alerts"| SRE
```
