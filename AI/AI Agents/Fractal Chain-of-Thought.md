---
title: Fractal Chain-of-Thought
source: "Agentic Architectural Patterns for Building Multi-Agent Systems (Ali Arsanjani)"
author: Ali Arsanjani
created: 2026-06-11
tags:
  - ai/agents/reasoning
  - ai/agents/architecture
  - type/pattern
  - status/permanent
aliases:
  - FCoT
  - Fractal CoT
  - FCoT Embedding
  - fractal-chain-of-thought
---

# Fractal Chain-of-Thought

> [!note] Definición
> **FCoT (Fractal Chain-of-Thought)** es el framework de razonamiento **propio y más original** del libro: estructura la mente del agente como un **"sistema operativo cognitivo"**, combinando un **Instruction Contract (IC)** inmutable con un **Recursive Loop** de N iteraciones. Su objetivo es combatir el **goal drift** y el efecto **lost-in-the-middle**, asegurando que el agente **constantemente vuelva a su misión** sin importar la longitud del contexto. Es el núcleo cognitivo de los use cases prácticos del libro.

## Los dos elementos

- **(1) Instruction Contract (IC)** — la **fuente de verdad inmutable**. Contiene: *mission*, *deliverables* exactos, *success criteria*, *hard constraints*, *safety policy* y un **IC-fingerprint** (ej. `LOAN-FCoT-v3-Δ0710`) que ancla e identifica la versión del contrato. No cambia durante el razonamiento.
- **(2) Recursive Loop** — el **proceso de pensamiento activo**, con N iteraciones (típicamente **N=3**: *Planning*, *Execution*, *Verification*). Cada iteración corre tres pasos: **RECAP · REASON · VERIFY**.
  - **RECAP** — re-cargar el IC y el estado para re-anclarse.
  - **REASON** — pensar/avanzar el trabajo de esa fase.
  - **VERIFY** — contrastar plan, razonamiento y output **contra el IC**.

> [!tip] La feature más potente: VERIFY en cada iteración
> El **VERIFY de *cada* iteración** (no solo al final) es lo que distingue a FCoT: verifica el plan, el razonamiento y el output contra el IC en cada vuelta → **self-correction estructurada**, análoga a una self-evaluation continua (ver [[Generator-Evaluator Pattern]]).

## Mecanismos avanzados (cap. 6 — FCoT Embedding)

- **Autocorrección interna y proactiva** — durante el razonamiento, **no después** (a diferencia de un eval post-hoc).
- **Temporal re-grounding** — re-anclar el contexto temporal en cada iteración.
- **Inter-agent reflectivity** — reflexión entre agentes en un sistema multi-agente.
- **Context aperture** — una "apertura de contexto" que se **cierra/enfoca** con una **dual objective function** por iteración (foco progresivo en lo relevante al IC).

## Diferencia con CoT / ToT / ReAct

- **Chain-of-Thought** → razonamiento **lineal**.
- **Tree-of-Thought** → **ramificado**.
- **[[ReAct]]** → agrega **acción** al razonamiento.
- **FCoT** → **recursivo/fractal y anclado a un contrato**: no lineal ni ramificado, sino **auto-similar** (la misma estructura RECAP·REASON·VERIFY se repite a cada escala) y **auto-verificante**.

## Detalle clave (cap. 13): N=3 es *cognitive control*, no código

- **`N=3` = cognitive control** — una **instrucción semántica al LLM "en inglés"** que le indica iterar 3 veces. **NO es un parámetro Python.**
- Se distingue del **code control** (ej. `thinking_budget`), que fija los **límites de recursos** de forma programática.
- Filosofía: **"programar el modelo en inglés"** — el control del razonamiento vive en el prompt/contrato, no solo en el código.

## Dónde aparece en el libro (8 capítulos)

- **Cap. 6** — patrón **FCoT Embedding**, contra *instruction drift* (ver [[Instruction Drift]]).
- **Caps. 13 y 14** — **núcleo cognitivo** de los use cases hands-on, materializado en código con **Google ADK**.
- **Cap. 9** — Self-Correction.
- **Cap. 16** — en el ejercicio *"think in patterns"*.

## Related

- [[06 - Explainability and Compliance Agentic Patterns]]
- [[09 - Agent-Level Patterns]]
- [[13 - Use Case - A Single Agent for Loan Processing]]
- [[14 - Use Case - A Multi-Agent System for Loan Processing]]
- [[_Agentic Architectural Patterns for Building Multi-Agent Systems]]
- [[Generator-Evaluator Pattern]] — el VERIFY es self-evaluation
- [[Instruction Drift]] — lo que FCoT combate
- [[ReAct]] — el espectro de razonamiento
