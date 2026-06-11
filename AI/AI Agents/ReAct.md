---
title: ReAct
source: (framework de la literatura — paper "ReAct: Synergizing Reasoning and Acting in Language Models", Yao et al. 2022; Reflexion de Shinn et al. 2023. Aparece como reasoning core en *Agentic Architectural Patterns for Building Multi-Agent Systems*, Ali Arsanjani. Verificá los detalles antes de citarlos.)
author: Yao et al. (2022) · Shinn et al. (Reflexion, 2023)
created: 2026-06-11
tags:
  - ai/agents/architecture
  - ai/agents/reasoning
  - type/concept
  - status/permanent
aliases:
  - Reason-Act
  - react
  - Reflexion
  - ReAct (Reason-Act)
---

# ReAct

> [!warning] Fuente: framework conocido, no destilado de un solo artículo
> ReAct viene del paper **"ReAct: Synergizing Reasoning and Acting in Language Models"** (Yao et al., 2022) y **Reflexion** de Shinn et al. (2023). El libro de Arsanjani los usa como *reasoning core*, pero los detalles canónicos son de los papers. Verificá antes de citar números o claims fuertes.

> [!note] Definición
> **ReAct** (Reason-Act) es un framework de razonamiento donde el agente **intercala razonamiento y acción**: alterna **Reason** (cadenas de pensamiento / chain-of-thought) y **Act** (llamadas a tools), **observando el resultado de cada acción** antes del próximo paso. La clave es la **sinergia razonar+actuar en un loop** — no son fases separadas, se alimentan mutuamente. Es el patrón base del ciclo **sense → reason → plan → act**.

## El loop ReAct

El agente no se apura a una conclusión alucinada: **planifica en el "thought" antes de tocar una tool**.

```
Thought  → razona qué hace falta y por qué
Action   → emite una tool call (ver [[Function Calling]])
Observation → recibe el resultado real de la tool
Thought  → re-razona con la nueva evidencia
... (repite hasta resolver)
```

- El **Thought** ancla cada paso en razonamiento explícito → reduce la confabulación y hace el reasoning **auditable**.
- La **Observation** mete **datos reales del mundo** en el loop → el modelo corrige rumbo en vez de inventar.
- El **Act** es la única vía de afectar el entorno; mecánicamente se materializa como [[Function Calling|function calling]] estructurado.

## Reflexion

**Reflexion** (Shinn et al., 2023) agrega una capa sobre ReAct: **auto-reflexión verbal sobre fallos pasados**.

- El agente mantiene una **memoria episódica** de sus errores: tras un intento fallido, **escribe en lenguaje natural qué salió mal** y por qué.
- En el **reintento**, esa reflexión se inyecta como contexto → el agente mejora sin fine-tuning. Es **self-feedback liviano pero potente**.
- En el libro, el **cap. 11** lo cita como la base **operacionalizada a escala** en el *Coevolved Agent Training*: la reflexión deja de ser ad-hoc y se vuelve señal de entrenamiento.
- Relación: es estructuralmente un **self-critique loop** → emparenta con el [[Generator-Evaluator Pattern]] (ahí el crítico es otro agente; en Reflexion el agente se critica a sí mismo).

## Ubicación en el espectro de razonamiento estructurado

De menos a más sofisticado:

- **Chain-of-Thought (CoT)** — razonamiento **lineal**: una sola cadena de pensamiento, sin acción ni ramas.
- **Tree-of-Thought (ToT)** — **ramificado**: explora múltiples paths de razonamiento en paralelo y poda los malos. Sigue sin actuar sobre el mundo.
- **ReAct** — agrega **acción/tools**: intercala thought con tool calls + observaciones reales.
- **Reflexion** — agrega **memoria de fallos**: self-reflection verbal que persiste entre reintentos.
- **FCoT / Fractal Chain-of-Thought** — **recursivo**, anclado a un **contrato**: descompone el problema fractálmente con re-grounding temporal (el más sofisticado del reasoning core en el libro).

## En el libro

- **Cap. 2** — los introduce como frameworks del **reasoning core** (junto a ToT); el LLM conduce el loop `sense→reason→plan→act`. La Tabla 2.1 ordena los frameworks por madurez creciente.
- **Cap. 3** — **Nivel 3 del Agentic Maturity Model**: *Introspective patterns (ReAct y Reflexion)* — agentes únicos con razonamiento step-by-step y self-reflection que se autocorrigen vía feedback.
- **Cap. 4** — los ubica en la **anatomía** del agente (el reasoning core que distingue un agente activo/stateful de una LLM call pasiva).
- **Cap. 9** — los vuelve **patrón** (*Single Agent Baseline*: `Act` → tool use vía ReAct) y los conecta con **Self-Correction**.
- **Cap. 16** — en el ejercicio *"think in patterns"*, dibuja ReAct como el **loop core**: *"¿la query da la respuesta completa? si no, planear un paso nuevo → el loop ReAct"*.

## Related

- [[02 - Agent-Ready LLMs - Selection, Deployment, and Adaptation]] — el reasoning core (ReAct/Reflexion/ToT) y la selección del modelo.
- [[04 - Agentic AI Architecture - Components and Interactions]] — la anatomía donde vive el loop.
- [[09 - Agent-Level Patterns]] — ReAct como mecanismo de acción del Single Agent Baseline.
- [[_Agentic Architectural Patterns for Building Multi-Agent Systems]] — MOC del libro.
- [[Generator-Evaluator Pattern]] — Reflexion ≈ self-critique loop (ahí el crítico es un segundo agente).
- [[Function Calling]] — el `Act` del loop se materializa como function calling estructurado.
- [[Agentic AI Maturity Model]] — ReAct/Reflexion = Nivel 3 (introspective patterns).
