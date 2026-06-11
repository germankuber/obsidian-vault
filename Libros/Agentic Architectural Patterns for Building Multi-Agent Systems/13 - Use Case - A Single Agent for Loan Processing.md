---
title: "13 - Use Case - A Single Agent for Loan Processing"
libro: Agentic Architectural Patterns for Building Multi-Agent Systems
autor: Ali Arsanjani
capitulo: 13
created: 2026-06-11
tags:
  - libros/agentic-architectural-patterns
  - type/case-study
  - status/permanent
aliases:
  - Use Case - A Single Agent for Loan Processing
---

# 13 - Use Case - A Single Agent for Loan Processing

> [!info] Capítulo 13 · *Agentic Architectural Patterns for Building Multi-Agent Systems* — Ali Arsanjani
> El primer use case hands-on: construir de cero un **agente monolítico (Level 3)** que automatiza un loan origination pipeline con **Google ADK** + Gemini, usando el patrón **FCoT (Fractal Chain-of-Thought)** como núcleo cognitivo. Se implementa, ejecuta (happy path + rejection) y *deliberadamente se exponen sus límites* (single point of failure, cognitive overload) para motivar el rediseño multi-agente del cap. 14. Navegá: [[_Agentic Architectural Patterns for Building Multi-Agent Systems|Mapa del libro]] · anterior [[12 - A Practical Roadmap - Implementing Agentic Patterns by Maturity Level]] · siguiente *cap. 14 (multi-agent loan processing)*.

## Resumen

Tras toda la teoría (anatomía del agente, maturity model, patrones de diseño), el libro pasa a la práctica con un **use case real: automatizar un loan origination pipeline**. Se ataca en dos fases (caps. 13 y 14): primero, construir todo el sistema con un **único agente monolítico** (este capítulo); aprender a implementar el enfoque/patrón **FCoT (Fractal Chain of Thought)** para estructurar el razonamiento y definir un toolbelt robusto. El objetivo no es solo prototipar: se usa esta implementación inicial para **exponer deliberadamente las tensiones arquitectónicas** inherentes a un diseño single-agent —específicamente *cognitive overload* y *fault isolation* ante complejidad de producción—. Identificando esas "grietas en la fundación", se prepara el terreno para el cap. 14, donde el sistema se re-arquitectura con un enfoque multi-agente más robusto, mantenible y escalable.

El recorrido: (1) **el challenge** —un loan pipeline es ideal para un agente porque es una *secuencia de etapas complejas* (validación de documentos, credit check, risk assessment, compliance review, decisión final), cada una con su lógica, data y potencial de fallo; la automatización tradicional con `if/then` rígidos sería frágil e inmanejable—; (2) **el FCoT framework** —un prompt que actúa de "sistema operativo" interno del agente, combatiendo el *goal drift* y el *lost in the middle* con un **Instruction Contract (IC)** inmutable + un **Recursive Loop** de planning/execution/verification—; (3) **diseño monolítico** —un único agente capaz ejecuta todo el workflow (FCoT reasoning core + state management vía Runner/SessionService + toolbelt)—; (4) **la implementación en Colab** con Google ADK (las 4 tools, el prompt FCoT, la config del agente con BuiltInPlanner/ThinkingConfig); (5) **ejecución y análisis** —happy path (Borrower-789, aprobado) y exception path (Borrower-400, score 450, denegado con explainability report)—; (6) **roadmap de mejora** —los límites del monolito (single point of failure, mezcla de business concerns en un prompt, finite cognitive capacity) que motivan evolucionar a multi-agente—. Conclusión: *"an effective agent is not merely prompted; it is engineered"* — el monolito es un milestone valioso (la forma más rápida de entregar una solución end-to-end autónoma) y la fundación perfecta sobre la cual construir sistemas más complejos.

## Technical requirements

- **Google account** (para Google Colab y Google AI Studio).
- **Google Colab** — los ejemplos corren en un notebook (entorno Python cloud gratuito); livianos, no requieren hardware local potente.
- **Google AI Studio API key** — para acceder a los modelos Gemini.
- **Python libraries** — `google-adk` (Google **ADK / Agent Development Kit**) y paquetes estándar. Se eligió ADK porque ofrece una *production-first architecture* que soporta nativamente el structured reasoning y el strong typing requeridos por el patrón FCoT.
- Código completo en el GitHub del libro: `PacktPublishing/Agentic-Architectural-Patterns-for-Building-Multi-Agent-Systems/tree/main/Chapter_13`.

## El challenge: un workflow high-stakes (Tabla 13.1)

Un loan origination pipeline es ideal para un sistema agéntico porque no es una sola tarea, sino una secuencia de etapas complejas, cada una con su lógica, data y potencial de fallo.

| Etapa | Descripción | Desafío clave para la automatización |
|---|---|---|
| 1. Document intake & validation | Recibir la aplicación y asegurar que todos los documentos de soporte (ej. income verification) estén completos y válidos. | (Nota: en el ejemplo se asume que todos los docs están provistos; no se trata el intake real / OCR.) Manejar varios formatos, identificar info faltante y aplicar reglas de negocio de completitud. |
| 2. Credit check | Interactuar con APIs externas de credit bureau para traer el historial y score crediticio del borrower. | Manejar credenciales de forma segura, parsear respuestas de API variadas y manejar gracefully errores de red o downtime de la API. |
| 3. Risk assessment | Aplicar la lógica de negocio interna y modelos dinámicos de risk-scoring sobre toda la data financiera reunida. | Ejecutar lógica compleja y a menudo no-lineal que sintetiza múltiples data points (no una simple regla `if/then`). |
| 4. Compliance review | Auditar el proceso para asegurar adherencia a todas las regulaciones relevantes, como el **ECOA (Equal Credit Opportunity Act)**. | Mantener un trail auditable del proceso de decisión y asegurar que ningún atributo protegido influyó el resultado. |
| 5. Final decision & generation | Sintetizar toda la info para tomar una decisión final approve/deny y generar la documentación necesaria. | Crear una justificación coherente y human-readable de la decisión basada en todo el workflow previo. |

Automatizar esto con scripts tradicionales crearía un sistema frágil y difícil de mantener: la automatización tradicional se apoya en reglas rígidas predefinidas que se rompen fácil ante data no estructurada (formatos de documento variables, detalles ambiguos del aplicante); manejar cada excepción requeriría una telaraña interminable de lógica `if/then` que se vuelve inmanejable. El workflow requiere **razonamiento dinámico** (manejar excepciones gracefully), **structured planning** (adaptar los pasos según el contexto) y la capacidad de generar un **trail auditable** de sus decisiones — ahí brilla la IA agéntica.

## Guiando la mente del agente con el framework FCoT

Para que el agente opere con el rigor requerido para esta tarea financiera, se lo equipa con un framework cognitivo sofisticado basado en el enfoque/patrón **FCoT (Fractal Chain-of-Thought)**. El prompt FCoT actúa de "sistema operativo" interno del agente, dando una estructura formal a su misión, constraints y proceso de razonamiento — es la *constitución* que gobierna sus acciones.

En procesos de negocio complejos y multi-step, los agentes más simples sufren **goal drift** (fallo de comportamiento donde el agente se desvía gradualmente de su objetivo original a medida que avanza la tarea). Este drift es causado frecuentemente por la limitación técnica conocida como **"lost in the middle"** (los LLMs luchan por recordar constraints o instrucciones críticas enterradas profundo en una context window larga). FCoT combate ambos problemas imponiendo una estructura rígida y auto-correctiva sobre el razonamiento, asegurando que el agente **constantemente vuelva a referirse a su misión y constraints core sin importar la longitud del contexto**.

FCoT se compone de **dos elementos primarios**:
- **Instruction Contract (IC)** — la *fuente de verdad inmutable* del agente. Define formalmente su misión, los deliverables exactos que debe producir, y los guardrails de seguridad y compliance que nunca debe violar.
- **Recursive Loop** — define el *proceso de pensamiento activo* del agente. Lo compele a iterar a través de ciclos de **planning** sus acciones, **executing**-las y —lo más importante— **verifying** su trabajo y razonamiento contra el IC en cada etapa.

## Diseñando un agente monolítico

Para esta primera implementación se adopta un enfoque **monolítico**: un único agente altamente capaz ejecuta todo el workflow de principio a fin. Es común en las etapas iniciales del desarrollo agéntico (reflejando el **Nivel 3** del Agentic AI Levels — full agentic autonomy vía introspective reasoning y self-correction, tras los Niveles 1-2 de prompting y workflows básicos). Concentra toda la lógica, tools y state management en un componente centralizado; la tarea primaria del agente es seguir su prompt FCoT interno, invocando secuencialmente la tool correcta en el momento correcto.

El agente monolítico se compone de **tres partes esenciales**:
- **The FCoT reasoning core** — el prompt FCoT como el "cerebro" del agente.
- **State management** — el agente necesita trackear el status de la aplicación a medida que progresa; `Runner` y `SessionService` de ADK lo manejan.
- **The toolbelt** — para interactuar con el mundo exterior, una colección de funciones que puede llamar.

La arquitectura es directa: el agente, guiado por su FCoT core, usa sus tools para reunir y procesar info hasta poder tomar una decisión final.

![[13-fig-13.1.png]]
*Figura 13.1 – Architectural diagram of the monolithic loan processing agent*

## Construyendo el agente en un Colab notebook

### Setup y dependencias

Instalar `google-adk` e importar los módulos clave (que mapean a la anatomía del agente):
- `LlmAgent` — la clase core del agente; une modelo, instrucciones y tools en una unidad cohesiva.
- `BuiltInPlanner` y `ThinkingConfig` — potencian el reasoning core; el planner gestiona el loop de ejecución, la config permite habilitar/controlar el proceso de pensamiento interno para observabilidad.
- `FunctionTool` — el wrapper que convierte funciones Python estándar en un toolbelt que el agente entiende e invoca.
- `Runner` e `InMemorySessionService` — manejan el state management y el lifecycle de ejecución (historial de conversación y contexto de la sesión).
- Librerías estándar (`os`, `time`, `random`, `uuid`) — gestionar API keys, simular latencia, generar mock data y crear session IDs únicos.

API key vía `getpass` → `os.environ["GOOGLE_API_KEY"]`; modelo **`gemini-3-flash`** (por velocidad y costo, ideal para aprender; en producción se recomienda Google Vertex AI). 

### Definiendo las tools

*"An agent's ability to reason is only as powerful as its ability to act."* El toolbelt es el componente crítico que conecta los procesos cognitivos del agente con el mundo real. Cuatro tools especializadas que simulan las etapas clave del workflow:
- `validate_document(document_ids: list[str]) -> dict` — simula el intake chequeando que los document IDs requeridos estén presentes; devuelve `validated` o `incomplete` (con `missing_docs`). Si hay <2 docs → incomplete.
- `run_credit_check(borrower_id: str) -> dict` — simula un request al credit bureau; devuelve un credit score y report summary, con un `time.sleep(2)` para imitar latencia de red. Si `borrower_id == "Borrower-400"` → score **450**, "Credit history is compromised"; si no → `random.randint(750, 850)`, "Credit history is clean".
- `assess_risk(credit_score: int, loan_amount: float) -> dict` — simula la lógica de underwriting interna; devuelve `low`/`medium`/`high`. Si `credit_score > 740` → `low`; si no → `high`.
- `check_compliance(risk_level: str) -> dict` — simula una auditoría regulatoria verificando adherencia a Fair Lending; devuelve `pass`.

Cada función se envuelve en `FunctionTool(func=...)`, lo que hace su descripción y parámetros disponibles al LLM core. Implementarlas como funciones Python simples con mock data crea un ejemplo self-contained, enfocando todo en la orquestación y el razonamiento del agente sin la complejidad de API keys live.

> [!note] **Production note** — en un deployment real, las firmas de las funciones serían las mismas pero su lógica interna cambiaría: en vez de devolver mock strings, actuarían de wrappers de operaciones complejas, llamando a document management systems internos, APIs de terceros, o comunicándose con otros servicios vía **MCP (Model Context Protocol)**.

> [!tip] **Production tip: strong typing for data contracts** — en el ejemplo se usan dicts simples para los outputs (para mantener el código accesible); en un sistema financiero real, las estructuras de datos *strongly typed* son esenciales para confiabilidad. Usar librerías como **Pydantic** para modelos rigurosos (ej. `class RiskAssessmentResult(BaseModel)`) impone schema validation, previene errores de tipo y sirve de "data contract" estricto entre agentes, asegurando que los componentes downstream reciban exactamente la estructura que esperan.

### Configurando la mente del agente (el prompt FCoT)

El prompt es una implementación directa del patrón FCoT, el master pattern de todo el razonamiento. Se compone de dos secciones: `INSTRUCTION CONTRACT` y `FCoT RECURSIVE LOOP`.

**INSTRUCTION CONTRACT (IC)** (contenido verbatim resumido):
- **Mission**: originar, evaluar y aprobar un préstamo con full policy compliance, factual grounding y fairness.
- **Deliverables** (JSON + Narrative summary): (a) borrower profile, (b) creditworthiness decision, (c) justification citing verified data, (d) compliance audit record, (e) explainability report.
- **Success Criteria**: Accuracy ≥ 95% vs gold truth; Policy compliance = 100%; Explainability coverage ≥ 90%; Latency < 5 min end-to-end.
- **Hard Constraints**: no PII en logs; debe seguir Fair Lending & ECOA; todos los campos numéricos validados de fuentes autoritativas.
- **Safety Policy**: rechazar data especulativa/alucinada; nunca fabricar detalles del borrower; **deferir casos ambiguos al Human-in-the-Loop agent**.
- **IC-Fingerprint**: `LOAN-FCoT-v3-Δ0710`.

**FCoT RECURSIVE LOOP (N = 3)** — cada iteración tiene RECAP / REASON / VERIFY:
- **Iteración 1 (Planning)**: RECAP (echo IC-FP, mapear subtasks); REASON (diseñar **DAG** de acciones, elegir fuentes de retrieval, inicializar PoF ledger); VERIFY (asegurar que todos los subtasks preservan las cláusulas del IC).
- **Iteración 2 (Execution)**: RECAP (IC-FP, ejecutar tools de credit scoring & data validation); REASON (computar risk score, validar fuentes contra la política); VERIFY (chequear *causal alignment* entre atributos del borrower y la lógica de decisión).
- **Iteración 3 (Verification & Explainability)**: RECAP (IC-FP, juntar deliverables, correr RAG verifier); REASON (resumir **SHAP values**, crear narrative justification); VERIFY (evaluar coherencia vs IC y dual objectives).

> [!note] **Pattern insight: semantic guardrails vs programmatic evaluation** — los success criteria (ej. `Latency < 5 min`) y deliverables del prompt ¿los impone el código Python? En este diseño monolítico Level 3 son **semantic guardrails**: no son assertions Python externas sino instrucciones para el reasoning engine interno del LLM. Al listar explícitamente Accuracy y Policy compliance como criterios, se fuerza al modelo a considerarlos durante sus pasos `VERIFY`. Idealmente el modelo se auto-corrige (ej. "necesito ser conciso para mantener baja la latencia"). En un sistema Level 5/producción, se emparejarían estos prompts con un **external evaluation harness** (herramientas como **DeepEval** o **Ragas**) para imponer las métricas programáticamente, pasando de verificación *in-context* a governance *system-level*.

> [!note] **Pattern insight: cognitive control vs code control** — la instrucción `FCoT RECURSIVE LOOP (N = 3)` es arquitectura cognitiva: `N=3` **no** es un parámetro Python pasado a `BuiltInPlanner`, sino una instrucción semántica al LLM para iterar tres repeticiones. Se está "programando el modelo en inglés", dirigiéndolo a estructurar su razonamiento interno en tres iteraciones (planning, execution, verification), cada una con sus objective functions, antes de considerar la tarea completa. El código Python (`thinking_budget=1024`) fija los *límites de recursos* (cuántos tokens puede usar); el prompt define el *algoritmo* (cómo debe usarlos).

> [!note] **Concept note: Explainable AI (XAI) y SHAP** — en la Iteración 3, `REASON: Summarize SHAP values` refiere a **SHAP (SHapley Additive exPlanations)** values, un método estándar de data science para interpretar modelos ML: asigna un valor numérico a cada feature (ej. Credit Score, Debt-to-Income) para cuantificar exactamente cuánto contribuyó a una predicción específica. Patrón híbrido potente: el **ML tradicional (la tool)** hace el cálculo de riesgo preciso y genera SHAP values crudos (ej. `Credit Score: -0.45 contribution`); el **GenAI (el agente)** actúa de *narrador*, traduciendo esos valores matemáticos secos en una justificación coherente y human-readable para el cliente (ej. "Your application was primarily impacted by your credit score..."). Se manifiesta en la sección `explainability_report` del JSON final — el agente no recita un resultado, sintetiza el *por qué* detrás del *qué*, satisfaciendo los requisitos de transparencia de sistemas Nivel 4 y 5.

**Patrones instanciados en el IC**:
- **Instruction Contract (IC) pattern** — todo el bloque `INSTRUCTION CONTRACT` es su implementación directa: un set fijo y no-negociable de reglas, goals y constraints que previene goal drift y asegura acciones auditables.
- **Guardrails pattern** — las secciones `Hard Constraints` y `Safety Policy`: `Must follow Fair Lending & ECOA` es un *compliance guardrail*; `Reject speculative or hallucinated data` es un *safety guardrail* que promueve factual grounding.
- **Human-in-the-Loop pattern** — `Defer ambiguous cases to Human-in-the-Loop agent` define el escalation path explícitamente: el agente sabe cuándo alcanzó el límite de sus capacidades y necesita ayuda.
- **Explainability and Audit Trail pattern** — la sección `Deliverables` exige (c) justification citing verified data, (d) compliance audit record, (e) explainability report → fuerza al agente a *mostrar su trabajo*.

**Patrones instanciados en el Recursive Loop** (el "motor" de FCoT):
- **Task Decomposition (Planner) pattern** — en Iteración 1, `REASON: Design DAG of actions`: el agente descompone la misión en una secuencia de subtasks manejables (un DAG) antes de ejecutar.
- **Tool Use pattern** — Iteración 2 (`execute tools for credit scoring & data validation`): FCoT asegura que el uso de tools no es aleatorio sino parte de una secuencia deliberada y pre-planeada.
- **Self-correction and Verification** — el paso `VERIFY` en *cada iteración* es la feature más potente de FCoT: en planning verifica que el *plan mismo* es compliant; en execution que el *razonamiento* es sólido (`Check causal alignment`); en la fase final que el *output* es coherente y cumple todo el IC.

### Instanciando el agente

Tres pasos de configuración:
1. **Configurar el planner (reasoning engine)** — `BuiltInPlanner` con `ThinkingConfig(include_thoughts=True, thinking_budget=1024)`. Crucialmente `include_thoughts=True` habilita la **observabilidad**: fuerza al agente a exponer su monólogo interno (el proceso FCoT) en el output, permitiendo debuggear su lógica de razonamiento en vez de solo ver el resultado final.
2. **Ensamblar el toolbelt** — agregar los 4 `FunctionTool` en una lista (`loan_processing_tools`), definiendo explícitamente el *action space* del agente (solo puede hacer lo listado).
3. **Instanciar el agente** — crear el `LlmAgent` (model=`gemini-3-flash`, name=`LoanProcessingAgent`, instruction=`agent_instructions`, planner, tools) que une modelo + cognitive core + planner + toolbelt en una entidad runnable.

## Ejecución y análisis

El goal primario es **observabilidad**: no solo saber si aprueba/deniega, sino *cómo* llega a esa conclusión. Verificar que respeta la estructura FCoT (planning/executing/verifying), mantiene estado a través de pasos, y maneja gracefully tanto aplicantes calificados como casos high-risk. Se monta un ADK `Runner` (con `InMemorySessionService`, USER_ID, SESSION_ID vía `uuid4()`) y una función helper `call_agent` que filtra el event stream e imprime un log limpio de los "thoughts" (🧠), tool calls (🛠️) y tool outputs (↩️) del agente.

`call_agent` incluye un bloque de **error handling profesional**: un error `429 Rate Limit` no debe ser un traceback críptico de Python sino un mensaje diagnóstico que le diga al operador qué pasó y cómo arreglarlo. Distingue `RESOURCE_EXHAUSTED`/`429` (quota exceeded) y `FreeTier`/`limit: 20` (free tier de ~20 requests/día, que el retry no puede bypassear) con acciones concretas ("Wait 24 Hours" / "Enable Billing").

### Happy path (Borrower-789)

El **happy path**: input válido, sin excepciones, outcome positivo. Verifica que el agente interpreta correctamente una aplicación bien calificada, dispara las tools en el orden correcto (validation → credit → risk → compliance) y produce una aprobación sin alucinar obstáculos. Request: `Borrower-789`, loan $250,000, docs `['doc_id_123', 'doc_income_456']`.

**Análisis del output** — el log muestra que el agente *explícitamente planea su approach en el bloque `THOUGHT` antes de tocar una tool* (la fase de Reasoning de FCoT en acción, previniendo que se apure a una conclusión alucinada): mapea el plan de 5 pasos, verifica antes de ir live (cada paso contribuye al goal; logging limpio sin PII; el paso de compliance atiende Fair Lending/ECOA; validar campos numéricos; rechazar data dudosa), y luego ejecuta `validate_document` → `{status: validated}` → `run_credit_check(Borrower-789)` → … → ✅ FINAL RESPONSE: "All checks are complete, and the application has passed compliance." El agente es funcional: razonó los pasos, ejecutó el plan, resultó en `Approved`.

### Exception path: handling rejection (Borrower-400)

Probar la robustez: un loan agent no sirve si aprueba a todos. Se introduce `Borrower-400` (historial crediticio comprometido). Verificar que el agente: identifica el credit score bajo, dispara un risk assessment `High`, *igual realiza el compliance check* (crucial para Fair Lending) y genera una denial con justificación clara y fact-based. Request: `Borrower-400`, loan $350,000, mismos docs.

**Observando el razonamiento crítico** — atención a la Iteración 3 (Verification & Explainability): aunque el préstamo se deniega, el agente produce un **Explainability Report** detallado citando los data points específicos (`Score 450`) que llevaron a la decisión — el patrón **Audit Trail** en acción. El flujo: `validate_document` → validated → `run_credit_check(Borrower-400)` → `{credit_score: '450', report_summary: 'Credit history is compromised.'}` → `assess_risk(450, 350000)` → `{risk_level: 'high', ...}` → `check_compliance('high')` → `{compliance_status: 'pass'}`. La Iteración 3 construye: **Creditworthiness Decision: Denied**; Justification (score 450 + compromised history → riesgo significativo; high risk level; compliance adherido); Explainability Report (Poor Financial History, High Risk Profile, Regulatory Adherence); VERIFY (coherencia vs IC, accuracy de fuentes, no speculative/hallucinated data).

**Outcome analysis** — confirma que el agente no es un "yes man": ingirió la data de `run_credit_check`, razonó que un score de 450 constituye high risk, y correctamente denegó el préstamo. Crucialmente, *igual corrió `check_compliance`*, asegurando que el proceso de denegación fue tan riguroso como el de aprobación.

## Roadmap: de Level 5 a 6 (mejora del diseño)

El sistema single-agent es un éxito y un ejemplo potente de implementación **Level 5** (en otra parte el texto lo llama Level 3 — refiere a la madurez introspective/autonomous): una solución completa y autónoma que sigue un instruction set complejo usando un toolbelt. Analizándolo con el lente de los Agentic AI Levels, se identifican las mejoras arquitectónicas (no debilidades) para avanzar al siguiente nivel:
- **Enhancing robustness and resilience** — el agente monolítico es un **single point of failure**: si la API call dentro de `run_credit_check` fallara por un issue temporal de red, toda la ejecución se detendría con un error sin manejar; el diseño no contempla fallos transitorios de sus dependencias. El camino a una arquitectura más resiliente (Level 4) aplica **fault isolation**: en vez de que el único agente sea responsable de todas las llamadas externas, introducir agentes especializados que encapsulen el riesgo de fallo de dependencias. Un `CreditCheckAgent` dedicado tendría su propia lógica de error-handling (retry automático con exponential backoff); si falla definitivamente, el error queda contenido en ese sub-sistema y el orquestador decide un fallback (invocar un backup credit bureau tool, o escalar vía **Human-in-the-Loop**) en vez de crashear todo el workflow.
- **Improving maintainability through specialization** — la lógica core vive en el variable `agent_instructions` (el prompt FCoT), un artefacto centralizado potente pero que **mezcla múltiples business concerns**: las reglas de document validation están en el mismo instruction set que las políticas de risk assessment y los compliance checks. Cuando el negocio evoluciona, un pedido del departamento de credit risk para actualizar su scoring logic obligaría a un developer a editar este prompt grande y complejo, con riesgo de romper accidentalmente la lógica de compliance o validation. Avanzar al **Level 4** aplica **separation of concerns**: descomponer el único agente en un *team de especialistas*, cada uno con su instruction set enfocado — un `RiskAssessmentAgent` (prompt más corto, owned por el equipo de credit risk), un `ComplianceAgent` (gobernado por legal/compliance) — permitiendo que cada parte evolucione independiente y segura.
- **Scaling to greater complexity with modularity** — el prompt FCoT funciona excelente para el proceso de 4 pasos definido, pero **un único LLM tiene capacidad cognitiva finita**. Expandir el workflow a 10-15 pasos (agregando `FraudDetection`, `CollateralValuation`, `InsuranceVerification`) volvería el prompt extremadamente largo y complejo, aumentando el riesgo del **"lost in the middle"**. Un sistema multi-agente (Level 5) resuelve este scaling con **división del trabajo cognitivo** (como un equipo humano bien dirigido): un `OrchestratorAgent` (implementación del patrón **Supervisor**) no necesita conocer los detalles de fraud detection, solo cuándo delegar al `FraudDetectionAgent`; cada especialista trabaja con un prompt más corto e hyper-enfocado en su dominio, reduciendo el cognitive load. Esta arquitectura modular plug-and-play permite agregar capacidades (un nuevo `InsuranceVerificationAgent`) sin aumentar la complejidad de los componentes existentes — pero realizarlo requiere imponer **standardized message schemas y shared state objects (data contracts)** para que los nuevos agentes interoperen sin lógica de integración custom.

## Citas

> "an effective agent is not merely prompted; it is engineered."
> "An agent's ability to reason is only as powerful as its ability to act."
> "Observability is non-negotiable." / "An agent's reasoning should not be a black box."
> "It successfully ingested data... reasoned that a score of 450 constituted high risk, and correctly denied the loan."
> "the monolithic agent is a valuable milestone"

## Para aplicar

- **Ingeniar el agente, no solo promptearlo** — construir las instrucciones sobre el patrón FCoT (Instruction Contract inmutable + Recursive Loop de planning/execution/verification) para embeber planning, safety y explainability en el core; combate goal drift y lost in the middle.
- **Usar el Instruction Contract como fuente de verdad** — mission, deliverables, success criteria, hard constraints, safety policy y un IC-Fingerprint; que el agente lo "eche" (RECAP) en cada iteración para no perder la misión.
- **Definir un toolbelt claro** — funciones Python envueltas en `FunctionTool` (descripción + params expuestos al LLM); empezar con mock data para enfocarse en la orquestación; en producción, wrappers de APIs reales vía MCP + **strong typing con Pydantic** como data contracts.
- **Habilitar observabilidad** — `ThinkingConfig(include_thoughts=True)` para exponer el monólogo interno (FCoT) y poder debuggear el razonamiento; no tratar al agente como black box.
- **Distinguir semantic guardrails vs programmatic evaluation** y **cognitive vs code control** — los criterios/N=3 del prompt son instrucciones al LLM (in-context); en producción emparejarlos con un evaluation harness externo (DeepEval/Ragas) para governance system-level.
- **Error handling profesional** — convertir un `429` en un mensaje diagnóstico accionable (causa, contexto, diagnóstico, acción), no un traceback.
- **Testear happy path Y exception path** — verificar que el agente aprueba lo calificado *y* deniega lo high-risk corriendo igual el compliance check, con explainability report (Audit Trail).
- **Reconocer los límites del monolito** — single point of failure, mezcla de business concerns, capacidad cognitiva finita → motivan fault isolation, separation of concerns y división del trabajo cognitivo (multi-agente).

## Conexiones

- [[_Agentic Architectural Patterns for Building Multi-Agent Systems|Mapa del libro]] — el MOC del libro; **este es el primer use case hands-on (Parte 3)**.
- [[12 - A Practical Roadmap - Implementing Agentic Patterns by Maturity Level]] — cap. 12 (anterior): este capítulo *implementa* un sistema Level 1/Foundational (monolítico/síncrono) del roadmap, con Supervisor + Watchdog implícito + Human Calls/Agent Calls Human + Simple RAG.
- [[09 - Agent-Level Patterns]] — cap. 9: el **Single Agent Baseline** y el FCoT que este capítulo materializa en código; la anatomía (tools = Act, SessionService = Memory).
- [[06 - Explainability and Compliance Agentic Patterns]] — cap. 6: el **FCoT Embedding** (autocorrección recursiva), el Instruction Fidelity / Persistent Instruction Anchoring (el IC anclado), y el lost in the middle que FCoT combate; explainability/audit trail.
- [[05 - Multi-Agent Coordination Patterns]] — cap. 5: la Supervisor Architecture y los agentes especializados (CreditCheckAgent, RiskAssessmentAgent…) que el roadmap L3→L4 anticipa y el cap. 14 implementará.
- [[07 - Robustness and Fault Tolerance Patterns]] — cap. 7: el fault isolation, retry con exponential backoff y fallback del roadmap de mejora; el rate limiting (429).
- [[08 - Human-Agent Interaction Patterns]] — cap. 8: el **Human-in-the-Loop / Agent Calls Human** (deferir casos ambiguos) del IC y del escalation path.
- [[01 - GenAI in the Enterprise - Landscape, Maturity, and Agent Focus]] — cap. 1: el mismo *loan processing* fue el ejemplo seminal del Task Delegation Framework; aquí se materializa.
- [[Function Calling]] · [[Tool Calling]] — el `FunctionTool` de ADK y el action space del agente.
- [[Grounding]] · [[Hallucinations]] — el "reject speculative or hallucinated data" y el factual grounding del IC.
- [[Generator-Evaluator Pattern]] — el paso VERIFY de cada iteración FCoT (auto-evaluación) lo emparenta.
- [[Orchestrator]] — el OrchestratorAgent (patrón Supervisor) del roadmap multi-agente.
- [[MCP]] — el Model Context Protocol mencionado como la vía de producción para las tools.
- **Google ADK (Agent Development Kit)** · **FCoT (Fractal Chain-of-Thought)** · **Instruction Contract (IC)** · **SHAP (SHapley Additive exPlanations) / XAI** · **DeepEval / Ragas** · **Pydantic (data contracts)** · **Gemini / Vertex AI** · **DAG of actions** · **Semantic guardrails vs programmatic evaluation** · **Cognitive control vs code control** — conceptos/herramientas del capítulo; candidatos a nota propia.
