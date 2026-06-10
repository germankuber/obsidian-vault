---
title: Agent Harness
source: https://boringbot.substack.com/p/ai-agent-harnesses-explained-architecture
author: Hamza Farooq, Aishwarya Ashok
created: 2026-06-10
tags:
  - ai/agents/architecture
aliases:
  - Agent Harness
  - agent-harness
  - Harness
  - AI Agent Harness
---

# Agent Harness

> [!note] Definición
> **Agent = Model + Harness.** El **modelo** es un motor de inferencia: lee
> tokens, genera *tool calls* estructuradas y **no hace nada en el mundo**. El
> **harness** es el entorno de ejecución que **intercepta** las llamadas, las
> **rutea por una capa de permisos**, las **ejecuta en un entorno controlado** y
> devuelve **resultados estructurados** al contexto del modelo.

## La fórmula

- **Modelo** → motor de inferencia: procesa tokens y produce una secuencia de
  salida ponderada por probabilidad. No puede por sí solo abrir un archivo,
  recordar el refactor del martes pasado, ni negarse a borrar una base de prod.
- **Harness** → "le da manos al modelo": intercepta [[Tool Calling|tool calls]] →
  capa de permisos → ejecuta acotado → devuelve resultado estructurado.
- **Ninguno es opcional**: un modelo sin harness es *"un autocomplete
  sofisticado"*; un harness sin un modelo capaz es *"un andamio vacío"*.

> [!tip] Principio rector
> *"Safety lives in the harness, not the model."* Si confiás en que el **modelo**
> rechace las acciones malas, **no tenés seguridad**. Una negativa (refusal) solo
> cuenta cuando el harness **valida el schema de la tool call y la rechaza antes
> de ejecutar** → ver [[Permission Enforcement]].

## Analogías del artículo

> [!example] El robot quirúrgico
> El **modelo** es la inteligencia de decisión del cirujano (percibe la situación
> y determina la acción). El **harness** es el **brazo robótico**: el que toca el
> mundo físico. Los protocolos de esterilización, los límites de envolvente de
> movimiento, los sensores de fuerza y el logging viven en el brazo, **no son
> responsabilidad del cirujano**. Un cirujano brillante con un brazo mal
> construido es peligroso; un brazo bien construido con un sistema de decisión
> débil también. **Las dos mitades importan.**

> [!example] El contratista (ecosistema)
> Si el modelo es un contratista: el **[[AI Framework|framework]]** es el software
> de gestión de proyecto (Jira, Linear); el **[[Orchestrator|orchestrator]]** es
> el *job scheduler*; y el **harness** es la **obra real** — con gabinetes de
> herramientas bajo llave, equipo de seguridad, el **permiso de obra colgado en
> la pared**, y un **foreman chequeando credenciales antes de que cualquier
> herramienta salga del estante**.

## Las cinco responsabilidades

El harness cubre cinco funciones centrales — **invariantes arquitectónicas**
(detalle en [[Harness Responsibilities]]):

1. **Tool Execution** — intercepta tool calls y las traduce en operaciones reales
   (escrituras de archivos, comandos de shell, invocaciones de API); devuelve
   resultado estructurado.
2. **Memory & Context Management** — qué recuerda el agente entre turnos, tareas y
   sesiones; cómo **resume, evita (evict) y recupera** contexto previo.
3. **[[Sandboxing]]** — aísla la ejecución para que una call malformada, una
   prompt injection o un error no escalen a daño irreversible.
4. **State Persistence** — mantiene el entorno (archivos abiertos, estado de
   branch, progreso) entre tareas interrumpidas/reanudadas.
5. **[[Permission Enforcement]]** — acceso autorizado a tools/archivos/repos/APIs,
   forzado **en tiempo de ejecución**.

> [!warning] Por qué son invariantes
> Cualquier sistema de agentes en producción al que le falte una de las cinco
> opera con *"una superficie de ataque abierta o un radio de daño (blast radius)
> no controlado"*.

## Madurez

- Cuatro niveles, de invocación manual a producción multi-usuario → ver
  [[Harness Maturity Spectrum]] (L0 Bare Invocation → L3 Multi-User Production).

## Contexto histórico

- En 2024 y principios de 2025 la mayoría de los equipos invirtió en la **mitad
  del modelo** (elegir modelos frontier, tunear prompts, evaluar salidas) y dejó
  el harness *"para construir después"*.
- Los equipos que **sí** construyeron infraestructura rigurosa de harness son los
  que hoy corren agentes con confianza en producción.

## Implementaciones de referencia (2026)

El mercado de harnesses se clarificó en **dos diseños de referencia**:

- [[OpenAI Codex]] — CLI open-source + agente cloud que levanta un **contenedor
  fresco por tarea**, aislando runs concurrentes por **límites de proceso** en
  vez de lógica de coordinación.
- [[Claude Code]] — harness **residente en la terminal** que corre **local** y
  pide **consentimiento explícito** en cada acción consecuente, tratando al
  desarrollador como **co-firmante** de cada tool call significativa.
- No son competidores sino **apuestas de diseño opuestas**, útiles justamente
  porque mapean limpio a **perfiles de workflow distintos**.

## Ecosistema — qué NO es un harness

- [[AI Framework]] (LangChain, LlamaIndex, CrewAI) — *"construction kit, not a job
  site"*; abstracciones de pipeline, **no** sandbox ni permisos.
- [[Orchestrator]] (Temporal, Prefect, Airflow) — secuenciación/retries/estado;
  no entiende memory scoping, **tool call injection prevention** ni per-user
  isolation.
- Un harness sofisticado **puede usar** un framework para estructurar su lógica y
  un orchestrator para encolar tareas, pero el harness es el concepto de **más
  alto nivel que contiene a ambos**.

## Multi-usuario y governance

- Usuarios concurrentes exigen [[Multi-User Agent Design|diseño multi-usuario]]:
  per-user permission inheritance, namespaced memory, tamper-evident audit logs.
- *"Harness governance is now an org-design problem"* — quién configura el **tool
  registry**, quién aprueba escrituras a **memoria compartida** y quién revisa los
  **audit logs** son preguntas de **política**, no tareas de DevOps. Tratarlas como
  DevOps es como los equipos terminan **sin política alguna**.

## Métricas de referencia (mayo 2026)

- *"El factor limitante para los agentes de coding en producción dejó de ser el
  modelo."*
- Modelos de coding frontier: **60%+** en [[SWE-bench]] Verified.
- Capacidad de contexto: contextos de **millón de tokens** (million-token).

## References

- Fuente: [AI Agent Harnesses Explained](https://boringbot.substack.com/p/ai-agent-harnesses-explained-architecture) — Hamza Farooq, Aishwarya Ashok, 2026-05-08 (Edition #35)
- [OpenAI Codex](https://github.com/openai/codex) · [Claude Code docs](https://code.claude.com/docs/en/overview)
- [Claude Code best-practices](https://www.anthropic.com/engineering/claude-code-best-practices) — Anthropic
- [Harness Engineering](https://martinfowler.com/articles/harness-engineering.html) — Martin Fowler
- [SWE-bench Verified](https://www.swebench.com/)

## Related

- [[Harness Responsibilities]]
- [[Harness Maturity Spectrum]]
- [[Sandboxing]]
- [[Permission Enforcement]]
- [[Multi-User Agent Design]]
- [[AI Framework]]
- [[Orchestrator]]
- [[OpenAI Codex]]
- [[Claude Code]]
