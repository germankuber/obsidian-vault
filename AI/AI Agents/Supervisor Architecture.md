---
title: Supervisor Architecture
source: "Agentic Architectural Patterns for Building Multi-Agent Systems (Ali Arsanjani)"
author: Ali Arsanjani
created: 2026-06-11
tags:
  - ai/agents/architecture
  - ai/agents/coordination
  - type/pattern
  - status/permanent
aliases:
  - Orchestrator Architecture
  - Supervisor Pattern
  - Task Delegation Framework
  - supervisor-architecture
---

# Supervisor Architecture

> [!note] Definición
> Arquitectura de coordinación **centralizada y jerárquica** donde un único agente **orquestador/supervisor** gestiona el workflow de "worker" agents especializados. Recibe un **goal de alto nivel**, lo **descompone**, **delega** sub-tareas a los especialistas apropiados, y **sintetiza** sus hallazgos en un resultado final. El agente actúa como ***manager*, no *doer*** — la marca definitoria del **Level 4** de madurez (cap. 14). Extiende la noción del [[Orchestrator]] del vault al terreno **multi-agente**.

## Cómo funciona

- El supervisor **no ejecuta lógica de dominio** (anti-**"God agent"**: estricta *separation of concerns*); solo **coordina** — rutea, trackea estado y decide el *branching*.
- Implementación práctica: los especialistas se empaquetan como **AgentTools** (cap. 14, ADK) — el patrón **Agent Delegates to Agent**. El supervisor "llama" a otro agente como si fuera una tool.
- Mantiene **contextual state**: lee los outputs JSON acumulados de cada worker y construye el *payload* de la próxima delegación, actuando como **semantic data mapper** (traduce la salida de un agente al input que el siguiente espera).
- El flujo es **explícito y top-down**: el orquestador es la única autoridad que conoce el plan global; los workers son *stateless* respecto al objetivo macro.

## Pros / Cons

- **Pros** — **predictibilidad** (flujo determinista), **governance / auditabilidad** (cada delegación queda registrada), **fácil de debuggear** (un solo punto de control de flujo).
- **Cons** — el orquestador puede ser **cuello de botella** y **single point of failure**. Mitigación: **checkpointing** y persistencia de estado (ej. **[[LangGraph]]**), o fallback con votación/backup agents.

## Contraste con Swarm Architecture

- La **[[Swarm Architecture]]** es **descentralizada y emergente**: una red *peer-to-peer* sobre un **shared task board**, **sin líder central**.
- Es **más resiliente y adaptativa** (no hay SPOF) pero **difícil de gobernar y debuggear**; apta para tareas **creativas / dinámicas**.
- En la práctica muchos sistemas usan un **modelo híbrido**: un orquestador *top-level* delega sub-goals a swarms/crews auto-organizados.
- La coordinación pasa de **explícita / top-down** (Supervisor, nivel 4) a **emergente / descentralizada** (Swarm, niveles 5-6). Ver la **Tabla 5.2** del libro para el contraste canónico.

## Dónde aparece en el libro

- **Ejemplo seminal** — el *Task Delegation Framework* / **loan processing** (cap. 1): el `LoanOrchestratorAgent` recibe la aplicación y delega a especialistas.
- **Lugar canónico teórico** — cap. 5, contrastado con Swarm y el híbrido (Tabla 5.2).
- **Materialización hands-on** — cap. 14: el orquestador **refactorizado de *doer* a *manager*** (4 tools → 4 agentes especialistas vía AgentTool).
- **Reapariciones** — caps. 3 (arquitectura jerárquica orchestrator↔sub-agents), 6 (`SupervisorAgent` que ancla y dispara el auditing), 7 (robustez: backup/watchdog orchestrators), 8 (supervisor en human-agent interaction).

## Related

- [[Orchestrator]] — la nota del vault que esta extiende a multi-agente (el *job scheduler* / orquestador de workflow).
- [[Swarm Architecture]] — la nota hermana descentralizada (semilla).
- [[05 - Multi-Agent Coordination Patterns]] — lugar canónico (Supervisor vs Swarm, Tabla 5.2).
- [[14 - Use Case - A Multi-Agent System for Loan Processing]] — la materialización hands-on (doer → manager).
- [[01 - GenAI in the Enterprise - Landscape, Maturity, and Agent Focus]] — el ejemplo seminal del loan processing.
- [[06 - Explainability and Compliance Agentic Patterns]] — el `SupervisorAgent` que ancla y audita.
- [[_Agentic Architectural Patterns for Building Multi-Agent Systems]] — MOC del libro.
- [[Saga]] — orquestación de transacciones distribuidas (análogo de control flow centralizado).
- [[Generator-Evaluator Pattern]] — patrón de coordinación de objetivos opuestos del mismo dominio.
