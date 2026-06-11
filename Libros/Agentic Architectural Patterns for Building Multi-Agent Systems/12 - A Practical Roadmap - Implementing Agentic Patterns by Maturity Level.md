---
title: "12 - A Practical Roadmap - Implementing Agentic Patterns by Maturity Level"
libro: Agentic Architectural Patterns for Building Multi-Agent Systems
autor: Ali Arsanjani
capitulo: 12
created: 2026-06-11
tags:
  - libros/agentic-architectural-patterns
  - type/case-study
  - status/permanent
aliases:
  - A Practical Roadmap - Implementing Agentic Patterns by Maturity Level
---

# 12 - A Practical Roadmap - Implementing Agentic Patterns by Maturity Level

> [!info] Capítulo 12 · *Agentic Architectural Patterns for Building Multi-Agent Systems* — Ali Arsanjani
> La síntesis de toda la Parte 2: *"¿por dónde empiezo?"*. Una hoja de ruta de **adopción progresiva** que ordena todos los patrones del libro en **3 niveles de madurez** (Foundational → Production-Ready → Self-Improving), cada uno con su objetivo, principio core, set de patrones y foco de implementación; más una **guía de reflexión estratégica** (4 preguntas) y la Tabla 12.1. Navegá: [[_Agentic Architectural Patterns for Building Multi-Agent Systems|Mapa del libro]] · anterior [[11 - Advanced Adaptation - Building Agents That Learn]].

## Resumen

Tras explorar un *pattern language* comprehensivo a lo largo de la Parte 2 —coordinación multi-agente (cap. 5), explicabilidad y compliance (cap. 6), robustez y fault tolerance (cap. 7), human-agent interaction (cap. 8), capacidades agent-level (cap. 9), infraestructura system-level (cap. 10) y adaptación avanzada (cap. 11)—, surge la pregunta natural y más apremiante: *"¿por dónde empiezo?"*. Implementar todos los patrones a la vez no solo es impráctico sino a menudo innecesario. La clave para desplegar IA agéntica con éxito es la **progressive adoption**: construir una base sólida de capacidades esenciales y luego *layerear* patrones más sofisticados a medida que crecen la complejidad, la escala y las responsabilidades del sistema. La hoja de ruta sirve a dos públicos: quien quiere una implementación simple se enfoca en el nivel foundational; quien necesita el mayor grado de sofisticación usa los niveles avanzados como *referencia enciclopédica*.

El capítulo **simplifica los 6 niveles del Agentic AI Maturity Model del cap. 3** mapeando 2-en-1 para un rollout enterprise más accesible, resultando en **3 niveles**: **Level 1 – Foundational System (basic)** (el mínimo set de patrones para un sistema agéntico funcional, single-process, desplegable en producción — validar el core business logic con un PoC confiable), **Level 2 – Production-Ready Service (intermediate)** (re-arquitecturar el sistema foundational en un set de microservicios desacoplados, resilientes y observables — scalability, comunicación asincrónica, fault tolerance robusto), y **Level 3 – Self-Improving Ecosystem (advanced)** (el cutting edge: self-optimization, especialización profunda de dominio y adaptive learning — un sistema que no solo ejecuta sino que mejora activamente con el tiempo). Cada nivel se documenta con la misma estructura (objetivo arquitectónico, principio core, patrones a implementar por categoría, foco de implementación, sistema resultante y consecuencias). El capítulo cierra con una **guía de reflexión estratégica** —cuatro preguntas para el equipo (¿dónde estás hoy? ¿cuál es tu MVA? ¿tu camino a producción? ¿tu North Star?)— y la **Tabla 12.1** que condensa nivel↔foco↔decisiones↔patrones↔pregunta clave. La lección: *"adopt progressively, align architecture with goals, strategy precedes implementation"*. No todo sistema necesita evolucionar a los niveles más altos — un Level 1 bien construido suele bastar para automatización interna no-crítica.

## Level 1 – The Foundational System (basic maturity)

- **Objetivo arquitectónico** — construir y validar rápidamente la lógica core de un workflow agéntico en una aplicación única, **monolítica y síncrona**. Crear un sistema funcional que pruebe el valor de negocio del enfoque agéntico, aunque todavía no esté optimizado para escala o resiliencia.
- **Principio core** — **simplicidad para habilitar validación rápida**. Probar el valor de negocio del workflow agéntico sin el overhead de los sistemas distribuidos. El sistema se construye como una única app monolítica donde un orquestador central llama síncronamente a los worker agents; diseño predecible, fácil de desarrollar y debuggear, el camino más rápido a un prototipo funcional.

![[12-fig-12.1.png]]
*Figura 12.1 – Level 1 patterns*

**Patrones a implementar** (los más esenciales de cada categoría):
- **Coordination & Planning (cap. 5)**: *Task Delegation Framework (Supervisor Architecture)* — el punto de partida natural: un único orquestador central responsable de todo el workflow, dando una cadena de mando clara. *Multi-Agent Planning* — el orquestador hace task decomposition básica, descomponiendo el request en una secuencia hardcoded o simple/dinámica de pasos.
- **Explainability & Compliance (cap. 6)**: *Basic Audit Logging* — aunque un *Causal Dependency Graph* completo sería overkill, toda acción y decisión debe loguearse a un archivo o consola con timestamps, agent IDs y outcomes. Es el **mínimo no-negociable** para debugging y accountability.
- **Robustness & Fault Tolerance (cap. 7)**: *Watchdog Timeout Supervisor* — safety net crítico: envolver todas las llamadas a agentes (sobre todo las de APIs externas) con un timeout para que un único agente colgado no congele toda la app. *Simple Retry Mechanism* — un loop básico para reintentar una tarea fallida un nº fijo de veces; maneja errores transitorios de red o blips de API.
- **Human-Agent Interaction (cap. 8)**: *Human Calls Agent* — el mecanismo de input primario para tareas transaccionales basadas en comandos. *Agent Calls Human* — la **válvula de seguridad esencial**: el sistema debe tener una forma simple y confiable de detenerse y escalar a un humano ante un error crítico o situación de baja confianza.
- **Agent-Level Capabilities (cap. 9)**: *Single Agent Baseline* — define la estructura de los worker agents, cada uno con un set específico de tools. *Agent-Specific Memory (Short-Term)* — como mínimo, los agentes necesitan gestionar el contexto de la sesión actual (ej. guardar el historial de conversación). *Context-Aware Retrieval (Simple RAG)* — para groundear el agente y prevenir alucinaciones básicas, conectarlo a una única knowledge source core vía un pipeline RAG directo.
- **System-Level Infrastructure (cap. 10)**: *Agent Authentication and Authorization* — aun en un sistema monolítico, si el agente accede a APIs o DBs internas, debe hacerlo con un service account seguro y permisos least-privilege claramente definidos.

**Implementation focus** (deliberadamente cambia scalability por velocidad y simplicidad):
- **Single-process architecture** — todo el workflow (del *Supervisor* al *Worker B*) corre en una única app o script, eliminando latencia de red y fallos de sistemas distribuidos; permite debuggear todo el flujo de lógica en una sola ventana del IDE.
- **Hardcoded orchestration** — la lógica que conecta el Supervisor con sus workers es estática; a diferencia de los sistemas avanzados que descubren agentes vía un registry, el Level 1 hardcodea las relaciones (ej. "si el paso A está hecho, llamá al Worker B") → workflow determinista y fácil de trazar.
- **In-memory state management** — la data no necesita DBs externas complejas; el estado se gestiona pasando un context object (ej. un dict de Python) entre funciones, manteniendo la arquitectura plana y sin el overhead de storage persistente durante la validación.

**Sistema resultante y consecuencias** — un sistema **funcional pero frágil (brittle)**: ejecuta su workflow en el "happy path" y maneja los errores más básicos, pero **no es escalable, tiene alta latencia** (por su naturaleza síncrona) y **no sobreviviría el crash de un componente**. Su valor primario es validar que el workflow agéntico y la lógica de razonamiento core resuelven efectivamente el problema de negocio. Esa prueba de valor justifica la inversión de ingeniería significativa para re-arquitecturar al Level 2.

> [!note] Muchas organizaciones pueden encontrar que un sistema Level 1 bien construido es **suficiente** para tareas de automatización internas y no-críticas. No todo sistema agéntico necesita evolucionar a los niveles más altos de madurez.

## Level 2 – The Production-Ready Service (intermediate maturity)

- **Objetivo arquitectónico** — re-arquitecturar el prototipo foundational en un set de **microservicios asincrónicos** desacoplados, escalables, resilientes y observables. Construir un sistema production-ready que sea cost-efficient, seguro y trustworthy para operaciones de negocio reales.
- **Principio core** — **decoupling para scalability y resilience**. Un sistema production-ready debe sobrevivir fallos de componentes y manejar cargas fluctuantes sin colapsar. *Pero la transición a microservicios debe estar guiada por la necesidad*: los microservicios introducen complejidad operativa significativa, así que este shift se recomienda **solo cuando el enfoque monolítico ya no cumple** los requisitos de escala, fault isolation o velocidad del equipo.

![[12-fig-12.2.png]]
*Figura 12.2 – Level 2 maturity*

Cuando el shift se justifica, la arquitectura se apoya en **tres cambios estructurales fundamentales**:
- **Asynchronous communication via message bus** — en vez de que *Agent A* llame directamente a *Agent B* y espere respuesta (blocking execution), *Agent A* publica un evento al message bus central; los agentes relevantes se suscriben y procesan en paralelo. Este decoupling previene que un único agente lento cree un cuello de botella que congele todo el sistema.
- **Service isolation** — cada agente opera como servicio independiente (típicamente containerizado); si *Agent A* crashea por un bug o memory leak, *Agent B* y *Agent C* quedan inafectados. Este aislamiento es lo que habilita el **Auto-Healing**: la infraestructura detecta el crash y reinicia el agente fallido específico automáticamente sin bajar la app.
- **Externalized state** — en Level 1 el estado vivía en la memoria del script; en Level 2 se empuja a **Shared Memory** (Redis o una DB), asegurando que si un agente se reinicia pueda recuperar el contexto/task status del store compartido y reanudar de inmediato, facilitando el **Incremental Checkpointing**.

(La infraestructura dedicada: *Tool and Agent Registry* permite al orquestador descubrir servicios disponibles dinámicamente —soportando el patrón *Agent Delegates to Agent*—, mientras *Shared Memory* tipo Redis persiste el estado fuera de los agentes individuales → si un agente crashea, se auto-healea sin perder el contexto de la transacción en curso.)

**Patrones a implementar** (construye sobre los foundational e introduce soluciones más sofisticadas):
- **Coordination & Planning (cap. 5)**: *Hybrid Delegation Framework* — el sistema puede evolucionar a un modelo híbrido donde un orquestador top-level delega tareas a swarms/crews auto-organizados. *Knowledge Sharing* — el simple estado in-memory se reemplaza por una **Shared Epistemic Memory** persistente (Redis cache o DB dedicada) accesible por todos los agentes; para estabilidad de producción debe incluir *concurrency controls* (prevenir race conditions), políticas **TTL (Time-to-Live)** para data hygiene, e indexación semántica para retrieval eficiente.
- **Explainability & Compliance (cap. 6)**: *Instruction Fidelity Auditing* y *Persistent Instruction Anchoring* — ahora formalmente implementados para prevenir instruction drift en workflows multi-hop más complejos. *Causal Dependency Graph* (cap. 7) — el logging básico se upgradea a un grafo estructurado y auditable que traza el linaje completo de cada decisión.
- **Robustness & Fault Tolerance (cap. 7)**: *Adaptive Retry with Prompt Mutation* — los retries simples se mejoran con lógica para modificar el prompt ante el fallo. *Auto-Healing Agent Resuscitation* y *Incremental Checkpointing* — el sistema puede reiniciar automáticamente agentes crasheados y reanudar tareas long-running desde el último estado guardado; para prevenir "crash loops" infinitos por errores persistentes, debe gobernarse con **exponential backoff** y **maximum retry thresholds**. *Fallback Model Invocation* — plan de continuidad de negocio para outages de la API del LLM. *Rate-Limited Invocation* — protege las APIs downstream y gestiona costos.
- **Human-Agent Interaction (cap. 8)**: *Agent Delegates to Agent* — la complejidad interna de la colaboración multi-agente ahora se abstrae del usuario. *Agent Calls Proxy Agent* — gestiona de forma segura todas las interacciones con sistemas externos de terceros.
- **Agent-Level Capabilities (cap. 9)**: *Advanced RAG* — el pipeline RAG se mejora con técnicas como re-ranking y query transformation para mejorar la calidad del retrieval.
- **System-Level Infrastructure (cap. 10)**: *Tool and Agent Registry* — se vuelve esencial para una arquitectura de microservicios, permitiendo a los agentes descubrirse dinámicamente. *Event-Driven Reactivity* — todo el sistema se re-arquitectura en torno a un message bus central (Kafka o Google Cloud Pub/Sub), habilitando comunicación asincrónica y escalable.

**Implementation focus** (el foco pasa de escribir lógica a *ingeniería de infraestructura*):
- **Containerization (Docker)** — cada agente se empaqueta en su propio container con sus dependencias específicas (librerías Python, drivers); aísla el "dependency hell" y asegura que un update al entorno de *Agent A* no rompa a *Agent B*.
- **Orchestration (Kubernetes)** — gestionar decenas de containers independientes requiere un orquestador; Kubernetes es la "mano invisible" que maneja el lifecycle: auto-healing (detectar el crash de *Agent B* y reiniciarlo) y horizontal scaling (levantar múltiples copias de *Agent C* si sube la carga).
- **Event infrastructure (message bus)** — implementar un message broker robusto (Apache Kafka, RabbitMQ, Google Cloud Pub/Sub); el foco está en definir topics y schemas claros para que los agentes publiquen/suscriban sin saber quién está del otro lado → verdadera Event-Driven Reactivity.
- **Infrastructure as Code (IaC) y CI/CD** — como el sistema ya no es un único script, el deploy manual es riesgoso y no escalable; definir la infra (registry, Redis, message bus) con herramientas como **Terraform** y establecer pipelines **CI/CD**, permitiendo actualizar/testear/desplegar agentes individuales independientemente, minimizando regresiones system-wide.

**Sistema resultante y consecuencias** — ahora es un **servicio de producción robusto, observable y seguro**: escala para manejar carga real, se recupera de fallos comunes y puede ser mantenido por un equipo de operaciones. La consecuencia de esa resiliencia es un **aumento significativo de complejidad arquitectónica**: gestionar un sistema distribuido de decenas de microservicios especializados requiere una **cultura DevOps y MLOps madura**. Pero estabilidad no es lo mismo que inteligencia: un sistema Level 2, con toda su robustez, sigue siendo **estático** — ejecuta la misma lógica mañana que hoy. Para desbloquear el potencial transformador real hay que pasar de la mera ejecución a la adaptación → Level 3.

## Level 3 – The Self-Improving Ecosystem (advanced maturity)

- **Objetivo arquitectónico** — evolucionar el production-ready service en un sistema **state-of-the-art, auto-mejorable** que desarrolla expertise profunda de dominio y optimiza su propia performance mediante feedback loops automatizados y análisis estratégico.
- **Principio core** — **self-optimization**. El sistema ya no es estático; es un ecosistema dinámico que aprende. Diseñado no solo para ejecutar sus tareas sino para **medir su propia performance, aprender de sus éxitos y fallos, y adaptar su comportamiento** para ser más efectivo y eficiente con el tiempo.

![[12-fig-12.3.png]]
*Figura 12.3 – Level 3 patterns*

Para lograr esa auto-mejora autónoma, la arquitectura se apoya en **tres pilares fundamentales**:
- **Closed-loop learning** — la arquitectura captura sus propios outputs (éxitos y fallos) y los usa como training data; integrando MLOps tuning pipelines directamente en el runtime, el sistema puede fine-tunear sus modelos (con patrones como *Coevolved Agent Training*) para adaptarse a distribuciones de data cambiantes sin intervención manual de ingeniería.
- **Emergent coordination** — en vez de depender solo de orquestación top-down, los agentes se empoderan con skills "sociales"; patrones como *Consensus & Negotiation* permiten al **Agent Swarm** resolver ambigüedad y conflictos de recursos autónomamente, reduciendo la carga del orquestador central y habilitando escenarios novedosos e indefinidos.
- **Safe evolution** — como el sistema cambia dinámicamente, la estabilidad se gestiona con testing automatizado riguroso; *Canary Testing* y *Trust Decay* aseguran que la lógica "mejorada" se valide contra métricas de performance real antes de confiar plenamente en ella, previniendo regresión a medida que el sistema evoluciona.

**Patrones a implementar** (los más avanzados, enfocados en aprendizaje, evaluación estratégica y colaboración compleja):
- **Coordination & Planning (cap. 5)**: *Consensus, Negotiation, and Conflict Resolution* — el sistema ahora tiene los skills "sociales" para manejar ambigüedad y desacuerdo autónomamente; los agentes debaten data conflictiva, negocian por recursos y resuelven planes en conflicto sin directiva top-down.
- **Explainability & Compliance (cap. 6)**: *Fractal CoT Embedding* — los agentes usan este patrón de razonamiento avanzado para self-correction recursiva, atrapando sus propias fallas lógicas y revisando sus planes según nueva evidencia.
- **Robustness & Fault Tolerance (cap. 7)**: *Majority Voting Across Agents* — para las decisiones más críticas, un panel de agentes logra confiabilidad extrema. *Trust Decay and Scoring* — el orquestador implementa un sistema de reputación para rutear tareas adaptativamente a los agentes más confiables. *Canary Agent Testing* — las versiones nuevas de agentes se despliegan seguras a producción sin arriesgar la estabilidad del sistema.
- **Agent-Level Capabilities (cap. 9)**: *Agentic RAG* y *Graph-Vector Hybrid Retrieval* — el sistema construye y mantiene su propio knowledge graph rico, combinándolo con vector search para expertise de dominio state-of-the-art.
- **Continuous Improvement & Tuning (caps. 3 y 11/14)**: *Hybrid Workflow Agent Architecture (Planner + Scorer)* — el sistema usa un pairing generator-evaluator para crear y vetar sus propios workflows. *Coevolved Agent Training* — los agentes planner y scorer se mejoran en tándem con una mezcla de SFT, DPO y aprendizaje iterativo. *Preference-Controlled Synthetic Data Generation* — para prevenir runaway divergence o collusion (donde los agentes refuerzan sesgos compartidos o derivan a satisfacer las manías del otro en vez del objetivo de negocio), este proceso requiere benchmarks estrictos de offline evaluation y validación periódica human-in-the-loop. *Custom Evaluation Metrics* — métricas domain-specific (como el **STEPScore**) para medir la calidad de workflows con precisión.

**Implementation focus** (el foco de ingeniería pasa de la lógica de aplicación a construir infraestructura sofisticada de MLOps y DataOps, cerrando el loop entre ejecución y training sin intervención manual):
- **Automated data synthesis** — el sistema no puede depender solo de data orgánica de usuarios (a menudo escasa o ruidosa); construir pipelines donde los planner agents generan escenarios y los scorer agents los etiquetan, creando un dataset limpio que alimenta el aprendizaje.
- **Continuous tuning pipelines** — el *MLOps Tuning Pipeline* es el motor; a diferencia del Level 2 (donde los modelos son assets estáticos), este nivel requiere infraestructura (Kubeflow o Vertex AI Pipelines) que ingiera data automáticamente, dispare jobs de fine-tuning (SFT) o DPO y produzca modelos actualizados.
- **Feedback loop integration** — el flujo de *Logs & Outputs* de vuelta al pipeline representa el desafío crítico de data engineering: transformar logs de ejecución crudos en training examples estructurados; requiere *Custom Evaluation Metrics* (como el STEPScore) para gradear performance programáticamente y señalar al pipeline qué reforzar.
- **Safe model promotion** — el flujo de *Updated Models* al *Agent Swarm* implica una estrategia de deployment robusta; requiere infraestructura de *Canary Testing* para evaluar automáticamente el nuevo modelo contra un grupo de control antes de dejarlo tomar el tráfico completo de producción.

**Sistema resultante y consecuencias** — un sistema state-of-the-art que actúa como un **experto dinámico y en evolución** para su dominio: no solo da respuestas precisas sino que mejora sus propias capacidades, endurece sus defensas y demuestra su valor estratégico vía métricas alineadas al negocio. El costo de esta sofisticación es **complejidad extrema** y una inversión significativa y continua en infraestructura y expertise para mantener los loops automatizados de creación de conocimiento y training.

## Guía de reflexión estratégica (tu agentic roadmap)

El modelo da el *qué* (un catálogo de posibilidades y patrones); el paso más importante es traducir el mapa a un plan concreto para tu proyecto — el *cómo* y el *cuándo*. No es un test sino una conversación estructurada con tu equipo:
- **The journey begins: ¿dónde estás hoy?** — primero determinar la ubicación de partida con una evaluación honesta del estado actual y los goals inmediatos. Preguntar: *¿es un PoC nuevo y exploratorio, o intentamos escalar una automatización existente, quizás frágil?* El objetivo de negocio del corto plazo es la **brújula**: validar que un workflow agéntico resuelve un problema → Level 1; construir un servicio resiliente y escalable para carga real → Level 2; crear un experto auto-mejorable que sea ventaja competitiva durable → Level 3. Considerar también la **expertise actual del equipo** (¿fluidos en sistemas distribuidos y MLOps, o es territorio nuevo?) — la honestidad previene sobre-arquitecturar un sistema que no podés mantener.
- **Laying the foundation: ¿cuál es tu Minimum Viable Agent (MVA)?** — sin importar el target último, el ciclo de desarrollo de un workflow agéntico nuevo debe **empezar estableciendo un Level 1 foundational como paso de validación** (la fase **MVA — minimum viable agent**). Incluso equipos sofisticados usan esta fase para aislar y perfeccionar la lógica de razonamiento, las interacciones con tools y los safety guardrails en un entorno monolítico controlado antes de agregar la complejidad distribuida. Resistir el impulso de sobre-ingenierizar: *¿cuál es la versión más simple posible que prueba que nuestro core business logic funciona?* ¿Puede empezar gestionado por una única *Supervisor Architecture*? Si el agente llama a una API externa, un *Watchdog Timeout* no es lujo sino necesidad; si el sistema puede atascarse en ambigüedad, un *Agent Calls Human* es no-negociable.
- **Building for scale: ¿cuál es tu camino a producción?** — un prototipo Level 1 exitoso inevitablemente crea demanda de más (usuarios, features, agentes), cuando se vuelven aparentes los límites del monolito. Empezar identificando los **triggers específicos** que necesitarán la evolución (¿cierto nº de usuarios diarios? ¿necesidad de >3-4 agentes especializados? ¿requerimiento de procesamiento asincrónico long-running?) — definirlos por adelantado convierte la decisión de escalar de un pánico reactivo en un evento planeado. Luego priorizar qué patrones Level 2 son más críticos: si uptime es la métrica clave → *Auto-Healing Agent Resuscitation* y *Fallback Model Invocation*; si la responsiveness real-time es clave → arquitectura *Event-Driven Reactivity* con message bus central; al sumar agentes especializados, el discovery se vuelve primordial → planear un *Tool and Agent Registry*.
- **Reaching for autonomy: ¿cuál es tu North Star?** — no todo sistema necesita el zenit de la madurez, pero entender lo posible informa la estrategia de largo plazo y previene decisiones arquitectónicas tempranas que bloqueen innovación futura. La pregunta grande: *¿nuestro problema de negocio core requiere que el sistema aprenda y se adapte con el tiempo para ser realmente efectivo?* Si sí, el camino al Level 3 es una **necesidad estratégica** (¿aprendería de feedback implícito del usuario para personalizar? ¿haría A/B testing automático de workflows? → *Trust Decay and Scoring*, *Coevolved Agent Training*). Considerar **las stakes**: ¿las decisiones individuales son tan críticas que un solo error tendría consecuencias mayores? Entonces el alto costo de *Consensus* y *Majority Voting* no solo se justifica: es un requisito para un sistema trustworthy.

## Tabla 12.1 – Resumen estratégico

| Nivel de madurez | Foco estratégico | Decisiones arquitectónicas clave | Patrones críticos a considerar | Pregunta clave para tu equipo |
|---|---|---|---|---|
| **Level 1 – Foundational** | Validar el concepto | Monolítico vs microservicios; Síncrono vs asincrónico | Supervisor Architecture · Watchdog Timeout · Agent Calls Human · Basic RAG | *"¿Cuál es la versión más simple de esto que funciona de forma segura y prueba el valor?"* |
| **Level 2 – Production-ready** | Lograr estabilidad y escala | Centralizado vs descentralizado; Servicios stateful vs stateless | Event-Driven Reactivity · Auto-Healing · Tool and Agent Registry · Causal Dependency Graph | *"¿Cómo manejará este sistema 10× la carga y se recuperará de fallos sin intervención humana?"* |
| **Level 3 – Self-improving** | Habilitar aprendizaje y adaptación | Workflows estáticos vs dinámicos; Comportamiento rule-based vs learned | Consensus and Negotiation · Fractal CoT · Trust Decay and Scoring · Coevolved Agent Training | *"¿Gana nuestro negocio una ventaja competitiva si este sistema aprende y se adapta por su cuenta?"* |

## Citas

> "Where do I start?"
> "the key to successfully deploying agentic AI is *progressive adoption*"
> "stability is not the same as intelligence."
> "Not every agentic system needs to evolve to the highest levels of maturity."
> "Adopt progressively, align architecture with goals, strategy precedes implementation."

## Para aplicar

- **Adoptar progresivamente** — empezar con la arquitectura más simple que entrega valor y *layerear* complejidad intencionalmente a medida que evolucionan las necesidades; no implementar todos los patrones a la vez.
- **Alinear arquitectura con goals** — la elección de patrones debe reflejar directamente los objetivos estratégicos: validar un concepto (L1), lograr estabilidad de producción (L2) o crear un activo auto-mejorable (L3).
- **La estrategia precede a la implementación** — antes de escribir una línea de código, recorrer la guía de reflexión (¿dónde estás? ¿MVA? ¿camino a producción? ¿North Star?) para definir scope, arquitectura y visión.
- **Siempre empezar por el MVA (Level 1)** sin importar el target — aislar y perfeccionar la lógica de razonamiento, tools y guardrails en un monolito controlado antes de distribuir; *Watchdog Timeout* y *Agent Calls Human* son no-negociables desde el inicio.
- **Definir los triggers de escala por adelantado** (nº de usuarios, >3-4 agentes, procesamiento asincrónico) para que el salto a L2 sea un evento planeado, no un pánico reactivo; migrar a microservicios solo cuando el monolito ya no alcanza.
- **Reservar el Level 3 para cuando el negocio realmente necesita que el sistema aprenda/adapte** o cuando las stakes de cada decisión justifican el costo de Consensus/Majority Voting; vigilar runaway divergence/collusion con offline benchmarks + human-in-the-loop.

## Conexiones

- [[_Agentic Architectural Patterns for Building Multi-Agent Systems|Mapa del libro]] — el MOC del libro; **este capítulo ES la síntesis ejecutable de toda la Parte 2**.
- [[11 - Advanced Adaptation - Building Agents That Learn]] — cap. 11 (anterior): los patrones de aprendizaje (Planner+Scorer, Coevolved Training, Synthetic Data, Custom Metrics/STEPScore) que pueblan el Level 3.
- [[03 - The Spectrum of LLM Adaptation for Agents - RAG to Fine-tuning]] — cap. 3: el Agentic AI Maturity Model de 6 niveles que este capítulo **simplifica a 3** (mapeando 2-en-1) para el rollout enterprise.
- [[05 - Multi-Agent Coordination Patterns]] — cap. 5: Supervisor Architecture (L1), Hybrid Delegation/Knowledge Sharing (L2), Consensus/Negotiation/Conflict Resolution (L3).
- [[06 - Explainability and Compliance Agentic Patterns]] — cap. 6: Basic Audit Logging (L1), Instruction Fidelity Auditing/Persistent Anchoring (L2), Fractal CoT (L3).
- [[07 - Robustness and Fault Tolerance Patterns]] — cap. 7: Watchdog/Simple Retry (L1), Adaptive Retry/Auto-Healing/Checkpointing/Fallback/Rate-Limited (L2), Majority Voting/Trust Decay/Canary (L3); el Causal Dependency Graph que sube de L1 a L2.
- [[08 - Human-Agent Interaction Patterns]] — cap. 8: Human Calls Agent/Agent Calls Human (L1), Agent Delegates to Agent/Agent Calls Proxy Agent (L2).
- [[09 - Agent-Level Patterns]] — cap. 9: Single Agent Baseline/Memory/Simple RAG (L1), Advanced RAG (L2), Agentic RAG/Graph-Vector Hybrid (L3).
- [[10 - System-Level Patterns for Production Readiness]] — cap. 10: Agent Auth (L1), Tool and Agent Registry/Event-Driven Reactivity (L2).
- [[Orchestrator]] — la Supervisor Architecture que ancla el L1 y persiste como orquestador en L2/L3.
- [[Generator-Evaluator Pattern]] — el Hybrid (Planner + Scorer) del L3.
- [[Circuit Breaker]] · [[Retry with Backoff]] · [[Health Check]] · [[Canary Deployment]] · [[Message Queue]] · [[Pub-Sub]] · [[Service Discovery]] — patrones de System Design del vault que el roadmap orquesta por nivel (retry/auto-healing/fallback en L2, canary en L3, message bus en L2, registry/service discovery en L2).
- [[Docker]] · [[Kubernetes]] · [[Terraform]] · [[CI-CD]] · [[_MLOps|MLOps]] — el stack de infraestructura del L2 (containers, orquestación, IaC, pipelines) y el MLOps/DataOps del L3 (Kubeflow, Vertex AI Pipelines).
- **Progressive adoption** · **Minimum Viable Agent (MVA)** · **Kubeflow / Vertex AI Pipelines** · **STEPScore** · **Trust Decay and Scoring** · **Coevolved Agent Training** — conceptos del capítulo; candidatos a nota propia.
