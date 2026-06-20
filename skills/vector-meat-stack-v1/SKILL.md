---
name: vector-meat-stack-v1
description: Use when the user asks about Vector Meat — which OSS components to deploy, at which AI-maturity stage (1-5), for a meat-processing plant. Triggers on "vector-meat stack", "стадия 2 стек", "что ставить на мясозаводе", "ИИ-трансформация мясопереработки", "Vector Meat стадия", "AI maturity meat plant".
version: 1.0.0
author: Hermes Agent
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [Vector-Meat, Meat-Plant, AI-Maturity, OSS-Stack, Manufacturing, FoodTech, Russia]
    related_skills: [github-repo-research]
---

# Vector Meat Stack v1

Recommended open-source component stack for Vector Meat (meat-processing plant, no slaughter), mapped to the 5-stage AI-maturity model from `docs/ai-implementation-plan.md`. Source of truth for component choices: `enterprise_ai_ecosystem_oss_catalog_ru_v2.pdf` (Малышев, 11.06.2026), Russian Federation regulations — Меркурий (эВСД), Честный Знак, ХАССП.

**Core rule (carries over from Vector Meat principle #1):** virtual worker prepares → human approves. AI never signs into Меркурий/ЧЗ, never signs financial documents. Component choices below enforce this at architecture level, not just policy level.

## When to Use

- "на какой мы стадии?" / "what stage are we at?"
- "что развернуть на стадии 2/3/4?" / "which OSS at stage N?"
- Building a new Vector Meat deployment — pick components by maturity stage
- Reviewing an existing deployment against the recommended stack
- Estimating budget for the next stage transition
- Comparing alternatives within one component class (vector DB, LLM gateway, etc.)

## When NOT to Use

- Slaughterhouses or livestock operations (different regulatory regime)
- Retail / HoReCa downstream — see Vector Marketing
- Pure software work (no production line) — see Vector Office / Vector Work
- Questions about HOW a specific component works — read that component's own docs

## Stage Matrix (One-Line View)

| Stage | Maturity | Components to add |
|-------|----------|-------------------|
| 1 | Resistance | None — audit only |
| 2 | Helper | Keycloak + OpenBao + LiteLLM + NeMo Guardrails + Ollama/vLLM (local) |
| 3 | Pilots | + OPA + Docling + pgvector + OpenSearch + Langfuse + OpenTelemetry + Prometheus/Grafana + OpenSandbox + Gryph + garak + promptfoo + Trivy |
| 4 | Platform | + Apache Kafka/Red Panda + ClickHouse/TimescaleDB + OpenMetadata + OpenCost + OpenMeter + OpenCode/Qwen Code + Gryph audit |
| 5 | AI-Native | + Skills publication pipeline (47 catalog) + multi-vendor + КИИ/ФСТЭК certification |

Pick ONE primary + ONE reserve per class. Do NOT deploy the whole matrix at once.

## Stage 1 — Resistance to AI

**No new components.** Goal: baseline measurement and audit.

- Audit узких мест (where manual work makes critical decisions)
- Audit данных (what is digitised, what is paper, what is in heads)
- Зафиксировать знания технологов (recipes, regimes, yield norms)
- Internal survey: where do people spend the most time on routine?
- Baseline metrics for each pain point (Меркурий errors, Честный Знак fines, ХАССП incidents, yield deviation)

Russian specifics: Меркурий and Честный Знак are already in production — this is the foundation to build on. Data already flows into state systems; harvest it before adding anything new.

## Stage 2 — AI as Helper

**Минимальный стек первой волны.** Six components. Each one closes a specific pain point in the plan.

| Class | Component | License | Why at this stage |
|-------|-----------|---------|-------------------|
| IAM / SSO | Keycloak | Apache-2.0 | OIDC, OAuth2, SAML, LDAP/AD federation, service accounts for AI workers. Without it every pilot invents its own creds. |
| Secrets | OpenBao | MPL-2.0 | Strict open-source Vault fork. Short-lived credentials for API Меркурий/ЧЗ. Without it, AI workers get long-lived API keys — catastrophe if leaked. |
| LLM Gateway | LiteLLM | MIT-like | One entry point for GigaChat / YandexGPT / local Ollama. Routing, fallback, rate limits, audit. Closes the "choose 1 LLM provider" problem: keep primary + reserve without rewriting clients. |
| Guardrails | NeMo Guardrails | Apache-2.0 | Programmable rails over LLM. Topic restrictions, no hallucinated эВСД numbers, allowlist tools. Critical for Спец Меркурий/ЧЗ — guard against auto-filling wrong shipment data. |
| Local LLM runtime | Ollama (workstation) / vLLM (GPU server) | Apache-2.0 / Apache-2.0 | On-premise inference. Enforces Vector Meat principle #2 ("recipes never leave the plant"). Ollama for demos, vLLM for production-grade GPU. |
| Observability (minimal) | Langfuse base + OpenTelemetry | OSS / Apache-2.0 | LLM tracing from day one. Without it, when pilot #2 starts you cannot compare token usage per worker. |

**Russian specifics (Stage 2):**
- Гибридный режим для Меркурий/ЧЗ: автоматический + ручной с быстрым переключением. LiteLLM should route to a "manual fallback" mode that pauses API calls when gov-system APIs are unstable.
- Choose 1 LLM provider with closed corporate API (YandexGPT corporate or GigaCloud) as primary; local Ollama as reserve for sensitive recipes.
- Policy enforcement at Keycloak level: which human operator approves which AI worker action (for ЭЦП flow).

**What NOT to deploy at Stage 2:** OPA, Docling, OpenSearch, OpenSandbox, Gryph, garak, promptfoo, Trivy, Kafka, ClickHouse, OpenMetadata, coding agents. These belong to Stage 3.

**Budget reference:** plan estimates 200-500k RUB for Stage 1→2 transition. Six components above fit comfortably within this (most are OSS with minimal infra cost).

## Stage 3 — Local Pilots

Three to four pilots in flight. Data flowing from production lines. Add the integration, observability, and AI security layers.

**Add to Stage 2 stack:**

| Class | Component | License | Why at this stage |
|-------|-----------|---------|-------------------|
| Policy-as-code | Open Policy Agent (OPA) | Apache-2.0 | Allowlist for AI worker actions, conditional access. When Gryph starts logging tool calls, OPA decides what is allowed. |
| Document parsing | Docling + Docling Serve | MIT | Ingest входящей нормативки (ГОСТы, ТР ТС, supplier specs, ХАССП protocols). Powers pilot #1 (recipe нормы закладки) by feeding the ML model. |
| Vector DB (start) | pgvector | PostgreSQL | First RAG over recipes — alongside existing 1С PostgreSQL. Simplest infra. Migrate to Qdrant/Milvus only if >5-10k chunks. |
| Hybrid search | OpenSearch | Apache-2.0 | Keyword + vector + filters in one query. Technologist UI uses this when searching recipes by both ingredient and semantic intent. |
| LLM tracing | Langfuse (full) | OSS (basic) | Trace ID per request, link to user/agent/model/tool. Tokens, latency, cost, retries, cache hit rate. Without this, "Аналитик производства" cannot optimise. |
| Metrics stack | OpenTelemetry + Prometheus + Grafana | Apache-2.0 | Production-line metrics + LLM metrics in one Grafana. Per the plan: "кнопочный интерфейс, крупные цифры на простом фоне, не Grafana с 30 графиками" — keep dashboards minimal. |
| Sandbox | OpenSandbox / E2B / Firecracker | Apache-2.0 (varies) | Isolate any executable code Куратор ИИ produces. Required BEFORE any coding agent. |
| Coding-agent audit | Gryph | Apache-2.0 | Pre-tool/post-tool hooks, audit log of file reads/writes and commands in SQLite. Required for any coding agent access. |
| LLM red team | garak | Apache-2.0 | Probe LLM for jailbreak, prompt injection, hallucination. Run before deploying any new model in production. |
| Functional evals | promptfoo | MIT | Scenario tests like "оформи эВСД по партии №..." with expected output patterns. |
| Container/CVE scan | Trivy | Apache-2.0 | CVE + misconfiguration scans for any new container image (Docling Serve, OpenSearch, custom workers). |
| Secrets scan | Gitleaks + Semgrep | MIT / LGPL-2.1 (engine) | Before any skill is published, scan for leaked creds and bad patterns. |

**Pilot ↔ Component mapping (cross-reference with plan):**

| Pilot from plan | Components that power it |
|-----------------|--------------------------|
| Pilot 1 — ИИ-расчёт норм закладки | Docling (parse past records) + pgvector (recipe RAG) + local vLLM (inference) + Langfuse (eval model output) |
| Pilot 2 — Видеоаналитика цеха | Separate CV stack (Macroscop/Ivideon in plan) — this skill does not cover CV vendors |
| Pilot 3 — Предиктивная ТО холода | Modbus/OPC-UA → TimescaleDB → Grafana; ML model is separate, not in this OSS stack |
| Pilot 4 — Авто-разметка и контроль маркировки | OpenCV-based pipeline; CV again outside this stack's scope |

**Russian specifics (Stage 3):**
- ClickHouse is a Russian-developed product — no import-substitution issues.
- Kafka → Red Panda (Aeroflot variant or upstream Red Panda) for import-substitution.
- LLM choice: GigaChat API (Сбер) or YandexGPT corporate.
- "Зоопарк" risk: cameras from vendor A, sensors from vendor B, LLM from vendor C. Address with integration bus (MQTT or Kafka) — but Kafka itself is Stage 4.

**What NOT to deploy at Stage 3:** ClickHouse, Apache Kafka/Red Panda, OpenMetadata, OpenCost, OpenMeter, coding agents (OpenCode, Qwen Code), Gryph as primary audit layer for coding agents (it IS used, but the coding agent itself waits for Stage 4).

**Budget reference:** plan estimates 2-5M RUB for Stage 2→3.

## Stage 4 — AI in the Core

Process reengineered. Production line sends data, ИИ suggests actions, human approves. Add the platform layer.

**Add to Stage 3 stack:**

| Class | Component | License | Why at this stage |
|-------|-----------|---------|-------------------|
| Message bus | Apache Kafka / Red Panda | Apache-2.0 | Single integration bus. Cameras, sensors, 1С, LLM workers all publish/subscribe. Closes the "зоопарк" pain point. |
| Data lake | ClickHouse / TimescaleDB | Apache-2.0 | Time-series from production + events. Both are Russian-friendly. |
| Data catalog / lineage | OpenMetadata | Apache-2.0 | Owner of each model/index/recipe/prompt. Lifecycle: draft → test → approved → deprecated. |
| FinOps | OpenCost + OpenMeter | Apache-2.0 | Cost per AI worker per day. Куратор ИИ sees the bill, not surprises. |
| Coding agent | OpenCode | MIT | Primary OSS coding agent. Runs in isolated workspace per task. |
| Coding agent (alternative) | Qwen Code | Apache-2.0 | Has built-in Skills + SubAgents. Better when skill ecosystem matters more. |
| Coding agent audit | Gryph (full integration) | Apache-2.0 | Hooks on OpenCode/Qwen Code. Logs every file read/write, every command. SIEM-bound. |

**Coding agent security (mandatory from plan + catalog раздел 42.1):**
- Isolated workspace per task
- Pre-tool policy hook via OPA
- Human approval for: package install, shell escalation, network, destructive filesystem actions
- Sandbox runtime: OpenSandbox / E2B / Firecracker / gVisor
- Egress proxy allowlist
- Gryph + OpenTelemetry events → SIEM (Wazuh / OpenSearch)
- Persist: diff, commit SHA, review result

**Russian specifics (Stage 4):**
- Astra Linux as OS baseline
- Сертификация ПО for integration with Меркурий/ЧЗ
- Multi-vendor where possible (single-vendor = sanction risk)

**What NOT to deploy at Stage 4:** public skills registry without quarantine pipeline, multi-region active-active setups (overkill for a single plant), blockchain traceability (Stage 5).

**Budget reference:** plan estimates 10-25M RUB for Stage 3→4.

## Stage 5 — AI-Native Business

Process reengineered across the supply chain. AI agents for procurement, quality, sales. Add the security/certification layer.

**Add to Stage 4 stack:**

- Agent Skills publication pipeline (раздел 47 of catalog): quarantine Git → ClamAV+YARA → Semgrep+Gitleaks → AI-Infra-Guard Skills Scan → human review → OpenSandbox tests → Syft SBOM + Cosign signature → internal Harbor/ORAS mirror → OPA allowlist → Gryph/OTel/SIEM runtime
- AI Security scanner: AI-Infra-Guard (Tencent) for OpenClaw scan + Agent Scan + MCP Server scan
- ClawSec for OpenClaw/Hermes drift detection
- Multi-vendor strategy: минимум 2 поставщика на каждый критический компонент
- Резервный контур на случай отключения от интернета
- КИИ сертификация (ФСТЭК) — обязательна для пищевых производств с 2025
- Пентесты и аудит безопасности ежеквартально
- Internal AI University / наставничество (not a component, but a process change)

**Russian specifics (Stage 5):**
- Премьер-министр подписал план внедрения «Индустрии 4.0» в пищевой отрасли до 2030 года — возможны субсидии и льготы
- Релокация ИИ-специалистов через программы Минсельхоза
- Льготная ипотека для аграриев и переработчиков

**Budget reference:** plan estimates 50+M RUB for Stage 4→5.

## Quick Reference: Component → License → Stage

```
Keycloak           Apache-2.0   S2
OpenBao            MPL-2.0      S2
LiteLLM            MIT-like     S2
NeMo Guardrails    Apache-2.0   S2
Ollama             Apache-2.0   S2 (workstation)
vLLM               Apache-2.0   S2 (GPU server)
Langfuse           OSS basic    S2 → S3
OpenTelemetry      Apache-2.0   S2 → S3
OPA                Apache-2.0   S3
Docling            MIT          S3
Docling Serve      MIT          S3
pgvector           PostgreSQL   S3
OpenSearch         Apache-2.0   S3
Prometheus         Apache-2.0   S3
Grafana            AGPL-3.0     S3
OpenSandbox        Apache-2.0   S3
E2B                Apache-2.0   S3 (alt)
Firecracker        Apache-2.0   S3 (alt)
gVisor             Apache-2.0   S3 (alt)
Gryph              Apache-2.0   S3 (audit) → S4 (full)
garak              Apache-2.0   S3
promptfoo          MIT          S3
Trivy              Apache-2.0   S3
Gitleaks           MIT          S3
Semgrep            LGPL-2.1     S3
Apache Kafka       Apache-2.0   S4
Red Panda          Apache-2.0   S4 (import-substitution)
ClickHouse         Apache-2.0   S4
TimescaleDB        Apache-2.0   S4 (alt)
OpenMetadata       Apache-2.0   S4
OpenCost           Apache-2.0   S4
OpenMeter          Apache-2.0   S4
OpenCode           MIT          S4
Qwen Code          Apache-2.0   S4 (alt)
Kilo Code          MIT          S4 (compare)
AI-Infra-Guard     Apache-2.0   S5
ClawSec            AGPL-3.0+    S5
Cosign             Apache-2.0   S5
Syft               Apache-2.0   S5
ClamAV             GPL-2.0      S5
YARA               BSD-3-Clause S5
Harbor / ORAS      Apache-2.0   S5
```

## Component Selection Within a Class

When more than one option fits, use these rules:

- **Vector DB:** pgvector if you already have 1С PostgreSQL and want to start fast. Qdrant if you need a dedicated retrieval service with filters. Milvus if you need distributed scale. OpenSearch if you also need keyword/hybrid. Chroma/LanceDB/FAISS for local PoC only.
- **Local inference runtime:** Ollama for laptops/demos. vLLM for production GPU servers (high throughput). SGLang for high-performance LLM/VLM on GPU. KServe for Kubernetes inference platform. llama.cpp for CPU/edge/GGUF. Triton for mixed ML with batching/ensembles. TEI or Infinity for embeddings/rerankers.
- **Coding agent (Stage 4):** OpenCode as primary (MIT, terminal-native). Qwen Code as alternative (Apache-2.0, has Skills/SubAgents built-in). Kilo Code for comparative testing. OpenHands/Aider in short list for complex tasks. Open Interpreter only in separate sandbox with strict human control. **Do not re-deploy Roo Code** — archive notice by owner 15.05.2026.
- **Graph DB for GraphRAG (Stage 3+):** Apache AGE if PostgreSQL-centric (no new service). JanusGraph for large distributed property graphs. Neo4j Community for single-server. **Memgraph** — useful in real-time GraphRAG scenarios, but upstream uses BSL (Community) and MEL (Enterprise), so do NOT auto-place in strict base set; verify license before use.

## Common Pitfalls (Vector Meat Specific)

1. **Stage-jumping.** "Let's deploy Kafka + ClickHouse + OpenMetadata at Stage 2 because the consultant said so." Result: infra team drowns, plant workers see zero benefit. Stick to the matrix.

2. **No IAM at Stage 2.** Starting with LiteLLM + NeMo Guardrails but skipping Keycloak/OpenBao. Each pilot ends up with its own API key in a `.env` file on someone's laptop. By Stage 3 you have 12 scattered creds and one already leaked.

3. **Treating Ollama = production.** Ollama is great for laptop demos. For Stage 2 production, vLLM. For multi-tenant GPU farms, KServe. Mixing them up wastes weeks.

4. **LLM with public recipes.** Letting technologists paste recipes into ChatGPT web. The plan's principle #2 ("данные локально") means local Ollama/vLLM only for proprietary recipes. YandexGPT corporate API has data residency but verify the SLA before treating it as "local".

5. **NeMo Guardrails without rail config.** Installing guardrails but not writing the rails. Result: guardrails ship in default mode, allow most topics, give false sense of security. For Спец Меркурий/ЧЗ write explicit rails: "do not invent эВСД numbers", "always cite the source shipment record", "refuse if shipment data is ambiguous".

6. **Langfuse as optional.** "We'll add tracing later when we have budget." By Stage 3 you have 5 workers, no idea which one burns tokens, and zero ability to optimise. Add it at Stage 2 even if dashboards are empty.

7. **Coding agent without Gryph.** Deploying OpenCode because it's MIT and looks productive. Without Gryph hooks, every `rm -rf` is one prompt-injection away. Stage 4 demands Gryph + OPA + sandbox + SIEM as a bundle, not à la carte.

8. **Forgetting import-substitution.** Kafka → Red Panda; ClickHouse is fine; Astra Linux instead of Ubuntu for КИИ. Plan says "Полный контур на российском ПО" by Stage 4 — start planning at Stage 3.

9. **Treating BSL/MEL/AGPL as MIT.** Memgraph BSL, ClawSec AGPL-3.0+, Grafana AGPL-3.0, Neo4j Community, ZITADEL AGPL-3.0 — these are NOT strict OSS for enterprise. Check with legal before commercial deployment, especially anything that touches the production line data.

10. **No audit baseline for Stage 1.** Deploying Keycloak at Stage 2 without having logged how Меркурий errors actually happen today. Result: cannot tell if IAM reduced Меркурий errors or not.

## Verification Checklist

- [ ] Confirm current stage against the plan (don't guess — re-read `docs/ai-implementation-plan.md`)
- [ ] Confirm each component in current stack has a license compatible with planned use (BSL/AGPL = legal review)
- [ ] Confirm IAM + secrets present before any new LLM/API integration
- [ ] Confirm guardrails configured (not just installed) for any worker that touches Меркурий/ЧЗ/1С
- [ ] Confirm Langfuse/OTel receiving traces from every active worker
- [ ] Confirm human-in-the-loop enforced at component level (Keycloak SSO, OPA policy, ЭЦП separation) — not just "policy doc says so"
- [ ] Confirm data residency: recipes never leave plant network (check LiteLLM routing config + Ollama/vLLM deployment)
- [ ] Budget per stage transition matches plan estimate within 2× (if not, re-evaluate scope)
- [ ] Plan was updated (`docs/ai-implementation-plan.md`) BEFORE component deployment, not after
- [ ] Куратор ИИ owns the stack map, not IT alone — cross-functional committee per plan раздел 3.3

## Related Skills & Resources

- `github-repo-research` — when investigating a new component before adding to stack
- Vector Meat repo: `https://github.com/Osmosy/vector-meat`
- Vector Work repo: `https://github.com/Osmosy/vector-work` (virtual worker roles)
- Vector Agent Ready repo: `https://github.com/Osmosy/vector-agent-ready` (methodology)
- Source catalog: `enterprise_ai_ecosystem_oss_catalog_ru_v2.pdf` (Малышев, 11.06.2026)
- Plan: `docs/ai-implementation-plan.md` (Vector Meat)
- Staffing: `docs/staffing-v1.md` (Vector Meat)
