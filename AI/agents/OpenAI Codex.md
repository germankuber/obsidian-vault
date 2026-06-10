---
title: OpenAI Codex
source: https://boringbot.substack.com/p/ai-agent-harnesses-explained-architecture
author: Hamza Farooq, Aishwarya Ashok
created: 2026-06-10
tags:
  - ai/agents/architecture
  - type/technology
  - status/permanent
aliases:
  - OpenAI Codex
  - openai-codex
  - Codex
---

# OpenAI Codex

> [!note] Definición
> Uno de los **dos diseños de referencia** (2026) de [[Agent Harness|harness]] de agentes, de OpenAI. Apuesta por el **aislamiento en la nube, con un contenedor fresco por tarea**.

## Estrategia de aislamiento

- Ships un **CLI open-source + un agente cloud**.
- Levanta un **contenedor fresco por tarea** (one container per task).
- Aísla los runs concurrentes mediante **límites de proceso** (process boundaries) en vez de **lógica de coordinación** → ver [[Sandboxing]].

## Posicionamiento frente a Claude Code

> [!tip] Apuestas opuestas, no competidores
> Codex (cloud, contenedor por tarea) y [[Claude Code]] (local, modelo de consentimiento) **no son competidores sino apuestas de diseño opuestas**, útiles justamente porque mapean limpio a **perfiles de workflow distintos**.

Ambos entregan hoy un harness de **Nivel 3** del [[Harness Maturity Spectrum]].

## References

- Fuente: [AI Agent Harnesses Explained](https://boringbot.substack.com/p/ai-agent-harnesses-explained-architecture) — Hamza Farooq, Aishwarya Ashok, 2026-05-08
- Repo: [github.com/openai/codex](https://github.com/openai/codex)

## Related

- [[Agent Harness]]
- [[Claude Code]]
- [[Sandboxing]]
- [[Harness Maturity Spectrum]]
