---
title: AgentOps
source: "Agentic Architectural Patterns for Building Multi-Agent Systems — Ali Arsanjani (cap. 2)"
author: Ali Arsanjani
created: 2026-06-11
tags:
  - ai/agents/operations
  - ai/agents/production
  - type/concept
  - status/permanent
aliases:
  - AgentOps
  - Agent Ops
  - agentops
  - AgentOps (operaciones agénticas)
---

# AgentOps

> [!note] Definición
> La **disciplina operacional** para gestionar agentes en producción: una extensión de los principios de **[[_MLOps|MLOps]]** y **LLMOps** a los desafíos únicos de gestionar **agentes, sus tools y sus dependencias de modelo**. Introducida en el **cap. 2** del libro de Ali Arsanjani. *"AgentOps extends MLOps/LLMOps to the unique challenges of managing agents, their tools, and their model dependencies in production."*

## Por qué MLOps no alcanza

- El agente es **no-determinista**: la misma entrada puede producir trayectorias distintas.
- Razona en **múltiples pasos** (*multi-step reasoning*) y usa **tools de forma autónoma** → la superficie de fallo no es solo el output.
- Por eso AgentOps necesita trazar **el *por qué* de cada decisión** (la cadena de razonamiento y las tool calls), no únicamente el output final como hace el monitoreo ML clásico.

## Prácticas core (cap. 2)

- **Monitoreo *agent-specific*** — más allá de latencia/error rate: **task success rate**, **tool use accuracy**, **response quality**, **latency/cost** por tarea, y **drift detection** (degradación del comportamiento en el tiempo).
- **Logging y traceability comprehensiva** — capturar **inputs**, **reasoning steps / chain-of-thought**, **tool calls y sus responses**, y **context snapshots**; el mecanismo de captura típico son los **callbacks** del framework.
- **Version control como artefactos** — **prompts**, **configs** y **tool schemas** versionados igual que el código y los modelos.
- **A/B testing y experimentación** — comparar versiones de agente sobre tráfico real (conecta con *Canary / Shadow Mode* del cap. 7).
- **Feedback loops** — para la **mejora continua** del agente a partir de outcomes observados.
- **Security y compliance monitoring** — detección de **prompt injection** y de **uso anómalo de tools**.

## Tooling

- Plataformas **LLMOps/MLOps** para monitoring, tracing, experiment tracking y model versioning: **Vertex AI**, **LangSmith**, **Arize AI**, **WhyLabs**, **ClearML**, **Weights & Biases**.

## Conexión transversal en el libro

- **Observability = cornerstone del responsible AI** (cap. 15): *"if you cannot trace why an agent made a decision, you cannot guarantee fairness, safety, or compliance."* Se apoya en **LangSmith** y **OpenTelemetry**.
- El **R⁵ model** (cap. 11: *Relax · Reflect · Reference · Retry · Report*) es el **contrato operacional** que extiende AgentOps a sistemas que **aprenden**.
- El **cap. 16** lo lista como uno de los **3 pilares** del *holistic approach* a producción: *"build your AgentOps muscle"* → **containerize → serve como API → log thoughts → monitor**.

## Related

- [[02 - Agent-Ready LLMs - Selection, Deployment, and Adaptation]]
- [[11 - Advanced Adaptation - Building Agents That Learn]]
- [[15 - Agent Frameworks - Loan Processing with CrewAI and LangGraph]]
- [[16 - Conclusion - Charting Your Agentic AI Journey]]
- [[_Agentic Architectural Patterns for Building Multi-Agent Systems]]
- [[_MLOps|MLOps]]
- [[Evals]]
- [[LLM as Judge]]
