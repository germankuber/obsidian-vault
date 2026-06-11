---
title: Instruction Drift
source: "Agentic Architectural Patterns for Building Multi-Agent Systems (Ali Arsanjani) — cap. 6"
author: Ali Arsanjani
created: 2026-06-11
tags:
  - ai/agents/architecture
  - ai/agents/safety
  - type/concept
  - status/permanent
aliases:
  - Goal Drift
  - instruction-drift
  - Deriva de instrucciones
---

# Instruction Drift

> [!note] Definición
> La **dilución progresiva de la intención original y de las constraints** a medida que una tarea se propaga por una jerarquía o cadena de agentes. Es la **causa raíz** que motiva toda la dimensión de *accountability/compliance* del libro. Cita rectora: *"autonomy without accountability is a liability"* (cap. 6). Cuanto más autónomo el colectivo, más se desvía silenciosamente del mandato inicial sin que nadie lo note.

## Causas

- **Lost in the middle** — los LLMs **olvidan información crítica enterrada en medio de un contexto largo** (atienden bien al inicio y al final, mal al centro). Las constraints sepultadas se pierden. → semilla: [[Lost in the Middle]].
- **Cadenas largas de delegación** agente→agente: cada salto reinterpreta el encargo y acumula error.
- **Resúmenes lossy** entre pasos: cada compresión intermedia descarta matices de la instrucción original.
- **Goal drift** — el agente se **desvía gradualmente de su objetivo** a medida que avanza la tarea, optimizando lo local y perdiendo de vista el mandato global.

## Los 4 patrones que lo combaten (cap. 6)

1. **Instruction Fidelity Auditing** — auditor **externo y reactivo** que compara el output contra la instrucción original, atrapando *silent failures*. Es estructuralmente un [[Generator-Evaluator Pattern]]: objetivos divergentes, el auditor desconfía por defecto.
2. **FCoT Embedding** — autocorrección **interna y proactiva** durante el razonamiento (Faithful Chain-of-Thought), con **temporal re-grounding** que re-ancla al objetivo paso a paso.
3. **Persistent Instruction Anchoring** — **tags** que mantienen las constraints **salientes** y visibles contra el *lost-in-the-middle*, re-inyectándolas para que no se hundan en el contexto.
4. **Shared Epistemic Memory** — **scratchpad central** que da [[Ground Truth|ground truth]] al colectivo, evitando el efecto **"Tower of Babel"** (cada agente con su propia versión de la verdad).

> [!tip] Best practice — defensa multicapa
> No elegir uno solo: **componer 2-3 de estos patrones concurrentemente**. Combinan auditoría externa (1) + corrección interna (2) + saliencia (3) + verdad compartida (4) como capas redundantes.

## Dónde aparece en el libro

- **Definido en el cap. 6** ([[06 - Explainability and Compliance Agentic Patterns]]) — el problema central del capítulo.
- El **lost-in-the-middle** reaparece en el cap. 9 ([[09 - Agent-Level Patterns]], gestión de memoria).
- El **FCoT** combate el goal drift en los caps. 13/14 ([[13 - Use Case - A Single Agent for Loan Processing]], [[14 - Use Case - A Multi-Agent System for Loan Processing]]).
- El **Instruction Fidelity Auditing** es el patrón estrella de los case studies y del action plan (cap. 16).

## Related

- [[_Agentic Architectural Patterns for Building Multi-Agent Systems]]
- [[06 - Explainability and Compliance Agentic Patterns]]
- [[09 - Agent-Level Patterns]]
- [[13 - Use Case - A Single Agent for Loan Processing]]
- [[14 - Use Case - A Multi-Agent System for Loan Processing]]
- [[Generator-Evaluator Pattern]]
- [[Ground Truth]]
- [[Lost in the Middle]]
