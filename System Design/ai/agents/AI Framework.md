---
title: AI Framework
source: https://boringbot.substack.com/p/ai-agent-harnesses-explained-architecture
author: Hamza Farooq, Aishwarya Ashok
created: 2026-06-10
tags:
  - ai/agents/ecosystem
aliases:
  - AI Framework
  - ai-framework
  - Agent Framework
  - Framework de IA
---

# AI Framework

> [!note] Definición
> Librería para desarrolladores que provee **abstracciones** para construir pipelines de agentes. Ejemplos: **LangChain, LlamaIndex, CrewAI**.

## Qué hace

- Ayuda a **cablear componentes** entre sí.
- Define flujos **chain-of-thought**.
- Integra con [[Vector Database|vector stores]].

## Qué NO hace

> [!warning] Límite clave
> **No hace sandboxing de tool calls ni enforcement de permisos.** No es un [[Agent Harness|harness]]. En palabras del artículo: *"It's a **construction kit, not a job site**"* (es un kit de construcción, no la obra).

## Lugar en el ecosistema

> [!example] Metáfora del contratista
> Si el modelo es un contratista, el **framework** es el **software de gestión de proyecto** (Jira, Linear). El **[[Orchestrator]]** es el *job scheduler*. El **[[Agent Harness]]** es la **obra real** (gabinetes con llave, equipo de seguridad, permiso en la pared, foreman chequeando credenciales).

- Un harness sofisticado **puede usar** un framework para estructurar su lógica de agente — pero el harness es el concepto de más alto nivel que lo contiene.

## References

- Fuente: [AI Agent Harnesses Explained](https://boringbot.substack.com/p/ai-agent-harnesses-explained-architecture) — Hamza Farooq, Aishwarya Ashok, 2026-05-08

## Related

- [[Agent Harness]]
- [[Orchestrator]]
