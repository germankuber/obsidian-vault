---
title: Multi-User Agent Design
source: https://boringbot.substack.com/p/ai-agent-harnesses-explained-architecture
author: Hamza Farooq, Aishwarya Ashok
created: 2026-06-10
tags:
  - ai/agents/architecture
  - ai/agents/safety
aliases:
  - Multi-User Agent Design
  - multi-user-agent-design
  - Diseño de Agentes Multi-Usuario
  - Multi-User Harness
---

# Multi-User Agent Design

> [!note] Definición
> El conjunto de requisitos que aparecen cuando varios usuarios comparten un
> [[Agent Harness|harness]] con agentes **concurrentes**. Es el **Nivel 3** del
> [[Harness Maturity Spectrum]].

## Requisitos de un harness multi-usuario

- **Per-user permission inheritance** — cada agente hereda el scope de permisos
  del usuario que lo invoca ([[Permission Enforcement]]).
- **Namespaced memory** — memoria con espacios de nombre por usuario para que los
  contextos no se filtren entre usuarios.
- **Tamper-evident audit logs** — registros de auditoría a prueba de manipulación,
  compartidos y **con atribución** de quién hizo qué.

## Por qué es difícil

> [!warning] El salto que rompe
> *"Single-developer architectures fail subtly at scale."* Lo que funciona para un
> solo desarrollador falla de forma **sutil** (no obvia) al escalar a multi-usuario:
> los bugs son de aislamiento y atribución, no crashes evidentes.

## Governance: un problema de org-design

> [!tip] Insight
> *"Harness governance is now an org-design problem."* Configurar el **tool
> registry**, aprobar el acceso a **memoria compartida** y revisar los **audit
> logs** son **decisiones de política**, no tareas de DevOps.

## References

- Fuente: [AI Agent Harnesses Explained](https://boringbot.substack.com/p/ai-agent-harnesses-explained-architecture) — Hamza Farooq, Aishwarya Ashok, 2026-05-08

## Related

- [[Agent Harness]]
- [[Harness Maturity Spectrum]]
- [[Permission Enforcement]]
- [[Sandboxing]]
