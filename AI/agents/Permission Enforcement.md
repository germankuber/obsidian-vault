---
title: Permission Enforcement
source: https://boringbot.substack.com/p/ai-agent-harnesses-explained-architecture
author: Hamza Farooq, Aishwarya Ashok
created: 2026-06-10
tags:
  - ai/agents/safety
  - type/pattern
  - status/permanent
aliases:
  - Permission Enforcement
  - permission-enforcement
  - Enforcement de Permisos
---

# Permission Enforcement

> [!note] Definición
> Determinar a qué tools, archivos, repositorios y APIs está autorizado a acceder un agente, y **hacer cumplir esos límites en tiempo de ejecución** —no confiando en que el modelo se autogobierne. Una de las cinco [[Harness Responsibilities|responsabilidades del harness]].

## La idea central

- El enforcement ocurre **en el momento de ejecución** (execution time), no como una promesa del modelo.
- Una negativa del modelo (refusal) **solo cuenta** cuando el harness valida el schema de la tool call y la **rechaza antes de ejecutar**.

> [!warning] Límite del "refusal" del modelo
> *"Refusals only count when the harness validates the tool-call schema and rejects before execution."* Un modelo que "dice que no" pero cuyo harness igual ejecuta la acción, no aporta seguridad. El control real es la validación previa.

## Relación con la seguridad estructural

- *"Safety lives in the harness, not the model"* — el enforcement es la contracara de [[Sandboxing]]: sandboxing limita el **daño**, el permission enforcement limita el **acceso**.

## Escalado multi-usuario

- En [[Multi-User Agent Design|sistemas multi-usuario]] hace falta **herencia de permisos por usuario** (per-user permission inheritance): cada agente opera con el scope de quien lo invoca.
- Es el salto de Nivel 2 a **Nivel 3** del [[Harness Maturity Spectrum]] (permisos *scoped*).

## References

- Fuente: [AI Agent Harnesses Explained](https://boringbot.substack.com/p/ai-agent-harnesses-explained-architecture) — Hamza Farooq, Aishwarya Ashok, 2026-05-08

## Related

- [[Agent Harness]]
- [[Harness Responsibilities]]
- [[Sandboxing]]
- [[Multi-User Agent Design]]
