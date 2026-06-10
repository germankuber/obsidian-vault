---
title: Sandboxing
source: https://boringbot.substack.com/p/ai-agent-harnesses-explained-architecture
author: Hamza Farooq, Aishwarya Ashok
created: 2026-06-10
tags:
  - ai/agents/safety
  - type/pattern
  - status/permanent
aliases:
  - Sandboxing
  - sandboxing
  - Aislamiento de Ejecución
---

# Sandboxing

> [!note] Definición
> Aislar la ejecución de un agente para que una tool call malformada, una **prompt injection** o un error de programación **no causen daño irreversible**. Es una de las cinco [[Harness Responsibilities|responsabilidades del harness]].

## Qué aísla

- Limita el radio de impacto de cualquier acción que el modelo proponga (escrituras de archivos, comandos de shell, llamadas a API).
- Protege contra tres fuentes de daño:
  - **Tool calls malformadas** — el modelo emite argumentos inválidos o peligrosos.
  - **Prompt injection** — input adversario que intenta secuestrar el comportamiento.
  - **Errores de programación** — bugs del propio agente o de las tools.

## Por qué la seguridad vive acá

> [!tip] Principio
> *"Safety lives in the harness, not the model."* El aislamiento es estructural: no depende de que el modelo "decida" portarse bien, sino del entorno que lo contiene. Ver [[Permission Enforcement]].

## Cómo se implementa según el nivel de madurez

- **Nivel 2** ([[Harness Maturity Spectrum]]): sandbox de un solo usuario.
- **Nivel 3**: aislamiento **por usuario** (per-user isolation) para soportar agentes concurrentes sin que se contaminen entre sí.
- Estrategias concretas de las implementaciones de referencia:
  - [[OpenAI Codex]] — contenedores cloud frescos **por tarea** (límites de proceso).
  - [[Claude Code]] — ejecución local con checkpoints de consentimiento.

## References

- Fuente: [AI Agent Harnesses Explained](https://boringbot.substack.com/p/ai-agent-harnesses-explained-architecture) — Hamza Farooq, Aishwarya Ashok, 2026-05-08

## Related

- [[Agent Harness]]
- [[Harness Responsibilities]]
- [[Permission Enforcement]]
- [[Multi-User Agent Design]]
