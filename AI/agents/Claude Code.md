---
title: Claude Code
source: https://boringbot.substack.com/p/ai-agent-harnesses-explained-architecture
author: Hamza Farooq, Aishwarya Ashok
created: 2026-06-10
tags:
  - ai/agents/architecture
  - type/technology
  - status/permanent
aliases:
  - Claude Code
  - claude-code
---

# Claude Code

> [!note] Definición
> Uno de los **dos diseños de referencia** (2026) de [[Agent Harness|harness]] de agentes, de Anthropic. Apuesta por un **harness local, residente en la terminal**, con un **modelo de consentimiento explícito**.

## Estrategia de aislamiento

- Ships un **harness residente en la terminal** que corre **localmente**.
- Pide **consentimiento explícito en cada acción consecuente**, tratando al desarrollador como **co-firmante (co-signer) de cada tool call significativa** → ver [[Permission Enforcement]].

## Posicionamiento frente a OpenAI Codex

> [!tip] Apuestas opuestas, no competidores
> [[OpenAI Codex]] aísla en la nube (contenedor por tarea); Claude Code aísla localmente vía consentimiento humano por acción. **No son competidores sino apuestas de diseño opuestas** que sirven a **perfiles de workflow distintos**.

Ambos entregan hoy un harness de **Nivel 3** del [[Harness Maturity Spectrum]].

> [!note] Filosofía del harness
> El [best-practices post de Anthropic](https://www.anthropic.com/engineering/claude-code-best-practices) trata al harness como **la capa donde se hacen cumplir las propiedades de seguridad**, en vez de aplicarlas decorativamente alrededor de un modelo que ya se asume seguro.

## References

- Fuente: [AI Agent Harnesses Explained](https://boringbot.substack.com/p/ai-agent-harnesses-explained-architecture) — Hamza Farooq, Aishwarya Ashok, 2026-05-08
- Docs: [code.claude.com/docs/en/overview](https://code.claude.com/docs/en/overview)
- Best practices: [Claude Code best-practices](https://www.anthropic.com/engineering/claude-code-best-practices) — Anthropic

## Related

- [[Agent Harness]]
- [[OpenAI Codex]]
- [[Permission Enforcement]]
- [[Harness Maturity Spectrum]]
