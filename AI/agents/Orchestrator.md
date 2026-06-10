---
title: Orchestrator
source: https://boringbot.substack.com/p/ai-agent-harnesses-explained-architecture
author: Hamza Farooq, Aishwarya Ashok
created: 2026-06-10
tags:
  - ai/agents/ecosystem
aliases:
  - Orchestrator
  - orchestrator
  - Orquestador
  - Workflow Orchestrator
---

# Orchestrator

> [!note] Definición
> Sistema que, usado en contextos de pipelines de agentes, gestiona **secuenciación de tareas, reintentos y estado de workflow**. Ejemplos: **Temporal, Prefect, Airflow**.

## Qué hace

- Maneja el **orden** de las tareas (task sequencing).
- Reintentos (retries) y **estado del workflow**.

## Qué NO hace

> [!warning] Límite clave
> No entiende las preocupaciones **específicas de agentes**:
> - **memory scoping**
> - **tool call injection prevention** (prevención de inyección en tool calls)
> - **per-user isolation** (aislamiento por usuario)
>
> No es un [[Agent Harness|harness]].

## Lugar en el ecosistema

> [!example] Metáfora del contratista
> Si el modelo es un contratista, el **orchestrator** es el **job scheduler**. El **[[AI Framework]]** es el software de gestión de proyecto. El **[[Agent Harness]]** es la obra real con los controles de seguridad.

- Un harness sofisticado **puede usar** un orchestrator para manejar el encolado de tareas (task queuing) — pero el harness es el concepto que lo contiene.

## References

- Fuente: [AI Agent Harnesses Explained](https://boringbot.substack.com/p/ai-agent-harnesses-explained-architecture) — Hamza Farooq, Aishwarya Ashok, 2026-05-08

## Related

- [[Agent Harness]]
- [[AI Framework]]
