---
title: Eval Observability
source: "AI Evals For Engineers, PMs & QAs: Complete Study Guide"
author: Om Bharatiya
created: 2026-06-11
tags:
  - ai/evals
  - type/concept
  - status/permanent
aliases:
  - Eval Observability
  - Trace
  - Span
  - Observability Platforms
  - Eval Frameworks
updated: 2026-06-11
---

# Eval Observability

> [!note] Definición
> La **capa de observabilidad** sobre la que se construyen las evals: el registro de lo que hizo el AI (**traces** y **spans**), las plataformas que lo capturan, y los frameworks de evaluación. **Principio fundacional: *"Sin traces no podés hacer evals."* Configuralo primero.**

## Trace y Span (definiciones core)

- **Trace** = el **registro completo** de todo lo que hizo el AI para responder: (1) system prompt, (2) user messages, (3) tool calls, (4) tool responses, (5) assistant responses, (6) todo el contexto que vio el LLM.
- **Span** = una **unidad de trabajo dentro de un trace** (ej: una tool call, una LLM call, un paso del pipeline).
- **Qué capturar:** mínimo = input, output, timestamp, ID único. Mejor = system prompts, tool calls+resultados, params del modelo (temperature, max_tokens), token counts, latencia, costo/request. Best practice = contexto de usuario (historial), error messages, versión de modelo, feature flags.

> [!info] No dupliques con [[AgentOps]]
> [[AgentOps]] ya toca observabilidad de agentes; esta nota se enfoca en el ángulo **eval** (traces como insumo de evaluación y el landscape de plataformas de eval). Cross-linkeá, no repitas.

## Landscape de plataformas de observabilidad

| Plataforma | Modelo | Licencia / costo | Notas |
|---|---|---|---|
| **Arize Phoenix** | OSS self-hosted, OTel-native | ELv2, Free, un Docker | full-featured |
| **Langfuse** | OSS cloud/self-hosted | MIT, Free tier + paid | UI pulida, comunidad fuerte |
| **Braintrust** | cloud | Paid | UI excelente, colaboración |
| **LangSmith** | cloud | Paid | para usuarios LangChain |
| **Build Your Own** | custom | Free | para aprender |

Todas soportan: traces, spans, datasets, evaluations, experiments.

## Prompt Management / versionado

> [!tip] Versioná los prompts (incluidos los judge prompts)
> Ambas plataformas dan prompt management versionado: Phoenix `PromptVersion`, Langfuse `create_prompt(type='chat', labels=['production'])` + `get_prompt().compile(...)`. Logueá qué versión corrió con cada trace; trackeá texto, few-shot, modelo+temperature, métricas Dev TPR/TNR, fecha y razón del cambio.

## Eval Frameworks landscape

- **Phoenix Evals** (`arize-phoenix-evals`: `llm_generate` / `llm_classify`) · **Langfuse Evals** (LLM-as-Judge built-in + custom vía SDK) · **OpenAI Simple Evals** (model-graded liviano) · **Ragas** ([[RAGAS]], especializado RAG, reference-free + synthetic test data, Apache 2.0) · **DeepEval**.
- **Eval Tooling Strategy Matrix (por empresa):** Anthropic (Safety/Red Teaming, constitutional classifiers) · OpenAI (Preparedness, SME probing) · Arize (Observability, OTel-native) · RAGAS (RAG, reference-free + synthetic) · Maxim (Agentic, simulation no-code) · Langfuse (Custom Pipelines, data sovereignty self-hostable) · Braintrust (Experimentation, colaborativo) · Galileo (Hallucinations, ChainPoll real-time) · Comet Opik (E2E observability) · METR (Catastrophic Risk, autonomous capability assessment).

## Seeds del grafo

Tools y frameworks mencionados, candidatos a nota propia: [[Opik]] · [[RAGAS]] · [[Galileo]] · [[METR]] · [[DeepEval]] · [[Braintrust]] · [[LangSmith]].

## References

- Fuente: [AI Evals For Engineers, PMs & QAs: Complete Study Guide](https://github.com/ombharatiya/ai-system-design-guide/blob/main/ai_evals_comprehensive_study_guide.md) — Om Bharatiya. Caps 2, 16; Apéndice F.

## Related

- [[AgentOps]]
- [[Eval Lifecycle]]
- [[Evals]]
- [[LLM as Judge]]
