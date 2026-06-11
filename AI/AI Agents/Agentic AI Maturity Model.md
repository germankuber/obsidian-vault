---
title: Agentic AI Maturity Model
source: "Agentic Architectural Patterns for Building Multi-Agent Systems (libro)"
author: Ali Arsanjani
created: 2026-06-11
tags:
  - ai/agents/architecture
  - type/concept
  - status/permanent
aliases:
  - Agentic AI Maturity Model
  - agentic-ai-maturity-model
  - Modelo de Madurez Agéntica
  - GenAI Maturity Model
  - Agentic AI Maturity Spectrum
---

# Agentic AI Maturity Model

> [!note] Definición
> El **andamio que ordena todo el libro**: un espectro de niveles que mide cuán autónomo y sofisticado es un sistema agéntico. El principio rector es que **la madurez es resultado directo de qué patrones implementás** — *"para subir de nivel, implementás los patrones del nivel siguiente"*. No es una etiqueta que te ganás, es una **consecuencia arquitectónica**.

## Agentic AI Maturity Spectrum (cap. 3)

Los **6 niveles** del eje arquitectónico, según la sofisticación del *reasoning loop*:

- **Nivel 1 — Basic agentic systems**: workflows fijos, pasos predeterminados, sin elección real.
- **Nivel 2 — Dynamic single-agent**: el agente **selecciona tools** dinámicamente según el contexto.
- **Nivel 3 — Introspective patterns**: razonamiento iterativo con [[ReAct]] / [[Reflexion]] (pensar, actuar, reflexionar).
- **Nivel 4 — Multi-agent systems**: varios agentes colaboran bajo **coordinación top-down explícita**.
- **Nivel 5 — Advanced coordination**: **meta-agents** orquestan; la coordinación empieza a ser **emergente**.
- **Nivel 6 — Self-correcting / multi-agent learning**: **feedback loops** y aprendizaje colectivo; coordinación **descentralizada**.

> [!tip] El eje de la coordinación
> Lo que cambia a lo largo del espectro es **dónde vive el control**: de **top-down explícita** (nivel 4) a **emergente y descentralizada** (niveles 5–6). Cuanto más arriba, menos director central y más comportamiento que surge de la interacción.

## Los 3 modelos que el libro alinea (cap. 16)

La síntesis final del libro es que **tres maturity models distintos describen la misma jornada desde ángulos complementarios**:

- **(a) GenAI Maturity Model** (cap. 1) — *strategic roadmap* / **organizational readiness**. Niveles 0–6: data foundation → prompting → RAG → tuning → grounding/eval → single-agent → multi-agent.
- **(b) Agentic AI Maturity Spectrum** (cap. 3) — *architectural blueprint* / **inteligencia de los reasoning loops** (los 6 niveles de arriba).
- **(c) Implementation Maturity Levels** (cap. 12) — *engineering discipline* / **código a producción**. L1 Foundational (monolítico) → L2 Production-Ready (microservicios) → L3 Self-Improving (closed-loop).

> [!note] Las 6 fases alineadas del cap. 16
> El capítulo final **fusiona los tres ejes** en una sola progresión: **Foundational → Augmented → Production → Autonomous → Orchestrated → Self-learning**.

## La advertencia clave: architectural overreach

> [!warning] No saltes ejes ni etapas
> El error capital es el **"architectural overreach"**: construir un *self-improving ecosystem* (Implementation **L3**) **antes** de dominar el grounding/evaluation (GenAI **L4**). Readiness organizacional, arquitectura y engineering discipline **deben evolucionar en tándem** — un eje muy por delante de los otros colapsa.

## Related

- [[03 - The Spectrum of LLM Adaptation for Agents - RAG to Fine-tuning]]
- [[05 - Multi-Agent Coordination Patterns]]
- [[12 - A Practical Roadmap - Implementing Agentic Patterns by Maturity Level]]
- [[16 - Conclusion - Charting Your Agentic AI Journey]]
- [[_Agentic Architectural Patterns for Building Multi-Agent Systems]]
- [[Harness Maturity Spectrum]]
- [[MLOps Maturity Stages]]
