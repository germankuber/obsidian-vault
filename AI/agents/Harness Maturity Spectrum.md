---
title: Harness Maturity Spectrum
source: https://boringbot.substack.com/p/ai-agent-harnesses-explained-architecture
author: Hamza Farooq, Aishwarya Ashok
created: 2026-06-10
tags:
  - ai/agents/architecture
  - type/concept
  - status/permanent
aliases:
  - Harness Maturity Spectrum
  - harness-maturity-spectrum
  - Espectro de Madurez del Harness
  - Niveles de Harness
---

# Harness Maturity Spectrum

> [!note] Definición
> Modelo de **cuatro niveles** (L0–L3) que clasifica la madurez de un [[Agent Harness|harness]] según la infraestructura de seguridad, memoria y aislamiento que provee. Saber en qué nivel estás *"es el primer paso para saber qué construir después"*.

## Los cuatro niveles

- **Nivel 0 — Bare Invocation**
  - El modelo recibe un prompt y devuelve texto. Un humano lo lee y ejecuta **manualmente** cualquier acción sugerida.
  - **Cero infraestructura de harness**: *"solo una interfaz de chat y un portapapeles (clipboard)"*.

- **Nivel 1 — Tool-Calling Wrapper**
  - El modelo invoca un schema de tools predefinido; un **wrapper fino** ejecuta esas calls.
  - **Sin** memoria persistente entre sesiones, **sin** [[Sandboxing|sandboxing]], **sin** rollback.
  - Un error es **permanente hasta que un humano lo arregla manualmente**.

- **Nivel 2 — Session-Aware Harness**
  - Memoria **persistente dentro de una sesión**.
  - Rollback básico vía **snapshotting**.
  - Ejecución *sandboxed* para **un solo usuario concurrente**.
  - *"Es más o menos donde estaba el tooling de desarrollador individual bien construido en 2025."*

- **Nivel 3 — Multi-User Production Harness**
  - Aislamiento de ejecución **completo por usuario** (per-user).
  - **Matrices de permisos scoped** ([[Permission Enforcement]]).
  - Audit logs compartidos **con atribución por usuario**.
  - Soporte robusto de **agentes concurrentes**.
  - Es lo que [[OpenAI Codex]] y [[Claude Code]] entregan hoy, y *"el baseline que cualquier organización de ingeniería seria debería estar apuntando"*.

## Lectura del espectro

> [!tip]
> Cada nivel agrega un control estructural sobre el anterior: del manual puro (L0) a la seguridad y concurrencia de producción multi-usuario (L3). El salto más difícil es **L2 → L3**: las arquitecturas que funcionaban para **un** desarrollador *"fallan de forma sutil cuando diez comparten una instancia"* → ver [[Multi-User Agent Design]].

## References

- Fuente: [AI Agent Harnesses Explained](https://boringbot.substack.com/p/ai-agent-harnesses-explained-architecture) — Hamza Farooq, Aishwarya Ashok, 2026-05-08

## Related

- [[Agent Harness]]
- [[Harness Responsibilities]]
- [[Multi-User Agent Design]]
- [[OpenAI Codex]]
- [[Claude Code]]
