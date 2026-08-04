---
title: 05 - Multi-Agent Coordination Patterns
libro: Agentic Architectural Patterns for Building Multi-Agent Systems
autor: Ali Arsanjani
capitulo: 5
created: 2026-06-11
tags:
  - libros/agentic-architectural-patterns
  - type/case-study
  - status/permanent
aliases:
  - Multi-Agent Coordination Patterns
updated: 2026-07-03
---

# 05 - Multi-Agent Coordination Patterns

> [!info] Capítulo 5 · *Agentic Architectural Patterns for Building Multi-Agent Systems* — Ali Arsanjani
> Los patrones que hacen posible la colaboración entre múltiples agentes: routing, delegación (Supervisor vs Swarm), topologías (Blackboard, Contract-Net, Supervision Tree), planning, knowledge sharing, tool routing, y la fricción (consensus, negociación, resource allocation, conflict resolution, formation control). Navegá: [[_Agentic Architectural Patterns for Building Multi-Agent Systems|Mapa del libro]] · anterior [[04 - Agentic AI Architecture - Components and Interactions]] · siguiente [[06 - Explainability and Compliance Agentic Patterns]].

## Resumen

Tras el blueprint del agente individual (cap. 4), este capítulo se dedica a los **coordination patterns** que hacen posible la colaboración entre múltiples agentes: los desafíos enterprise más complejos y valiosos suelen exceder las capacidades de cualquier agente único, así como una empresa depende de un equipo de empleados especializados, las soluciones de IA avanzadas requieren un equipo de agentes especializados trabajando en concierto. Cuando múltiples agentes autónomos operan en un entorno compartido, sus acciones deben coordinarse para evitar conflictos, gestionar recursos y lograr goals globales. El enfoque "map before the journey": primero una guía estratégica alineada al GenAI Maturity Model que muestra cómo la implementación de estos patrones se vuelve más dinámica, compleja y descentralizada a medida que la organización progresa de single-agent (Niveles 1-3) a multi-agente (Nivel 4) y más allá.

La relación con el modelo de madurez **no es asignar un patrón a un nivel**, sino una *evolución*: en sistemas multi-agente fundacionales (Nivel 4) la coordinación es típicamente **explícita y top-down** (Supervisor Architecture, planning estático, shared memory simple, conflict resolution centralizado); en sistemas avanzados y self-correcting (Niveles 5-6) se vuelve **emergente y descentralizada** (Swarm/Hybrid, planning dinámico, capacidades "sociales" — consensus, negociación). El capítulo recorre 12 patrones agrupados conceptualmente en: **routing y delegación** (Agent Router, Task Delegation Frameworks: Supervisor vs Swarm), **topologías de composición** (Blackboard Knowledge Hub, Contract-Net Marketplace, Supervision Tree with Guarded Capabilities), **mecanismos de colaboración** (Multi-Agent Planning, Knowledge Sharing, Tool Routing), y **gestión de la fricción** (Consensus, Negotiation, Resource Allocation, Conflict Resolution, Formation Control). Conclusión: la coordinación se *arquitectura, no se asume*; la elección de framework/topología dicta el balance entre predictibilidad y adaptabilidad.

## Guía estratégica: el Agentic AI Maturity Model (Tabla 5.1)

(Revisión de los 6 niveles del cap. 3, ahora con los *coordination patterns* mapeados a cada nivel. Nota: de aquí en más, "niveles" se refiere a los de esta Tabla 5.1.)

| Nivel | Descripción | Scalability | Compliance | Patrones/métodos clave |
|---|---|---|---|---|
| 1. Basic agentic systems | Agentes únicos, tareas bien definidas semi-autónomas, workflows predefinidos + function calls. | Adaptable pero rígido. | Directo de gestionar. | Single-Agent Baseline, Static Function Calling, Watchdog Timeout, Agent Calls Human. |
| 2. Dynamic single-agent workflows | Un agente elige dinámicamente de tools/APIs pre-seleccionadas. | Más versátil y eficiente. | Manejable por pre-selección, pero requiere monitoreo. | Agent Router (Basic), Dynamic Tool Selection, Simple RAG, Simple Retry. |
| 3. Introspective (ReAct y Reflexion) | Razonamiento step-by-step y self-reflection; aprenden y autocorrigen vía feedback. | Feedback y autocorrección → paths de escalabilidad. | Monitoreo real-time y mecanismos correctivos esenciales. | ReAct, Reflexion, Instruction Fidelity Auditing, Adaptive Retry with Prompt Mutation. |
| 4. Multi-agent systems | Múltiples agentes especializados colaboran; funciones no superpuestas, procesamiento paralelo. | Ideal high-scale (distribuir tareas). | Más complejo; se requieren sistemas de monitoreo. | **Supervisor Architecture, Multi-Agent Planning, Shared Epistemic Memory, Event-Driven Reactivity, Tool/Agent Registry.** |
| 5. Advanced coordination con meta-agents | Un "meta-agent" supervisa y coordina; reasignación dinámica y ajustes de planning real-time. | Adaptabilidad mejorada. | Meta-agents como overseers de adherencia a políticas. | **Meta-agents, Blackboard Topology, Resource Allocation, Contract-Net Marketplace, Supervision Trees.** |
| 6. Self-correcting agents | Multi-agente con feedback loops multi-turn; critican, corrigen y refinan outputs iterativamente. | Altamente escalable, mejora continua. | El más complejo; compliance checks automatizados. | **Consensus, Agent Negotiation, Conflict Resolution, Fractal CoT, Coevolved Agent Training, Trust Decay.** |

### Coordinación fundacional (Nivel 4) vs avanzada (Niveles 5-6) — Tabla 5.3

En **Nivel 4** (foundational), el goal es un sistema funcional, predecible y auditable donde múltiples agentes colaboran en un workflow bien definido; coordinación explícita y top-down. Características: *task delegation* vía **Supervisor Architecture** (orquestador central fácil de construir/debuggear/gobernar) usando **Multi-Agent Planning** (plan estático o semi-estático); *knowledge/resource management* vía **Shared Epistemic Memory** simple (DB compartida) + Resource Allocation y Conflict Resolution centralizados por el supervisor; los agentes generalmente **no negocian** entre sí.

En **Niveles 5-6** (advanced/self-correcting), salto en autonomía: maneja ambigüedad, se adapta a lo imprevisto, resuelve problemas cuyo path no se conoce de antemano; coordinación emergente y descentralizada. Características: *delegation* hacia **Swarm Architecture** descentralizado o **Hybrid Model**, planning dinámico; *interacciones "sociales"* (el shift más significativo del Nivel 6): **Consensus** (debate iterativo para converger ante datos conflictivos) y **Negotiation** (agentes con goals en competencia negocian un compromiso); *coordinación especializada*: **Formation Control** para el mundo físico (drones/robots auto-organizándose).

| Aspecto arquitectónico | Multi-agente (Nivel 4) | Avanzado/self-correcting (Niveles 5-6) |
|---|---|---|
| Goal primario | Workflow funcional, predecible y auditable. | Manejar ambigüedad, adaptarse a lo imprevisto, problemas dinámicos. |
| Modelo de coordinación | Top-down y explícito, gestionado por autoridad central. | Bottom-up y emergente, surge de las interacciones. |
| Arquitectura primaria | Supervisor: un orquestador central dirige. | Swarm o híbrida: red peer-to-peer o equipos auto-organizados. |
| Método de planning | Estático: el supervisor descompone y dicta un plan fijo. | Dinámico: el plan es adaptable y se modifica en tiempo real. |
| Knowledge sharing | Shared memory simple: pasar estado entre agentes. | Rich shared epistemic memory: construir inteligencia colectiva. |
| Conflict resolution | Centralizado y policy-based: el supervisor resuelve por reglas predefinidas. | Autónomo y negociado: los agentes resuelven directo vía negociación y consenso. |

## 1. Agent Router (intent-based routing)

Patrón fundacional para **desacoplar la intención del usuario del agente específico que la ejecuta**. En sistemas tempranos se usaba lógica condicional hardcoded (`if "sales" in query: call_sales_agent`), que a escala enterprise (decenas de agentes) se vuelve frágil e inmanejable. Es el "Hello World" de la coordinación agéntica: el core mínimo viable para dispatch inteligente.
- **Contexto** — sistema con una suite de agentes especializados, cada uno con capacidades distintas; los usuarios interactúan en lenguaje natural (ambiguo, variado, con ruido).
- **Problema** — ¿cómo mapear con precisión un request de lenguaje natural variable a el agente mejor capacitado, sin "alucinar" capacidades ni depender de keyword matching frágil? *Fuerzas*: ambigüedad vs precisión, escalabilidad vs mantenimiento, safety vs alucinación.
- **Solución** — proceso de dos pasos: (1) **semantic intent extraction** (un LLM con schema estricto traduce la query cruda a un "intent object" con actions/verbs y resources/nouns estandarizados); (2) **graph-constrained routing** (consultar un lookup table / knowledge graph para encontrar qué agente reclama la capacidad de hacer esa action sobre ese resource). Si existe un path válido en el grafo, se rutea; si no, se rechaza como unsupported.
- **Ejemplo (compliance request)** — *"Where is the latest security audit for the Q3 finance project?"* → (1) **intent extraction**: `{Action: "Find", Resource: "Document", Params: {"type": "audit", "period": "Q3"}}`; (2) **graph lookup** del tuple `(Find, Document)`; (3) **evaluación**: SalesAgent registrado para `(Find, SalesReport)` → Mismatch; ComplianceAgent para `(Find, Document)` → Match; (4) **dispatch** al ComplianceAgent. (Código: schema `RoutingIntent` con Pydantic + `capability_graph` que mapea `(action, resource) → agent`; safety check que bloquea si no hay link.)
- **Consecuencias** — *Pros*: **decoupling** (la capa de extracción no conoce nombres de agentes; los agentes no parsean lenguaje natural), **scalability** (agregar un agente = registrar sus capacidades en el grafo, la lógica de routing no cambia), **safety** (el grafo es un whitelist: no puede mandar "delete database" a un agente salvo que el link esté explícitamente definido). *Cons*: **latency** (la extracción requiere una llamada LLM antes del trabajo real), **schema rigidity** (si la query no encaja en los enums Action/Resource, la extracción falla o degrada).
- **Guía** — schema con abstracción "Goldilocks": ni demasiado granular (`FindPDF`/`FindWordDoc` → grafo masivo) ni demasiado amplio (`DoWork` → pierde poder de distinguir); 10-20 actions/resources canónicos suele bastar. Usar **function calling** (no prompt engineering simple) para forzar JSON estricto. Considerar un **semantic cache** en la capa de routing (embeber la query, chequear vector DB por requests previos similares) para bypassear la extracción en queries repetidas, reduciendo latencia y costo.

![[05-fig-5.1.png|401]]
*Figura 5.1 – The Agent Router pattern*

## 2. Task Delegation Frameworks

El "sistema operativo" del multi-agente: el modelo de alto nivel que dicta cómo fluye el control y la comunicación (cómo se asignan las tareas, quién es accountable, cómo se asegura el goal final). Elegir el framework correcto es una de las decisiones tempranas más importantes. Dos enfoques principales: **Supervisor (Orchestrator) Architecture** y **Swarm Architecture**.

### 2a. Supervisor Architecture (orquestación centralizada)

- **Qué es** — un único **orchestrator agent** central gestiona y dirige el workflow de "worker" agents especializados (espeja una estructura jerárquica de management: recibe un goal de alto nivel, lo descompone en sub-tareas y delega). Control top-down claro; ideal para procesos de negocio estructurados y predecibles.
- **Contexto** — tarea compleja que requiere múltiples pasos secuenciales/condicionales; el sistema debe asegurar que el proceso se siga correctamente con cadena de mando clara.
- **Problema** — ¿cómo ejecutar de forma confiable y predecible una tarea compleja multi-step que requiere capacidades especializadas, gestionando el flujo de datos y el orden de operaciones sin intervención humana constante? *Fuerzas*: predictibilidad vs flexibilidad, centralización vs bottleneck, especialización vs overhead de coordinación.
- **Solución** — un único agente maneja toda la coordinación: interpreta el request, formula un plan (pre-programado o dinámico), llama a los workers según necesita, recibe el output de cada uno y decide el próximo paso.
- **Ejemplo (loan processing centralizado)** — el `LoanOrchestratorAgent` recibe "process this loan application" → delega al `DocumentValidationAgent` → según el resultado (documentos válidos) delega al `CreditCheckAgent` → recibe el risk score del `RiskAssessmentAgent`, ensambla un resumen y decide. (Código: el orquestador NO hace validación/credit checks; mantiene un equipo de workers especializados y maneja el branching condicional — ej. halt si un documento es inválido.)
- **Consecuencias** — *Pros*: **predictability** (flujo claro, simple de monitorear/debuggear/auditar), **governance** (control centralizado simplifica enforcement de reglas de negocio y compliance). *Cons*: **scalability** (el orquestador único puede volverse cuello de botella), **single point of failure** (si el orquestador falla, todo el workflow se detiene).
- **Guía** — mantener **separación estricta de concerns** para evitar un "God agent": el orquestador solo coordina (rutea, trackea estado, decide), no ejecuta lógica de dominio. **Robust state management**: como es single point of failure, persistir el estado ("checkpointing") tras cada paso (frameworks como **LangGraph** lo hacen, persistiendo el graph state a una DB → reanudar exactamente donde quedó). Comunicación **determinista** (output schemas estrictos vía Pydantic/JSON mode, no lenguaje libre). El supervisor como **central fault handler** (retry, route a backup, fail gracefully).

![[05-fig-5.2.png|421]]
*Figura 5.2 – Supervisor Architecture workflow*

### 2b. Swarm Architecture (coordinación descentralizada emergente)

- **Qué es** — **no hay líder central**; los agentes operan como una red peer-to-peer, colaborando de forma emergente y auto-organizada. Una tarea se broadcastea al grupo; los agentes "bid" o se auto-seleccionan según sus capacidades; el workflow emerge de las interacciones. Ideal para tareas creativas, problem-solving dinámico y entornos que requieren alta resiliencia.
- **Contexto** — tarea dinámica y no estructurada, o el sistema necesita ser muy resiliente y adaptativo; un single point of failure es inaceptable; el problem-solving se sirve mejor de acción paralela y autónoma que de un flujo secuencial rígido.
- **Problema** — ¿cómo colaboran efectivamente agentes autónomos hacia un goal común **sin orquestador central**? Se necesita un mecanismo de task discovery, handoff y completion descentralizado y resiliente. *Fuerzas*: autonomía vs coordinación, escalabilidad vs overhead, comportamiento emergente vs predictibilidad.
- **Solución** — típicamente un **shared task board**: una tarea se postea y cualquier agente puede "pull"-earla cuando esté listo; al completar su parte, actualiza el estado de la tarea en el board, dejándola disponible para el siguiente especialista. Procesamiento asíncrono y paralelo, sin dependencia de un punto de control único.
- **Ejemplo (content creation descentralizado)** — "write a blog post about solar power" → (1) **task broadcast** al shared board; (2) **self-selection**: un `ResearchAgent` poll-ea, ve status `new` y se auto-selecciona; (3) **execution/update**: junta facts, actualiza la tarea, cambia status a `researched`; (4) **handoff**: un `DraftingAgent` ve `researched`, escribe el draft, status `drafted`; (5) **completion**: un `EditorAgent` hace proofreading y marca `complete`. (Código: cada agente tiene un `check_for_tasks(shared_task_board)` que actúa solo si el status matchea su capacidad; modelo pull-based decoupled.)
- **Consecuencias** — *Pros*: **resilience** (sin controlador central → sin single point of failure; sigue operando aunque algunos agentes caigan), **scalability** (peer-to-peer escala horizontal agregando agentes). *Cons*: **debuggability** (el flujo emergente y no-lineal es difícil de debuggear/predecir), **governance** (enforcing reglas/compliance sin autoridad central es difícil).
- **Guía** — para la mayoría de las apps enterprise, empezar con **Supervisor** (más fácil de construir/debuggear/gobernar). Algunas apps se benefician de descentralización con el tiempo (Swarm: más resiliente y adaptativo). En la práctica, muchos sistemas complejos adoptan un **modelo híbrido**: un orquestador top-level gestiona el proceso de negocio pero delega sub-goals grandes a "swarms"/"crews" auto-organizados que manejan los detalles entre sí — combinando la claridad del control central con la adaptabilidad de la decisión distribuida.

![[05-fig-5.3.png|565]]
*Figura 5.3 – Swarm Architecture workflow*

### Comparación Supervisor vs Swarm (Tabla 5.2)

| Feature | Supervisor (centralizado) | Swarm (descentralizado) |
|---|---|---|
| Control flow | Jerárquico: un orquestador delega a workers. | Peer-to-peer: los agentes se auto-seleccionan o se pasan tareas. |
| Coordinación | Explícita y top-down: el supervisor gestiona el workflow. | Emergente y bottom-up: surge de interacciones locales. |
| Modularidad | Alta: especialistas fáciles de agregar/reemplazar bajo el supervisor. | Alta: agentes autónomos, se agregan/quitan del swarm. |
| Beneficio clave | Predictibilidad y oversight claro; fácil de debuggear/gobernar. | Resiliencia y adaptabilidad; sin single point of failure. |
| Drawback clave | El supervisor puede ser cuello de botella o single point of failure. | Difícil de gobernar/debuggear; comportamiento menos predecible. |
| Mejor para | Procesos de negocio estructurados, workflows con pasos claros (ej. loan processing). | Tareas creativas, problem-solving dinámico, entornos de alta resiliencia. |

## 3. Agent Composition Topologies

Mientras los frameworks de delegación definen las "rules of engagement" (centralizado vs descentralizado), las **topologías de composición** describen el arreglo estructural de agentes y datos en más detalle, atacando knowledge convergence, market-based task allocation y fault tolerance.

### 3a. Blackboard Knowledge Hub

- **Qué es** — repositorio central (el *Blackboard*) que tiene facts e hipótesis **tipadas y versionadas**; los knowledge sources (agentes) NO se comunican directamente sino que postean updates al Blackboard; un *controller* arbitra las fases (post → evaluate → integrate). Para problemas complejos donde múltiples especialistas aportan facts parciales o inciertos a una solución que evoluciona dinámicamente.
- **Contexto** — problemas mal definidos donde ningún agente único tiene todo el conocimiento; la solución requiere acumulación incremental de contribuciones de varios "expertos" cuyo orden no puede predeterminarse.
- **Problema** — ¿cómo colaboran agentes independientes en una solución en desarrollo sin canales directos y tightly-coupled que se volverían inmanejables a escala? *Fuerzas*: shared understanding vs race conditions, openness vs quality control, consistencia global vs autonomía.
- **Solución** — el Blackboard como repositorio central de facts/hipótesis tipadas y versionadas; el controller asegura que las contribuciones se validen y la solución converja lógicamente.
- **Ejemplo (diagnóstico médico colaborativo)** — (1) **posting**: un `SymptomAnalysisAgent` postea "Patient has fever and rash"; (2) **triggering**: al ver "rash", el `DermatologyAgent` analiza la imagen y postea "Rash indicates potential viral infection (Confidence: 0.8)"; (3) **refining**: el `VirologyAgent` lee la hipótesis y pide "Blood Test Results"; (4) **convergence**: el controller evalúa los facts colectivos hasta que un `DiagnosisAgent` sintetiza el diagnóstico final con suficiente confianza. (Código: `Blackboard` thread-safe con `post_hypothesis(agent, hypothesis, confidence, timestamp)`; `Controller.run_cycle` con selección → ejecución → evaluación/convergencia.)
- **Consecuencias** — *Pros*: **flexibility** (excelente para problemas ill-posed que requieren contribuciones iterativas), **auditability** (el log append-only da una historia clara de cómo evolucionó la solución, crucial para el "chain of thought" del sistema). *Cons*: **latency** (escritura central + evaluación → más lento que message passing directo), **bottleneck** (el controller puede ser cuello de botella si el blackboard no se shardea/indexa).
- **Guía** — mejor con muchos "weak experts" (agentes especializados pero limitados) o cuando se necesita convergencia trazable. Evitar para tareas simples low-latency (el overhead supera el beneficio). Implementar una "cleanup strategy" / "forgetting mechanism" para prunear facts viejos o invalidados, o el blackboard se vuelve un scratchpad ruidoso que degrada la performance.

![[05-fig-5.4.png|384]]
*Figura 5.4 – Blackboard topology*

### 3b. Contract-Net Marketplace (Mediator + Bids)

- **Qué es** — mecanismo de negociación basado en mercado (**Contract-Net Protocol**): un *solicitor*/manager broadcastea anuncios de tarea a workers potenciales; los *bidders* evalúan y responden con un bid formal (capability, cost, ETA, confidence score); el solicitor actúa de *awarder*, asignando la tarea al agente con el mayor utility score. Para seleccionar el best-fit agent en runtime en vez de hardcodear la asignación.
- **Contexto** — entorno distribuido con un pool diverso de agentes cuyas capacidades se solapan pero su disponibilidad, costo y performance varían dinámicamente; el routing estático es frágil e ineficiente (no considera load real-time ni matices de la tarea).
- **Problema** — ¿cómo asignar una tarea al agente más adecuado cuando la elección óptima depende de factores dinámicos (disponibilidad, costo, confidence) conocidos solo en runtime? *Fuerzas*: especialización vs overhead de routing, competitive bidding vs costo de coordinación, exploración vs SLAs.
- **Ejemplo (seleccionar cloud provider agent)** — "Train Model X. Max cost $100" → bids: `AWS_Agent` $90/2h, `Azure_Agent` $85/2.5h, `OnPrem_Agent` $10/12h → el solicitor pesa tiempo vs dinero y adjudica al `AWS_Agent` (mejor balance). (Código: `Solicitor.request_task_fulfillment` broadcastea, evalúa bids por utility function y adjudica; `BidderAgent.receive_announcement` rechaza si no puede, o devuelve un `Bid(cost, confidence)`.)
- **Consecuencias** — *Pros*: **adaptive selection** (se adapta dinámicamente a disponibilidad/capacidades cambiantes sin cambios de código), **high utilization** (desacopla requester de provider; el trabajo fluye a los mejores en ese momento). *Cons*: **auction latency** (la negociación introduce overhead antes de empezar), **risk of gaming** (sin incentivos para bidding honesto, los agentes pueden sobrestimar su confidence para ganar tareas).
- **Guía** — usar con un toolset grande/variable o cuando se optimiza por factores dinámicos (costo/velocidad). Evitar para workflows fijos y predecibles (routing estático es más simple/rápido). El solicitor debe imponer **deadlines estrictos** para recibir bids (evitar espera infinita). Considerar un **"reputation score"** para penalizar agentes que ganan contratos pero no entregan calidad.

![[05-fig-5.5.png|546]]
*Figura 5.5 – Contract-Net Protocol*

### 3c. Supervision Tree with Guarded Capabilities

- **Qué es** — patrón para **contener fallos e imponer fronteras de seguridad**; organiza agentes en un árbol jerárquico donde los supervisores monitorean la salud y el comportamiento de sus subordinados. Combina la filosofía "let it crash" del **Actor Model** (Erlang/Akka) con **capability guarding** estricto (los agentes solo acceden a las tools requeridas para su rol), más recovery automático y self-healing.
- **Contexto** — derivado del actor model, aplicado a IA agéntica; esencial donde muchos agentes hacen tool calls autónomos y potencialmente riesgosos (ejecutar código generado, scrapear web, interactuar con APIs externas inestables).
- **Problema** — ¿cómo contener fallos de agentes autónomos sin crashear toda la aplicación, dándoles suficiente libertad para operar? *Fuerzas*: safety vs velocidad, aislamiento vs coordinación, agilidad del developer vs enforcement de políticas (least privilege).
- **Solución** — un **Supervision Tree**: los supervisores son agentes especializados que gestionan únicamente el lifecycle de sus children. Si un child crashea o viola una política, el supervisor lo detecta y aplica una recovery strategy (ej. restart). Las capabilities se otorgan **por subtree** (una rama "Research" no puede acceder a tools de "Billing").
- **Ejemplo (web scraper resiliente)** — (1) **spawn**: el `Root Supervisor` spawnea un `ResearchSupervisor` que spawnea tres `ScraperAgents`; (2) **failure**: un ScraperAgent encuentra un captcha bloqueante y lanza un error fatal; (3) **detection**: el ResearchSupervisor detecta la señal de crash; (4) **recovery**: con estrategia **"one-for-one"**, reinicia *solo* el ScraperAgent fallido con estado fresco, dejando los otros corriendo. Fallo contenido. (Código: `SupervisorAgent` con `strategy`, `spawn_child(agent_cls, tools)` que aísla capabilities pasando solo tools específicas, `monitor_loop` y `handle_failure` con `ONE_FOR_ONE` o `ESCALATE`.)
- **Consecuencias** — *Pros*: **high resilience** (recovery automático sin intervención humana), **blast-radius control** (un crash en una rama riesgosa no afecta las ramas seguras ni el root). *Cons*: **complexity** (más coreografía arquitectónica; pensar en términos de árboles y lifecycle), **communication overhead** (la comunicación cross-tree requiere gateways claros / mailboxes; los agentes no pueden "agarrar" data de un sibling).
- **Guía** — crítico para producción con tools inestables (web browsing, code execution). Evitar árboles profundos para utilities simples single-shot (el overhead domina). Implementar **backoff logic**: si un child crashea 5 veces en 1 segundo, dejar de reiniciarlo (evitar "crash loop" que quema recursos). Imponer que los children no bypasseen a su supervisor para comunicarse con el root.

![[05-fig-5.6.png|636]]
*Figura 5.6 – Supervision Tree with Guarded Capabilities*

## 4. Multi-Agent Planning

- **Qué es** — el motor cognitivo: dado un framework de delegación, el método concreto para atacar goals complejos descomponiendo un objetivo de alto nivel (ej. "launch a new product") en una serie de acciones coordinadas más chicas asignables a agentes especializados (*collaborative task decomposition*, típicamente del orquestador). El plan se vuelve un *shared artifact* que guía el comportamiento colectivo.
- **Contexto** — un problema complejo demasiado grande/multifacético para un agente único; el goal es claro pero la secuencia de pasos y la división del trabajo no.
- **Problema** — ¿cómo crear y ejecutar colaborativamente un plan unificado hacia un goal común? Sin plan compartido, los agentes hacen trabajo redundante, ejecutan en orden equivocado o no combinan resultados → ineficiencia o fallo. *Fuerzas*: descomposición vs cohesión, especialización vs overhead, planning estático vs dinámico.
- **Solución** — descomponer el goal en un grafo/secuencia de sub-tareas manejables y asignarlas a los agentes más apropiados.
- **Ejemplo (market analysis report)** — "Generate a comprehensive market analysis report for Product X" → el orquestador descompone en: (1) `gather_sales_data` → `DataRetrieverAgent`; (2) `analyze_competitor_chatter` → `SocialMediaMonitoringAgent`; (3) `summarize_analyst_reports` → `FinancialDocsAgent`; (4) `synthesize_findings_and_draft_report` → `ReportWriterAgent`. Las sub-tareas se ejecutan en paralelo o secuencialmente, con dependencias gestionadas por el orquestador. (Código: `MarketAnalysisOrchestrator` usa `concurrent.futures.ThreadPoolExecutor` para correr las tareas independientes en paralelo, luego maneja la dependencia final pasando todos los hallazgos al report writer.)
- **Consecuencias** — *Pros*: **efficiency** (especialización + ejecución paralela), **leverages specialization** (resultados de mayor calidad que un agente general). *Cons*: **coordination overhead** (el planning consume recursos, puede ser cuello de botella), **risk of rigidity** (un plan estático falla si el entorno cambia o una sub-tarea falla).
- **Guía** — el plan debe ser **flexible, no estático**: adaptarse ante info nueva o sub-tareas fallidas. Definir claramente las **dependencias** entre sub-tareas para handoffs fluidos y evitar errores de ejecución.

![[05-fig-5.7.png|733]]
*Figura 5.7 – Multi-Agent Planning workflow*

## 5. Knowledge Sharing

- **Qué es** — mecanismo para crear **inteligencia colectiva**: asegurar que la info valiosa descubierta por un agente sea accesible por todos. Implementa una **Shared Epistemic Memory** (data store global y persistente que todos los agentes leen y escriben) — knowledge graph, vector DB u otro storage persistente —, yendo más allá del simple message passing.
- **Contexto** — los agentes adquieren info valiosa o aprenden skills por experiencia (uno aprende la forma más eficiente de consultar una DB, otro a identificar un nuevo tipo de queja); sin sharing, ese conocimiento queda siloed.
- **Problema** — ¿cómo compartir el conocimiento/experiencia de un agente con los demás para mejorar la inteligencia colectiva? *Fuerzas*: siloed knowledge vs inteligencia colectiva, facilidad de escritura vs esfuerzo de retrieval, propagación vs integridad.
- **Ejemplo (soluciones de customer service compartidas)** — (1) `Agent_A` descubre que "Error 503 on ProWidget X" se resuelve haciendo que el usuario limpie la cache del device; (2) escribe esa solución exitosa en una vector DB compartida; (3) semanas después, `Agent_B` encuentra un issue similar, hace una búsqueda semántica en el knowledge base compartido y encuentra de inmediato la solución de `Agent_A`, resolviéndolo al primer intento. (Código: `SHARED_KNOWLEDGE_BASE = VectorDatabase()`; `AgentA.handle_issue` hace `add_entry(knowledge_entry)`; `AgentB.handle_issue` hace `semantic_search(user_query)` — previene la "amnesia" de conocimiento siloed en sesiones individuales.)
- **Consecuencias** — *Pros*: **collective intelligence** (el sistema se vuelve más capaz que la suma de sus partes; aprende de la experiencia colectiva), **efficiency** (resuelve problemas recurrentes más rápido). *Cons*: **data integrity** (riesgo de propagar info incorrecta o maliciosa), **governance overhead** (se necesita un sistema para gestionar/verificar/prunear el knowledge base).
- **Guía** — **knowledge representation**: formato estructurado (JSON) para facts objetivos específicos; texto no estructurado en vector DBs para conocimiento experiencial matizado. **Provenance**: trackear la fuente de todo conocimiento compartido (evaluar confiabilidad, debuggear propagación de info incorrecta). **Trust y verification**: permitir que los agentes raten/validen info de sus peers, o delegar un "governance agent" especializado que revise y prunee periódicamente.

![[05-fig-5.8.png|653]]
*Figura 5.8 – Agent information sharing*

## 6. Tool Routing in Multi-Agent Contexts

- **Qué es** — framework para gestionar y dirigir el uso de tools a través del sistema: asegurar que el agente correcto llame a la tool correcta para una tarea específica. Mejora el foco y reduce decision fatigue dando a cada agente/supervisor un prompt que describe **solo sus tools relevantes**: las capabilities se *scopean*, en vez de que cada agente acceda a cada tool.
- **Contexto** — sistema multi-agente con variedad de tools (APIs, funciones, DBs); cuando una tarea requiere una capacidad específica, el sistema debe decidir qué agente invoca qué tool.
- **Problema** — ¿cómo asegurar que la tool correcta se seleccione para una sub-tarea y sea invocada por el agente más apropiado? El misalignment goal↔tool degrada performance, da resultados incorrectos o desperdicia recursos. *Fuerzas*: accuracy vs flexibilidad, centralización vs bottleneck, especialización de tools vs tool discovery.
- **Ejemplo (asistente personal inteligente)** — "What is the current stock price of Google?" → (1) el `Router Agent` central clasifica como `financial_query`; (2) delega al `FinancialAgent` (que tiene las API keys/tools de market data); (3) el FinancialAgent usa su tool `get_stock_price`; (4) el resultado vuelve al usuario, mientras el `WeatherAgent` y el `TravelAgent` quedan sin tocar (no alucinan respuestas ni usan mal sus tools). (Código: `CentralOrchestrator` con `AGENT_ROUTING_MAP` (request_type → agent), `classify_request` vía LLM, `handle_request` que delega; cada agente especializado solo conoce sus propias tools.)
- **Consecuencias** — *Pros*: **higher accuracy** (limitar las opciones de tools por agente reduce invocaciones incorrectas), **focus** (agentes muy especializados, más eficientes). *Cons*: **rigidity** (menos flexible si una tarea inesperadamente requiere una tool fuera del set predefinido), **upfront design** (requiere diseño y mantenimiento cuidadoso del routing map).
- **Guía** — para muchos tools, crear un **tool registry** que los agentes puedan consultar (routing más dinámico que asignaciones hardcoded). La lógica de routing puede delegarse a un **router agent LLM-powered** que use function-calling para seleccionar el agente y su tool asociada.

![[05-fig-5.9.png|451]]
*Figura 5.9 – Centralized tool routing example implementation*

## 7. Consensus

- **Qué es** — familia de protocolos para que un grupo de agentes **acuerde sobre un dato o el estado del sistema**; no es simple votación sino un proceso estructurado de comunicación y convergencia, asegurando que el sistema opere desde un punto de vista único y confiable (previene errores de actuar sobre info contradictoria).
- **Contexto** — múltiples agentes con info distinta, incompleta o conflictiva sobre el estado del entorno; para una acción coordinada deben acordar primero un entendimiento compartido.
- **Problema** — ¿cómo lograr un acuerdo garantizado sobre un valor/estado aun con data ruidosa o desacuerdos menores? *Fuerzas*: acuerdo vs precisión individual (forzar un compromiso a costa de perder precisión individual), convergencia vs tiempo, honest actors vs malicious actors.
- **Solución** — un protocolo donde los agentes convergen a un estado común, a menudo vía **debate iterativo**: broadcastean sus beliefs actuales, reciben los de otros, y ajustan los propios según una regla predefinida; se repite hasta que todos convergen dentro de una tolerancia aceptable.
- **Ejemplo (debate de forecasting financiero)** — generar un forecast consensuado de revenue del próximo trimestre. (1) **Round 1**: `OptimistAgent` $110M, `PessimistAgent` $95M, `RealistAgent` $102M; (2) **debate iterativo (Round 2)**: comparten forecasts, calculan el promedio ($102.3M) y ajustan parcialmente hacia esa media; (3) **convergence**: el rango entre el más alto y el más bajo se estrecha cada round hasta caer dentro de la tolerancia; (4) **action**: forecast final $103M. (Código: `ConsensusManager.get_consensus_forecast` con `tolerance` y `max_rounds`, chequea convergencia y hace share-and-adjust; `FinancialAgent.adjust_forecast` con `adjustment_factor=0.5` hacia el promedio.)
- **Consecuencias** — *Pros*: **reliability** (decisiones basadas en un entendimiento compartido y validado, no un único data point potencialmente viciado), **fault tolerance** (el fallo de un agente no necesariamente detiene el proceso). *Cons*: **latency** (múltiples rounds de comunicación → no apto para sistemas real-time), **complexity** (manejar edge cases: fallo de agentes, network partitions, malicious actors).
- **Guía** — **termination condition** clara (máximo de rounds o threshold de convergencia, evitar loops infinitos). El **convergence algorithm** como lógica primaria (de un promedio numérico simple a métodos que pesan opiniones según la reliability histórica del agente). Priorizar **explainability** (registrar el razonamiento y estados intermedios del debate como audit trail).

![[05-fig-5.10.png|340]]
*Figura 5.10 – Agent consensus workflow*

## 8. Agent Negotiation

- **Qué es** — protocolo estructurado para que agentes con goals en competencia resuelvan el conflicto entre sí (en vez de una decisión top-down subóptima), vía un diálogo back-and-forth de ofertas y contraofertas para hallar un compromiso. Fuertemente influenciado por **game theory** (agentes como actores racionales maximizando su utility).
- **Contexto** — múltiples agentes autónomos con self-interests o goals conflictivos que necesitan un acuerdo mutuamente aceptable.
- **Problema** — ¿cómo alcanzan agentes self-interested un acuerdo mutuamente beneficioso sin una autoridad central que dicte el resultado? Un enfoque fijo y no negociable lleva a deadlocks o resultados subóptimos donde se pierde un potencial "win-win". *Fuerzas*: autonomía vs alineación, fairness vs eficiencia, comportamiento estratégico vs transparencia.
- **Solución** — protocolo típico: iniciación (oferta inicial), evaluación, respuesta (accept/reject/counter-offer); itera hasta acuerdo o termination.
- **Ejemplo (negociar un recurso compartido)** — `AnalyticsAgent` (modelo financiero high-priority, ventana 2h, idealmente 2:00 AM, flexibilidad baja) y `TrainingAgent` (retraining low-priority, ventana 4h, también 2:00 AM, flexibilidad media), mediados por un `ResourceManagerAgent`. (1) **conflict detection**: ambos piden lock del GPU a las 2:00 AM; (2) **iniciación**: el manager les pide prioridad y flexibilidad; (3-4) responden con sus dicts; (5) **evaluación**: la política prioriza "high" → pide al TrainingAgent una nueva hora; (6) **counter-offer**: "I can defer. I propose to start my 4-hour job at 4:00 AM"; (7) **agreement**: el manager confirma; (8) **confirmation**: AnalyticsAgent 2:00-4:00 AM, TrainingAgent 4:00-8:00 AM. (Código: `ResourceManagerAgent.handle_requests` detecta conflicto, identifica el lower-priority, le pide `propose_new_time()` y chequea si resuelve.)
- **Consecuencias** — *Pros*: **flexibility** (acuerdos dinámicos con mejores outcomes que políticas rígidas), **optimality** (descubre soluciones "win-win" no obvias para una autoridad central). *Cons*: **time y complexity** (puede ser lento e intensivo, sin garantía de acuerdo), **no guarantee** (puede fallar → deadlock si no hay fallback).
- **Guía** — definir **termination conditions y fallback positions** claras (¿qué pasa si no hay deal? el agente necesita un plan B). Loggear toda la secuencia de ofertas/contraofertas para auditabilidad y oversight humano.

![[05-fig-5.11.png|422]]
*Figura 5.11 – Agent Negotiation workflow*

## 9. Resource Allocation

- **Qué es** — enfoque estructurado para distribuir recursos finitos (cómputo, bandwidth, acceso a una API, assets físicos como brazos robóticos) entre agentes con demandas en competencia, yendo más allá del "first-come, first-served" hacia un modelo inteligente y goal-oriented.
- **Contexto** — pool finito de recursos que debe distribuirse entre múltiples agentes con necesidades en competencia.
- **Problema** — ¿cómo distribuir recursos limitados de forma eficiente, justa y alineada con los goals del sistema? Sin estrategia clara → contención, bottlenecks, performance subóptima. *Fuerzas*: throughput vs fairness (prevenir *starvation* de los low-priority), control centralizado vs overhead, predictibilidad vs adaptabilidad.
- **Solución** — mecanismos clave: **centralized allocator** (un agente manager decide con vista global de prioridades/disponibilidad), **auction mechanisms** (los agentes "bid" con una currency/priority score interno; el mejor bid gana por un período), **fair division algorithms** (cuando la fairness es primordial, ej. división envy-free donde ningún agente envidia la share de otro).
- **Ejemplo (robots autónomos en una smart factory)** — un `AMR_DispatcherAgent` central asigna una flota limitada de **AMRs (autonomous mobile robots)** según prioridad. (1) `ProductionLine_A_Agent` pide high-priority (componente crítico, line stoppage inminente); (2) `WarehouseAgent` pide low-priority (inventory count rutinario); (3) `ShippingAgent` pide medium-priority (finished goods, shipment en 2h). El dispatcher asigna inmediato el próximo AMR a ProductionLine_A (evitar el costoso stoppage), luego uno al ShippingAgent (deadline), y deja al WarehouseAgent en cola hasta que haya un robot libre y nada de mayor prioridad pendiente. (Código: `AMR_DispatcherAgent` con `high/medium/low_priority_queue` y `available_amrs`; `dispatch()` procesa primero la high.)
- **Consecuencias** — *Pros*: **optimization** (recursos escasos hacia las tareas más críticas, maximiza utility global), **stability** (previene contención, deadlocks, race conditions). *Cons*: **overhead** (el proceso de allocation introduce latencia), **risk of starvation** (reglas mal diseñadas → los low-priority nunca reciben recursos; se necesita un safeguard).
- **Guía** — lógica de allocation **transparente y explícita** (prioridad/bidding/fairness; clave para debugging y explainability). En sistemas de subasta, diseñar las reglas para fomentar bidding del valor real (**incentive compatibility** — previene misrepresentación de necesidades). El mecanismo debe adaptarse a condiciones cambiantes y **preempt** tareas low-priority si surge una crítica.

![[05-fig-5.12.png|737]]
*Figura 5.12 – Resource Allocation*

## 10. Conflict Resolution

- **Qué es** — framework estructurado para identificar puntos de contención y resolverlos evitando fallo del sistema y alineando con los goals globales (ej. el plan de un agente de mover un brazo robótico choca con el de otro; dos agentes financieros generan recomendaciones de trade opuestas para la misma acción). Crítico para la estabilidad del sistema.
- **Contexto** — dos o más agentes con acciones planeadas o goals conflictivos (ej. dos logistics agents intentan rutear sus camiones por la misma calle angosta a la vez).
- **Problema** — ¿cómo resolver desacuerdos/planes conflictivos para evitar deadlocks, condiciones inseguras o resultados subóptimos? *Fuerzas*: safety vs velocidad operativa, autoridad centralizada vs agilidad distribuida, consistencia lógica vs goal attainment.
- **Solución — cuatro enfoques comunes**:
  - **Hierarchical resolution** — un supervisor/orquestador tiene autoridad para sobreescribir agentes conflictivos e imponer una decisión; el más directo, outcomes claros y predecibles; default en enterprise donde compliance/safety/auditabilidad son primordiales.
  - **Policy-based resolution** — políticas/reglas predefinidas gobiernan automáticamente cómo se resuelven ciertos tipos de conflicto (ej. "safety-critical agents always have priority over efficiency-optimizing agents"); determinista, consistente, externaliza la lógica de decisión (fácil de entender/modificar/auditar para humanos).
  - **Negotiation** — los agentes conflictivos entran en el proceso de Negotiation para un compromiso mutuamente aceptable; bottom-up, apto cuando hay margen para "win-win"/"less-lose".
  - **Game-theoretic resolution** — modelar el conflicto como un juego formal; cada acción tiene payoffs/costs; identificar un outcome estable como un **Nash equilibrium** (ningún agente se beneficia cambiando unilateralmente su estrategia); computacionalmente intensivo pero diseña sistemas probadamente estables donde la cooperación emerge del interés propio racional.
- **Ejemplo (conflicto de workflow enterprise)** — `ThroughputAgent` (procesar el máximo de loan applications/hora por un KPI de velocidad) vs `FairnessAgent` (análisis intensivo de bias demográfico por batch, que ralentiza). Conflicto: el Throughput intenta empujar un batch directo a approval mientras el Fairness lo flaggea para un review de 20 min. (1) **conflict detection**: ambos planes ("advance to approval" vs "hold for fairness review") logueados como mutuamente exclusivos en un `SupervisorAgent`; (2) **policy-based resolution**: el supervisor consulta su policy framework y halla la política no-negociable "All loan batches must receive a FAIRNESS_PASSED status before proceeding... Compliance and ethical guidelines override speed-related KPIs"; (3) **resolution**: invalida el plan del Throughput, le manda "Halt and await fairness check completion", confirma al Fairness su prioridad; (4) **continuation**: cuando el Fairness marca `FAIRNESS_PASSED`, el supervisor deja al Throughput reanudar. (Código: `SupervisorAgent` con `POLICY_FRAMEWORK`, `is_conflicting` (mismo target, distinta action), `approve_plan`/`deny_plan`.)
- **Consecuencias** — *Pros*: **coherence** (el sistema no termina en estado contradictorio/deadlock), **safety** (previene situaciones peligrosas — colisiones de robots, corrupción lógica de datos). *Cons*: **latency** (detección+resolución añaden overhead), **complexity** (diseñar políticas/protocolos robustos para cada escenario aumenta mucho el esfuerzo de ingeniería).
- **Guía (rica)** — **conflict detection** primero (el "early warning system": un supervisor central que monitorea acciones/planes; resource locking; registrar acciones intencionadas en un shared space antes de ejecutar). **Explainable resolutions / audit trail** (loggear por qué se eligió cada resolución, ej. "Agent B's plan approved over Agent A's because policy [N] states safety-critical tasks have priority"). **Defined escalation paths / human-in-the-loop** (fallback último: un humano que revisa el contexto y da el juicio final). **Simulate to understand** (stress-test de las estrategias bajo condiciones variadas antes de deployar, identificar deadlocks/comportamientos emergentes indeseados).

![[05-fig-5.13.png|509]]
*Figura 5.13 – Conflict Resolution workflow*

## 11. Formation Control

- **Qué es** — principio de diseño para **movimiento colectivo y organización espacial**: un grupo de agentes mantiene una estructura física o lógica definida unos respecto de otros mientras se mueven por un entorno. Permite que un "swarm"/"squad" actúe como una entidad única y coordinada, reaccionando fluidamente sin un controlador centralizado.
- **Contexto** — un grupo de agentes que necesita mantener una estructura física/lógica específica entre sí mientras se mueve/actúa; común en robótica, simulaciones complejas, acción colectiva unificada.
- **Problema** — ¿cómo mantener dinámicamente una formación colectiva sin un controlador centralizado rígido que dicte la posición exacta de cada agente? Depender de un líder único crea single point of failure y le cuesta adaptarse a obstáculos → colisiones, formación rota, navegación ineficiente. *Fuerzas*: coherencia global vs sensing local, rigidez estructural vs evasión de obstáculos, latencia de comunicación vs velocidad de sincronización.
- **Solución** — **descentralizar la lógica de control**: cada agente decide según las posiciones/estados de sus *vecinos inmediatos*, no según comandos de un líder central. Cada agente programado con un set simple de *control laws* (distancia y bearing deseados respecto de sus vecinos designados); la formación se adapta fluidamente porque la reacción localizada de cada agente cascadea por la formación → respuesta colectiva emergente.
- **Ejemplo (swarm de drones agrícolas)** — flota fumigando un campo en grilla precisa. (1) **formation rule**: cada drone con "Maintain a position 10 meters to the right of your left neighbor and directly aligned with your forward neighbor"; (2) **coordinated movement**: al avanzar el lead, cada drone sigue ajustando velocidad/posición según sus vecinos; (3) **dynamic adaptation**: `Drone_C` (centro) detecta un árbol en su path y autónomamente lo esquiva; (4) **self-organization**: los vecinos (`Drone_B` izq, `Drone_D` der, y el de atrás) sensan el cambio y reducen velocidad para evitar colisión; (5) **re-formation**: una vez libre, `Drone_C` acelera de vuelta a su posición y los vecinos se reajustan, restableciendo la formación sin comando central. (Código: `DroneAgent` con `DESIGNATED_OFFSET` y un `control_loop`: sense neighbor → calcular desired position → calcular position_error → si supera TOLERANCE, ajustar velocidad.)
- **Consecuencias** — *Pros*: **scalability** (la formación crece a cientos/miles de agentes sin aumentar la carga de ningún controlador, decisiones locales), **resilience** (robusto al fallo de agentes individuales; si uno cae, los vecinos cierran el gap). *Cons*: **local optima** (agentes con info local pueden quedar atascados en obstáculos complejos — ej. un cul-de-sac — que un global planner evitaría), **stability risks** (control laws mal tuneadas → oscilación, la formación tiembla por sobre-corrección).
- **Guía** — **neighbor discovery** confiable (comunicación directa low-latency tipo mesh local, u observar un shared state). Las **control laws** son el core: diseñarlas con principios de **control theory** para estabilidad (que no oscile ni se rompa bajo estrés). **Simulaciones** esenciales para testear/refinar las control laws antes de deployar en un entorno físico.

![[05-fig-5.14.png|407]]
*Figura 5.14 – A single agent's control loop for Formation Control*

## Citas

> "Just as a company relies on a team of specialized employees, advanced AI solutions require a team of specialized agents working in concert."
> "Coordination is architected, not assumed"
> A2A como mecanismo; el Agent Router como "the 'Hello World' of agentic coordination"
> "Frameworks and topologies dictate control flow"
> "Protocols for disagreement are essential"

## Para aplicar

- **Arquitecturar la coordinación, no asumirla** — elegir explícitamente patrones de delegación, planning e interacción; mapear su complejidad al nivel de madurez (Nivel 4 centralizado/predecible → Niveles 5-6 dinámico/descentralizado).
- **Empezar con un Agent Router** — desacoplar intención↔agente vía intent extraction (function calling, schema "Goldilocks" de 10-20 actions/resources) + graph-constrained routing (whitelist de capacidades); cachear semánticamente queries repetidas.
- **Elegir el framework de delegación según el dominio** — Supervisor (enterprise estructurado, auditable; checkpointing con LangGraph, schemas estrictos, separación de concerns anti-"God agent") vs Swarm (creativo/dinámico, resiliente, shared task board pull-based); o híbrido (orquestador + crews auto-organizados).
- **Aplicar la topología que el problema pide** — Blackboard (convergencia trazable de muchos weak experts, con forgetting mechanism), Contract-Net (asignación market-driven con deadlines + reputation score), Supervision Tree (fault tolerance "let it crash" + capability guarding por subtree + backoff anti crash-loop).
- **Dar contexto compartido** — Multi-Agent Planning (descomposición flexible con dependencias claras, paralelizar lo independiente) + Knowledge Sharing (Shared Epistemic Memory con provenance y trust/verification) + Tool Routing (capabilities scopeadas, tool registry).
- **Gestionar la fricción con protocolos** — Consensus (debate iterativo con termination + convergence algorithm, ponderar por reliability), Negotiation (ofertas/contraofertas con fallback y log), Resource Allocation (priority/auction/fair-division, anti-starvation, incentive compatibility), Conflict Resolution (detección temprana + audit trail + escalación humana + simulación; hierarchical/policy/negotiation/game-theoretic), Formation Control (control laws con control theory, neighbor discovery, simular).

## Conexiones

- [[_Agentic Architectural Patterns for Building Multi-Agent Systems|Mapa del libro]] — el MOC del libro.
- [[04 - Agentic AI Architecture - Components and Interactions]] — cap. 4: la anatomía y los modelos de interacción (directa/stigmergy) que estos patrones de coordinación operacionalizan; el componente Coordinate (A2A).
- [[03 - The Spectrum of LLM Adaptation for Agents - RAG to Fine-tuning]] — cap. 3: el Agentic AI Maturity Model (Tabla 5.1 lo revisa) y la arquitectura jerárquica orquestador/sub-agents (Supervisor Architecture aquí); Router-Executor/Planner-Worker formalizados como Tool Routing + Supervisor.
- [[06 - Explainability and Compliance Agentic Patterns]] — cap. 6 (siguiente): la Shared Epistemic Memory (Knowledge Sharing/Consensus) y el shift de coordinación a accountability; Instruction Fidelity Auditing.
- [[07 - Robustness and Fault Tolerance Patterns]] — cap. 7: el Supervision Tree, el Watchdog Timeout y el Trust Decay (Nivel 6) reaparecen como patrones de robustez; Parallel Execution Consensus ↔ Consensus; Majority Voting ↔ resource/conflict.
- [[09 - Agent-Level Patterns]] — cap. 9: el Single Agent Baseline (Nivel 1) que estos patrones componen en colectivos.
- [[Orchestrator]] — el orquestador central de la Supervisor Architecture, Tool Routing, Multi-Agent Planning y Conflict Resolution.
- [[Generator-Evaluator Pattern]] — el debate iterativo del Consensus y el critique loop emparentan.
- [[A2A]] · [[MCP]] — la comunicación agente↔agente que subyace a la delegación/negociación/consensus (candidatos a nota propia).
- [[Quorum]] · [[Distributed Lock]] · [[Saga]] · [[Two-Phase Commit]] — patrones de System Design del vault que calzan: Quorum ↔ Consensus, Distributed Lock ↔ Resource Allocation/resource locking, Saga ↔ Multi-Agent Planning (orquestación de pasos con compensación).
- [[Load Balancing]] · [[Message Queue]] · [[Pub-Sub]] — Load Balancing ↔ el balanceo de instancias del Swarm/scalability; Pub-Sub/Message Queue ↔ el shared task board y la comunicación indirecta (stigmergy).
- [[_RAG|RAG]] · [[Hybrid Search]] · [[Reranking]] — la vector DB del Knowledge Sharing y el semantic cache del Agent Router.
- [[LangGraph]] — el framework de checkpointing recomendado para la Supervisor Architecture.
- [[Function Calling]] · [[Tool Calling]] — la base del Agent Router (intent extraction) y el Tool Routing.
- **Swarm Architecture** · **Blackboard topology** · **Contract-Net Protocol** · **Supervision Tree** · **Actor Model** · **Nash equilibrium** · **Stigmergy** · **Meta-agent** · **Incentive compatibility** · **Formation Control / control laws** — conceptos del capítulo; candidatos a nota propia.
