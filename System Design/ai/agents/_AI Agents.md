---
title: AI Agents — Mapa del tema
created: 2026-06-10
tags:
  - ai/agents
  - moc
aliases:
  - AI Agents
  - AI Agents MOC
  - Agentes de IA
  - Agent Harness MOC
---

# AI Agents — Mapa del tema

> [!note] Cómo usar esta nota
> Es el índice (MOC) de la carpeta `ai/agents`. Empezá por arriba y bajá: cada
> sección va de lo fundamental a lo avanzado. Abrí esta nota, no la carpeta.

## 🚀 Empezá por acá

- [[Agent Harness]] — el concepto raíz: **Agent = Model + Harness**. El modelo
  propone tool calls; el harness intercepta, valida, ejecuta acotado y devuelve
  resultados. *"Safety lives in the harness, not the model."* Todo lo de abajo son
  piezas de esta idea.

## 🧱 Responsabilidades — qué hace el harness

- [[Harness Responsibilities]] — las **cinco** funciones centrales (Tool
  Execution · Memory & Context · Sandboxing · State Persistence · Permission
  Enforcement).
- [[Sandboxing]] — aislar la ejecución contra calls malformadas, prompt injection
  y errores.
- [[Permission Enforcement]] — acceso autorizado, **en tiempo de ejecución**; los
  refusals del modelo solo cuentan si el harness rechaza antes de ejecutar.

## 📈 Madurez — cuán completo es el harness

- [[Harness Maturity Spectrum]] — cuatro niveles, L0 (Bare Invocation) → L3
  (Multi-User Production).

## 🏗️ Implementaciones de referencia (2026)

- [[OpenAI Codex]] — aislamiento por contenedores cloud **por tarea**.
- [[Claude Code]] — harness **local** con **modelo de consentimiento** (co-firma
  por acción). Apuestas opuestas, no competidores.

## 🧩 Ecosistema — qué NO es un harness

- [[AI Framework]] — LangChain/LlamaIndex/CrewAI: abstracciones de pipeline, **sin**
  sandbox ni permisos.
- [[Orchestrator]] — Temporal/Prefect/Airflow: secuenciación/retries, **sin**
  memory scoping ni per-user isolation.
- [[LangGraph]] — orquestación de agentes como state machine + conditional edges
  (familia LangChain).

## 🎯 Patrones y casos aplicados

- [[Generator-Evaluator Pattern]] — dos LLMs con objetivos opuestos en loop: uno
  genera, otro audita. *"Add a second LLM whose only job is to catch the first one
  leaving its source."*
- [[Grounded Eval Harness]] — caso de estudio concreto del patrón: una IA que se
  fact-checkea a sí misma para cazar alucinaciones en RAG (LangGraph + Groq/Llama 3.1).

## 👥 Escala y governance

- [[Multi-User Agent Design]] — per-user isolation, namespaced memory,
  tamper-evident audit logs; *"harness governance is an org-design problem"*.

## 🌱 Por escribir (semillas del grafo)

Conceptos ya enlazados desde las notas de arriba pero que todavía no tienen nota
propia — candidatos a promover cuando un próximo artículo aporte material:

- [[Tool Calling]] · [[Function Calling]] — el formato de la llamada que el harness intercepta.
- [[SWE-bench]] — benchmark de coding (60%+ Verified, mayo 2026).
- [[Prompt Injection]] · [[Vector Database]] (este último ya enlazado desde [[AI Framework]]).
- [[Grounding]] · [[Hallucinations]] · [[Evals]] — enlazados desde [[Grounded Eval Harness]]; conceptos de fact-checking/eval que cruzan con `ai/rag/`.
- [[RLHF]] — el loop de feedback en inference time es "estructuralmente análogo a RLHF".
