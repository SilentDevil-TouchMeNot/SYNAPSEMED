<div align="center">

# SynapseMed — Healthcare Context Intelligence Engine (HCIE)

### A general-purpose Context Intelligence Engine, specialized for healthcare emergency orchestration.

**Build context. Not diagnosis. Orchestrate action, not medicine.**

[![Version](https://img.shields.io/badge/version-v0.3.1-6EE7B7?style=flat-square)](#roadmap)
[![Python](https://img.shields.io/badge/python-3.11%2B-3776AB?style=flat-square&logo=python&logoColor=white)](#)
[![MCP Native](https://img.shields.io/badge/MCP-native-4F46E5?style=flat-square)](#)
[![License](https://img.shields.io/badge/license-Open--Source-black?style=flat-square)](#license)
[![Status](https://img.shields.io/badge/status-blueprint%20%2F%20MVP-orange?style=flat-square)](#roadmap)

[Overview](#overview) · [Architecture](#architecture) · [The 7-Layer Engine](#the-7-layer-orchestration-engine) · [MCP Tools](#mcp-tool-surface) · [Roadmap](#roadmap) · [Contributing](#contributing)

</div>

---

## Overview

**HCIE** merges two ideas into a single, coherent system:

- A **general Context Intelligence Engine (CIE)** — one that treats *context construction*, not raw retrieval and not generation, as the first-class problem to solve before any LLM acts.
- A **healthcare emergency orchestrator** — a buildable, text-only system that turns a noisy emergency report into a structured, explainable, human-approved action plan.

Both problems are, at their core, the same problem: given an underspecified request and a large ecosystem of heterogeneous knowledge sources, **construct the best possible structured context, reason over it, and decide what should happen next — before invoking any generative or execution step.**

HCIE is the healthcare vertical demonstration of that engine. The same retrieval → context-graph → planning → validation core that could help you write a JWT auth module is, here, routing a chest-pain patient to the nearest cardiac-capable hospital.

> **HCIE is not a diagnostic system, an image classifier, or a hospital management platform.**
> It is a context-assembly and orchestration engine that happens to operate in the healthcare emergency domain — clinical judgment always remains with clinicians and emergency responders.

---

## First Principle

```
Given the current context, what should happen next?
```

That is the *only* question HCIE answers. It never asks "what is wrong with this patient," and it will never produce a diagnosis, a treatment plan, or medication guidance.

| The Challenge | The HCIE Approach |
|---|---|
| Healthcare systems face noisy, non-standardized emergency input and fragmented, siloed data sources. | HCIE sits between an unstructured request and a structured knowledge ecosystem, and builds an optimal **context graph** — not a diagnosis. |
| AI models face serious hallucination risk when acting as clinical classifiers. | Every plan is schema-checked, safety-checked for diagnostic/dosing language, and gated behind **explicit human approval** before anything executes. |

---

## Architecture

HCIE is organized into three coordination zones — identical in shape to the general CIE:

```
┌─────────────────────────────────┐
│   Client Layer                   │   Claude / ChatGPT / Ops console
│   (any MCP-capable client)       │   — talks JSON-RPC / SSE / HTTP
└───────────────┬───────────────────┘
                │
┌───────────────▼───────────────────┐
│   MCP Gateway                     │   Auth · Session · Tool Registry
│   (single stable entrypoint)      │   Routing · Cache · Approval Gate
└───────────────┬───────────────────┘
                │
┌───────────────▼───────────────────┐
│   HCIE Engine (7 layers)          │   Intent → Plan → Discover →
│                                    │   Retrieve → Graph → Generate →
│                                    │   Validate
└───────────────┬───────────────────┘
                │
┌───────────────▼───────────────────┐
│   MCP Servers (mock-data backed)  │   Patient · Hospital · Contact ·
│                                    │   Maps
└─────────────────────────────────────┘
```

The **gateway** is the one stable endpoint any MCP client talks to. The **engine** is the reasoning core. The **servers** are thin, swappable, JSON-mock-backed tool surfaces — designed so a real EHR/hospital integration can drop in behind the same interface later, with zero change to the engine above it.

---

## The 7-Layer Orchestration Engine

| # | Layer | What it does |
|---|---|---|
| 1 | **Intent Understanding** | Parses raw free text into structured, machine-operable intent (emergency type, symptoms, location, a rules-based severity hint). |
| 2 | **Planning** | Builds the retrieval/execution task graph — what to fetch, in what order, and why. |
| 3 | **Knowledge Discovery** | Dispatches specialized agents — patient, hospital, doctor, contact, maps — concurrently against MCP servers. |
| 4 | **Hybrid Retrieval & Re-ranking** | BM25 lexical search + embedding semantic search, merged and cross-encoder re-ranked, to surface the most relevant hospitals/doctors. |
| 5 | **Context Graph & Priority Inference** | Assembles a typed graph (patient → symptom → history → hospital → doctor → contact) and reasons over it — e.g. *cardiac history + chest pain → prioritize Cardiology + ICU, shortest distance*. |
| 6 | **Generation** | Compresses the graph into a token-budgeted prompt and generates a ranked, explainable execution plan — not code, not a diagnosis. |
| 7 | **Validation** | Schema-checks entity references, rejects diagnostic/dosing language, confirms plan completeness, and generates a human-readable rationale. |

Every layer mirrors its counterpart in the general-purpose CIE — same graph-reasoning mechanism, same validation discipline, different node semantics.

### Example

**Input**

```
Patient: John
Emergency Type: Medical
Symptoms: Chest pain, difficulty breathing, sweating, unconscious
Location: Kakkanad
```

**Structured Intent (Layer 1 output)**

```json
{
  "task": "emergency_orchestration",
  "emergency_type": "cardiac_suspected",
  "patient_name": "John",
  "symptoms": ["chest_pain", "difficulty_breathing", "sweating", "unconscious"],
  "location": "Kakkanad",
  "severity_hint": "high"
}
```

**Generated Plan (Layer 6→7 output — proposed, not executed)**

```
1. Call ambulance
2. Notify family
3. Navigate to Hospital A
4. Share patient summary
5. Connect to available physician
```

Nothing in that plan fires until a human explicitly approves it.

---

## Architectural Evolution — v0.3 → v0.3.1

v0.3.1 collapses the original 9-agent wrapper hierarchy into two core agents (**Chief** and **Execution**) and moves discovery from a sequential chain to an **async, concurrent fan-out** — bounding discovery latency to the slowest single lookup instead of the sum of all four.

| Dimension | v0.3 (legacy) | v0.3.1 (optimized) | Impact |
|---|---|---|---|
| Agent hierarchy | 9 wrapper agents | 2 core agents (Chief, Execution) | Fewer redundant hops; execution authority isolated |
| Discovery flow | Sequential chain calls | `asyncio.gather` concurrent dispatch | Latency bound to `max()`, not `sum()` |
| File placement | Orphaned top-level folders | Deep, one-concept-per-folder integration | Easier to validate and maintain |
| Est. file footprint | ~85–95 files | ~78–85 files | Leaner, more maintainable MVP |

**Measured effect:** discovery latency drops from **1,250 ms** (sequential) to **~350 ms** (max-bound, async fan-out) — up to a **72% reduction** in API transaction bottlenecks.

---

## Context Intelligence Score

Every candidate resource (hospital, doctor) is ranked with a single composite score that blends semantic relevance with healthcare-specific quality signals:

```
CIS = 0.35 · semantic_similarity
    + 0.25 · proximity_score
    + 0.20 · department_match
    + 0.10 · resource_availability
    + 0.05 · rating
    + 0.05 · freshness
```

Priority inference then layers rule-based reasoning on top of the graph — for example:

```python
def infer_priority_resources(context_graph):
    priorities = []
    if has_symptom(context_graph, "chest_pain") and has_history(context_graph, "heart_disease"):
        priorities.append("cardiology_priority")
    if has_symptom(context_graph, "unconscious"):
        priorities.append("icu_priority")
    if not has_edge(context_graph, edge_type="ambulance_available"):
        priorities.append("self_transport_flag")
    return priorities
```

---

## MCP Tool Surface

Every knowledge source and every action is exposed as an MCP tool, so **any** MCP-capable client — Claude, ChatGPT, or a hospital ops console — can drive HCIE without custom integration work.

| Tool | Purpose | Gated? |
|---|---|---|
| `get_patient_summary` / `get_allergies` / `get_medications` | Patient identity & history lookup | — |
| `search_hospitals` / `nearest_hospital` / `available_departments` | Hospital discovery & filtering | — |
| `current_location` / `route` / `distance` | Maps & routing | — |
| `notify_family` / `share_location` | Contact notification | — |
| `call_ambulance` | Dispatch ambulance | ✅ Approval-gated |
| `connect_physician` | Initiate contact with an available doctor | ✅ Approval-gated |

Execution-class tools (`call_ambulance`, `connect_physician`) are **registered but blocked at the gateway** until an explicit human approval event is received — nothing fires without a human saying go.

---

## Repository Structure

```
healthcare-context-intelligence-engine/
├── apps/            # REST/HTTP entrypoint for the demo UI
├── gateway/          # Auth, routing, session, cache, capability registry [9 files]
├── engine/
│   ├── intent/       # Intent classification & parsing        [4 files]
│   ├── planning/      # Task graph & execution plan builder     [3 files]
│   ├── discovery/     # Per-source discovery agents             [5 files]
│   ├── retrieval/     # BM25 + embeddings + reranking           [5 files]
│   ├── context/       # Context graph + priority inference      [2 files]
│   ├── generation/    # Plan builder & compressor                [3 files]
│   └── validation/    # Schema / safety / completeness checks   [3 files]
├── agents/           # Chief, Intent, Context, Planning, Retrieval, Execution [9 files]
├── servers/           # Patient · Hospital · Contact · Maps MCP servers [4 files]
├── mock_data/         # JSON-backed mock patients/hospitals/doctors/contacts
├── schemas/            # Shared Pydantic-style interfaces
├── tests/              # unit / integration / evaluation suites
├── examples/            # Cardiac, trauma, respiratory, pediatric benchmark cases
├── docs/                # PRD, architecture diagrams, research notes
└── scripts/             # seed_mock_data.py, run_demo.py
```

Full v0.3.1 build: **~78–85 files**. A scoped hackathon-weekend slice (gateway + one discovery agent per server + hybrid retrieval + context graph + plan builder + validation) comes in around **~35 files**.

---

## Roadmap

```
●───────────────●───────────────●───────────────●
Hackathon MVP    v0.3.1 Core     Clinical Pilot   EHR Scale
```

| Milestone | Scope |
|---|---|
| **Hackathon MVP** | Python engine core, 4 mock MCP servers, intent parser, basic sequential orchestration |
| **v0.3.1 Core** *(current)* | Async parallel discovery, hybrid BM25 + embedding search, context graph assembly, validation gates |
| **Clinical Pilot** | Session-aware multi-turn cases, real hospital capacity metrics, physician feedback loops |
| **EHR Scale** | FHIR-compliant connectors, OAuth2 enterprise auth, HIPAA-hardened guardrails |

### Future ecosystem expansion

- **Biometric Edge** — smartwatch telemetry (heart-rate spikes, fall detection) to auto-trigger context assembly.
- **Logistics Hub** — MCP endpoints for medicine delivery, drone dispatch, ambulance marketplaces.
- **Clinical EHR Integration** — mock JSON replaced by production-grade HL7/FHIR gateways behind the same stable tool schemas.

---

## Evaluation

- Retrieval recall@K and reranker precision@K against a labeled mock benchmark.
- Priority-inference accuracy on symptom + history combinations.
- Plan completeness rate (ambulance + hospital + doctor + contact all addressed).
- Safety-checker catch rate against adversarial diagnostic/dosing prompts.
- End-to-end latency, text input → approved plan.
- Benchmarked against a plain-RAG baseline (no graph, no reranking) to demonstrate value-add.

Benchmark categories: cardiac, trauma, respiratory, and pediatric emergencies, plus ambiguous/incomplete input for graceful-degradation testing.

---

## Research Differentiation

> A Context Intelligence Engine that uses MCP, hybrid retrieval, reranking, and agentic planning to orchestrate healthcare resources based on **context** — not diagnosis — while preserving patient privacy.

The shift from *resource retrieval* to *context-aware orchestration* is the differentiating claim:

Instead of *"here are five nearby hospitals,"* HCIE produces: *"Two Cardiology-capable hospitals were found within 3 km. Hospital A has ambulance and ICU availability; Hospital B does not. Plan will route to Hospital A and notify the registered emergency contact."*

Because the underlying engine — intent → planning → discovery → hybrid retrieval → context graph → generation → validation — is domain-agnostic, HCIE also serves as a proof-of-concept that the general CIE architecture generalizes **beyond software engineering.**

---

## Contributing

HCIE is designed to be naturally parallelizable. Good first issues include:

- Adding a new discovery agent for an additional data source
- Extending the MCP tool surface with a new capability
- Adding benchmark cases to `examples/`
- Improving the plain-RAG baseline comparison in `tests/evaluation/`

See `docs/prd/`, `docs/architecture/`, and `docs/research/` for design context before opening a PR.

---

## Disclaimer

> All patient, hospital, and doctor data in this repository is **mock, JSON-backed data**. HCIE is an orchestration and context-assembly framework — it does **not** provide medical diagnoses, clinical triage, or treatment recommendations of any kind. Execution-class tools are approval-gated and disabled by default outside the demo mock servers. Clinical judgment always remains with licensed clinicians and emergency responders.

---

<div align="center">

**Secure Open Innovation for Healthtech**

Built as an MCP-native, open-source orchestration platform · Python 3.11+

</div>
