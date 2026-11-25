# Engineering Edge Cases, Algorithms & Offline Test Specification

This specification documents the subtle failure modes and exact algorithms required so implementation avoids common security and streaming bugs.

---

## 1. Critical Implementation Algorithms & Edge Cases

### 1.1 Chunk-Boundary-Safe `StreamingPlaceholderRestorer`

LLM tokenizers split placeholders across SSE frames (e.g., `"Contact ["`, `"EMAIL"`, `"_1"`, `"] today"`). Naive chunk regexes miss every split token.

- **Algorithm (Per-Channel Prefix Holdback):**
  1. Maintain `buffer: str = ""` inside each `StreamingPlaceholderRestorer` instance.
  2. On `feed(chunk)`: append `chunk` to `buffer`, emit `out = []`, and loop while `buffer`:
     - Find `idx = buffer.find("[")`.
     - If `idx == -1`: flush all of `buffer` to `out`, clear `buffer`, break.
     - If `idx > 0`: flush `buffer[:idx]` to `out`, slice `buffer = buffer[idx:]`.
     - Now `buffer` starts with `"["`:
       - If any `p in vault.active_placeholders` equals `buffer[:len(p)]`: restore `vault.restore(p)` to `out`, slice `buffer = buffer[len(p):]`, continue.
       - Else if any `p in vault.active_placeholders` starts with `buffer` (`p.startswith(buffer)`): **hold back** `buffer` (break and wait for next SSE chunk), provided `len(buffer) <= max_placeholder_len`.
       - Else: `"["` is ordinary text (e.g., `arr[0]` or `[NOTE]`). Emit `"["` to `out`, slice `buffer = buffer[1:]`, continue.
  3. On `flush()` (called on `finish_reason` or `[DONE]`): return `vault.restore(buffer)` and clear `buffer`.
- **Multi-Choice / Tool-Call Isolation:** Keep a dictionary of restorers keyed by `("content", choice_index)` and `("tool_call", choice_index, tool_call_index)` so interleaved choices or parallel tool arguments never cross-contaminate buffers.

### 1.2 Deterministic PII Checksums (`gateway.dlp.detectors.pii`)

Avoid naive digit regexes that trigger on timestamps or order IDs:

- **Credit Cards (Luhn):** Strip spaces/hyphens; require 13–19 digits; double every second digit from right (subtract 9 if `> 9`); sum `% 10 == 0`.
- **Singapore NRIC / FIN (`^[STFGM]\d{7}[A-Z]$`):**
  - Weights: `[2, 7, 6, 5, 4, 3, 2]`.
  - Prefix offsets: `S -> 0`, `F -> 0`, `T -> 4`, `G -> 4`, `M -> 3`.
  - `check = (offset + sum(w * d for w, d in zip(weights, digits))) % 11`.
  - Lookup tables: `S/T -> "JZIHGFEDCBA"`, `F/G -> "XWUTRQPNMLK"`, `M -> "XWUTRQPNJLK"`.
- **IBAN (`^[A-Z]{2}\d{2}[A-Z0-9]{11,30}$`):** Move first 4 chars to end, replace `A..Z` with `10..35`, validate `int(digits_str) % 97 == 1`.

### 1.3 Zero-Width Obfuscation & `TextNormalizer` Offset Mapping

Attackers attempt evasion with zero-width characters (`\u200b`–`\u200f`, `\ufeff`, `\u00ad`) or fullwidth homoglyphs (`\uff21` -> `A`).

- Build `normalized_chars: list[str]` and `norm_to_orig: list[int]` in one pass over `original_text`:
  - Strip invisible characters while recording `invisible_count`.
  - Apply `unicodedata.normalize("NFKC", ch)` per character (mapping each emitted normalized char back to its source index `orig_idx`).
- Map every detector span `(n_start, n_end)` back to `(norm_to_orig[n_start], norm_to_orig[n_end - 1] + 1)` so `TokenVault` replaces the exact obfuscated original span!
- If `invisible_count >= 3` inside alphanumeric tokens, emit a high-severity obfuscation signal / heuristic guardrail flag.

### 1.4 Deterministic Span Overlap Resolution (`gateway.dlp.resolver`)

Multiple detectors can match overlapping text (e.g., a database URL containing an IP and password).
1. Sort candidates by `(-SEVERITY_RANK[f.severity], -(f.end - f.start), -f.confidence, f.start)`.
2. Greedily accept finding `f` iff `[f.start, f.end)` does not overlap any already accepted `[a.start, a.end)` (`not (f.start < a.end and a.start < f.end)`).
3. Replace accepted spans right-to-left (sorted by `start` descending) so string indices never shift during rewriting.

### 1.5 ReDoS Prevention

- Never write nested quantifiers (`(a+)+`, `(\w+\s*)+`).
- Enforce `max_request_bytes` in FastAPI middleware before parsing JSON.
- Bound per-detector execution with `detector_timeout_ms` (e.g., threadpool/async timeout guard) and fail closed on timeout for external routes.

### 1.6 Local LLM Judge Prompt-Injection Defense & Noisy Output Parsing

- Generate `nonce = secrets.token_hex(8)` per evaluation.
- Format user + untrusted tool messages inside `<UNTRUSTED_INPUT_{nonce}>\n...\n</UNTRUSTED_INPUT_{nonce}>`.
- Truncate judge input deterministically to `max_input_chars_for_judge` (`8000` chars) to protect local GPU latency.
- Parse judge responses defensively:
  1. Strip optional ```` ```json ... ``` ```` fences and extract the outermost `{...}` JSON object.
  2. Validate `verdict in {"allow", "warn", "block"}` and clamp `confidence` into `[0.0, 1.0]`.
  3. On timeout, HTTP error, or unparseable output: increment `gateway_guardrail_degraded_total` and apply `degradation_mode` (`fail_open_with_warning` emits heuristic verdict + `degraded=True`; `fail_closed` raises `503`).

### 1.7 Client-Safe Warning Injection

Always set `X-Gateway-Guardrail-*` HTTP headers on `warn` and `degraded` responses. Inject in-band markdown warning text **ONLY** when ALL of the following hold:
- `guardrails.inject_in_band_banner == True`
- Request is `/v1/chat/completions` (never `/v1/embeddings`)
- `X-Gateway-Warning-Mode` header is not `"headers-only"`
- `payload.get("response_format", {}).get("type")` is not `"json_object"` or `"json_schema"`
- Response is not pure `tool_calls` (for SSE, inject an initial assistant content delta frame `data: {"id":...,"object":"chat.completion.chunk","choices":[{"index":0,"delta":{"role":"assistant","content":banner},"finish_reason":null}]}\n\n` only when appropriate or prepend to the first content chunk).

### 1.8 Serialized Hash Chain & Secret Scrubbing

- `SQLiteAuditStore` holds a single `asyncio.Lock` around `SELECT entry_hash FROM audit_logs ORDER BY id DESC LIMIT 1` + `INSERT INTO audit_logs`.
- `structlog` installs `scrub_secrets_processor` as the final processor before JSON rendering, masking `gw_live_[A-Za-z0-9_-]+`, `sk-[A-Za-z0-9_-]+`, `Bearer\s+[A-Za-z0-9._-]+`, and `-----BEGIN [A-Z ]+PRIVATE KEY-----[\s\S]+?-----END [A-Z ]+PRIVATE KEY-----`.

---

## 2. Offline Test Suite Specification Matrix

All tests run completely offline inside `.venv` using `tmp_path` SQLite DBs, deterministic test pepper (`b"test-pepper"`), and `httpx.MockTransport` emulating OpenAI, OpenRouter, local vLLM/Ollama, and the local judge model.

| Test File | Mandatory Cases & Assertions |
| :--- | :--- |
| `tests/test_detectors.py` | **Classification:** `TOP SECRET`, `CONFIDENTIAL`, `TLP:RED` match while `"keep it a secret"` does NOT. **Secrets:** OpenAI/OpenRouter/AWS/GitHub/PEM/JWT/DB URI/high-entropy match while normal UUIDs/git SHAs do NOT. **PII:** valid vs invalid Luhn card, mod-11 NRIC, mod-97 IBAN, SSN, email, phone. **Infra/Org:** RFC1918 vs public IPs (`8.8.8.8`), `.internal`, `org_terms.yaml` codenames. **Code:** copyright headers & large code blocks. **Anti-evasion:** `A\u200bKIA...` detected; overlapping `SECRET` beats `PII`. |
| `tests/test_policy_and_vault.py` | Tiered precedence (`BLOCK > REROUTE_LOCAL > REDACT > PSEUDONYMIZE > ALLOW`); fail-closed `503` when detector raises/times out on external target; multimodal `image_url` blocked externally unless role grants permission; `X-Gateway-No-Fallback: true` flips reroute into `403`; `TokenVault` mints consistent `[EMAIL_1]`, ignores unminted `[FAKE_1]`, clears cleanly in `finally`. |
| `tests/test_guardrails.py` | Heuristic detection (`< 2 ms`) for evasion/jailbreak; `LocalLLMJudge` assertions verifying nonce wrapper and `trust_zone == "local"`; `WARN` headers + markdown banner on normal chat; automatic banner suppression for `json_object`, `json_schema`, `tool_calls`, and `X-Gateway-Warning-Mode: headers-only`; high-confidence `BLOCK`; timeout/garbage JSON under `fail_open_with_warning` (`degraded=True`) vs `fail_closed` (`503`); post-flight `ToolCallGuard` flagging `rm -rf /` in sync and SSE tool calls. |
| `tests/test_stream_restorer.py` | Exhaustive fuzzing splitting multi-placeholder responses at every character index (`step=1, 2, 3, random`) across `delta.content` and `delta.tool_calls[].function.arguments`; preserves ordinary brackets `arr[0]` and `[INFO]` without stalling. |
| `tests/test_auth_and_quotas.py` | Raw SQL inspection proves zero plaintext `gw_live_` keys in `api_keys`; invalid/revoked/expired keys return `401`; RBAC rejects disallowed external models (`403`) and filters `/v1/models`; RPM sliding window returns `429` + `Retry-After`; daily token/USD budget blocks overrun. |
| `tests/test_audit_chain.py` | Runs block, reroute, pseudonymize, and guardrail-warn requests; asserts zero raw secrets/PII exist anywhere in SQLite `audit_logs` or JSONL file; asserts `verify_chain()` passes cleanly, then mutates one SQL column directly and asserts `verify_chain()` fails pointing to the tampered `request_id`. |
| `tests/test_e2e_proxy.py` | Full ASGI `httpx.AsyncClient` E2E tests: OpenAI sync & SSE restore with zero raw PII on mock wire; OpenRouter payload asserts `"provider": {"data_collection": "deny"}`; proprietary code auto-reroutes to `local/qwen2.5-coder` with governance headers; secret returns `403` with zero mock calls; guardrail warning banner + headers appear end-to-end; `/v1/embeddings`, `/healthz`, `/readyz`, `/metrics`, `/admin/policy/dry-run`, `/admin/audit/verify` all verified. |
| `tests/test_performance.py` | Benchmarks deterministic `TextNormalizer` + `DetectorEngine` + `PolicyEngine` + `HeuristicGuardrail` over 100 iterations on realistic 4KB prompts offline; asserts **p95 `< 10 ms`**. |

### Final Verification Command

```bash
.venv/bin/pytest -v && .venv/bin/ruff check src tests && .venv/bin/mypy src
```
