---
title: "03 - The Spectrum of LLM Adaptation for Agents - RAG to Fine-tuning"
libro: Agentic Architectural Patterns for Building Multi-Agent Systems
autor: Ali Arsanjani
capitulo: 3
created: 2026-06-11
tags:
  - libros/agentic-architectural-patterns
  - type/case-study
  - status/permanent
aliases:
  - The Spectrum of LLM Adaptation for Agents - RAG to Fine-tuning
  - The Spectrum of LLM Adaptation for Agents
---

# 03 - The Spectrum of LLM Adaptation for Agents - RAG to Fine-tuning

> [!info] Capítulo 3 · *Agentic Architectural Patterns for Building Multi-Agent Systems* — Ali Arsanjani
> Cómo adaptar un LLM genérico al rol específico de un agente: el espectro RAG (contexto dinámico) → fine-tuning (especialización profunda) → ICL (flexibilidad on-the-fly) → grounding (confiabilidad). Navegá: [[_Agentic Architectural Patterns for Building Multi-Agent Systems|Mapa del libro]] · anterior [[02 - Agent-Ready LLMs - Selection, Deployment, and Adaptation]].

## Resumen

El cap. 2 estableció el LLM como motor; este cubre el paso siguiente: **el viaje a un agente verdaderamente competente raramente termina con elegir un LLM general-purpose potente**. La ola inicial de GenAI se enfocó en frontier models monolíticos, pero el landscape evoluciona hacia una *sociedad distribuida de agentes inteligentes*. Para desbloquear valor de negocio, los agentes deben operar correcta, confiable y óptimamente dentro de un **contexto enterprise específico**, lo que requiere **adaptación significativa**. Mientras los LLMs pre-entrenados son generalistas por diseño, los agentes enterprise necesitan ser **especialistas** (finanzas, salud, legal — con vocabularios únicos, workflows complejos y data propietaria que los modelos genéricos no entienden out of the box). Otro shift clave: dejar de depender de un único LLM centralizado y pasar a sistemas distribuidos de múltiples agentes, cada uno potenciado por su propio "cerebro" (que puede variar por costo, performance, latencia, calidad, modalidad o arquitectura según el rol) — práctica emergente conocida como *Router-Executor* o *Planner-Worker* (formalizada en el libro como Tool Routing y Supervisor Architecture en el cap. 5).

El capítulo presenta cuatro técnicas de adaptación que forman un **espectro**, y subraya que rara vez se elige una sola: lo correcto es una **combinación pensada** alineada con los requisitos del agente, los recursos disponibles y el nivel de customización necesario. (1) **RAG** — enriquecer dinámicamente el conocimiento del LLM en tiempo de inferencia con info externa actualizada; (2) **Fine-tuning** — modificaciones más profundas y persistentes de los parámetros del modelo (de PEFT a full fine-tuning); (3) **ICL (in-context learning)** — ajustes de comportamiento on-the-fly vía ejemplos en el prompt, sin tocar pesos; y (4) **Grounding** — conectar sistemáticamente los outputs a fuentes verificables (no-negociable para responsible AI). Beneficios de la especialización: enhanced accuracy/relevance (el agente "habla el idioma" de su dominio), improved reliability/trustworthiness (grounding + menos alucinaciones), optimized efficiency (modelos chicos focalizados con menor latencia/costo) y better goal alignment. El capítulo también introduce un **Agentic AI Maturity Model** (expandiendo los niveles 5-6 del modelo del cap. 1) y una **arquitectura agéntica jerárquica** (orchestrator + sub-agents + tools/AgentTools + callbacks) como blueprint concreto.

![[03-fig-3.1.png]]
*Figura 3.1 – An LLM as the brain of agents (image generated with Google Imagen)*

## De LLMs genéricos a agentes especializados

Los LLMs pre-entrenados son **generalistas por diseño**; los agentes enterprise necesitan ser **especialistas**. Adaptar el LLM es el proceso crítico para cerrar la brecha entre potencial amplio y aplicación especializada, transformando el núcleo cognitivo del agente en un experto de su rol. Beneficios directos: **enhanced accuracy/relevance** (habla el idioma del dominio), **improved reliability/trustworthiness** (grounding, menos alucinaciones), **optimized efficiency** (modelos chicos focalizados → mejor performance con menor latencia/costo) y **better goal alignment** (comportamiento que matchea objetivos y constraints operativos). El éxito requiere una **selección estratégica de técnicas** —a menudo una combinación, no un único método—.

![[03-fig-3.2.png]]
*Figura 3.2 – Agent specialization: adapting agents for task-specific execution*

### Agentic AI Maturity Model (Tabla 3.1)

Expande los últimos dos niveles del GenAI Maturity Model del cap. 1 (single- y multi-agent) en su propio espectro de sofisticación, de modelos centralizados simples a redes descentralizadas de agentes colaborando. Enmarca por qué distintos patrones (Multi-Agent Planning, Consensus, Resource Allocation, Trust Modeling) se vuelven esenciales al crecer la complejidad:

| Nivel de madurez | Descripción | Scalability insight | Compliance insight | Patrones/métodos clave |
|---|---|---|---|---|
| **1. Basic agentic systems** | Agentes únicos manejan tareas específicas bien definidas semi-autónomamente, con workflows predefinidos simples y function calls a APIs/tools externas. | Adaptable pero rígido. Workflows relativamente fijos, limita la innovación. | Directo de gestionar (tareas bien definidas), minimiza riesgo de violación de políticas. | Single-agent, single-LLM llamando tools con function calling. |
| **2. Dynamic single-agent workflows** | Un único agente elige dinámicamente de una variedad de tools/APIs pre-seleccionadas según el problema. | Más versátil y eficiente (problemas más complejos eligiendo las tools adecuadas). | Manejable por la pre-selección de tools aprobadas, pero requiere monitoreo cuidadoso al crecer la autonomía. | Single-LLM llamando tools con selección dinámica de tools. |
| **3. Introspective patterns (ReAct y Reflexion)** | Agentes únicos con razonamiento step-by-step y self-reflection (ReAct, Reflexion); aprenden de sus acciones y se autocorrigen vía feedback loops. | Feedback y autocorrección permiten tareas más complejas y mejora en el tiempo → paths de escalabilidad. | Monitoreo real-time y mecanismos correctivos esenciales para alineación con políticas mientras aprenden. | ReAct (Reasoning and Action) y Reflexion. |
| **4. Multi-agent systems** | Múltiples agentes especializados colaboran; cada uno cubre funciones distintas y no superpuestas; su coordinación permite procesamiento paralelo. | Ideal para entornos high-scale (distribuir tareas, paralelizar workflows complejos). | Más complejo de gestionar; se requieren sistemas de monitoreo para que los agentes semi-autónomos colaboren respetando regulaciones. | Multi-agent systems. |
| **5. Advanced coordination con meta-agents** | Se introduce un "meta-agent" que supervisa y coordina a los demás; permite reasignación dinámica de tareas y ajustes de planning real-time. | Adaptabilidad mejorada (escala bien aun en entornos cambiantes por la distribución optimizada del meta-agent). | Los meta-agents actúan de overseers, ayudando a mantener adherencia a políticas ajustando workflows y redistribuyendo. | Meta-agent policy adherence, coordinación avanzada y conflict resolution. |
| **6. Self-correcting agents (feedback para self-learning)** | Sistemas multi-agente avanzados con feedback loops complejos multi-turn; los agentes critican, corrigen y refinan los outputs de otros iterativamente → mejora continua. | Altamente escalable con mejora continua incorporada; evolucionan en tiempo real, muy eficientes para tareas dinámicas. | El más complejo; compliance checks automatizados y acciones auto-correctivas necesarias para mantenerse alineados mientras se adaptan. | Multi-agent learning systems. |

### La granularidad de los agentes

Pregunta de diseño clave: ¿qué tan grande/granular debe ser un agente? No hay una respuesta única — depende del problema. Un agente puede ser una **unidad atómica fine-grained** (ej. uno cuyo único trabajo es chequear niveles de inventario) o un **sistema coarse-grained** que gestiona un proceso compound multi-step (ej. orquestar todo el proceso de order fulfillment). La arquitectura jerárquica (siguiente sección) muestra cómo coexisten y colaboran distintas granularidades.

## Arquitectura agéntica jerárquica para business process automation

Evolución significativa respecto del agente único navegando una lista plana de tools (enfoque monolítico que no escala ni mantiene claridad al crecer la complejidad). Establece un **ecosistema estructurado y observable** que espeja una organización bien gestionada, diseñado para modularidad y resiliencia. En la cima: **orchestrator agents coarse-grained** (como conductores de IA; su responsabilidad no es ejecutar cada paso sino entender el proceso end-to-end y conducir el workflow diseñando y supervisando la colaboración entre agentes especializados). Delegan tareas compound especializadas a un equipo de **sub-agents fine-grained** (cada uno experto en una función: analizar un documento, chequear compliance, acceder a una DB).

![[03-fig-3.3.png]]
*Figura 3.3 – Orchestrator agent architecture*

Ejemplo: el `NewCustomerOnboarding_Agent` (orquestador / "AI conductor") gestiona el onboarding end-to-end; invoca un `KYC_Agent` (know your customer) y un `CreditCheck_Agent`, y para tareas atómicas simples (ej. mandar el welcome email) invoca directamente un `send_email_tool` fine-grained. El orquestador se enfoca en el flujo del proceso, delegando trabajo especializado sin conocer los detalles internos.

### Componentes core: agentes y sus capacidades

Implementación en **Python** (dominancia en el ecosistema AI + OOP para componentes modulares/reusables). Dos aspectos fundamentales por agente:
- **Agent definition** — cada agente es instancia de una clase base `LLMAgent` (propiedades/métodos comunes → consistencia). El core de cada instancia es su **cognición**, potenciada por un LLM asignado ("cerebro") responsable de interpretar el entorno, razonar, planificar y decidir en su contexto.
- **Capability spectrum (de tools simples a AgentTools sofisticados)** — el "toolbox" no es una lista simple sino un *espectro*:
  - **Tools (funciones fine-grained)** — el building block más básico de acción: wrapper directo y stateless alrededor de una única función/API endpoint, para una acción atómica específica. Son los "verbos de acción" del agente. *Ejemplos*: `send_email`, `query_database`, `calculate_amortization`. *Uso*: tareas simples y deterministas que no requieren razonamiento complejo ni proceso multi-step.
  - **AgentTools (agentes compound sofisticados)** — un **sub-agente completo, self-contained y fine-grained empaquetado y expuesto como tool** para que otros agentes lo usen. Puede tener su propio estado interno, workflow multi-step y su propio set de tools. *Ejemplos*: un `CustomerSentimentAnalyzer` que toma texto crudo, usa internamente varias NLP tools y devuelve un sentiment score estructurado; un `DataProfiler` que acepta un dataset, usa tools estadísticas y devuelve un data profile comprehensivo. *Uso*: invocado por un orquestador para delegar una sub-tarea compleja multi-step sin conocer sus detalles internos.

### Estructura jerárquica: orquestadores y especialistas

Deliberadamente **no plana** — jerarquía de control y especialización que espeja una organización efectiva (mayor modularidad, oversight, escalabilidad):
- **Sub-agents (especialistas fine-grained)** — los workers/domain experts; cada uno una instancia `LLMAgent` adaptada para ser experta en una tarea compound o dominio estrecho. Se empaquetan en AgentTools. *Ejemplo*: `KnowYourCustomer_Agent` (único propósito: KYC checks; tiene su workflow interno y usa tools simples como `query_identity_database`, `check_sanctions_list`, `verify_address_api`).
- **Orchestrator agents (coarse-grained)** — managers/process coordinators; su responsabilidad es orquestar sub-agents para completar un proceso de negocio end-to-end (workflows que reflejan directamente un objetivo de negocio de alto nivel). *Ejemplo*: `NewCustomerOnboarding_Agent` como "AI conductor". *Toolbox composition*: consiste principalmente en **AgentTools** (los sub-agents que gestiona), más quizás unas pocas tools simples propias (notificación final, logging).

### Governance y observabilidad vía callbacks

Una arquitectura multi-agente sofisticada vale tanto como nuestra capacidad de gestionarla y entenderla. Para evitar que se vuelva una "black box" opaca, un framework de observabilidad robusto es esencial. (Ej. si la adaptación de un agente falla —un agente alucinando una tool call con parámetros inválidos en una transacción high-stakes— sin tracing granular eso es un *silent failure*; los callbacks dan la visibilidad para detectarlo de inmediato. Las mitigaciones específicas son **Instruction Fidelity Auditing** (cap. 6) para verificación y **Adaptive Retry** (cap. 7) para recovery.) Es la implementación práctica de los principios de **AgentOps** del cap. 2. Callbacks como hooks en los momentos críticos del lifecycle de cada componente:

- **Monitoring del cognitive core (`on_model_start`/`on_model_end`)** — disparan cuando el "cerebro" de cualquier agente consulta su LLM; loguean el prompt exacto, pasos de razonamiento intermedios, la respuesta final y métricas (token counts, latencia). Esencial para auditar decisiones para compliance, diagnosticar alucinaciones (inspeccionando el contexto exacto usado) y gestionar costos. Base de la explicabilidad.
- **Tracking de acciones/interacciones (`on_tool_start`/`on_tool_end`)** — disparan cuando se invoca cualquier tool o AgentTool; crean un **audit trail inmutable** (qué agente llamó qué tool, los input arguments precisos, la data devuelta, errores). Vital para seguridad y debugging: aísla fallos (si el fault está en el razonamiento del agente o en la ejecución de la tool). En multi-agente, da una vista clara de la cadena de delegación orquestador→sub-agente.
- **Oversight de proceso de alto nivel (`on_agent_start`/`on_agent_end`)** — trackean el lifecycle completo de una corrida de un agente individual; loguean el goal inicial, el outcome final (éxito/fallo) y la duración total. Alimentan dashboards de **BPM** (business process monitoring) y son esenciales para trackear/enforcing **SLAs**, conectando la performance técnica directamente con valor de negocio.

**Separación de concerns** resultante: orquestadores poseen el *qué/cuándo* (flujo del proceso de negocio + orquestación); sub-agents poseen el *cómo* (ejecución experta de una tarea específica); tools representan las capacidades atómicas más básicas. El sistema de callbacks integrado es la pieza final crítica: hace al ecosistema no solo potente sino transparente, gobernable y production-ready.

## RAG (contextual enhancement — stage 1)

*Context is king* (cap. 1) vale especialmente para LLMs, y aún más para agentes que deben tomar decisiones bien informadas y ejecutar acciones precisas —dependientes de info actual, relevante y precisa—. **RAG** suministra dinámicamente al LLM core el contexto esencial justo cuando se necesita, en tiempo de inferencia, en vez de operar solo con el conocimiento estático pre-entrenado (que se desactualiza o carece de detalles nicho). Mejora el **factual grounding** y reduce sustancialmente las alucinaciones; permite dar **citas/referencias** a las fuentes (aumenta transparencia y confianza) y mejora la relevancia general de las acciones.

### Ejemplo 1 — customer support agent con RAG (Tabla 3.2)

Escenario: *"My ProWidget X is showing 'Error E404' after the update last night."*
- **Sin RAG (el fracaso del generalista)**: el LLM accede a su conocimiento general; si "ProWidget X" o "Error E404" son nuevos/específicos de un update reciente, no están en su training. Cae en consejo genérico ("Please try restarting your ProWidget X..."). *Outcome*: inútil (el error es por un firmware bug conocido), frustración, mayor tiempo de resolución.
- **Con RAG (el éxito del especialista)**: el LLM reconoce las entidades clave → dispara RAG. **Retrieve**: query `search("ProWidget X Error E404 firmware update June 2025")` contra vector DB de manuales y service bulletins. **Augment**: devuelve el Service Bulletin #SB2025-06-26 (causa: firmware mismatch con server auth; solución: update a patch v1.2.3 vía Settings>Update, o rollback en KB #FW789) y el KB Article #FW789 (pasos de rollback). **Generate**: respuesta específica, empática y accionable (update a v1.2.3 o rollback). *Outcome*: resolución más rápida, mayor satisfacción, construye confianza.

| Aspecto clave | Sin RAG | Con RAG |
|---|---|---|
| Knowledge source | Data estática del modelo pre-entrenado | Knowledge base externa dinámica (service bulletins) |
| Insight del agente | Sin contexto del error/update específico | Entiende que el error es un issue reciente conocido con fix documentado |
| Respuesta generada | Troubleshooting genérico ("restart device") | Solución específica y accionable ("update to patch v1.2.3") |
| Business outcome | Frustración, mayor tiempo de soporte | Resolución rápida, alta satisfacción, mayor confianza |

### Ejemplo 2 — financial analyst agent con RAG (Tabla 3.3)

Escenario: *"Generate a concise summary of current market sentiment regarding CompanyCorp (Ticker: SMCP) following their Q1 earnings release yesterday."* Desafío: timeliness extrema (el sentiment cambia en minutos; un earnings report altera el outlook overnight). Arquitectura típica: retrieval pipeline con vector DB indexando streams real-time.
- **Sin RAG**: conocimiento limitado al último training (meses viejo); produce un resumen genérico y desactualizado ("CompanyCorp is a leading technology company..."). *Outcome*: inútil, riesgo de errores estratégicos.
- **Con RAG**: orquesta queries a múltiples fuentes — financial news APIs (Bloomberg/Reuters), press release DB, stock market data API. **Augment**: Reuters (Q1 EPS $1.25 vs $1.10 esperado, revenue +15% YoY, stock +7% pre-market), earnings highlight (net income $500M, +25% en AI services), market data (precio $150.00, +7.5% desde el cierre). **Generate**: resumen coherente y data-rich, market sentiment "strongly positive". *Outcome*: resumen oportuno y accionable.

![[03-fig-3.4.png]]
*Figura 3.4 – Example flow for RAG with a real-time specialist*

| Aspecto clave | Sin RAG | Con RAG |
|---|---|---|
| Knowledge source | Training data estática y desactualizada | Financial news APIs real-time, press releases, market data feeds |
| Insight del agente | Sin conocimiento de eventos recientes | Sintetiza earnings de ayer, reacción del mercado y noticias |
| Respuesta generada | Descripción genérica e irrelevante | Resumen de mercado data-rich, oportuno y accionable |
| Business outcome | Decisiones mal informadas, pérdida financiera potencial | Planeamiento estratégico informado, decisiones de inversión oportunas |

### Ejemplo 3 — compliance agent con RAG en transaction monitoring (Tabla 3.4)

Escenario: *"A $75,000 wire transfer from an account in Country A to a personal account of Mr. John Doe in Country B."* Stakes altísimos (penalidades regulatorias, pérdidas, daño reputacional); navegar reglas AML jurisdiction-dependent, políticas internas (a veces más estrictas que la regulación pública) y sanctions lists actualizadas.
- **Sin RAG (el generalista riesgoso)**: conoce principios AML generales pero no las regulaciones AML actuales específicas de Country A/B, los matices de política interna, ni si John Doe apareció recién en una sanctions list. Decide con reglas simplistas ("Transaction amount is high ($75,000). Flag for manual review."). *Outcome*: o ineficiente (flaggea demasiadas transacciones benignas) o, peor, non-compliance si se pierde una regulación recién promulgada. (Fig. 3.5 ilustra el "better safe than sorry" que crea cuellos de botella y deja vulnerable a breaches.)

![[03-fig-3.5.png]]
*Figura 3.5 – The risky generalist workflow*

- **Con RAG (el gatekeeper preciso)**: **Retrieve** de fuentes especializadas — regulatory intelligence DB (large outbound transfers de Country A, inbound a Country B), repositorio version-controlled de políticas internas, sanctions screening API. **Augment**: regulación (transfers >$50K USD de Country A a recipients individuales en non-FATF states requieren Form XYZ y enhanced due diligence), política interna (Bank Policy Manual v3.1.2: transfers >$25K a personal accounts en Country B requieren proof of purpose y disparan Level 2 Manager review), sanctions check (John Doe — No active matches). **Generate**: recomendación detallada y grounded — Sanctions: Cleared; External regulation: Breach (>$50K → Form XYZ + EDD); Internal policy: Breach (>$25K → proof of purpose + Level 2 review); Recommendation: Hold transaction, iniciar EDD, request Form XYZ, route a Level 2 Manager queue. *Outcome*: recomendación precisa, compliant, risk-aware, con audit trail.

![[03-fig-3.6.png]]
*Figura 3.6 – Example workflow leveraging a RAG-enabled agent*

| Aspecto clave | Sin RAG | Con RAG |
|---|---|---|
| Knowledge source | Conocimiento general y desactualizado de principios AML | Regulatory DBs real-time, políticas internas version-controlled, sanction-screening APIs |
| Insight del agente | No puede aplicar reglas específicas y actuales al contexto | Identifica y cross-referencia múltiples obligaciones de compliance en capas de distintas fuentes |
| Respuesta generada | Acción simplista (flag for review) | Recomendación detallada, multi-step y auditable |
| Business outcome | Alto riesgo de non-compliance y penalidades | Riesgo de compliance minimizado, proceso auditable, eficiencia operativa |

### El espectro RAG mapeado al GenAI Maturity Model (Tabla 3.5)

RAG no es solo acceder a más data, sino **la data correcta en el momento correcto**. Su sofisticación es un espectro que corresponde directamente al GenAI Maturity Model:

| Nivel RAG | Descripción | Nivel(es) de madurez correspondiente(s) |
|---|---|---|
| **"Vanilla" RAG** | Enfoque directo de augmentation contextual básica desde una única knowledge source. | **Nivel 2 – Contextual enhancement**: aplicación fundacional de RAG, supera limitaciones básicas del modelo dando contexto externo de una única knowledge base. |
| **Advanced RAG** | Métodos más sofisticados para mejorar calidad, relevancia y trustworthiness del contexto recuperado, a menudo de múltiples fuentes. | **Nivel 4 – Grounding and evaluation**: re-ranking, fusion y citing de fuentes alinean con la necesidad de outputs confiables, verificables y bien grounded. |
| **Agentic RAG** | Patrón emergente que usa agentes autónomos para gestionar y orquestar todo el proceso de retrieval, a menudo con colaboración. | **Niveles 5 y 6 – Single y multi-agent systems**: hallmark de arquitecturas agénticas maduras donde agentes especializados colaboran para retrieval sofisticado. |

## Fine-tuning para capacidades agénticas

Método más profundo y persistente de adaptación: **modifica directamente los parámetros internos (pesos) del modelo** vía training adicional en datasets especializados. A diferencia de RAG (que aumenta con info externa en el momento de uso), el fine-tuning **altera la knowledge base inherente y las tendencias de respuesta por defecto** → el modelo es intrínsecamente más apto en ciertas tareas/dominios. Tres outcomes estratégicos:
- **Domain specialization** — entrenar en un corpus domain-specific (legal case files, medical journals, financial regulatory docs) → más fluido en la terminología, mejor entendimiento de entidades clave y sus relaciones, outputs más apropiados/precisos. El agente "habla el idioma" de su entorno.
- **Task-specific skills** — excel en funciones particulares (generar código en un lenguaje específico, resumir en formato estructurado predefinido, extraer named entities, un tipo de diálogo). Ej.: fine-tunear en pares `"user request" -> "correct API tool_call_json"` mejora drásticamente la habilidad de invocar tools con los parámetros correctos.
- **Behavioral alignment** — moldear estilo conversacional (customer service empático vs technical support conciso y directo), seguir instrucciones multi-step complejas con mayor fidelidad, reforzar guidelines éticas/protocolos de safety (ej. fine-tunear para siempre pedir confirmación explícita antes de una acción irreversible).

### El espectro de tuning: PEFT a FFT (Tabla 3.6)

De técnicas simples/baratas (**PEFT**, parameter-efficient fine-tuning) a **FFT** (full fine-tuning, se cambian todos los pesos). FFT actualiza todos o una porción muy sustancial de los parámetros. PEFT (LoRA, adapter tuning, prompt tuning) modifica solo un subconjunto muy pequeño de parámetros existentes o agrega pocos parámetros nuevos entrenables, manteniendo congelada la mayoría del modelo original.

**Datos de fine-tuning**: *unsupervised fine-tuning* (alias *domain-adaptive pretraining*) — continuar el pretraining con un corpus grande de texto no etiquetado del dominio target (aprende vocabulario, matices estilísticos, conocimiento de fondo). *Supervised fine-tuning (SFT)* — entrenar con ejemplos curados y etiquetados (input-output pairs: instrucción→respuesta deseada, pregunta→respuesta correcta); crucial para enseñar tareas específicas, seguir instrucciones diversas, o generar outputs en un formato preciso. Para agentes, SFT enseña a ejecutar acciones correctamente o comunicar según su persona/función.

| Feature/consideración | FFT | PEFT |
|---|---|---|
| Depth of specialization | Alta. Adaptación profunda a dominios, tareas o comportamientos. | Moderada a alta. Efectiva para muchas necesidades, potencialmente menos profunda que FFT en tareas extremadamente complejas/novedosas. |
| Costo computacional (training) | Muy alto. Requiere recursos GPU/TPU y tiempo significativos. | Bajo a moderado. Mucho menos intensivo que FFT. |
| Requisito de dataset | Grande. Típicamente data etiquetada task-specific de alta calidad y sustancial. | Chico a moderado. Buenos resultados con datasets más chicos. |
| Riesgo de catastrophic forgetting | Mayor. Actualizar todos los pesos puede hacer perder capacidades generales. | Menor. Congelar la mayoría de los parámetros base ayuda a retener conocimiento general. |
| Impacto en recursos (múltiples agentes) | Alto. Almacenar/servir muchos modelos grandes FFT es costoso y complejo. | Bajo. Múltiples adaptaciones PEFT (ej. capas LoRA) pueden **compartir el mismo modelo base** → reduce drásticamente el overhead. |
| Impacto en pesos del modelo base | Todos o la mayoría se modifican. | Solo una fracción pequeña se modifica, o se entrenan pocos sets nuevos chicos (adapters, matrices LoRA). |
| Facilidad de implementación/iteración | Más complejo y lento de iterar/experimentar. | Generalmente más rápido y fácil de experimentar por los menores recursos. |
| Use cases típicos (agentes) | Agentes que requieren expertise de dominio profunda o rasgos de comportamiento muy especializados, más allá de PEFT. | Crear agentes especializados diversos (skills, roles, estilos) desde un base LLM común; agentes en entornos resource-constrained. |
| Beneficio primario | Máxima performance y especialización más profunda para una tarea/dominio dado. | Eficiencia, escalabilidad para múltiples especializaciones, retención de capacidades generales. |
| Drawback primario | Alto costo, resource-intensive, riesgo de perder conocimiento general. | Puede no alcanzar el pico absoluto de performance de FFT en algunas especializaciones muy exigentes/matizadas. |
| Técnicas de ejemplo | N/A (reentrena la mayoría/todas las capas) | LoRA, adapter tuning, prefix-tuning, prompt-tuning. |

Para muchas apps agénticas (suites de agentes especializados, adaptación a tareas cambiantes sin costos masivos de reentrenamiento), **PEFT ofrece un balance convincente** de especialización y eficiencia. FFT sigue siendo viable cuando se requiere el nivel absoluto más alto de especialización y los costos se justifican por el rol crítico del agente.

## In-context learning (ICL) para adaptación de agentes

Mientras el fine-tuning da especialización profunda y persistente, los agentes a menudo se benefician de **adaptar su comportamiento más inmediatamente** ante situaciones novedosas o instrucciones específicas dentro de una interacción en curso. **ICL** permite a los LLMs aprender tareas nuevas o modificar respuestas basándose **solo en la info y ejemplos provistos dentro del contexto del prompt actual — sin cambios a los pesos**. Valiosísimo para agentes que se adaptan dinámicamente a escenarios nuevos, preferencias cambiantes o instrucciones evolutivas en tiempo real.

Mecanismo: presentar ejemplos ilustrativos directo en el prompt — *few-shot* (un puñado de demostraciones) o, con context windows grandes, *many-shot*. El LLM infiere el patrón subyacente y lo aplica a inputs nuevos similares. Estructura conceptual del prompt: `System: You are a specialized assistant for [Task Domain]` → pares Example input/output → Current situation input → Agent's LLM output.

### Aplicaciones prácticas de ICL (Tabla 3.7)

| Use case / ventaja | Escenario | Approach con ICL | "Pensamiento" interno del agente |
|---|---|---|---|
| Task demonstration para necesidades ad hoc | Producir output en un formato ad hoc nuevo para el que no fue fine-tuneado (ej. reporte de 3 secciones desde texto no estructurado). | Construye un prompt con pocos ejemplos demostrando la transformación deseada (Raw Feedback → Formatted Summary). | "Este request requiere un formato de output nuevo. Voy a construir un prompt con ejemplos para guiar la generación del LLM en esta instancia." |
| Dynamic style adaptation | Detecta frustración del usuario por cues conversacionales y determina que su tono factual estándar podría escalar la situación. | Aumenta dinámicamente el contexto del LLM con ejemplos de respuestas empáticas/tranquilizadoras para guiar el tono. | "El sentiment es negativo. Necesito ajustar mi estilo. Voy a agregar ejemplos de respuestas empáticas a mi contexto antes de la próxima respuesta." |
| Clarificar tool usage en casos imprevistos | Llamar una tool/API con una combinación rara o compleja de parámetros, con alto riesgo de formato incorrecto. | Incluye un ejemplo completo y exitoso de una invocación compleja similar antes de pedir formatear la nueva. | "Esta invocación es compleja con varios parámetros opcionales. Voy a recuperar un ejemplo exitoso similar y agregarlo al prompt para asegurar el formato preciso." |

Especialmente potente en entornos dinámicos: **handling de tareas novel/evolving** (feature nuevo con doc mínima → feature notes + un par de QA pairs en el prompt), **adaptación a preferencias in-session** (un personal assistant que cambia su estilo de itinerario cuando el usuario provee un ejemplo breve), **comportamiento altamente contingente** (reglas que cambian por venue/nº de invitados → reglas actuales + un sample plan en el contexto). La eficacia de ICL depende de las capacidades del LLM (generalizar desde pocos ejemplos) y, muy importante, del **tamaño del context window** (cap. 2: ventanas grandes → ejemplos más comprehensivos → few/many-shot más robusto).

### Ejemplo end-to-end — Product feedback analyzer con ICL

Agente que analiza reviews extrayendo **sentiment por feature** (no overall). Review del telescopio AstroZoom: *"The magnification on the AstroZoom is incredible, I can see Saturn's rings clearly! However, the tripod is a bit wobbly and feels cheap."*
- **Comportamiento inicial (sin ICL específico)**: output poco estructurado ("Overall sentiment: Mixed. Positive mention of magnification... Negative mention of a wobbly/cheap tripod.") — no separa features de sentiment de forma machine-readable.
- **El agente construye dinámicamente un prompt ICL**: system message ("identify key product features... and the sentiment toward each... output as a JSON list of objects with 'feature' and 'sentiment' keys") + 3 ejemplos few-shot (smartwatch battery/display, phone camera/speaker/setup, e-reader screen/buttons/software) + la review real.
- **Output del LLM (tras ICL)**: JSON estructurado con `magnification (AstroZoom): Positive`, `clarity (seeing Saturn's rings): Positive`, `tripod (wobbliness): Negative`, `tripod (feel/build quality): Negative`.
- **Significancia**: análisis matizado y output estructurado útil, **sin cambios persistentes a los pesos**; "aprendió" la tarea de los ejemplos para esa interacción. El product team puede ingerir el JSON por feature; el próximo review puede usar un prompt distinto con ejemplos distintos.

ICL no da la especialización profunda y persistente del fine-tuning, pero dota a los agentes de una capa crucial de **flexibilidad y responsividad inmediata** — complemento valioso de RAG y fine-tuning.

## Grounding del output del modelo

Sea que el LLM se adapte vía RAG, fine-tuning o ICL, una práctica final es esencial para la confiabilidad: **grounding** = conectar sistemáticamente el output a fuentes de información verificables. Para agentes no es solo best practice sino **requisito crítico**: como sus outputs disparan acciones reales o informan decisiones high-stakes, un fallo de factual accuracy puede tener consecuencias negativas significativas (de frustración del usuario a repercusiones operativas/financieras). Es clave para mitigar **alucinaciones** (outputs plausibles pero incorrectos o fabricados).

### Técnicas de grounding (Tabla 3.8)

| Aspecto de grounding | Descripción y propósito | Ejemplo de implicación para el agente |
|---|---|---|
| **Citing sources y attributions** | El agente da referencias claras al material fuente original (sobre todo si vino vía RAG). Aumenta transparencia, permite verificar a usuarios/auditores, construye confianza y accountability. | Un HR agent, tras responder sobre la política de parental leave, da un link directo a la sección específica del documento de política interna que referenció. |
| **Fact verification y cross-referencing** | Para info crítica, especialmente si fue generada internamente por el LLM (no citada directamente), el agente cross-referencia programáticamente las aserciones contra knowledge bases/DBs/fuentes autoritativas confiables antes de actuar/presentar. Algunas arquitecturas usan tools de fact-checking o sub-agentes dedicados. | Antes de finalizar una config de orden compleja sugerida por su LLM, un sales assistant cross-verifica la compatibilidad de componentes contra una DB de productos interna. |
| **Leveraging confidence scores** | El agente interpreta y actúa sobre los confidence scores que el LLM puede proveer. Según el score, procede, expresa incertidumbre, busca validación adicional o escala a un humano. | Un diagnostic support agent, si su LLM expresa baja confianza (ej. <70%) en un diagnóstico de fallo, lo flaggea para review obligatorio de un técnico senior. |
| **Proactive ambiguity reduction** | El agente asegura que su propio entendimiento de la query/instrucción/estado sea preciso e inequívoco **antes** de generar respuesta o plan; pregunta clarificación si el input es vago. | Un travel planning agent, ante "Book a flight to Springfield soon", responde "¿A qué Springfield te referís y qué fechas considerás 'soon'?" antes de proceder. |

Esto es la base de la fase **Grounding and evaluation** del GenAI Maturity Model. La conclusión del capítulo: desarrollar LLMs verdaderamente *agent-ready* suele involucrar una **combinación pensada** de las cuatro estrategias — RAG para contexto actual, fine-tuning para especializaciones core, ICL para flexibilidad inmediata, y grounding para outputs confiables.

## Citas

> "context is king"
> "While pretrained LLMs possess a vast range of general skills, they are, by design, generalists. In contrast, enterprise agents need to be specialists"
> "RAG isn't just about accessing more data; it's about providing agents with the right data at the right time"
> "Grounding is not just a technical process; it's the cornerstone of building responsible and reliable AI agents."

## Para aplicar

- **Combinar las cuatro estrategias, no elegir una sola** — RAG (contexto actual), fine-tuning (especialización core), ICL (flexibilidad inmediata) y grounding (confiabilidad), alineadas con los requisitos del agente, recursos y nivel de customización.
- **Usar RAG para data dinámica/propietaria/sensible al tiempo** — pipeline con vector DB; mapear el nivel de RAG al de madurez (vanilla → advanced con re-ranking/fusion/citas → agentic RAG). Dar siempre citas a la fuente.
- **Elegir PEFT (LoRA/adapters) por defecto para suites de agentes** — comparte el modelo base entre múltiples adaptaciones, menor costo y catastrophic forgetting; reservar FFT para la especialización más profunda cuando el rol crítico lo justifique. Usar SFT con pares `user request → tool_call_json` para mejorar la invocación de tools.
- **Aprovechar ICL para tareas ad hoc/novel y preferencias in-session** — construir prompts few/many-shot dinámicamente; depende de un context window grande (cap. 2).
- **Hacer grounding obligatorio** — citing de fuentes, fact verification/cross-referencing (incluso con tool o sub-agente de fact-checking), interpretar confidence scores (escalar a humano si baja), y proactive ambiguity reduction (preguntar antes de actuar sobre input vago).
- **Diseñar la arquitectura jerárquica** — orquestadores coarse-grained (qué/cuándo) + sub-agents fine-grained empaquetados como AgentTools (cómo) + tools atómicas; instrumentar con callbacks (`on_model_*`, `on_tool_*`, `on_agent_*`) para observabilidad/governance (AgentOps), feeding de dashboards BPM/SLA.
- **Adoptar las técnicas según el Agentic Maturity Model** — de single-agent con function calling → tool selection dinámica → ReAct/Reflexion → multi-agente → meta-agents → multi-agent learning systems.

## Conexiones

- [[_Agentic Architectural Patterns for Building Multi-Agent Systems|Mapa del libro]] — el MOC del libro.
- [[02 - Agent-Ready LLMs - Selection, Deployment, and Adaptation]] — cap. 2: este capítulo desarrolla la dimensión de *adaptability* (PEFT/LoRA, RAG, ICL) que el cap. 2 introdujo como criterio de selección; cierra el camino RAG→fine-tuning que el cap. 2 apuntaba.
- [[01 - GenAI in the Enterprise - Landscape, Maturity, and Agent Focus]] — cap. 1: *context is king*, el GenAI Maturity Model (Nivel 2 RAG, Nivel 3 tuning, Nivel 4 grounding) que las Tablas 3.1/3.5 expanden; el ejemplo del loan/KYC orchestrator.
- [[06 - Explainability and Compliance Agentic Patterns]] — cap. 6: el *Instruction Fidelity Auditing* que mitiga la adaptación fallida (alucinación de tool call) referenciada aquí.
- [[07 - Robustness and Fault Tolerance Patterns]] — cap. 7: el *Adaptive Retry* para recovery referenciado aquí; prompt injection y grounding como mitigaciones.
- [[09 - Agent-Level Patterns]] — cap. 9: el patrón *Sensing with RAG* y *Structured Reasoning* operacionalizan lo de este capítulo; ReAct reaparece.
- [[_RAG|RAG]] · [[Chunking Strategies]] · [[Hybrid Search]] · [[Reranking]] · [[Enterprise RAG Assistant]] · [[Grounding]] · [[Hallucinations]] — el patrón RAG y su grounding; el *advanced RAG* (re-ranking/fusion) conecta con reranking.
- [[Ground Truth]] · [[Evals]] · [[LLM as Judge]] — grounding y la fase de evaluation; golden datasets para validar.
- [[_MLOps|MLOps]] · [[RLHF]] — fine-tuning (SFT/PEFT/FFT) y el lifecycle; AgentOps via callbacks extiende MLOps.
- [[Orchestrator]] — la arquitectura jerárquica orquestador↔sub-agents (AgentTools).
- [[Function Calling]] · [[Tool Calling]] — function calling como base de los niveles 1-2 del Agentic Maturity Model; SFT para `tool_call_json`.
- [[LLM]] — el sujeto adaptado (candidato a nota propia).
- **PEFT · LoRA · Adapter tuning · Prefix/Prompt-tuning · SFT · FFT · Domain-adaptive pretraining · Catastrophic forgetting · In-context learning (ICL) · Few-shot/Many-shot · ReAct · Reflexion · Agentic RAG · AgentTools · Router-Executor / Planner-Worker** — técnicas/conceptos del capítulo; candidatos a nota propia.
