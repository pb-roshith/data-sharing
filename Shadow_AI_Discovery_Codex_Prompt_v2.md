# Codex Prompt v2 – Shadow AI Discovery Prototype (2-week build)

> v2 adds: the full demo environment (repositories, keys, local model, MCP server, proxy, DLP, gateway), exact reference data, the signal schema, the correlation rules, and the complete **confidence and risk scoring model** with expected results per scenario.

---

## 1. Context

Build a working prototype of a **Shadow AI Discovery engine**. It has four jobs:

1. Detect AI activity from real local telemetry on one machine.
2. Correlate related signals into **AI usage candidates**.
3. Reconcile candidates against an approved AI inventory.
4. Score **discovery confidence** and **AI risk** separately, then make an explainable decision.

The decision is one of: **Genuine**, **Monitor**, **Raise issue** or **False positive**.

The USP is the pipeline **detect → correlate → reconcile → score → explainable decision**. This is not a statistics dashboard. Every decision must show its evidence, the points each factor earned and why, and which rule fired.

## 2. Environment and constraints

- **Machine:** one Windows Azure Virtual Desktop (AVD) with admin rights. No other servers; everything runs as local processes.
- **Stack:** Python 3.11, FastAPI, SQLite, React (Vite) + Tailwind. Keep dependencies minimal and permissively licensed.
- **Root folder:** `C:\ShadowAI-Demo\`. **All scanning is limited to this folder.** Never scan the user profile, OneDrive or other drives.
- **Metadata only:** never store prompts, payloads, file contents or secret values. Secrets are stored as a SHA-256 fingerprint plus `first4…last4`.
- **Fake keys only:** all keys in repos are fake and contain `DEMO`.
- **Deterministic decisions:** detection, correlation, scoring and decisions are rule-based. An LLM is optional and may only write the issue narrative.
- **Fictional data only:** all people, customers and account numbers are fictional.

## 3. Folder structure

```
C:\ShadowAI-Demo\
  repos\        credit-dossier\ loan-ops-helper\ readme-only\ treasury-reports\  (ahmed-scratch\ created live by S7)
  endpoint\     models\ (Ollama model store)   ahmed\.vscode\mcp.json   endpoint_profile.yaml
  services\     proxy\ (mitmproxy add-ons)   gateway\ (LiteLLM config)   mcp_creditdb\ (our MCP server)   creditdb\ (SQL)
  logs\         proxy.jsonl  dlp.jsonl  gateway.jsonl  mcp_audit.jsonl
  reference\    users.csv devices.csv cmdb_applications.csv repo_register.csv approved_ai_inventory.csv ai_signatures.yaml
  scenarios\    s0_background.py s1_approved.py s2_shadow_api.py s3_pii_upload.py s4_local_model.ps1 s5_agent.py s7_live.py
  engine\       collectors\ detect.py correlate.py reconcile.py score.py decide.py db.py api.py
  app\          React UI
  setup\        install.ps1  start.ps1  reset_demo.ps1
  tests\        test_scenarios.py  test_no_secrets_stored.py
```

---

## 4. Demo environment to build (Days 1–3)

### 4.1 Proxy: mitmproxy on `127.0.0.1:8080`

- Proxy authentication with one login per demo identity: `app-svc`, `gateway-svc`, `dev-svc`, `priya`, `ahmed` (password `demo`).
- **Add-on `log_addon.py`** writes one line per request to `logs\proxy.jsonl`:
  `{ts, proxy_user, host, method, path_class, bytes_out, status}`.
  - `path_class` is a coarse label such as `api` or `upload`. Never store the full URL query or body.
- **Add-on `dlp_addon.py`** runs only for requests to AI hosts in the signature catalogue. It checks the request body against these test patterns:
  - **Customer PII:** account number `\bACC-\d{10}\b`, or fake PAN `\b[A-Z]{5}\d{4}[A-Z]\b`.
  - **Confidential:** the marker `CONFIDENTIAL-INTERNAL`.
- When a pattern matches, the DLP add-on does three things:
  1. Writes `{ts, proxy_user, host, label, match_count, bytes}` to `logs\dlp.jsonl`. It never writes the body.
  2. Returns **HTTP 403** so that the data never leaves the AVD.
  3. Still lets the request be recorded in `proxy.jsonl`.
- Scenario scripts use the proxy via `HTTPS_PROXY=http://<user>:demo@127.0.0.1:8080` and trust the mitmproxy CA (`REQUESTS_CA_BUNDLE` / `SSL_CERT_FILE`). Only scenario scripts are configured this way, not the whole machine.

### 4.2 AI gateway: LiteLLM proxy on `127.0.0.1:4000`

- One route: `mistral-small` → `mistral/mistral-small-latest`.
- `MISTRAL_API_KEY` is read from an environment variable, never from a repo.
  - A real key is optional. Without one, the calls fail but are still logged.
- Outbound traffic goes through mitmproxy as `gateway-svc`.
- A custom success/failure callback writes to `logs\gateway.jsonl`:
  `{ts, app (from request metadata/header x-app-id), model, provider, tokens, status}`.

### 4.3 Repositories (local git only, never pushed)

| Repo | Contents | Commit history |
|---|---|---|
| `credit-dossier` | `requirements.txt` with `mistralai`, `httpx`. `dossier/llm.py` imports `mistralai` and calls the gateway at `http://127.0.0.1:4000` with header `x-app-id: credit-dossier`. `config/gateway.yaml` contains the model name. **No key anywhere.** | 3 commits |
| `loan-ops-helper` | `requirements.txt` with `openai`. `helper/summarise.py` uses `from openai import OpenAI` and calls the default OpenAI endpoint (bypassing the gateway). | Commit 1: code. Commit 2: adds `.env` with `OPENAI_API_KEY=sk-proj-DEMO<40 random chars>`. Commit 3: deletes `.env` and adds `.gitignore`. **The key survives only in history.** |
| `readme-only` | `README.md` contains the sentence "Future idea: summarise reports with OpenAI." Plain Python with no AI imports. | 2 commits |
| `treasury-reports` | Plain Python report code (pandas/csv). No AI at all. This is the control repo. | 2 commits |
| `ahmed-scratch` | **Created live by S7**: `test_claude.py` imports `anthropic`. | 1 commit |

### 4.4 Local model (S4)

- Install Ollama for Windows.
- Set `OLLAMA_MODELS=C:\ShadowAI-Demo\endpoint\models`, then run `ollama pull qwen2.5:0.5b`.
- `s4_local_model.ps1` starts `ollama serve` (port 11434) and runs one local prompt. There is no internet traffic.
- `endpoint\endpoint_profile.yaml` provides the simulated device context for this single machine:
  `device_label: AVD-ANL-03, primary_user: ahmed`.

### 4.5 Agent and MCP (S5)

- **Database:** a PostgreSQL database `creditdb` with tables `loans(loan_id, customer_ref, amount, status)` and `limits(...)`. Load about 20 fictional rows.
- **MCP server:** our own Python MCP server (official `mcp` Python SDK) in `services\mcp_creditdb\server.py`, running persistently over Streamable HTTP on `127.0.0.1:8765`.
  - Tool `query_loans` (read-only, annotation `readOnlyHint=true`).
  - Tool `execute_sql` (write, annotation `destructiveHint=true`).
  - Every tool call is appended to `logs\mcp_audit.jsonl`: `{ts, server, tool, caller}`. Arguments are never logged.
- **Config:** `endpoint\ahmed\.vscode\mcp.json` registers two servers:
  - `creditdb` at `http://127.0.0.1:8765/mcp`.
  - A harmless read-only `docs` server, for contrast. This can be a second tiny server we write.
- **Agent:** `s5_agent.py` (caller `risk-agent`) connects and calls `query_loans` once and `execute_sql` once (an UPDATE on a demo row).

### 4.6 Scenario scripts (each is idempotent and re-runnable)

| Script | Does |
|---|---|
| `s0_background.py` | About 200 benign requests through the proxy (e.g. example.com, microsoft.com, wikipedia.org) as random demo users, so AI traffic is not the only traffic |
| `s1_approved.py` | About 20 calls credit-dossier → gateway (`x-app-id: credit-dossier`). Proxy user for the client is `app-svc` |
| `s2_shadow_api.py` | About 12 calls from `loan-ops-helper/helper/summarise.py` directly to api.openai.com as `dev-svc`, using the DEMO key from env. Expect 401 responses, which is fine |
| `s3_pii_upload.py` | As `priya`, a single POST of `dummy_customer_extract.csv` (contains 14 `ACC-##########` values) to chatgpt.com. Gets a DLP hit and a 403 |
| `s4_local_model.ps1` | Starts Ollama and runs one local prompt |
| `s5_agent.py` | Runs the agent as described in 4.5 |
| `s7_live.py` | **Live moment.** Creates `repos\ahmed-scratch`, commits `test_claude.py` (imports `anthropic`), then runs about 5 calls as `ahmed` to api.anthropic.com with a DEMO key from env. One request includes the marker `CONFIDENTIAL-INTERNAL`, which gets a DLP hit labelled Confidential |

- `reset_demo.ps1` clears logs, the SQLite DB and `ahmed-scratch`, and stops Ollama.
- `start.ps1` starts mitmproxy, LiteLLM, the MCP server, the API and the UI, then runs s0–s5 (not s7).

---

## 5. Reference data (exact contents)

**users.csv**: `user_id, display_name, department, dept_criticality, type`

```
app-svc, Credit Dossier service, Credit Technology, High, service
gateway-svc, AI Gateway service, Enterprise AI Platform, High, service
dev-svc, Lending dev service, Lending Technology, High, service
priya, Priya Sharma, Retail Lending, High, human
ahmed, Ahmed Khan, Risk Analytics, Medium, human
```

**devices.csv**: `device_label, primary_user`

```
AVD-APP-01, app-svc
AVD-DEV-02, dev-svc
AVD-RL-07, priya
AVD-ANL-03, ahmed
```

The proxy user maps to a device through this file.

**cmdb_applications.csv**: `ci_name, ci_type, owner, criticality, data_classification, holds_customer_data`

```
credit-dossier, application, Credit Technology, High, Confidential, yes
creditdb, datastore, Credit Data Platform, High, Confidential, yes
treasury-reports, application, Treasury IT, Medium, Internal, no
ai-gateway, platform, Enterprise AI Platform, High, Internal, no
```

loan-ops-helper is deliberately **absent** from the CMDB.

**repo_register.csv** (source-code platform metadata): `repo, owner, business_unit, service_account, criticality, handles_customer_data`

```
credit-dossier, Credit Technology, Credit, app-svc, High, yes
loan-ops-helper, Ravi Menon, Lending, dev-svc, High, yes
readme-only, Sanjay Iyer, Operations, dev-svc, Low, no
treasury-reports, Treasury IT, Treasury, app-svc, Medium, no
```

**approved_ai_inventory.csv**: `inv_id, asset, match_type, match_value, provider, required_route, status, owner`

```
AI-INV-1022, Credit Dossier GenAI, app, credit-dossier, Mistral, gateway, Approved, Credit Technology
AI-INV-1001, Microsoft 365 Copilot, host, copilot.microsoft.com, Microsoft, any, Approved, IT Productivity
AI-INV-1030, Public ChatGPT, host, chatgpt.com, OpenAI, n/a, Restricted, CISO Office
```

**ai_signatures.yaml**

| Signature type | Values | Provider mapping |
|---|---|---|
| `hosts` | api.openai.com, chatgpt.com | OpenAI |
| | api.anthropic.com, claude.ai | Anthropic |
| | api.mistral.ai | Mistral |
| | generativelanguage.googleapis.com, gemini.google.com | Google |
| | copilot.microsoft.com | Microsoft |
| `sdks` | openai, anthropic, @anthropic-ai/sdk, mistralai, google-generativeai, langchain, llama-index, ollama | — |
| `key_patterns` | OpenAI: `sk-(proj-)?[A-Za-z0-9_-]{20,}` | OpenAI |
| | Anthropic: `sk-ant-[A-Za-z0-9_-]{20,}` | Anthropic |
| | Mistral: 32-char value assigned to a `*MISTRAL*` variable | Mistral |
| `processes` | ollama → port 11434, lmstudio → port 1234 | — |
| `model_file_ext` | .gguf, .safetensors, .onnx, Ollama blob store (`models\blobs\sha256-*`) | — |
| `doc_files` | README*, *.md | Weak evidence only |
| `mcp_config_files` | mcp.json, claude_desktop_config.json | — |

---

## 6. Collectors and signals

Each collector writes raw `observations` (one per item inspected), then `detect.py` turns matches into **signals**.

**Signal schema:** `signal_id, run_id, channel, signal_type, provider, entity, user_id, device_label, fingerprint, host, event_count, ts_first, ts_last, attributes(JSON), source_ref`

| Collector | Channel | signal_type values |
|---|---|---|
| code | Code | `sdk_dependency`, `sdk_import`, `doc_mention` (README/markdown only) |
| secrets | Secret | `credential_in_file`, `credential_in_history` (stores fingerprint and commit hash only) |
| endpoint | Endpoint | `ai_process`, `ai_port`, `model_file` |
| mcp | MCP | `mcp_config_server` (server name and URL), `mcp_server_running` (port open and tool list fetched via MCP `tools/list`, each tool tagged read/write using annotations or names), `mcp_tool_calls` (from mcp_audit, aggregated count) |
| proxy | Network | `ai_network_call` (aggregated per user + host + 10-minute bucket, with event_count and bytes_out) |
| dlp | DLP | `dlp_label` (label, match_count) |
| gateway | Gateway | `gateway_call` (aggregated per app + model) |

**Entity resolution during detection:**

- Code and secret signals get `entity` = repo name.
- `user_id` comes from `repo_register.service_account`, with flag `identity_source=register`.
- Proxy, DLP and MCP signals get `user_id` directly from the evidence, with flag `identity_source=direct`.
- **Gateway egress:** a proxy event from `gateway-svc` is attributed to the gateway app(s) calling the same provider within the same 10-minute bucket. It gets `entity` = that app and the attribute `via_gateway=true`.

---

## 7. Correlation rules (`correlate.py`)

Build a graph where **signals are nodes**. Add an edge when any rule below matches. Each connected component becomes one **candidate**. Use union-find.

| # | Join rule | Example |
|---|---|---|
| J1 | Same `entity` (repo/app name) **and** same provider | credit-dossier code ↔ gateway ↔ gateway egress |
| J2 | Same credential fingerprint | key in history ↔ key seen elsewhere |
| J3 | Repo signal ↔ network signal where `repo_register.service_account == proxy_user` **and** same provider | loan-ops-helper code/secret ↔ dev-svc → api.openai.com |
| J4 | Network ↔ DLP: same user **and** same host, with timestamps within ±2 minutes | priya → chatgpt.com ↔ DLP Customer PII |
| J5 | Endpoint signals on the same device **and** same tool family (process ↔ port ↔ model file of that runtime) | ollama process ↔ 11434 ↔ model blob |
| J6 | MCP config server URL ↔ running server port ↔ audit server name | creditdb config ↔ :8765 ↔ tool calls |

**Over-merge guard.** User alone, device alone, or provider alone is **never** enough to join. Ahmed's local model (S4), agent (S5) and Anthropic use (S7) must remain **three separate candidates**.

**Candidate naming:** `<entity or user> → <provider/tool>`. Store `join_rules_fired` on each candidate for display.

---

## 8. Reconciliation (`reconcile.py`)

Check these in order:

1. If the candidate's entity matches an inventory row with `match_type=app`, the provider matches, **and** the route matches (`required_route=gateway` needs `via_gateway=true` or a gateway_call signal) → **Approved**, linked to `inv_id`.
2. If any network host in the candidate matches an inventory row with `match_type=host` → that row's status (Approved / Restricted / Prohibited).
3. Otherwise → **Unknown**.

Also attach a CMDB match (owner, criticality, data_classification, holds_customer_data) when the entity or its connected datastore (the MCP target) is in the CMDB.

---

## 9. Scoring model (`score.py`)

Both scores range from 0 to 100. For each factor, take the **highest** matching tier (tiers do not add up within a factor). Store, per factor: `points`, `max`, `reason`, and the `signal_ids` used. The UI shows all of these.

### 9.1 Discovery confidence ("How sure are we this is real AI usage?")

| Factor (max) | Points |
|---|---|
| **Direct evidence (30)** | **30:** runtime activity observed (`ai_network_call`, `gateway_call`, `ai_process`). **25:** `mcp_server_running` or a credential. **15:** `sdk_import` / `sdk_dependency`. **10:** `model_file` only. **5:** `doc_mention` only |
| **Identity link (20)** | **20:** user/service account taken directly from evidence (`identity_source=direct`, or endpoint profile). **10:** only via repo_register. **0:** none |
| **Technology correlation (20)** | Distinct signal_types in the candidate. **≥3 → 20.** **2 → 15.** **1 → 5** |
| **Recurrence (15)** | Sum of underlying observations/events. **≥10 → 15.** **2–9 → 10.** **1 → 0** |
| **Independent channels (15)** | Distinct channels. **≥3 → 15.** **2 → 10.** **1 → 0** |

### 9.2 AI risk ("How harmful could it be?")

| Factor (max) | Points |
|---|---|
| **Data sensitivity (25)** | **25:** DLP Customer PII. **20:** DLP Confidential, or an MCP target with data_classification Confidential. **15:** plaintext credential exposed (file or history). **0:** none observed |
| **External exposure (15)** | **15:** direct calls to an external AI host. **5:** external AI only via the approved gateway. **0:** local only |
| **Autonomy / permissions (20)** | **20:** MCP server exposes a write/destructive tool. **10:** MCP read-only, or programmatic API use (service account or script). **5:** interactive human use or local runtime. **0:** no runtime evidence |
| **Business criticality (15)** | From CMDB, then repo_register, then the user's dept_criticality, in that order. **High 15.** **Medium 10.** **Low 5.** **Unknown 8** |
| **Regulatory / customer impact (15)** | **15:** DLP Customer PII, or write access to a store with holds_customer_data=yes. **10:** DLP Confidential, or the app/repo handles customer data. **0:** otherwise |
| **Control weakness (10)** | **+5** if inventory status is not Approved. **+5** if it bypasses the gateway, has a plaintext credential, or has no registered owner for the AI use. Cap at 10 |

### 9.3 Decision rules (`decide.py`). Evaluate in order; the first match wins.

| Rule | Condition | Decision |
|---|---|---|
| R1 | confidence < 40 | **False positive** (auto-close) |
| R2 | inventory = Approved **and** confidence ≥ 70 | **Genuine** (close) |
| R3 | inventory ≠ Approved **and** risk ≥ 70 | **Raise issue** |
| R4 | otherwise | **Monitor** |

Thresholds live in `reference\scoring.yaml` so they can be tuned without code changes.

### 9.4 Expected results. These are the acceptance tests; the engine must reproduce them.

Confidence is the sum of: direct + identity + tech + recurrence + channels.
Risk is the sum of: data + external + autonomy + criticality + regulatory + control.

| Candidate | Signals (types) | Inventory | Confidence | Risk | Decision |
|---|---|---|---|---|---|
| **S1** credit-dossier → Mistral | sdk_dependency/import, gateway_call, ai_network_call (via gateway) | Approved AI-INV-1022 | 30+20+20+15+15 = **100** | 0+5+10+15+0+0 = **30** | **Genuine** (R2) |
| **S2** loan-ops-helper → OpenAI | sdk_import, credential_in_history, ai_network_call | Unknown | 30+20+20+15+15 = **100** | 15+15+10+15+10+10 = **75** | **Raise issue** (R3) |
| **S3** priya → chatgpt.com | ai_network_call, dlp_label(PII) | Restricted AI-INV-1030 | 30+20+15+10+10 = **85** | 25+15+5+15+15+10 = **85** | **Raise issue** (R3) |
| **S4** ahmed → Ollama (local) | ai_process, ai_port, model_file | Unknown | 30+20+20+10+0 = **80** | 0+0+5+10+0+10 = **25** | **Monitor** (R4) |
| **S5** risk-agent → creditdb MCP | mcp_config_server, mcp_server_running(write), mcp_tool_calls | Unknown | 25+20+20+10+0 = **75** | 20+0+20+15+15+10 = **80** | **Raise issue** (R3) |
| **S6** readme-only → OpenAI mention | doc_mention | Unknown | 5+10+5+0+0 = **20** | 0+0+0+5+0+5 = **10** | **False positive** (R1) |
| **S7 (live)** ahmed → Anthropic | sdk_import, ai_network_call, dlp_label(Confidential) | Unknown | 30+20+20+10+15 = **95** | 20+15+10+10+10+10 = **75** | **Raise issue** (R3) |

`treasury-reports` must produce **zero** signals.

Baseline run (s0–s5) → **6 candidates, 3 issues** (S2, S3, S5). After s7 and a re-run → **7 candidates, 4 issues**.

---

## 10. Issues

Create an issue automatically for every R3 decision, or manually from the UI. Each issue contains:

- Title.
- Evidence pack (signal list with source_refs).
- Both score breakdowns with reasons.
- Rule fired.
- Recommended action, from a template per pattern (exposed key, PII to public AI, write-capable agent, unapproved external API).
- Suggested owner, taken from CMDB, then repo_register, then the user's department.

Status workflow: **Open → Investigating → Resolved** (Registered / Remediated / False positive).

**Re-runs must not duplicate issues.** Use a stable candidate key, e.g. a hash of the sorted join identifiers.

---

## 11. API (FastAPI)

| Endpoint | Purpose |
|---|---|
| `POST /discovery/run` | Runs all collectors → detect → correlate → reconcile → score → decide. Returns counts per stage |
| `GET /runs/latest` | Latest run summary |
| `GET /signals?channel=` | Signal list, filterable by channel |
| `GET /candidates` | Candidate list |
| `GET /candidates/{id}` | Signals, join_rules_fired, reconciliation, factor breakdowns, decision trace |
| `GET /issues` | Issue list |
| `GET /issues/{id}` | Issue detail |
| `PATCH /issues/{id}` | Update status, owner or resolution |
| `GET /reference/{name}` | Reference file contents |

## 12. UI: 5 screens (match the wireframe file `Shadow_AI_Discovery_Prototype_Wireframe.html`)

1. **Run Discovery:** collector cards, a Run button, the funnel (items scanned → signals → candidates → issues) and the decision counts.
2. **AI Signals:** table with a channel filter.
3. **AI Usage Candidates:** channels involved, inventory status, confidence, risk and decision.
4. **Candidate Detail (core screen):** evidence chain, join rules fired, reconciliation, both factor breakdowns (points/max **and reason**), the decision trace with the fired rule highlighted, and a Raise/close action.
5. **Issues:** list and detail, evidence pack, reasoning, recommended action, owner and status workflow.

Reference data appears as a read-only tab.

## 13. Two-week plan

| Days | Work |
|---|---|
| 1–3 | Section 4 environment, section 5 reference data, scenario scripts, reset/start scripts |
| 4–6 | Collectors and detection (section 6) |
| 7–8 | Correlation, reconciliation, scoring and decision (sections 7–9). `tests\test_scenarios.py` asserts table 9.4 |
| 9–11 | API and the 5 UI screens |
| 12 | Issue workflow, de-duplication, S7 live run |
| 13–14 | End-to-end dry runs from a clean reset, fixes, demo script |

## 14. Acceptance criteria

- After `reset_demo.ps1` → `start.ps1` → Run discovery, the results exactly match table 9.4 for S1–S6, and treasury-reports has no signals.
- Running `s7_live.py` then Run discovery adds exactly one candidate and one issue.
- Running discovery twice without new activity creates no duplicate issues.
- `test_no_secrets_stored.py` scans the SQLite DB and logs and finds no `sk-` key bodies, DLP payloads or file contents.
- Every candidate detail shows its signals, join rules, per-factor points with reasons, and the fired rule.

**Instruction to Codex:** build in the order of section 13. After each stage, run `tests\test_scenarios.py` and report which S1–S7 expectations pass. If a real tool (LiteLLM callback, Ollama path, MCP transport) behaves differently from this spec, keep the **signal schema and the expected results** unchanged and adapt the collector.
