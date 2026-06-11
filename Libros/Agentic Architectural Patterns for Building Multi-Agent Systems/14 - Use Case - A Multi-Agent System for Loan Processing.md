---
title: "14 - Use Case - A Multi-Agent System for Loan Processing"
libro: Agentic Architectural Patterns for Building Multi-Agent Systems
autor: Ali Arsanjani
capitulo: 14
created: 2026-06-11
tags:
  - libros/agentic-architectural-patterns
  - type/case-study
  - status/permanent
aliases:
  - Use Case - A Multi-Agent System for Loan Processing
---

# 14 - Use Case - A Multi-Agent System for Loan Processing

> [!info] Capítulo 14 · *Agentic Architectural Patterns for Building Multi-Agent Systems* — Ali Arsanjani
> La segunda fase del use case: re-arquitecturar el agente monolítico del cap. 13 en un **sistema multi-agente jerárquico (Level 4)** con la **Supervisor/Orchestrator Architecture** y el patrón **Agent Delegates to Agent** (vía `AgentTool` de ADK), promoviendo las "tools" a un equipo de agentes especializados colaborativos. Resuelve los límites del monolito (cognitive overload, single point of failure, falta de especialización) y desbloquea scalability/resilience/maintainability. Navegá: [[_Agentic Architectural Patterns for Building Multi-Agent Systems|Mapa del libro]] · anterior [[13 - Use Case - A Single Agent for Loan Processing]].

## Resumen

Tras construir y testear el agente monolítico single-agent (cap. 13, un Level 3/5 capaz pero frágil), este capítulo lo evoluciona a **Level 4** re-arquitecturándolo en un **sistema multi-agente**. El monolito, aunque milestone necesario, tenía tres límites inherentes: **cognitive overload y maintainability** (el prompt FCoT central se vuelve cada vez más grande y complejo; agregar tools o lógica de negocio lo hace difícil de debuggear/mantener sin side effects), **single point of failure** (todo depende del reasoning loop de un agente; si malinterpreta un paso o falla por un issue del modelo, todo el workflow se detiene), y **lack of specialization** (el agente es un generalista que debe conocer las intricacies de document validation, credit policies, risk modeling y compliance, aumentando su cognitive load y dificultando actualizar un área sin afectar las otras).

La solución: aplicar la **Supervisor (Orchestrator) Architecture**, promoviendo las "tools" especializadas a un *team de agentes colaborativos single-purpose*. Esto transforma la app de una entidad única y compleja en un sistema de componentes más simples y altamente interoperables. El recorrido: (1) **hierarchical agent architectures** —el patrón Supervisor/Orchestrator (un manager supervisando un crew de especialistas) habilitado por **Agent Delegates to Agent** (la clase `AgentTool` de ADK), fault tolerance & isolation, y modularity—; (2) **building the system** —definir las tools especializadas, crear el specialist crew (cada uno su propio `LlmAgent` con instrucciones enfocadas y una sola tool, envueltos en `AgentTool`; con la opción de *heterogeneous model selection*), agregar production guardrails (rate limiting, exponential backoff, thinking budgets), y refactorizar la mente del orquestador (de *doer* a *manager*, con contextual state management)—; (3) **execution & analysis** —dos escenarios (happy path approval + denied path por policy violation) con los traces FCoT del orquestador—; (4) **patterns in practice** —Supervisor, Agent Delegates to Agent, FCoT—; (5) **mapping a los Agentic AI Levels** —L3 monolito frágil → L4 team resiliente, y el horizonte beyond L4 (meta-agents L5, emergent swarms/self-improving L6, el protocolo A2A bajo la Linux Foundation)—. Conclusión: el shift de *doer* a *manager of specialized workers* es la característica definitoria del Level 4, y los principios de la buena arquitectura de software (problem decomposition, separation of concerns) aplican directamente a la IA agéntica.

## Hierarchical agent architectures

Aunque los sistemas multi-agente pueden organizarse en varias topologías (swarms descentralizados, redes peer-to-peer aptas para tareas creativas open-ended), los workflows enterprise regulados requieren control más estricto. Para un loan processing pipeline —donde auditabilidad y adherencia a pasos específicos son no-negociables— el diseño más efectivo es la **arquitectura jerárquica**, también conocida como patrón **Supervisor/Orchestrator**: organiza a los agentes en una estructura tipo equipo corporativo, con un manager supervisando un crew de especialistas. En vez de un único agente monolítico que intenta hacer todo, las responsabilidades se dividen claramente.

Dos roles de agente:
- **The supervisor (parent) agent** — actúa de *manager*: planificar el workflow general, delegar sub-tareas al especialista apropiado, monitorear el progreso y sintetizar el resultado final de los hallazgos de los especialistas.
- **The specialist (sub-agent)** — los expertos individuales; cada uno diseñado para hacer *una cosa excepcionalmente bien* (validar un documento, calcular un risk score); recibe una tarea, la ejecuta con sus tools dedicadas y devuelve un resultado a su supervisor.

**Patrones arquitectónicos que habilitan la orquestación y modularidad**:
- **Agent Delegates to Agent** — el patrón core: el supervisor no hace el trabajo sino que lo delega a otro agente autónomo. La clase **`AgentTool` de ADK** es su implementación directa: encapsula un agente independiente completo (con su propio LLM, instrucciones y tools) detrás de una interfaz de tool estándar, permitiendo al supervisor invocarlo síncronamente como cualquier función — *con el trade-off de latencia anidada y costos de tokens*.
- **Fault Tolerance and Isolation** — al romper el workflow en agentes independientes, se aíslan los fallos: si el `CreditCheckAgent` falla, no crashea todo el sistema; el supervisor puede atrapar el error y decidir un curso de acción (detener el proceso, escalar a un humano). Mejora significativa sobre el diseño monolítico.
- **Modularity** — cada agente especialista es un módulo de expertise self-contained; se puede actualizar, mejorar o reemplazar el `RiskAssessmentAgent` sin tocar ninguna otra parte del sistema → arquitectura mucho más mantenible y escalable.

![[14-fig-14.1.png]]
*Figura 14.1 – A simplified model of a hierarchical agent architecture*

![[14-fig-14.2.png]]
*Figura 14.2 – Contrasting monolithic with multi-agent architectures*

## Building the multi-agent system

### Equipando especialistas con tools dedicadas

Se definen las funciones Python que servirán de tools dedicadas para cada especialista (con type hints estándar por simplicidad; un sistema production-grade debería imponer schemas estrictos input/output con **Pydantic** para garantizar la integridad de datos entre agentes). Cada función contiene la lógica de negocio precisa de su tarea, incluyendo las reglas de los escenarios "unhappy path" (ej. flaggear inmediatamente un credit score < 600 como condición de fallo high-risk). Las 4 funciones:
- `validate_document_fields(application_data: str) -> str` — simula el parsing de los documentos entrantes, asegurando que la estructura JSON sea válida y todos los campos requeridos estén presentes.
- `query_credit_bureau_api(customer_id: str) -> str` — interfaz a data externa, recuperando un credit score para un customer ID dado.
- `calculate_risk_score(loan_amount: int, income: str, credit_score: int) -> str` — encapsula la lógica financiera core, computando una métrica de riesgo basada en la relación entre income, loan amount y credit score.
- `check_lending_compliance(credit_history: str, risk_score: int) -> str` — impone la governance de negocio, verificando que el risk profile calculado cumpla los estándares regulatorios de la institución.

Cada función se envuelve en `FunctionTool` (`validation_tool`, `credit_tool`, `risk_tool`, `compliance_tool`); el wrapper inspecciona las firmas y genera la schema description que el LLM necesita para saber cómo llamar la tool.

### Creando el specialist agent crew

Con las tools definidas, se crean los agentes especialistas. Crucialmente, **cada agente es ahora su propia instancia `LlmAgent`**, con un modelo, un instruction set altamente enfocado y su única tool dedicada. Sus instrucciones son ahora más simples, diciéndoles solo que usen su tool y qué inputs esperar — imponiendo un **"data contract"** claro entre agentes. Ejemplo de instrucciones (`document_validator`): *"You are a Document Validation Agent. Your ONLY task is to call the `validate_document_fields` tool. INPUT REQUIREMENT: You must receive the complete, original loan application as a JSON string. If you receive the required input, call the tool and return its exact output. If the input is missing or malformed, respond with an error: 'ERROR: Missing or invalid application_data input.'"*

**Heterogeneous model selection** — este decoupling habilita una estrategia de producción potente: ya no estás atado a un único modelo para todo el workflow. Podés asignar un modelo liviano y low-latency (ej. **Gemini Flash**) al `document_validator` para extracción rápida, mientras reservás un modelo de razonamiento más capaz (ej. **Gemini Pro**) para el `compliance_checker` que maneja interpretación matizada de políticas. Optimiza costo y performance, pero mezclar modelos puede introducir variaciones sutiles de comportamiento; apoyarse en definiciones estrictas de `FunctionTool` es la mejor forma de mantener consistencia en un equipo de agentes diverso.

Cada agente especialista se crea como `LlmAgent(model, instruction, name, description, tools=[...])` y luego se envuelve en un **`AgentTool`** (`validator_agent_tool = AgentTool(agent=document_validation_agent)`, etc.) — el paso más crítico: el `AgentTool` actúa de *adaptador*, haciendo a un agente invocable por otro y habilitando el patrón **Agent Delegates to Agent**. Transforma los sub-agentes autónomos en tools invocables que el orquestador puede llamar.

> [!warning] **Production warning: the cost of abstraction** — cuidado con las cadenas de delegación profundas (ej. supervisor → manager → team lead → worker): cada capa agrega un round-trip completo de inferencia LLM, aumentando significativamente latencia y costos de tokens.

> [!note] **Design note: production configuration via GenerateContentConfig** — el código muestra el wiring estructural, pero los sistemas de producción requieren control de comportamiento estricto. En ADK (y el Google GenAI SDK) se gestiona vía el parámetro `generate_content_config` (un objeto `types.GenerateContentConfig`). Parámetros clave a configurar:
> - **Temperature** (`temperature`): `0.0` para agentes especialistas (validator, compliance) para forzar outputs determinísticos y analíticos; valores más altos (ej. `0.7`) solo para roles creativos/conversacionales.
> - **Output limits** (`max_output_tokens`): límites estrictos para prevenir runaway loops o verbosidad excesiva.
> - **Safety** (`safety_settings`): configurar block thresholds para que los agentes procesen data financiera sensible sin disparar false-positive refusals.
>
> Para la demostración arquitectónica se usan los defaults (foco en el patrón de orquestación), pero siempre configurarlos explícitamente en deployments live.

### Agregando production guardrails

Un sistema Level 4 también debe manejar el ruido de un entorno de producción. Se introducen patrones de robustez enterprise-grade para que el sistema no colapse bajo presión de la API. La execution logic se envuelve en un robust runner que implementa:
- **Rate limiting** — el decorador `@limits(calls=15, period=60)` throttlea proactivamente los requests para mantenerse dentro de la quota del provider (ej. 15 llamadas por minuto).
- **Exponential backoff** — usando la librería **`tenacity`** (`@retry` con `stop_after_attempt(5)`, `wait_exponential(multiplier=2, min=4, max=30)`, `retry_if_exception(is_rate_limit_error)`), el sistema reintenta inteligentemente si encuentra un error `RESOURCE_EXHAUSTED` (429): en vez de fallar la aplicación, espera y reintenta con intervalos crecientes.
- **Thinking budgets** — un `ThinkingConfig(include_thoughts=True, thinking_budget=1024)` permite al orquestador gastar más compute cycles en razonamiento interno, crucial para parsear el nested JSON devuelto por los sub-agentes.

(Más el decorador `@sleep_and_retry` de la librería `ratelimit`, envolviendo `start_agent_run`.)

### Revisando la mente del orquestador

El rol del agente principal ahora pasa de **doer a manager**.

> [!note] **Architectural note: contextual state management** — a diferencia de los workflow engines que pasan un dict Python estructurado (ej. `state = {'id': 123}`) entre pasos, este orquestador se apoya en **contextual state**: el "estado" es la historia acumulada de outputs JSON de los agentes especialistas. Por ejemplo, cuando el document validator devuelve `{"status": "valid", "extracted_data": {"customer_id": "CUST-123", "income": "5000"}}`, el orquestador lo lee de su context window y construye dinámicamente el payload para la próxima llamada delegada: `credit_checker_tool(customer_id="CUST-123")`. Esto requiere instruir explícitamente al orquestador sobre *qué campos extraer y reenviar*, actuando efectivamente de **semantic data mapper**.

Se actualiza el prompt FCoT del agente principal: en vez de una lista de tools, sus instrucciones ahora refieren a su *specialist team*, y lo hacen explícitamente responsable de pasar el data context correcto de un agente al siguiente (resolviendo el problema de "loss of context"). El prompt del orquestador incluye:
- **Failure Handling Policy** (3 pasos): (1) **Reflect** — si un especialista devuelve un error, primero analizar el mensaje; (2) **Resolve** — si el error es por info faltante, revisar el request original; (3) **Escalate** — solo si no podés resolverlo por tu cuenta, escalar.
- **Your Specialist Team** (roster con input requirements estrictos): `document_validator` (devuelve la data validada), `credit_checker` (pasarle el `customer_id` de la data validada), `risk_assessor` (pasarle `loan_amount`, `income`, `credit_score`), `compliance_checker` (pasarle `credit_history` AND `risk_score`).

Estas instrucciones definen los principios operativos del supervisor: al definir explícitamente una failure handling policy y un roster con input requirements estrictos, se asegura que el orquestador *gestione* el workflow en vez de intentar ejecutar cada paso él mismo.

## Ejecución y análisis

Se monta el ADK `Runner` y una función `call_agent` (implementando el patrón **Observability**): filtra y muestra los eventos crudos del execution loop —thoughts (🧠), tool calls (🛠️), outputs (↩️)— separando los thoughts (monólogo interno guiado por FCoT) de los tool calls (acciones), esencial para verificar que el orquestador planifica y delega correctamente. Session init con `InMemorySessionService`, USER_ID, SESSION_ID vía `uuid4()`.

### Scenario 1: happy path (successful approval)

Aplicación válida de un aplicante bien calificado (`CUST-12345`, income USD 5000/mes, loan $50,000, credit history "Very Solid", 4 documentos). El trace muestra **dos loops FCoT completos**:
- **Iteración 1 (Planning)**: el primer thought del orquestador es un loop FCoT completo — **RECAP** (reconoce el request e inventaría sus recursos: los agentes especialistas), **REASON** (formula un plan multi-step preciso; crucialmente no solo secuencia los agentes sino que *razona sobre el data flow entre ellos*, mencionando explícitamente que va a "extract the customer ID" de un paso para usar en el siguiente → demuestra awareness de los data contracts embebidos en el prompt), **VERIFY** (critica su propio plan contra sus reglas internas: *"I've verified that each agent's input matches their expected data contract"* — acto de self-correction que reduce el riesgo de errores).
- Luego los **TOOL CALLs** (`document_validator` → `{status: validated}` → `credit_checker` → score **810** → `risk_assessor` → risk score **6** → `compliance_checker` → green light).
- **Iteración 2 (Synthesis)**: el thought final antes de responder es otro loop FCoT enfocado en síntesis — **RECAP** (resume el estado del workflow), **REASON** (sintetiza los hallazgos individuales en una narrativa coherente: "document validation was a clean pass... credit check came back with a solid score - 810... risk score of 6, which falls within acceptable parameters... compliance check? That's a green light"), **VERIFY** (confirma que tiene todo para producir el deliverable final). → ✅ **Loan Decision: Approved**.

Este proceso de razonamiento estructurado en dos fases es el sello del patrón FCoT: fuerza al agente a ser deliberado en su planning y riguroso en sus conclusiones.

### Scenario 2: denied path (high-risk failure)

Testea la capacidad de observar, moderar e imponer reglas de negocio críticas (*policy adherence*). Aplicación de alto riesgo (`CUST-55555`, income USD 1000/mes, loan **$1,000,000**, credit history "Presenting Gaps", solo 1 documento `drivers_license.pdf`). El trace:
- **Iteración 1 (Planning)**: loop FCoT *idéntico* al happy path (RECAP/REASON/VERIFY "Data dependencies are solid") → demuestra la **consistent planning** del sistema, mismo proceso riguroso sin importar el contenido de la aplicación.
- Una serie de TOOL CALLs exitosos (validator, credit checker score **680**, risk assessor) → `risk_assessor` devuelve `{"risk_score": 8}` → `compliance_checker(risk_score: 8)` → `{"is_compliant": false, "reason": "Policy violation: Risk score of 8 is too high for approval."}`.
- **Iteración 2 (Synthesis)** — la parte más reveladora: **synthesizing contradictory evidence**. El agente enfrenta evidencia conflictiva: los primeros pasos fueron exitosos ("documents looked good... credit check showed a score of 680...") pero el paso final fue un hard failure ("The compliance check flagged it right away... It's a clear-cut policy violation"). El agente correctamente razona que el **compliance failure es el factor más importante y decisivo**. En VERIFY groundea su conclusión final directamente en el output del especialista: *"The primary reason for denial is the high-risk score of 8, which exceeds the acceptable threshold"* → cumple el patrón **Explainability and Audit Trail** dando una razón transparente y verificable del outcome negativo. → ✅ **Rejected** (policy violation).

Esta capacidad de *pesar evidencia y priorizar correctamente un fallo crítico sobre éxitos iniciales* es una capacidad de razonamiento sofisticada que FCoT habilita de forma confiable, demostrando cómo construir agentes que no solo ejecutan workflows sino que hacen juicios sólidos y auditables basados en los resultados.

## Examinando los patrones en práctica

El loan processor multi-agente no es un solo trozo de código corriendo un agente; es un *ecosistema agéntico dinámico* donde los patrones convergen para crear un sistema con inteligencia que espeja necesidades de negocio/dominio, verificado por evaluación rigurosa y asegurado por safety guardrails — una capacidad mucho mayor que la suma de sus partes.

### Pattern: Supervisor (AI orchestrator / meta-agent) architecture
El **master blueprint** de todo el sistema: ataca el desafío de gestionar complejidad abrumadora estableciendo una jerarquía clara de control. En vez de un agente monolítico experto en todo, una estructura "manager and team". El supervisor no *realiza* tareas: entiende el goal high-level, lo descompone en pasos lógicos, delega a los especialistas apropiados y sintetiza sus hallazgos en un resultado coherente. En el use case, el orquestador es el supervisor por excelencia: *su prompt FCoT no contiene lógica de negocio para validar documentos o evaluar riesgo*; todo su proceso cognitivo se dedica a workflow management. Se ve claro en el thought final del denied path (puro razonamiento managerial: recibe, interpreta y prioriza los reportes de su equipo para una decisión ejecutiva). Esta *separation of concerns* es el beneficio core. (Otro ejemplo: un supply chain management system donde un supervisor monitorea inventario y delega a `SupplierContactAgent`/`LogisticsAgent`/`PurchaseOrderAgent`.)

### Pattern: Agent Delegates to Agent
El **mecanismo primario** por el cual se implementa la Supervisor Architecture: una forma estandarizada de que un agente invoque las capacidades de otro agente autónomo. El agente que llama no necesita saber *cómo* funciona el otro, solo *qué* hace y qué inputs requiere → abstracción potente para sistemas modulares e interoperables. Se implementó directamente con el **`AgentTool` de ADK**: envolviendo cada especialista (ej. `document_validation_agent`) en un `AgentTool`, se vuelven componentes invocables que el orquestador trata como cualquier tool de su toolbelt. La encapsulación es clave: toda la lógica compleja (instrucciones del especialista, su tool Python dedicada, el LLM específico que usa) está completamente contenida dentro del agente especialista → permite desarrollar, testear y reemplazar especialistas sin modificar el orquestador. (Otro ejemplo: un smart home system donde un `HomeManager` recibe "Movie time" y delega a `LightingAgent`/`BlindsAgent`/`MediaAgent`.)

### Pattern: FCoT
El patrón que gobierna la mente del supervisor, su *cognitive operating system*: asegura que el razonamiento sea deliberado, auditable y alineado con sus instrucciones core, previniendo el *goal drift* en tareas complejas multi-step. Lo logra forzando un loop recursivo y adaptativo **RECAP → REASON → VERIFY** tanto para planning como para sintetizar la respuesta final. Usa una **dual objective function** para optimizar cada iteración a medida que empieza a *cerrar y enfocar su context aperture*. Este razonamiento estructurado es lo que hace al orquestador un manager confiable: no actúa impulsivamente, cada fase mayor está precedida por reflexión estructurada y self-correction. En el denied path, el synthesis loop de FCoT fue crítico para pesar la evidencia contradictoria. (Otro ejemplo: un scientific research agent resumiendo un paper — RECAP "necesito resumir este paper", REASON "leeré abstract y conclusión, luego la metodología, luego draft", VERIFY "es un plan lógico y eficiente".)

## Mapeo a los Agentic AI Levels

Este use case de dos capítulos ilustra prácticamente el viaje por los niveles más altos del Agentic AI Levels, mostrando no solo qué es cada nivel sino *por qué* la progresión es crítica para el éxito enterprise.

- **Level 3: the capable but brittle monolith** — el sistema single-agent del cap. 13 es el sistema Level 5 por excelencia (un agente autónomo único capaz de ejecutar un proceso de negocio end-to-end complejo): el `LoanProcessing` agent validaba documentos, chequeaba crédito, evaluaba riesgo y aseguraba compliance, entregando valor real. Pero su naturaleza monolítica —todo el cognitive load, business logic y workflow control concentrados en un único prompt masivo— introduce desafíos de producción: es inherentemente **frágil** (un pequeño error de lógica en una parte del prompt puede tener consecuencias no deseadas en todo el workflow), y a medida que el proceso de negocio evoluciona, mantener y actualizar este "cerebro" único y complejo se vuelve cada vez más difícil y riesgoso, limitando la scalability de largo plazo.
- **Level 4: the resilient, collaborative team** — el sistema multi-agente jerárquico de este capítulo es la implementación directa de un Level 4. El salto de L3 a L4 no es meramente agregar más agentes: es un **shift arquitectónico fundamental centrado en el principio de problem decomposition**. Se tomó un problema único y complejo y se lo rompió en una serie de tareas más pequeñas y simples; luego se construyó un team de agentes expertos altamente especializados, cada uno diseñado para hacer solo una cosa excepcionalmente bien (el equivalente agéntico de pasar de un único generalista brillante a un team coordinado de especialistas). Esta evolución desbloquea capacidades enterprise-grade reales: el sistema se vuelve dramáticamente más **resiliente** (un fallo en un especialista se aísla y el supervisor lo maneja gracefully sin bajar todo el proceso), más **mantenible** (la lógica de risk assessment se actualiza dentro de su agente dedicado sin riesgo de afectar el credit-checking), y más **escalable** —no solo técnicamente sino *cognitivamente*: distribuyendo el cognitive load, esta arquitectura L4 permite atacar problemas de un orden de complejidad mucho mayor—. El shift de **doer a manager of specialized workers** es la característica definitoria del Level 4, la cima de los Agentic AI Levels.

### Beyond Level 4: el futuro de la colaboración agéntica
El sistema jerárquico L4 representa el state of the art actual para apps production-grade, pero el viaje no termina ahí; los patrones construidos son la fundación de sistemas aún más dinámicos:
- **Level 5 con meta-agents** — sistemas que usan meta-agents para reasignación de tareas real-time.
- **Level 6 — emergent swarms** — *no hay supervisor fijo*: una colección de agentes especializados recibe un problema complejo y novel, y autónomamente forma un team *ad hoc* para resolverlo. Ej. en un escenario de corporate loan restructuring complejo, un planning agent podría tomar temporalmente el liderazgo, reclutando un data-analysis agent para modelar cash flow projections y un reporting agent para resumir legal risks, formando una jerarquía temporal que se disuelve al completarse el plan.
- **Level 6 — self-improving agentic systems** — un loan processor no solo procesa aplicaciones sino que *analiza su propia performance*: podría identificar que su especialista `risk_assessor` es un bottleneck frecuente y sugerir mejoras, o incluso intentar reescribir el código de la tool subyacente para mejor eficiencia; podría correr sus propios A/B tests sobre distintas versiones de sus prompts para optimizar accuracy y costo. Pasa de la ejecución a un estado de optimización continua, autónoma y self-correcting.
- **Protocolos de comunicación** — estos niveles futuros se apoyarán en protocolos estándar de la industria. El protocolo **A2A (Agent-to-Agent)**, ahora un *proyecto oficial bajo la Linux Foundation* (gobernado por un cross-industry steering committee), provee el estándar certificado para esta interoperabilidad, permitiendo a los agentes descubrir, negociar y colaborar sin importar su framework subyacente — habilitando una verdadera *economía de agentes*.

## Citas

> "the shift from a *doer* to a *manager of specialized workers* is the defining characteristic of Level 4"
> "a capability that is far greater than the sum of its individual parts."
> "the principles of good software architecture are directly applicable to the world of agentic AI."
> "I've verified that each agent's input matches their expected data contract."
> "It's a clear-cut policy violation; that's the bottom line."

## Para aplicar

- **Refactorizar el monolito a Supervisor/Orchestrator** cuando aparezcan cognitive overload, single point of failure o falta de especialización — promover las "tools" a un team de agentes single-purpose (un manager que delega + especialistas que ejecutan).
- **Implementar Agent Delegates to Agent con `AgentTool`** — envolver cada especialista (su propio `LlmAgent` con instrucciones enfocadas + una sola tool) en un `AgentTool` para que el orquestador lo invoque como una tool; cuidado con las cadenas de delegación profundas (cada capa = un round-trip LLM, latencia + tokens).
- **Imponer data contracts estrictos** entre agentes (input requirements en las instrucciones, Pydantic en producción) para que interoperen sin lógica de integración custom.
- **Heterogeneous model selection** — asignar modelos livianos (Gemini Flash) a tareas de extracción rápida y modelos potentes (Gemini Pro) a interpretación matizada (compliance); mantener consistencia con `FunctionTool` estrictas.
- **Production guardrails** — rate limiting (`@limits`), exponential backoff (`tenacity`/`@retry` ante 429), thinking budgets, y `GenerateContentConfig` (temperature 0.0 para especialistas, max_output_tokens, safety_settings).
- **El orquestador como semantic data mapper** — contextual state management: leer los outputs JSON acumulados del context window y construir dinámicamente el payload de la próxima delegación; instruirlo explícitamente sobre qué campos extraer/reenviar.
- **Failure Handling Policy en el orquestador** — Reflect (analizar el error) → Resolve (revisar el request original) → Escalate (solo si no podés resolver) → Human-in-the-Loop.
- **Verificar los patrones con observabilidad** — `call_agent` que filtra thoughts/tool calls/outputs; testear happy path Y denied path, confirmando que el FCoT pesa evidencia contradictoria y prioriza el fallo crítico (Audit Trail).

## Conexiones

- [[_Agentic Architectural Patterns for Building Multi-Agent Systems|Mapa del libro]] — el MOC del libro.
- [[13 - Use Case - A Single Agent for Loan Processing]] — cap. 13 (anterior): el **monolito que este capítulo refactoriza**; las 4 tools se promueven a 4 agentes especialistas, el FCoT pasa del agente único al orquestador.
- [[05 - Multi-Agent Coordination Patterns]] — cap. 5: la **Supervisor Architecture** (vs Swarm) y el Task Delegation Framework que aquí se materializan; el horizonte de emergent swarms (L6) es la Swarm Architecture descentralizada del cap. 5.
- [[12 - A Practical Roadmap - Implementing Agentic Patterns by Maturity Level]] — cap. 12: este capítulo *implementa* el salto L1→L2 del roadmap (monolítico → microservicios desacoplados) con su fault isolation y modularity.
- [[07 - Robustness and Fault Tolerance Patterns]] — cap. 7: el rate limiting, exponential backoff (Adaptive Retry), fault isolation y escalación que los production guardrails materializan.
- [[06 - Explainability and Compliance Agentic Patterns]] — cap. 6: el FCoT Embedding (el RECAP/REASON/VERIFY del orquestador) y el Explainability/Audit Trail del denied path; el *context aperture* y dual objective function.
- [[08 - Human-Agent Interaction Patterns]] — cap. 8: el **Agent Delegates to Agent** (colaboración invisible al usuario) y el Human-in-the-Loop del escalation path.
- [[11 - Advanced Adaptation - Building Agents That Learn]] — cap. 11: el horizonte L6 self-improving (A/B testing de prompts, reescribir tools) que este capítulo anticipa.
- [[Orchestrator]] — **el lugar canónico del vault**: el orquestador supervisor de este use case.
- [[A2A]] — el protocolo Agent-to-Agent (ahora bajo la Linux Foundation) para la economía de agentes futura.
- [[Generator-Evaluator Pattern]] — el paso VERIFY de cada loop FCoT.
- [[Circuit Breaker]] · [[Retry with Backoff]] · [[Rate Limiting]] — los production guardrails (tenacity, @limits, 429).
- [[Function Calling]] · [[Tool Calling]] — el `FunctionTool` y `AgentTool` de ADK.
- **Google ADK / AgentTool** · **Heterogeneous model selection (Gemini Flash vs Pro)** · **GenerateContentConfig** · **tenacity (exponential backoff)** · **Contextual state management / semantic data mapper** · **Context aperture / dual objective function** · **A2A (Linux Foundation)** · **Emergent swarms / self-improving systems** — conceptos/herramientas del capítulo; candidatos a nota propia.
