---
title: "15 - Agent Frameworks - Loan Processing with CrewAI and LangGraph"
libro: Agentic Architectural Patterns for Building Multi-Agent Systems
autor: Ali Arsanjani
capitulo: 15
created: 2026-06-11
tags:
  - libros/agentic-architectural-patterns
  - type/case-study
  - status/permanent
aliases:
  - Agent Frameworks - Loan Processing with CrewAI and LangGraph
  - Agent Frameworks - Use Case CrewAI and LangGraph
---

# 15 - Agent Frameworks - Loan Processing with CrewAI and LangGraph

> [!info] Capítulo 15 · *Agentic Architectural Patterns for Building Multi-Agent Systems* — Ali Arsanjani
> Comparación práctica de **tres frameworks agénticos** —Google **ADK**, **CrewAI**, **LangGraph**— reimplementando el use case de loan processing (cap. 13/14) en cada uno. Sus filosofías core: ADK = *production-ready agents*, CrewAI = *role-playing team/crew*, LangGraph = *stateful graph/state machine*. Cierra con observability + responsible AI, recomendaciones de elección (evitar el lock-in) y el mapeo a la madurez. Navegá: [[_Agentic Architectural Patterns for Building Multi-Agent Systems|Mapa del libro]] · anterior [[14 - Use Case - A Multi-Agent System for Loan Processing]].

## Resumen

Traducir los constructos teóricos del libro a aplicaciones production-ready requiere herramientas y librerías que manejen la complejidad subyacente de creación, ejecución y comunicación de agentes — ahí entran los **agent frameworks**. Construir un sistema agéntico de cero implica gestionar numerosas piezas (definir roles/capacidades, manejar state y memory, orquestar workflows, integrar LLMs, gestionar tool use/function calling, facilitar comunicación entre agentes); los frameworks dan abstracciones y componentes pre-construidos que simplifican esto, dejando al developer enfocarse en la lógica de aplicación en vez de reinventar el "plumbing" fundacional. El capítulo presenta tres ejemplos prominentes —**Google ADK**, **CrewAI**, **LangGraph**— *ilustrativos, no exhaustivos*: los design patterns del libro son universales y generalizan a otros frameworks. Para hacer la comparación concreta, reimplementa el use case de loan processing de los caps. 13/14 en **CrewAI** y **LangGraph** (ADK ya se usó extensamente, no se repite).

El recorrido: introduce cada framework y su filosofía core (ADK = production-ready agents como componentes modulares/testeables/desplegables; CrewAI = *collaborative intelligence* vía role-playing agents en un "crew"; LangGraph = workflows agénticos como *state machines* de nodos y edges); compara similarities (los 3 usan un LLM como reasoning engine, el patrón Function Calling/Tool Use, y son goal-oriented multi-step) y key differences (su core abstraction: team vs state machine vs production agent, y de ahí derivan control flow, state management, callbacks); reimplementa el loan agent con las mismas 4 tools de negocio (`BaseTool`) más un helper `get_document_content`, mostrando CrewAI con un *hierarchical process* (un manager que delega a especialistas con role/goal/backstory) y LangGraph con un *explicit state machine* (`LoanGraphState` TypedDict, nodos por paso, conditional edges para fail-fast routing al `compile_rejection`); discute observability (LangSmith para LangGraph/CrewAI, OpenTelemetry para ADK) y responsible AI (transparency/explainability, safety/robustness, accountability/governance); da recomendaciones de cuándo usar cada uno (evitando el *framework lock-in* diseñando alrededor de interfaces estables: tool contracts, state schemas, pattern-based orchestration); y mapea los frameworks a los niveles de madurez (Tabla 15.3). La lección: *no hay un "best" framework único*; el framework es una *herramienta para implementar los patrones* — entender sus core abstractions (el *team* de CrewAI, el *state machine* de LangGraph, el *production service* de ADK) permite elegir el que mejor calce con el problema.

## Los tres frameworks

### Google's Agent Development Kit (ADK)
Provee un entorno estructurado para construir y desplegar agentes, diseñado con *production readiness* en mente. Apunta a dar el scaffolding fundacional para agentes que razonan, planifican e interactúan confiablemente con sistemas externos y otros agentes. Features clave:
- **Agent abstraction** — clase/estructura base para definir la lógica core del agente (instrucciones, tools, cómo maneja tasks/mensajes).
- **Tool integration** — mecanismos para definir/registrar tools (funciones o APIs) que los agentes usan para interactuar con el mundo.
- **Planning and reasoning** — integración con LLMs (Gemini) para el reasoning loop; incluye built-in planners o lógica de planning custom.
- **State management** — mecanismos para mantener state y memory a través de interacciones.
- **Communication protocol** — soporte nativo del protocolo **A2A (Agent-to-Agent)**; usa sus formatos de mensaje estandarizados y task life cycles para comunicar/colaborar entre frameworks y fronteras enterprise.
- **Runtime environment** — un agent runtime/agent engine que gestiona deployment, task distribution, ejecución paralela, retries, e integración con observability.

### CrewAI
Enfoca explícitamente la **collaborative intelligence** vía *role-playing agents* que trabajan juntos como un "crew" cohesivo. Filosofía core: las tareas complejas se descomponen y asignan a agentes con roles, responsabilidades y hasta "backstories" específicas que guían su comportamiento y expertise; estos agentes colaboran compartiendo info y resultados intermedios. Conceptos clave:
- **Agents** — definidos con un *role*, *goal*, *backstory*, el LLM que usan y tools específicas; el aspecto role-playing ayuda al LLM a encarnar una persona/expertise particular.
- **Tasks** — asignaciones específicas dadas a agentes; cada una con descripción y expected output, asignada a un agente; se pueden encadenar (output de una → input de otra).
- **Tools** — funciones o capacidades; en CrewAI a menudo heredan de una clase `BaseTool`.
- **Crew** — la colección de agentes y las tasks que deben realizar; define cómo colaboran.
- **Process** — el workflow/metodología que sigue el crew: **sequential** (tasks una tras otra) o **hierarchical** (un manager delega tasks).

Enfatiza el *aspecto social* de la interacción, intuitivo para sistemas que imitan equipos humanos. *Consideración enterprise*: este estilo "persona-driven" puede introducir **variabilidad** en el output del modelo; aplicado a workflows regulados/high-stakes (como loan adjudication), es crítico contrabalancear esa flexibilidad con **strict tool contracts y testing riguroso** para asegurar outcomes determinísticos y compliant.

### LangGraph
Extiende la librería LangChain, dando una forma robusta de construir aplicaciones stateful y multi-actor (incl. sistemas agénticos complejos) usando **grafos**. Mientras LangChain enfoca el chaining de llamadas (secuencias lineales), LangGraph permite **ciclos** — apto para modelar comportamientos de agentes más flexiblemente (loop, retry, decidir dinámicamente el próximo paso según el estado). Representa los workflows agénticos como **state machines**: cada paso es un **node** del grafo, las transiciones son **edges**. Conceptos clave:
- **StateGraph** — el objeto core que representa el grafo del workflow; contiene el estado de la aplicación.
- **State** — una estructura de datos definida (a menudo una clase Python o dict, como `TypedDict`) que contiene toda la info relevante al progreso (user input, resultados intermedios, mensajes de agentes).
- **Nodes** — funciones u objetos runnable que representan pasos o actores (agentes); cada node recibe el estado actual, ejecuta una acción (llamar un LLM, usar una tool, procesar data) y devuelve updates al estado.
- **Edges** — definen las transiciones entre nodos; determinan el próximo node según el estado actual o el output del previo.
- **Conditional edges** — permiten *branching logic*: según el estado/output, el grafo rutea a diferentes nodos subsiguientes, habilitando decision-making complejo y loops.

Especialmente apto cuando: el explicit state management es crucial; se necesitan procesos cíclicos (un agente reflexionando sobre su output y reintentando, o interacciones **HITL/Human-in-the-Loop**); se requiere control flow complejo con branching y dynamic routing; o modelar interacciones entre múltiples agentes/actores (incluidos humanos).

## Similarities y key differences

**Similarities** (fundación conceptual común): (1) **LLM as the reasoning engine** — los tres usan un LLM (Gemini, GPT-4, open source) como el "cerebro" responsable de razonar, planificar y decidir según prompt, estado y tools; (2) **Tool integration** — todos construidos alrededor del patrón **Function Calling / Tool Use** (la capacidad del agente de actuar sobre el mundo: buscar una DB, leer un archivo, llamar una API); (3) **Goal-oriented** — no son para Q&A simple single-shot, sino para apps que ejecutan tareas complejas multi-step hacia un goal definido por el developer.

**Key differences** (la diferencia primaria está en su *core abstraction* — el modelo mental para representar un workflow agéntico, que influye todo desde control flow a state management):
- **Core philosophy / abstraction**: **CrewAI** = *role-based collaboration* (un team de especialistas definidos por role/goal/backstory; la colaboración es la feature central; intuitivo para workflows que imitan equipos humanos). **LangGraph** = *stateful graphs* (un flowchart/state machine de nodes y edges; el foco pasa de los *agentes* al *proceso*; su poder está en hacer el estado explícito y el control flow determinista). **Google ADK** = *production-ready agents* (el agente como componente de software modular, testeable y desplegable; enfoque code-first familiar a los software engineers; foco en el lifecycle del agente; con dos mecanismos para robustez enterprise: **Callbacks** —middleware para filtering activo, PII detection, HITL control— y **Workflow agents** —scaffolding para tasks sequential/looping/parallel junto al razonamiento autónomo—).
- **Control flow / cyclical behavior**: **CrewAI** gestiona el control flow a alto nivel (proceso *sequential* o *hierarchical*); simple y efectivo para tareas lineales/delegadas. **LangGraph** da control fino y completo (al ser un grafo, fácil crear ciclos, branches, loops; conditional edges tipo "si validación falló → nodo Rejection; si no → nodo Credit Check"); clave para muchos patrones avanzados. **ADK** balancea ambos (puede correr workflows determinísticos como `SequentialAgent` y también planning dinámico LLM-driven donde el agente decide los próximos pasos, gestionados por el runtime).
- **State management**: el *superpoder* de **LangGraph** es su explicit state management (un objeto `State`/`TypedDict` con toda la info, pasado a *cada* node; cada node trabaja y devuelve updates → debugging mucho más fácil, inspeccionable en cada paso). **CrewAI** es más implícito (el output de una task se formatea y pasa automáticamente de contexto a la siguiente dependiente; rápido para chains simples pero con menos control/inspección directa). **ADK** usa managed state (el runtime y session service persisten el state/memory entre interacciones, abstrayendo la complejidad y permitiendo que agentes long-running retomen donde quedaron).

### Tabla 15.1 – Comparación de frameworks (ecosistema y filosofía)

| Framework | Primary supporter | Announced/launched | Core philosophy y focus |
|---|---|---|---|
| Google ADK | Google | 2024 (prototype) / 2025 (public) | Toolkit open source comprehensivo para construir, evaluar y desplegar agentes production-grade robustos. Optimizado para el ecosistema Google (Gemini, Vertex AI) pero diseñado model-agnostic. |
| CrewAI | CrewAI (fundado por João Moura) | 2023 (open source launch) | Framework para orquestar role-playing autonomous agents. Enfatiza la collaborative intelligence, donde los agentes trabajan juntos como un "crew" para lograr goals. |
| LangGraph | LangChain | 2024 | Extensión de LangChain para apps stateful, multi-actor. Excele en crear apps con procesos cíclicos y control flows complejos modelándolos como grafos (state machines). |

### Tabla 15.2 – Comparación técnica ADK / CrewAI / LangGraph

| Feature | Google ADK | CrewAI | LangGraph |
|---|---|---|---|
| **Core abstraction** | Production-grade agents y runtimes. | Role-playing teams (un "crew"). | Stateful graph (un "state machine"). |
| **Control flow** | Planner-driven; gestionado por un agent runtime. Sequential o parallel. | Proceso high-level (sequential o hierarchical). | Fine-grained; definido por graph edges. Excelente para ciclos y branching. |
| **State management** | Managed: por la session y el runtime del agente. | Implicit: pasado entre tasks automáticamente vía context. | Explicit: un objeto `State` central pasado a y actualizado por cada node. |
| **Callbacks y hooks** | Middleware/Interceptor pattern. Callbacks como "guardrails" que interceptan input/output antes/después de las LLM calls. Clave: modificar data in flight (PII redaction) o bypassear el LLM (caching). | Event-driven hooks. Callbacks disparados en eventos del lifecycle (`on_task_start`, `on_task_end`). Clave: observabilidad y side effects (logging, UI, webhook) sin alterar la lógica core. | State listeners e interrupts. Callbacks para tracing (LangSmith) pero se apoya en "interrupts" para control. Clave: pausar el grafo en un node (checkpointing) para esperar input HITL antes de reanudar. |
| **Best for...** | Sistemas de producción, integración enterprise (especialmente Google Cloud), agentes robustos y testeables. | Prototipado rápido de tareas colaborativas, workflows role-defined ("researcher", "writer"). | Workflows complejos y dinámicos; manejo explícito de errores; loops; y HITL. |

## Reimplementando el loan agent: comparación práctica

Goal: tomar `applicant_id` y `document_id`, traer el contenido del documento y producir una decisión de préstamo final y auditable. El workflow tiene 5 tasks mapeadas a su componente en cada implementación:

| Task | LangGraph | CrewAI |
|---|---|---|
| 1. Document fetch | `node_fetch_document` | Pasado como input inicial/preprocessing |
| 2. Document validation | `node_validate_document` | Document validation specialist |
| 3. Credit check | `node_check_credit` | Credit check agent |
| 4. Risk assessment | `node_assess_risk` | Risk assessment analyst |
| 5. Compliance check | `node_check_compliance` | Compliance officer |

**Tools de negocio compartidas** (clases que heredan de `BaseTool` de CrewAI), todas devolviendo **JSON-encoded strings**:
- `ValidateDocumentFieldsTool` — valida que el JSON tenga los campos requeridos (`customer_id`, `loan_amount`, `income`, `credit_history`); devuelve `{status: validated, data: ...}` o `{error: ...}`.
- `QueryCreditBureauAPITool` — simula una API de credit bureau; mock scores: **CUST-12345 → 810** (happy path), **CUST-55555 → 620** (high risk), borrower_good_780 → 810, borrower_bad_620 → 620.
- `CalculateRiskScoreTool` — risk basado en `loan_amount`, `income`, `credit_score`: parsea el income (×12 si es mensual), `loan_to_income_ratio`; risk_score base 1, **+4 si credit<650 / +2 si <720**, **+5 si ratio>0.8 / +2 si >0.5**, capeado a 10 (10 si income inválido/cero).
- `CheckLendingComplianceTool` — usa `credit_history` y `risk_score`: "No History" → denial automático; **risk_score ≥ 8 → non-compliant**; si no → compliant.
- Helper `get_document_content(document_id)` — `document_valid_123` → CUST-12345/loan 50000/income "USD 120000 a year"/7 years; `document_invalid_456` → CUST-55555/loan 200000/**income faltante**/1 year.

> [!note] **Tool output patterns** — las tools devuelven JSON-encoded *strings*, no dicts Python, deliberadamente: como el consumidor primario del output suele ser el *LLM mismo* (que procesa text tokens), un JSON string explícito le da un formato estructurado y legible fácil de parsear/razonar. Además establece un **data contract**: la tool garantiza devolver o `{"status": "validated", "data": ...}` (éxito) o `{"error": ...}` (fallo), permitiendo a los agentes downstream manejar errores determinísticamente.

> [!tip] **Data normalization patterns** — el `CalculateRiskScoreTool` hace string parsing básico del income; en producción es demasiado frágil. Implementar un *normalization node / preprocessing tool* upstream que maneje conversión de moneda, formato de locale (ej. "$100k" vs "100,000 EUR") y estandarización *antes* de que la data llegue a los agentes de risk assessment.

> [!note] **Production pattern – the proxy tool (Agent Calls Proxy Agent)** — la lógica `if/else` es una heurística simplificada para demostración; en una arquitectura enterprise real, esta tool funcionaría como un *proxy* (patrón **Agent Calls Proxy Agent** del cap. 8): el método `_run` actuaría de wrapper que construye un API request seguro a un *external risk decision engine* o un endpoint de modelo ML desplegado, ejecuta la llamada y parsea la respuesta — manteniendo al agente liviano y la lógica de negocio crítica centralizada y gobernable.

### Implementation 1: CrewAI (the collaborative team)

Usa un **hierarchical process** donde un `manager` agent delega tasks a agentes especializados:
1. **Definir el LLM y agentes** — el LLM (Gemini) vía la abstracción `LLM` de CrewAI con **`temperature=0.0`**. Los 5 agentes: `doc_specialist` (role "Document Validation Specialist", su única tarea: llamar la tool y devolver su output exacto, `allow_delegation=False`), `credit_analyst`, `risk_assessor`, `compliance_officer` (cada uno con role/goal/backstory enfocado + su única tool, sin delegación), y el `manager` ("Loan Processing Manager", *sin tools* pero `allow_delegation=True` — orquesta y compila el reporte final).
2. **Definir las tasks** — 5 tasks (`task_validate` con placeholder `{document_content}`, `task_credit`, `task_risk`, `task_compliance`, `task_report`); el parámetro **`context`** pasa implícitamente los outputs entre tasks dependientes (ej. `task_risk` depende de `[task_validate, task_credit]`).
3. **Ensamblar y correr el crew** — `Crew(agents=[...], tasks=[...], process=Process.hierarchical, manager_agent=manager)`; se hace fetch del documento *antes* del kickoff y se pasa como input. Se corre con inputs válidos e inválidos.

> [!tip] **Minimizing variance with temperature** — se setea `temperature=0.0` deliberadamente: en workflows que dependen de tool usage preciso y outputs estructurados (JSON), minimizar la aleatoriedad es crucial. Nota: `0.0` reduce significativamente la varianza pero *no garantiza* comportamiento 100% determinístico (por la no-determinación inherente de las operaciones floating-point en GPUs); sí provee la máxima estabilidad posible para tareas de lógica y orquestación.

**Comportamiento**: el enfoque hierarchical deja al manager orquestar, delegando cada task al especialista apropiado según la descripción y tools disponibles. El **error handling es algo implícito**: si `task_validate` devuelve un error (ej. el campo `income` faltante en el caso inválido), las tasks subsiguientes que dependen de su output *igual pueden correr* pero probablemente fallen o produzcan resultados incorrectos, mientras el manager intenta proceder. El reporte final del caso inválido refleja el fallo de validación, pero los pasos intermedios (credit check, risk assessment) *se ejecutan igual*, potencialmente realizando acciones innecesarias.

### Implementation 2: LangGraph (the state machine)

Usa un **explicit state machine** con nodos por paso y conditional edges para error handling robusto:
1. **Definir el state** — `LoanGraphState(TypedDict)` con todos los campos (`applicant_id`, `document_id`, `document_content`, `validation_status`, `customer_id`, `loan_amount`, `income`, `credit_history`, `credit_score`, `risk_score`, `risk_level`, `compliance_status`, `final_decision`, y un campo **`error`** explícito para trackear errores).
2. **Definir los graph nodes** — funciones Python como nodos: `node_fetch_document` (simula el fetch, setea `error` si falla), `node_validate_document` (llama la tool, actualiza el state con la data extraída *o* un error; los nodos siguientes chequean `error` antes de proceder), `node_check_credit`, `node_assess_risk` (**LLM-powered**: usa el LLM directamente vía un `ChatPromptTemplate` para generar el risk assessment, parseando LOW/MEDIUM/HIGH → score_map {LOW:3, MEDIUM:6, HIGH:9, UNKNOWN:10}), `node_check_compliance` (setea `error` si non-compliant), `node_compile_report` (success path, sólo alcanzado si todos los pasos pasaron) y `node_compile_rejection` (failure path, con razones específicas según qué etapa falló).
3. **Definir el grafo y sus edges** — `StateGraph(LoanGraphState)`, entry point `fetch_doc`, y **conditional edges** tras cada nodo (`decide_after_fetch/validation/credit_check/risk/compliance`) que chequean el campo `error` y rutean a `compile_rejection` inmediatamente si hay error, o `continue` al siguiente; `compile_report` y `compile_rejection` → `END`.
4. **Correr el grafo** — `app.stream()` para observar las transiciones de estado, con inputs válidos (`document_valid_123`) e inválidos (`document_invalid_456`).

> [!tip] **Enforcing structured output** — el ejemplo usa string matching simple para parsear la respuesta del LLM (riesgoso: el modelo puede ser verboso, ej. "The risk is relatively LOW"). En producción, usar **structured output features** (soportadas por LangChain y Gemini): pasando un schema **Pydantic** al modelo se lo fuerza a devolver un JSON válido (ej. `{"risk_level": "LOW"}`), garantizando que el output matchee los requisitos downstream sin parsing frágil.

**Comportamiento**: demuestra control flow y state management explícitos. El `node_fetch_document` hace que el proceso arranque limpio. Las conditional edges basadas en `error` aseguran que si el fetch o la validación fallan, el grafo rutea inmediatamente al `compile_rejection`, **previniendo tool calls innecesarias** (credit check, risk) sobre data inválida. Este routing explícito da una **ganancia de eficiencia tangible**: elimina costos de API y latencia innecesarios al detener la ejecución inmediatamente ante un fallo — *contraste directo con el ejemplo CrewAI, donde los agentes intermedios seguían operando pese al error inicial de validación*. El uso de un LLM directamente en `node_assess_risk` muestra cómo LangGraph integra pasos generativos junto a tool calls determinísticas. Este enfoque graph-based da superior robustez y traceability que el proceso más simple de CrewAI para este workflow específico, especialmente en error handling.

## Observability y responsible AI

Elegir un framework no es solo sobre la dev experience; es sobre tu capacidad de gestionar, monitorear y gobernar la app resultante. En IA agéntica, donde la no-determinación es un factor, **la observabilidad es piedra angular del responsible AI**: si no podés trazar *por qué* un agente tomó una decisión, no podés asegurar que sea justa, segura o compliant.

**Observability en práctica**:
- **LangGraph y CrewAI (con LangSmith)** — el ecosistema LangChain integra nativamente con **LangSmith** (plataforma de observability para trazar apps LLM complejas). Como el state de LangGraph es explícito, sus traces en LangSmith son muy detallados, permitiendo *"time-travel" debug* viendo el full state y las LLM calls en cada node. Los traces de CrewAI también se benefician de LangSmith (muestra las acciones y tool calls del agente) → audit trail completo de thoughts y actions.
- **Google ADK** — instrumentado con **OpenTelemetry** (el estándar de la industria para tracing y metrics), integrando directamente con soluciones de monitoring enterprise como **Google Cloud's operations suite** (Cloud Trace, Cloud Logging) → trata al agente menos como un script y más como un microservicio gestionable.

**Enabling responsible AI** (los principios —fairness, transparency, accountability, safety— se habilitan con elecciones arquitectónicas concretas + governance sostenido):
- **Transparency y explainability** — el explicit state graph de LangGraph *es* una forma de explainability (el grafo documenta la lógica de decisión, el final state object contiene toda la data intermedia). Escenarios: *demographic parity testing* (fairness eval en el CI/CD con el testing framework de ADK contra un "golden dataset" de queries de demografías diversas, midiendo si la calidad de respuesta se mantiene consistente entre grupos antes de promover una versión); *reasoning transparency* (la vista **Trace** de Google Cloud expone el loop "thought, action, observation", mostrando por qué el agente eligió `getUserBalance` en vez de `getLoanStatus`); *explicit state graph visualization* (correr contra un golden dataset de perfiles de aplicantes diversos para asegurar que la lógica de aprobación no exhiba *disparate impact* por atributos protegidos —ZIP code, gender— aun si no se usan como features).
- **Safety y robustness** — las conditional edges de LangGraph actúan de *programmatic safety pattern* imponiendo lógica **"fail-fast"**: en vez de dejar al LLM seguir razonando sobre data incompleta/corrupta, el grafo monitorea el `error` key y reroutea a un terminal rejection node, previniendo que agentes downstream procesen inputs inválidos (reduce el riesgo de alucinar una decisión o hacer API calls no autorizadas). Escenarios: *safety guardrails y PII protection* (configurar `safety_settings` en ADK para bloquear hate speech/harassment en `BLOCK_LOW_AND_ABOVE`; input guardrails que interceptan/redactan PII antes del context window); *conditional edge guardrails* (hardcodear un `stop` que halt-ea si la income verification API devuelve null/negativo, previniendo alucinar una credit decision con data mala).
- **Accountability y governance** — un workflow trazable/observable es prerequisito de accountability: cuando un auditor pregunta por qué se denegó un préstamo, podés dar el trace completo de LangSmith o Cloud Trace (data, tool outputs, LLM reasoning en cada paso, especialmente con el explicit state de LangGraph) → transforma al agente de "black box" a componente transparente y auditable. Escenarios: *immutable audit trail* (data access logs + exportar logs de interacción a BigQuery → record inmutable donde cada API call tiene timestamp y service account identity, permitiendo queries de *cuándo* y *quién* autorizó una transacción); *full execution traces* (recuperar el trace ID de LangSmith con el prompt específico, el credit score recuperado y el paso de razonamiento intermedio del LLM que llevó al output `Denied`).

## Recomendaciones para elegir un framework

**No hay un "best" framework único**: depende de la complejidad del proyecto, la familiaridad del equipo y los requisitos de producción. Estrategia crítica de largo plazo: **evitar el framework lock-in** (el landscape es volátil, el líder de hoy puede deprecarse mañana). Diseñar el sistema alrededor de **interfaces estables** —tool definitions estandarizadas (contracts), explicit state schemas, pattern-based orchestration logic— en vez de acoplar cada componente a las clases propietarias de un framework específico; esa abstracción permite migrar/swapear frameworks con mínima fricción.
- **Considerá CrewAI cuando** — prototipás rápido y querés un multi-agente corriendo ya; tu workflow mapea naturalmente a un team colaborativo de especialistas ("researcher"/"writer"/"editor"); te beneficia una estructura hierarchical (manager/worker) donde la delegación es clave; el implicit state passing vía context te alcanza.
- **Considerá LangGraph cuando** — necesitás control flow complejo no-lineal (loops, branches, dynamic routing según estado); el explicit state management e inspección en cada paso es crítico para lógica/debugging; requerís error handling robusto con routing específico según fallos; necesitás debugging/traceability high-fidelity (ver el full state en cada paso); construís agentes long-running que necesitan control de estado preciso; querés implementar patrones HITL fácil agregando nodos que esperan input.
- **Considerá Google ADK cuando** — construís para un entorno enterprise de producción (especialmente en Google Cloud); necesitás un enfoque más estructurado y software-engineering-centric que trate a los agentes como componentes modulares/testeables/desplegables; la integración con observability enterprise estándar (OpenTelemetry) y governance es requisito primario; necesitás gestionar el full lifecycle del agente (dev/eval/deploy/monitoring); necesitás inspeccionar el payload hacia tools/agents/models e inspeccionar/actuar sobre los outputs; necesitás orquestar patrones multi-agente complejos (sequential pipelines, parallel fan-outs, iterative loops) con workflow agents especializados que imponen estructura determinística sobre el razonamiento no-determinístico de los LLMs.

En última instancia, el framework es una *herramienta para implementar los patrones* discutidos; entendiendo sus core abstractions elegís el mejor para tu problema: el **team** de CrewAI, el **state machine** de LangGraph, el **production service** de ADK.

## Frameworks como enablers de la madurez (Tabla 15.3)

Mapeo de los frameworks al GenAI Maturity Model (cap. 1): son los enablers que ayudan a una organización a progresar de la generación básica data-enhanced (Nivel 2) a sistemas agénticos autónomos y colaborativos (Niveles 4-5/6).

| Nivel de madurez | Descripción | Framework approach / enabling tools |
|---|---|---|
| **Level 1 – Prompting** | Prompting simple single-turn | LLM API calls directas (Gemini, OpenAI). Frameworks generalmente no requeridos. |
| **Level 2 – RAG** | Context-enhanced generation (RAG) | LangChain (para RAG pipelines) o código custom que llama una vector DB e inserta contexto. |
| **Level 3 – Tuning** | — | N/A al framework agéntico. |
| **Level 4 – Grounding & evaluation** | — | **CrewAI** (*integrated/enterprise-focused*, "turnkey"): Hallucination Guardrail enterprise (faithfulness score 0-10 + self-correction bajo threshold), Built-in RAG/Knowledge (PDFs/CSVs), native utility tools (`TimeAwarenessTool` contra temporal hallucinations); evaluación: CrewAI Test CLI (corre N iteraciones → tablas de score), Patronus AI + `crew.train()` (fine-tuning human-feedback). **LangGraph** (*architectural/developer-driven*, control fino del "reasoning path"): Self-correction loops (conditional edges que detectan outputs pobres y rutean a un nodo "Refinement"), State checkpoints (persistencia nativa → rollbacks a estados buenos conocidos), HITL (interrupts explícitos que pausan antes de tool calls high-stakes); evaluación: LangSmith integration (trace-level, cada transición medida por latency/cost/accuracy con "LLM-as-a-judge"), unit-testable nodes (nodos = funciones Python aisladas → unit testing determinístico). |
| **Level 5 – Single-agent systems** | Un agente autónomo con planner, tools y memory ejecuta una tarea multi-step | **LangGraph**: un grafo con uno o más agent nodes que llaman múltiples tools según un explicit state, potencialmente looping (reflecting). **ADK**: el caso de uso primario (definir un `Agent` con sus tools y correrlo en el runtime). **CrewAI**: usable con un "crew" de uno, pero menos común. |
| **Level 6 – Multi-agent systems** | Múltiples agentes colaboran, negocian y delegan para resolver un problema complejo | **LangGraph**: ideal para interacciones complejas (cada agente/función = un node, los edges definen comunicación/handoffs/control flow; el explicit state facilita shared understanding). **CrewAI**: su filosofía de diseño primaria (un crew con roles distintos y un process sequential/hierarchical). **ADK**: un sistema de múltiples agent services ADK desplegados independientemente, comunicándose vía messaging o A2A. |

Los frameworks son **el puente de los modelos "agent-ready" (Nivel 3) a los sistemas agénticos funcionales (Niveles 4-5)**: dan el *cómo* esencial para construir las apps sofisticadas que el libro diseñó.

## Citas

> "the underlying design patterns discussed in this book are universal and generalize to other frameworks."
> "There is no single 'best' agent framework."
> "avoid framework lock-in"
> "observability is a cornerstone of responsible AI. If you cannot trace why an agent made a decision, you cannot ensure it is fair, safe, or compliant."
> "the framework is a tool to implement the patterns we've discussed."
> CrewAI's *team* · LangGraph's *state machine* · ADK's *production service*

## Para aplicar

- **Elegir el framework por su core abstraction**, no por moda: CrewAI (*team* role-based, prototipado rápido, delegación hierarchical), LangGraph (*state machine*, control flow complejo/cíclico, error handling explícito, HITL, debugging high-fidelity), ADK (*production service*, enterprise/Google Cloud, lifecycle completo, OpenTelemetry, callbacks/workflow agents).
- **Evitar el framework lock-in** — diseñar alrededor de interfaces estables (tool contracts estandarizados, explicit state schemas, pattern-based orchestration) en vez de acoplar todo a clases propietarias; así migrás/swapeás con mínima fricción.
- **Devolver JSON strings con data contracts** desde las tools (`{status/data}` o `{error}`) para que el LLM las parsee y los agentes downstream manejen errores determinísticamente; en producción, structured output con Pydantic (no string matching frágil).
- **Imponer fail-fast con conditional edges** (LangGraph) — un `error` key en el state + routing inmediato a rejection ahorra costos/latencia y previene alucinar decisiones sobre data inválida (vs CrewAI que sigue corriendo pasos intermedios).
- **temperature=0.0** para agentes de lógica/orquestación/tool-use (máxima estabilidad, aunque no 100% determinístico por las GPUs).
- **Normalización upstream y proxy tools** — un preprocessing node para currency/locale antes del risk assessment; las tools de lógica crítica como proxies (Agent Calls Proxy Agent) a un risk engine/ML endpoint externo, manteniendo el agente liviano.
- **Observability como pilar del responsible AI** — LangSmith (LangGraph/CrewAI, time-travel debug) u OpenTelemetry/Cloud Trace (ADK); habilitar transparency (golden dataset / disparate impact testing), safety (fail-fast, safety_settings, PII redaction) y accountability (immutable audit trail a BigQuery, trace IDs).

## Conexiones

- [[_Agentic Architectural Patterns for Building Multi-Agent Systems|Mapa del libro]] — el MOC del libro.
- [[14 - Use Case - A Multi-Agent System for Loan Processing]] — cap. 14 (anterior): el mismo use case implementado en ADK, que aquí se reimplementa en CrewAI y LangGraph para comparar paradigmas.
- [[13 - Use Case - A Single Agent for Loan Processing]] — cap. 13: el origen del use case y de las 4 tools de negocio.
- [[05 - Multi-Agent Coordination Patterns]] — cap. 5: la Supervisor Architecture (hierarchical de CrewAI) vs el control flow explícito; el routing y delegación que los frameworks materializan.
- [[01 - GenAI in the Enterprise - Landscape, Maturity, and Agent Focus]] — cap. 1: el GenAI Maturity Model (Tabla 15.3) y el stack de interoperabilidad (A2A nativo en ADK).
- [[08 - Human-Agent Interaction Patterns]] — cap. 8: el patrón **Agent Calls Proxy Agent** (la proxy tool) y el HITL (interrupts de LangGraph).
- [[07 - Robustness and Fault Tolerance Patterns]] — cap. 7: el fail-fast / conditional edges, el error handling robusto y los safety guardrails.
- [[06 - Explainability and Compliance Agentic Patterns]] — cap. 6: la observability/audit trail como base de la explainability y el compliance.
- [[11 - Advanced Adaptation - Building Agents That Learn]] — cap. 11: el "LLM-as-a-judge" de LangSmith, el `crew.train()` y la evaluación que conectan con el flywheel de auto-mejora.
- [[LangGraph]] — **ya existe en el vault** (AI Agents): el framework de state machine de este capítulo.
- [[Orchestrator]] — el manager de CrewAI / el control flow de LangGraph.
- [[Generator-Evaluator Pattern]] — los self-correction loops de LangGraph y el Hallucination Guardrail de CrewAI.
- [[Function Calling]] · [[Tool Calling]] — la base común de los tres frameworks (`BaseTool`, `FunctionTool`, nodes con tools).
- [[A2A]] · [[MCP]] — el A2A nativo de ADK para multi-agente cross-framework.
- [[Evals]] · [[LLM as Judge]] — la evaluación (CrewAI Test CLI, LangSmith trace-level, Patronus AI).
- [[Circuit Breaker]] · [[Retry with Backoff]] — calzan con el fail-fast y el error handling.
- [[State Machine]] — el paradigma core de LangGraph (candidato a enlace si existe en el vault).
- **Google ADK** · **CrewAI** · **LangGraph** · **LangSmith** · **OpenTelemetry** · **BaseTool / FunctionTool** · **StateGraph / TypedDict / conditional edges** · **Pydantic structured output** · **Patronus AI** · **Hallucination Guardrail** · **Framework lock-in** — conceptos/herramientas del capítulo; candidatos a nota propia.
