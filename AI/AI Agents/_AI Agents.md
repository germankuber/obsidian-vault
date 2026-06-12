---
title: AI Agents — Mapa del tema
created: 2026-06-10
tags:
  - ai/agents
  - type/moc
  - status/permanent
aliases:
  - AI Agents
  - AI Agents MOC
  - Agentes de IA
  - Agent Harness MOC
updated: 2026-06-11
---

# AI Agents — Mapa del tema

> [!note] Cómo usar esta nota
> Es el índice (MOC) de la carpeta `ai/agents`. Empezá por arriba y bajá: cada sección va de lo fundamental a lo avanzado. Abrí esta nota, no la carpeta.

## 🚀 Empezá por acá

- [[Agent Harness]] — el concepto raíz: **Agent = Model + Harness**. El modelo propone tool calls; el harness intercepta, valida, ejecuta acotado y devuelve resultados. *"Safety lives in the harness, not the model."* Todo lo de abajo son piezas de esta idea.

## 🧱 Responsabilidades — qué hace el harness

- [[Harness Responsibilities]] — las **cinco** funciones centrales (Tool Execution · Memory & Context · Sandboxing · State Persistence · Permission Enforcement).
- [[Sandboxing]] — aislar la ejecución contra calls malformadas, prompt injection y errores.
- [[Permission Enforcement]] — acceso autorizado, **en tiempo de ejecución**; los refusals del modelo solo cuentan si el harness rechaza antes de ejecutar.

## 📈 Madurez — cuán completo es el harness

- [[Harness Maturity Spectrum]] — cuatro niveles, L0 (Bare Invocation) → L3 (Multi-User Production).

## 🏗️ Implementaciones de referencia (2026)

- [[OpenAI Codex]] — aislamiento por contenedores cloud **por tarea**.
- [[Claude Code]] — harness **local** con **modelo de consentimiento** (co-firma por acción). Apuestas opuestas, no competidores.

## 🧩 Ecosistema — qué NO es un harness

- [[AI Framework]] — LangChain/LlamaIndex/CrewAI: abstracciones de pipeline, **sin** sandbox ni permisos.
- [[Orchestrator]] — Temporal/Prefect/Airflow: secuenciación/retries, **sin** memory scoping ni per-user isolation.
- [[LangGraph]] — orquestación de agentes como state machine + conditional edges (familia LangChain).

## 🎯 Patrones y casos aplicados

- [[Generator-Evaluator Pattern]] — dos LLMs con objetivos opuestos en loop: uno genera, otro audita. *"Add a second LLM whose only job is to catch the first one leaving its source."*
- [[Grounded Eval Harness]] — caso de estudio concreto del patrón: una IA que se fact-checkea a sí misma para cazar alucinaciones en RAG (LangGraph + Groq/Llama 3.1).

## 🔧 La llamada — qué intercepta el harness

- [[Tool Calling]] — el modelo propone una llamada a una herramienta; el sistema la valida y ejecuta. "El modelo pide; el sistema decide."
- [[Function Calling]] — la versión estructurada: argumentos que matchean un schema. "Schema primero, ejecución después."

## 🔌 Interoperabilidad / Protocolos

- [[MCP]] — **Model Context Protocol**: protocolo cliente-servidor JSON-RPC 2.0 para que el agente obtenga contexto (tools/datos/prompts) de servers externos. *Layer 2 del stack agéntico (vertical, intra-org); la contraparte horizontal es [[A2A]].*
- [[MCP Primitives]] — los 7 primitivos: tools · resources · prompts (server) + sampling · elicitation · logging (client) + tasks (experimental).
- [[MCP Transports]] — STDIO (local) vs Streamable HTTP (remoto, SSE + OAuth).
- [[MCP Lifecycle]] — handshake `initialize`, capability negotiation y notifications event-driven.

## 👥 Escala y governance

- [[Multi-User Agent Design]] — per-user isolation, namespaced memory, tamper-evident audit logs; *"harness governance is an org-design problem"*.

## 🌱 Por escribir (semillas del grafo)

Conceptos ya enlazados desde las notas de arriba pero que todavía no tienen nota propia — candidatos a promover cuando un próximo artículo aporte material:

- [[SWE-bench]] — benchmark de coding (60%+ Verified, mayo 2026).
- [[A2A]] — Agent-to-Agent protocol (Google → Linux Foundation): la capa **horizontal** agente↔agente, contraparte de [[MCP]] (ya resuelto). Muy referenciada desde los libros; candidata a promover.
- [[Prompt Injection]] · [[Vector Database]] (este último ya enlazado desde [[AI Framework]]).

Conceptos relacionados que ya tienen nota en otros subdominios: [[Grounding]], [[Hallucinations]], [[Evals]] (en `AI/fundamentals`) · [[RLHF]] (en `AI/fundamentals`, "análogo" al loop de feedback del harness).

## 🔍 Todas las notas de esta carpeta (auto)

```dataview
LIST
FROM "AI/AI Agents"
WHERE file.name != this.file.name
SORT file.name ASC
```
