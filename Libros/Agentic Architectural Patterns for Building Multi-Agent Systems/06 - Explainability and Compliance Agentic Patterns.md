---
title: 06 - Explainability and Compliance Agentic Patterns
libro: Agentic Architectural Patterns for Building Multi-Agent Systems
autor: Ali Arsanjani
capitulo: 6
created: 2026-06-11
tags:
  - libros/agentic-architectural-patterns
  - type/case-study
  - status/permanent
aliases:
  - Explainability and Compliance Agentic Patterns
updated: 2026-07-08
---

# 06 - Explainability and Compliance Agentic Patterns

> [!info] Capítulo 6 · *Agentic Architectural Patterns for Building Multi-Agent Systems* — Ali Arsanjani
> Cuatro patrones para que un sistema agéntico sea *trustworthy*: explicabilidad ("¿por qué hizo eso?") y compliance ("¿puedo verificar que siguió las reglas?"), combatiendo el *instruction drift* en jerarquías de agentes. Navegá: [[_Agentic Architectural Patterns for Building Multi-Agent Systems|Mapa del libro]] · relacionado [[09 - Agent-Level Patterns]] (de aquí provienen Anchoring y Fidelity Auditing).

## Resumen

Tras los patrones de coordinación entre agentes, este capítulo afronta lo que separa un prototipo de un sistema enterprise en producción: la efectividad debe ir acompañada de *trustworthiness*, porque **autonomía sin accountability es un pasivo** (*"Autonomy without accountability is a liability"*). Define dos dominios críticos: **Explainability** (hacer transparente y entendible el proceso de decisión del agente — *¿por qué hizo eso?*) y **Compliance** (asegurar que las acciones del agente cumplen el entramado de regulaciones externas y políticas internas — *¿puedo verificar si el agente siguió las reglas?*). Para incorporarlos, introduce cuatro patrones arquitectónicos: **Instruction Fidelity Auditing**, **Fractal Chain-of-Thought (FCoT) Embedding**, **Persistent Instruction Anchoring** y **Shared Epistemic Memory**. El enfoque, otra vez, es "map before the journey": primero una guía estratégica alineada al GenAI Maturity Model que muestra cómo la necesidad de estos patrones de accountability se profundiza a medida que el sistema evoluciona de un agente único a un colectivo multi-agente.

El problema transversal que todos atacan es el **instruction drift**: en sistemas jerárquicos, la intención original de una tarea se diluye o se pierde cuando agentes subordinados agregan su propio razonamiento. Cada patrón ataca un punto de falla distinto y complementario: Instruction Fidelity Auditing es el *check externo y reactivo* (un auditor compara el output contra la instrucción original antes de finalizar), FCoT Embedding es la *autogobernanza interna y proactiva* (razonamiento recursivo multinivel con autocorrección y reflectividad entre agentes), Persistent Instruction Anchoring es el *recordatorio constante de la misión* (tags que mantienen las instrucciones críticas salientes contra el *lost in the middle*), y Shared Epistemic Memory es la *capa fundacional de verdad* (un scratchpad central mutable que da ground truth compartido al colectivo). El capítulo cierra con una sección de **composición de patrones**: su valor real emerge al combinarlos en una defensa multicapa; la best practice production-grade es instanciar al menos dos o tres concurrentemente. Conclusión que cruza el capítulo: la confiabilidad sistémica no viene de una técnica sino de combinar checks internos (FCoT) y externos (auditing) sobre una base de verdad compartida y con la misión anclada.

## Guía estratégica: mapeo al GenAI Maturity Model (Tabla 6.1)

Cómo se profundiza la aplicación de cada patrón al madurar de single-agent (Nivel 5) a multi-agent (Nivel 6). (Nota: el capítulo usa los niveles de forma algo ambivalente — la tabla habla de Nivel 5 single-agent / Nivel 6 multi-agent, mientras el takeaway final menciona "Nivel 4" single complejo y "Nivel 5" multi-agente; el sentido es el mismo: a mayor colectividad, más esenciales son estos patrones.)

| Aspecto arquitectónico | Nivel 5 (single-agent) | Nivel 6 (multi-agent) |
|---|---|---|
| **Objetivo primario** | Asegurar que un agente autónomo único sea accountable y su razonamiento complejo auditable. | Asegurar que todo un sistema de agentes colaborando se mantenga alineado con el goal de nivel superior y sea colectivamente confiable. |
| **Instruction Fidelity Auditing** | Auditar el output final de un único agente antes de que tome una acción crítica. | Auditar los *handoffs* entre agentes, asegurando fidelidad de instrucción en cada paso de la jerarquía. |
| **Fractal CoT Embedding** | Habilita autocorrección interna de un único agente, refinando su propio razonamiento multi-step. | Habilita autocorrección interna **e inter-agent reflectivity**: los agentes revisan sus planes según el razonamiento de sus pares. |
| **Persistent Instruction Anchoring** | Mantiene a un único agente enfocado en su goal y constraints durante una tarea larga y multi-step. | Crucial para que la instrucción y constraints originales sobrevivan el viaje por una jerarquía profunda de múltiples agentes. |
| **Shared Epistemic Memory** | Menos crítico para un único agente, pero útil para loguear su estado o compartir contexto con un supervisor humano. | Esencial para la colaboración inter-agente. Se vuelve el "sistema nervioso central", proveyendo el "ground truth" del que depende todo el sistema. |

## Instruction Fidelity Auditing

- **Qué es** — capa de verificación que asegura que las acciones de un agente se adhieren estrictamente a las instrucciones originales que recibió. Combate los **"silent failures"**: sub-tareas que parecen completarse con éxito pero cuyo resultado global está desalineado con la intención de negocio original.
- **Contexto** — sistema multi-agente donde un agente supervisor delega tareas a uno o más worker agents; el éxito depende del cumplimiento estricto de todas las constraints de la instrucción original.
- **Problema** — ¿cómo asegurar que un agente autónomo, mientras optimiza su sub-tarea local, se mantenga estrictamente alineado con las instrucciones de alto nivel y constraints globales, previniendo silent failures donde el goal global se pierde?
- **Solución** — introducir un **auditor agent especializado** que actúa como checkpoint automatizado. Su rol NO es hacer la tarea sino **comparar el output del worker contra la instrucción original** que recibió, verificando que todas las constraints y goals iniciales se cumplieron antes de finalizar la acción. Crea una capa de verificación explícita que impone accountability y previene instruction drift.
- **Ejemplo (descuento e-commerce)** — `SupervisorAgent` (maneja requests según reglas de negocio), `DiscountAgent` (aplica descuentos), `ComplianceAuditAgent` (verifica compliance). Flujo: (1) **Instrucción**: "Apply a 15% 'WELCOME' discount for user 'user123' on order #5678, **only if** this is their first order". (2) **Delegación/ejecución**: el DiscountAgent aplica el 15% y devuelve `{"status": "SUCCESS", "action": "15% discount applied to order #5678"}` — pero **omite chequear** la constraint "first order". (3) **Audit**: antes de finalizar, el Supervisor manda la instrucción original + el output al ComplianceAuditAgent. (4) **Verificación**: el auditor consulta la base de clientes y encuentra que 'user123' tiene varias órdenes previas. (5) **Verdict**: `{"audit_status": "FAIL", "reason": "Worker agent failed to check the 'first order' constraint. User 'user123' is not a new customer."}`. (6) **Resolución**: el Supervisor recibe el fallo, **revierte** el descuento y avisa al usuario, evitando la violación de política.
- **Consecuencias** — *Pros*: **accountability y explainability** (checkpoint explícito de compliance; las justificaciones del auditor generan un trail auditable de cómo/por qué se validaron las decisiones), **reliability** (reduce silent failures, sistema más robusto y predecible). *Con*: **performance overhead** (la capa de auditing agrega latencia y costo; cada check puede requerir tool use o llamadas LLM extra → trade-off directo velocidad vs confiabilidad).
- **Guía** — seleccionar cuidadosamente los **junctures críticos** donde el auditing es más necesario; aplicarlo en cada paso impacta severamente la performance. Enfocarse en los handoffs donde el riesgo de malinterpretación o violación de política es más alto. Balancear compliance estricto vs latencia aceptable.
- **Naturaleza** — es un check **reactivo** (sobre el output ya producido); por eso se complementa con FCoT, que es proactivo.

![[06-fig-6.1.png|545]]
*Figura 6.1 – Instruction Fidelity Auditing workflow*

## Fractal Chain-of-Thought (FCoT) Embedding

- **Qué es** — estructura el razonamiento de un agente como un framework **recursivo y multinivel**, habilitando autocorrección dinámica y mejor alineación en tareas colaborativas. Supera la rigidez del CoT tradicional (lineal, estático, aislado) que no sirve para entornos multi-agente dinámicos donde hay que coordinar planes, adaptarse a info nueva y corregir el razonamiento según el feedback de otros.
- **Contexto** — sistema multi-agente con un problema complejo que requiere razonamiento colaborativo y adaptación; las conclusiones iniciales de un agente pueden invalidarse cuando otros aportan datos nuevos o conflictivos, requiriendo una forma dinámica de revisar pasos de razonamiento previos.
- **Problema** — ¿cómo estructurar el razonamiento para permitir autocorrección dinámica, sincronización con el razonamiento de otros agentes y análisis en múltiples niveles de detalle, superando la naturaleza estática y aislada del CoT tradicional?
- **Solución — framework recursivo y multi-escala** organizado en unidades autocontenidas (*thought units*) que se refinan recursivamente. Cuatro principios core:
  - **Recursive self-correction** — el razonamiento se guía por una *objective function* que simultáneamente **maximiza insight y minimiza error**, creando un loop continuo de autocorrección.
  - **Dynamic context aperture** — el agente puede ampliar o estrechar su foco, haciendo zoom-in para análisis detallado o zoom-out para incorporar contexto más amplio de otros agentes.
  - **Inter-agent reflectivity** — los agentes evalúan y reflexionan sobre el razonamiento de *otros* agentes, habilitando mejor alineación y delegación más inteligente.
  - **Temporal re-grounding** — pasos de razonamiento anteriores pueden **revisarse formalmente** a la luz de evidencia nueva, creando un proceso de pensamiento dinámico y auditable (no un log append-only estático).
- **Ejemplo (síntesis de investigación colaborativa)** — literatura review sobre "climate-resilient urban design"; `SynthesizerAgent` (lidera y sintetiza), `ClimateScienceAgent` y `MaterialSustainabilityAgent` (especialistas). Flujo: (1) **Investigación inicial**: el MaterialSustainabilityAgent produce un reporte destacando beneficios del *hempcrete*. (2) **Inter-agent reflection**: el Synthesizer comparte los summaries; el ClimateScienceAgent analiza la recomendación y reporta que la alta humedad en ciudades costeras podría comprometer la integridad estructural del material. (3) **Temporal re-grounding**: FCoT prompt-ea al MaterialSustainabilityAgent a **revisar su evaluación original**; su razonamiento inicial se corrige formalmente a "Hempcrete is a viable sustainable material, but may degrade under high humidity without specialized treatment" — una revisión directa de una conclusión pasada, no solo una adición. (4) **Granularity control**: el Synthesizer trabaja primero a nivel párrafo para combinar insights, luego ajusta a perspectiva nivel-documento para asegurar una narrativa coherente.
- **Implementación** — vía un prompt template sofisticado que guía al LLM por pasos recursivos y reflexivos: fuerza al modelo a (1) externalizar su *Thought*, (2) hacer un *Self-Correction Check (Objective Function)* — ¿maximiza insight y se alinea con el goal? ¿entra en conflicto con el shared context o el razonamiento previo? ¿introduce redundancia? — con verdict y justificación, y (3) *Action/Revision*: si PASS, indicar la acción; si FAIL, indicar la corrección (que puede revisar un *thought unit previo* — temporal re-grounding). El template incluye el goal original, el shared context con summaries de peers, y los pasos de razonamiento previos.
- **Consecuencias** — *Pros*: **reliability y self-correction** (razonamiento autocorrectivo en tiempo real; los agentes detectan y arreglan sus propias desalineaciones), **explainability** (los checks de autocorrección y el temporal re-grounding crean un trace transparente y auditable: se ve no solo *qué* decidió un agente sino *cómo y por qué cambió de opinión*). *Con*: **complejidad y overhead** (más complejo que el CoT estándar; requiere una capa de orquestación sofisticada para manejar shared context y disparar loops reflexivos → más latencia y costo de tokens).
- **Guía** — requiere una capa de orquestación robusta capaz de manejar memoria compartida, disparar loops reflexivos y manejar updates recursivos. Empezar definiendo una *objective function* clara para la autocorrección. Tener en cuenta la mayor latencia/costo de tokens de los checks recursivos y aplicar el patrón a tareas complejas de alto valor donde correctness y adaptabilidad importan más que la velocidad cruda.
- **Naturaleza** — check **proactivo e interno** (durante el trabajo del agente); complementa al Instruction Fidelity Auditing (reactivo y externo). (FCoT también aparece como framework de razonamiento en el [[09 - Agent-Level Patterns|cap. 9]].)

![[06-fig-6.2.png|880]]
*Figura 6.2 – FCoT internal reasoning loop*

![[06-fig-6.3.png|674]]
*Figura 6.3 – FCoT use case example*

## Persistent Instruction Anchoring

- **Qué es** — usa tags especiales para hacer que las instrucciones críticas se destaquen, evitando que sean olvidadas por agentes downstream. En cadenas jerárquicas largas, la instrucción inicial del orquestador (al principio del contexto) queda empujada al *medio* de la ventana a medida que los subordinados agregan razonamiento y datos — y los LLMs sufren el **"lost in the middle"** (dan menos peso a lo que no está al inicio o al final del prompt).
- **Contexto** — sistema multi-agente jerárquico profundo donde una instrucción inicial baja por una cadena de agentes; al crecer la ventana de contexto, la instrucción original se aleja de las posiciones de alta prioridad (inicio/fin).
- **Problema** — ¿cómo asegurar que las instrucciones y constraints críticas de alto nivel no sean olvidadas o ignoradas por agentes downstream a medida que el contexto se vuelve más largo y desordenado? Sin un mecanismo que contrarreste el *recency bias* del modelo, ocurre un **instruction drift** gradual que desalinea el output final de la intención original.
- **Solución** — mantener las instrucciones críticas *salientes* sin importar su posición, embebiéndolas en **tags semánticamente significativos** (ej. `<CRITICAL_INSTRUCTION>`, `[GOAL]`, `#DO_NOT_FORGET:`). Estos "anchors" se pasan por la cadena y están diseñados para que el LLM los reconozca fácilmente, recordándole los objetivos core en cada paso.
- **Ejemplo (reporting financiero)** — `ReportingSupervisor` (generar resumen Q3 *sin* forward-looking statements ilegales), `DataExtractionAgent`, `SummarizationAgent`. Flujo: (1) **Instrucción + anchoring**: el Supervisor define la tarea con una constraint negativa estricta como ancla: "Generate a summary of Q3 financial performance. PERSISTENT_GOAL: [NO_FORWARD_LOOKING_STATEMENTS]". (2) **Delegación**: pasa la constraint anclada junto con la instrucción al DataExtractionAgent. (3) **Procesamiento/handoff**: el DataExtractionAgent pasa sus hallazgos al SummarizationAgent incluyendo el ancla: "Data: {revenue: 10M, profit: 2M}. PERSISTENT_GOAL: [NO_FORWARD_LOOKING_STATEMENTS]. Please summarize this." (4) **Output final**: el prompt del SummarizationAgent ahora tiene la instrucción crítica al lado de los datos, así que su LLM core es mucho más probable que vea y cumpla la constraint negativa, evitando lenguaje especulativo sobre Q4.
- **Consecuencias** — *Pros*: **reliability** (mejora significativamente el recall de instrucciones upstream, reduce instruction drift), **explainability** (la presencia de instrucciones ancladas en los mensajes agente-a-agente da un trace auditable de cómo se mantuvieron las constraints). *Con*: **prompt overhead** (agrega un overhead menor al prompt y requiere una estructura de templating consistente en todos los agentes para ser efectivo).
- **Guía** — establecer un **formato estandarizado** para los anchors (ej. `PERSISTENT_GOAL: [...]` o `<CONSTRAINT>...</CONSTRAINT>`) y usarlo consistentemente en todos los agentes, para que la instrucción se pueda parsear y re-insertar de forma confiable en cada paso, maximizando su visibilidad para el LLM.

![[06-fig-6.4.png|439]]
*Figura 6.4 – Persistent Instruction Anchoring workflow*

## Shared Epistemic Memory

- **Qué es** — establece una **única fuente de verdad mutable** que existe *fuera* de las ventanas de contexto individuales, asegurando que todo el colectivo comparta el mismo entendimiento del world state. En multi-agente, depender de contextos individuales crea un efecto **"Tower of Babel"**: si un agente sabe que un servidor está caído pero otro cree que sigue activo, sus acciones coordinadas fallan.
- **Contexto** — sistema multi-agente donde los agentes operan con memoria local y reciben tareas distintas; una observación de un agente (output de tool, cambio de estado) puede no estar naturalmente disponible para otro, llevando a decisiones basadas en información fragmentada, incompleta o inconsistente.
- **Problema** — sin una fuente de verdad unificada y evolutiva, los agentes de una jerarquía se desalinean fácilmente: un agente actualiza su entendimiento con un dato nuevo, pero si ese "hecho" no se propaga, otros operan sobre supuestos desactualizados. Tres fuerzas core lo causan:
  - **Fragmented knowledge** — el "worldview" de cada agente se limita a su contexto local.
  - **Lossy communication** — pasar estado por cadenas largas de conversación suele perder matices.
  - **Lack of "ground truth"** — no hay un punto de referencia autoritativo para chequear el estado canónico de la tarea.
- **Solución** — establecer un **"scratchpad" global** o módulo de memoria centralizado que todos los agentes del workflow puedan leer y escribir. Sirve como la fuente de verdad colectiva y autoritativa: todos los agentes, sin importar su posición en la jerarquía, operan desde el mismo conjunto de hechos y supuestos. Es feature clave de arquitecturas como el framework **"chain-of-agents"**, manteniendo coherencia aunque los agentes individuales razonen aislados.
- **Ejemplo (disrupción supply chain)** — `MonitoringAgent` (news feeds), `LogisticsAgent` (tracking de envíos), `CustomerNotificationAgent` (comunicación con clientes). Estado baseline en memoria compartida: `{"shipment_A1": {"status": "On Time"}, "shipment_B2": {"status": "On Time"}}`. Flujo: (1) **Event detection**: el MonitoringAgent detecta una tormenta en un hub clave y escribe `SharedMemory.update({"event_log": ["Storm reported at Port X"]})`. (2) **Impact analysis**: el LogisticsAgent lee la memoria periódicamente, ve el event log, confirma con sus tools que shipment_B2 pasa por Port X, y escribe `{"shipment_B2": {"status": "Delayed", "reason": "Storm at Port X"}}`. (3) **Proactive action**: el CustomerNotificationAgent lee la memoria, ve el status "Delayed" de shipment_B2 y dispara una notificación al cliente afectado, evitando una queja. Sin el patrón, el CustomerNotificationAgent no se enteraría del retraso hasta una intervención humana o hasta perder el deadline.
- **Consecuencias** — *Pros*: **consistency** (reduce drásticamente el semantic drift; todos los agentes con entendimiento sincronizado → comportamiento colectivo coherente), **efficiency** (más eficiente que pasar grandes cantidades de estado por mensajes conversacionales directos; los agentes pulfrom el contexto específico que necesitan cuando lo necesitan). *Cons*: **centralization risk** (la memoria compartida puede volverse single point of failure o cuello de botella si no se diseña para alta disponibilidad y acceso concurrente), **complexity** (manejar estado compartido requiere mecanismos para **race conditions** si múltiples agentes escriben el mismo dato a la vez).
- **Guía (rica — no perder)**:
  - **Backing store**: para producción, evitar diccionarios in-memory simples que desaparecen al reiniciar el proceso. Usar key-value stores persistentes de baja latencia como **Redis** o **Memcached**, que soportan **operaciones atómicas** (esenciales para prevenir race conditions cuando varios agentes actualizan a la vez).
  - **TTL / timestamp**: la información en un sistema agéntico "se pudre" rápido (un hecho sobre el server status de hace 5 minutos puede ser falso ahora). Imponer un schema donde cada entrada de memoria requiera **timestamp** y **source_agent_id**, para que los agentes downstream puedan ponderar la confiabilidad del dato ("este hecho tiene 20 minutos; debería re-verificarlo") en vez de confiar ciegamente.
  - **Tools tipados estrictos**: exponer la memoria vía tools estrictas y tipadas (ej. `update_order_status(id, status)`), NO una tool genérica `write_memory(text)` — esto evita que la memoria compartida se vuelva un vertedero de texto no estructurado e improcesable.

![[06-fig-6.5.png|501]]
*Figura 6.5 – Shared Epistemic Memory Workflow*

## Composición de patrones para confiabilidad sistémica

El valor real se realiza al **componer** los patrones en una defensa multicapa contra instruction drift y misalignment; no son opciones mutuamente excluyentes sino componentes complementarios. Un sistema jerárquico resiliente combina:

- **Shared Epistemic Memory** — capa fundacional de verdad; todos los agentes trabajan desde un set común y sincronizado de hechos y estado.
- **Persistent Instruction Anchoring** — recordatorio constante de la misión; los objetivos core nunca quedan "lost in the middle" sin importar cuántos agentes intervengan.
- **Fractal CoT Embedding** — mecanismo interno y proactivo de autogobernanza; la mejora continua que ocurre *durante* el trabajo del agente, manteniendo su razonamiento alineado.
- **Instruction Fidelity Auditing** — checkpoint externo y reactivo; el gate final de QA que inspecciona el output verificando compliance con las instrucciones originales antes de finalizar o pasar a la siguiente etapa.

**Ejemplo de composición**: un SupervisorAgent escribe primero el goal primario en la Shared Epistemic Memory; delega una sub-tarea a un WorkerAgent pasándole un Persistent Instruction Anchor; el WorkerAgent usa FCoT Embedding para alinear continuamente su razonamiento con el sub-goal; tras producir su output, este pasa a un auditing agent separado que verifica compliance con las instrucciones top-level de la memoria compartida. **Best practice production-grade**: instanciar al menos **dos o tres** de estos patrones concurrentemente para tener safeguards redundantes.

## Citas

> "Autonomy without accountability is a liability."
> "Trust requires transparency"
> "Shared knowledge prevents misalignment"

## Para aplicar

- **Arquitecturar safeguards contra instruction drift** explícitamente en sistemas jerárquicos — no asumir que el goal original sobrevive la cadena de agentes.
- **Combinar checks internos y externos** — FCoT (autocorrección interna y proactiva) + Instruction Fidelity Auditing (verificación externa y reactiva); los sistemas más robustos usan ambos.
- **Auditar selectivamente en los junctures críticos** — no en cada paso (mata la performance); enfocar en los handoffs de mayor riesgo de violación de política.
- **Estandarizar el formato de los anchors** (ej. `PERSISTENT_GOAL: [...]`) y usarlo consistentemente en todos los agentes para que se parsee y re-inserte de forma confiable.
- **Diseñar la Shared Epistemic Memory para producción** — Redis/Memcached con operaciones atómicas (contra race conditions), schema con timestamp + source_agent_id + TTL (la info se pudre), y tools tipadas estrictas en vez de `write_memory(text)`.
- **Definir una objective function clara** para la autocorrección de FCoT y reservarlo para tareas complejas de alto valor (su overhead de latencia/tokens no se justifica para tareas simples).
- **Componer 2-3 patrones concurrentemente** para confiabilidad production-grade (Shared Memory de base + Anchoring + FCoT + Auditing como defensa en capas).

## Conexiones

- [[_Agentic Architectural Patterns for Building Multi-Agent Systems|Mapa del libro]] — el MOC del libro.
- [[09 - Agent-Level Patterns]] — cap. 9: Persistent Instruction Anchoring e Instruction Fidelity Auditing se *referencian desde ahí* como provenientes de este capítulo; FCoT también aparece como framework de razonamiento del Single Agent Baseline.
- [[01 - GenAI in the Enterprise - Landscape, Maturity, and Agent Focus]] — cap. 1: el GenAI Maturity Model (niveles 5/6) al que la Tabla 6.1 mapea estos patrones; *context is king* y *lost in the middle*.
- [[08 - Human-Agent Interaction Patterns]] — cap. 8: la orquestación supervisor→workers y el rol del human-in-the-loop como evaluador high-stakes se conectan con el auditing de este capítulo.
- [[Generator-Evaluator Pattern]] — el patrón generador-evaluador del vault; calca tanto el loop de autocorrección de FCoT como el check del auditor.
- [[Ground Truth]] — la Shared Epistemic Memory ES el "ground truth" del colectivo; conecta con la noción de ground truth del vault.
- [[Orchestrator]] — el SupervisorAgent que delega, ancla y dispara el auditing.
- [[Distributed Lock]] · [[Quorum]] · [[Write-Ahead Log]] — patrones del vault para las race conditions / atomicidad / persistencia que la Shared Epistemic Memory necesita (Redis/Memcached, operaciones atómicas).
- [[Cache-Aside]] · [[Write-Through]] — patrones de acceso a un key-value store compartido (Redis/Memcached), relevantes al backing store de la memoria.
- [[Hallucinations]] · [[Grounding]] — lo que el grounding compartido y el auditing mitigan.
- [[LLM]] — núcleo de razonamiento sobre el que operan FCoT y el auditor (candidato a nota propia).
- **FCoT / Temporal re-grounding / Inter-agent reflectivity / Instruction drift / Lost in the middle / Chain-of-agents** — conceptos sembrados, candidatos a nota propia.
