---
title: 01 - GenAI in the Enterprise - Landscape, Maturity, and Agent Focus
libro: Agentic Architectural Patterns for Building Multi-Agent Systems
autor: Ali Arsanjani
capitulo: 1
created: 2026-06-11
tags:
  - libros/agentic-architectural-patterns
  - type/case-study
  - status/permanent
aliases:
  - GenAI in the Enterprise - Landscape, Maturity, and Agent Focus
updated: 2026-06-11
---

# 01 - GenAI in the Enterprise - Landscape, Maturity, and Agent Focus

> [!info] Capítulo 1 · *Agentic Architectural Patterns for Building Multi-Agent Systems* — Ali Arsanjani
> El marco estratégico de todo el libro: qué es GenAI y por qué *context is king*, la anatomía del agente, el GenAI Maturity Model (Niveles 0–6), el nuevo stack agéntico (function calling/MCP/A2A) y los desafíos que frenan el paso a producción. Navegá: [[_Agentic Architectural Patterns for Building Multi-Agent Systems|Mapa del libro]] · siguiente [[02 - Agent-Ready LLMs - Selection, Deployment, and Adaptation]].

## Resumen

El capítulo arma el andamiaje conceptual de todo el libro. Define **GenAI** (Generative AI) como un campo de la IA que permite a los sistemas **crear contenido nuevo o sintetizado, razonar, entender contexto y recomendar** aprendiendo patrones de datasets enormes —a diferencia de la IA tradicional que principalmente *analiza* información existente, GenAI **excele en producir artefactos novedosos** (marketing copy, código funcional, etc.)—. Reconoce desde el inicio que el potencial es inmenso pero **pasar de conceptos experimentales a sistemas production-grade presenta desafíos significativos** para las empresas: requiere foco estratégico en seguridad, confiabilidad y governance, con guardrails robustos (input validation/sanitization contra ataques) y policy enforcement para compliance. El capítulo traza el camino de diseño inicial a soluciones responsables y production-ready, y deja claro *por qué* la IA agéntica representa un shift pivotal en la tecnología empresarial.

La tesis central, que el autor consagra como **Principle #1: *context is king***, es que sin contexto relevante, oportuno y preciso, hasta el LLM más fluido produce respuestas plausibles pero erróneas (*hallucinations*) o correctas en general pero inaplicables al caso concreto. Lo ilustra con el caso del underwriter hipotecario. El capítulo recorre luego: las aplicaciones de negocio de GenAI (horizontales y verticales), la definición de **sistemas agénticos** (agent-based vs multi-agent), la **anatomía del agente** (Goals · Sense · Reason · Plan · Act · Memory · Coordinate, operando en un loop continuo con feedback), el **GenAI Maturity Model** (Niveles 0–6: datos → prompting → RAG → tuning → grounding/eval → single-agent → multi-agent) como hoja de ruta, el **nuevo stack agéntico** de interoperabilidad (function calling, [[MCP]], [[A2A]]), y los **desafíos para llegar a producción** —estratégicos, de datos, técnicos, de recursos y éticos— ilustrados con el fracaso del *Returns & Order Status Agent*.

## La potencia transformadora de GenAI

GenAI sintetiza capacidades análogas a facetas complejas de la cognición humana, yendo más allá de la computación simple. Cuatro capacidades sintetizadas son la base de su potencial transformador:
- **Crear (creating)** — como la imaginación humana fuel-ea el arte/storytelling/invención, GenAI crea contenido nuevo o sintetizado: texto coherente (marketing copy, descripciones de producto, emails, posts), música, imágenes, código y otros artefactos novedosos basados en patrones aprendidos, no meras copias.
- **Razonar (reasoning)** — sintetiza info de múltiples fuentes/formatos, descubre patrones, hace inferencias lógicas, identifica relaciones significativas (incluso posibles vínculos causales) y construye approaches step-by-step para resolver problemas o responder preguntas sofisticadas.
- **Entender contexto (understanding context)** — ingiere y entiende contexto con profundidad notable, más allá de keyword matching: interpreta matices del lenguaje, considera el historial conversacional, incorpora preferencias del usuario (*user modeling*) e integra conocimiento externo. Crucial para respuestas no solo relevantes sino *apropiadas* a la situación específica.
- **Recomendar (recommending)** — como un advisor experimentado que anticipa necesidades, identifica patrones en comportamiento/datos para sugerir productos, pathways de información o acciones potenciales, personalizando interacciones y apoyando la toma de decisiones.

Para dar vida a estas capacidades, GenAI se apoya en tecnologías subyacentes sofisticadas, sobre todo los **LLMs (large language models)** como núcleo cognitivo: diseñados para entender, procesar y generar texto human-like y otra data compleja.

### Context is king (Principle #1)

La aplicación efectiva de estas capacidades depende críticamente de un elemento indispensable: el **contexto**. Un LLM es como un conversador increíblemente fluido y knowledgeable que cae en medio de una discusión: sin entender el diálogo previo, el tema o los matices de la situación, hasta el más articulado daría contribuciones irrelevantes, incorrectas o sin sentido. A pesar de su vasta data de pretraining, los LLMs **requieren contexto relevante, oportuno y preciso** para outputs útiles, seguros y alineados. Sin contexto suficiente, producen respuestas incorrectas de dos formas: (1) **hallucinations** (suenan plausibles pero son factualmente incorrectas o sin sentido); (2) respuestas correctas en general pero **inaplicables al caso concreto** por faltar detalles contextuales cruciales.

> [!note] **Caso del underwriter hipotecario (DTI)** — Un AI assistant ayuda a un mortgage underwriter que pregunta "What is the maximum allowable debt-to-income ratio for this application?". El LLM, con conocimiento general, responde **43%** — correcto como guía común para muchos *qualified mortgages (QMs)* convencionales en EE.UU. Pero si el contexto no enunciado es una aplicación **FHA (Federal Housing Administration)** para un borrower en Florida buscando financiación de un lender específico (MegaBank USA), el 43% probablemente es incorrecto y engañoso: FHA generalmente permite DTI más altos, hasta **50% o incluso 57%** con ciertos *compensating factors*. Además, MegaBank USA podría tener *lender overlays* internos con un cap más estricto (ej. **48%**), y las regulaciones de Florida agregar otros matices. El máximo DTI verdaderamente correcto depende de la **intersección** de esos factores contextuales: loan program (FHA) + compensating factors del aplicante + políticas del lender (overlays de MegaBank USA) + regulaciones geográficas (Florida). Dar contexto insuficiente o ambiguo es el driver primario de outputs inexactos o engañosos en escenarios reales complejos.

Esto vale especialmente en sistemas agénticos: crear buenos prompts iniciales es solo el comienzo. Para resultados confiables y de alta calidad —sobre todo en tareas multi-step complejas de agentes en entornos enterprise donde accuracy y trustworthiness son primordiales— hay que **arquitectar sistemas que equipen consistentemente el reasoning core del agente (a menudo un LLM) con la info contextual correcta en el momento correcto de su loop operativo**, yendo más allá del conocimiento interno estático (que puede estar desactualizado, incompleto o sin detalles de dominio). Así como el contexto insuficiente lleva a respuestas contextualmente erróneas (alucinaciones) en un Q&A simple, dentro de un agente la mala gestión de contexto puede descarrilar el planning, llevar a acciones incorrectas y socavar los goals.

Aquí los **agentic design patterns** del libro se vuelven esenciales: soluciones estructuradas y repetibles para desafíos comunes, incluido el manejo efectivo de contexto. Patrones como **Task Delegation Framework**, **Collaborative Task Decomposition** o **Iterative Debate for Robust Reasoning** dan blueprints para agentes y multi-agentes que manejan flujos de info complejos, mantienen situational awareness y refinan su entendimiento/planes.

> [!example] **Patrón ejemplo — Task Delegation Framework (Supervisor Architecture)**
> - **Context**: una institución financiera necesita automatizar un proceso multi-step complejo como loan underwriting. Un único agente monolítico tendría problemas para gestionar todas las reglas, fuentes de datos e interacciones requeridas.
> - **Problem**: ¿cómo automatizar este workflow de forma confiable, asegurando que cada paso lo maneje un experto y el proceso global se gestione coherentemente de principio a fin?
> - **Solution**: estructura jerárquica con un agente "supervisor"/"orchestrator" central que actúa de project manager — no hace los checks individuales, sino que recibe la tarea high-level, la descompone y delega las sub-tareas a un equipo de "worker" agents especializados.
> - **Example in action (loan processing)**: (1) un `LoanOrchestratorAgent` (el supervisor) recibe una nueva aplicación; (2) primero delega verificar los documentos a un `DocumentValidationAgent`; (3) una vez validados, delega al `CreditCheckAgent` para traer el historial crediticio; (4) finalmente manda toda la info verificada a un `RiskAssessmentAgent` para un score final.
> - **Outcome**: el orquestador junta los outputs de cada especialista y ensambla el resultado final para decidir. Hace el workflow modular, predecible y más fácil de gobernar (cada agente con responsabilidad claramente definida).

Considerar e implementar estos patrones establece una base más fuerte para **diseñar guardrails implícitos** en la arquitectura (boundaries para el comportamiento del agente → decisiones mejor informadas y acciones más alineadas con el contexto requerido). Además, las interacciones estructuradas (iterative debate, feedback loops) crean oportunidades de **self-correction**: un agente o sistema de agentes puede atrapar inconsistencias o refinar el razonamiento según contexto dinámico o peer review *antes* de tomar acciones erróneas. Por eso un foco significativo será **RAG (retrieval-augmented generation)** —que fetchea dinámicamente info relevante de fuentes externas para informar la respuesta del LLM—, el *grounding* de respuestas en material verificable con citas, y data structures más sofisticadas (databases, knowledge graphs) para contexto más rico y estructurado.

## Aplicaciones de negocio

La versatilidad de GenAI permite aplicarlo a través de funciones de negocio (**horizontales**) y dentro de dominios industriales específicos (**verticales**).

### Horizontales (cross-functional)
- **Marketing y sales** — experiencias hiper-personalizadas a escala (ej. una cruise line sugiriendo dinámicamente actividades/reservas/excursiones según perfiles de pasajeros derivados del comportamiento pasado; incorpora feedback para refinar). Para advertising: entender patrones de compra y relaciones entre productos (knowledge graphs) → estrategias de campaña más matizadas; generar assets creativos (ad copy, materiales) por segmento/plataforma.
- **Customer service** — chatbots/asistentes virtuales 24/7 que manejan inquiries complejas (knowledge bases, backend: order status, returns), reconocen los límites de su conocimiento/autoridad y **escalan inteligentemente a un humano** transfiriendo el contexto; adaptan su estilo de comunicación (narrativa, tono, complejidad) según el usuario.
- **Human resources** — recruitment (analizar resumes vs job descriptions, generar preguntas iniciales), guardrails éticos, onboarding/training personalizado por rol y learning style, knowledge sharing interno (benefits, IT, políticas) con consistencia y sensibilidad al elemento humano.
- **Finance y accounting** — más allá de automatizar análisis y reportes, mejora detección de anomalías/fraude (patrones sutiles de actividad ilícita); enforcement de adherencia a políticas y guardrails financieros (alineación con reglas internas y regulaciones externas).
- **Operations y supply chain** — interpreta outputs de modelos predictivos e inicia acciones: optimizar inventario (ajustar stock orders según forecasts), logística (routing dinámico con data de traffic/weather, dispatch), production line management y predictive maintenance (interpretar alertas, sensor data, generar work orders o reschedule).
- **IT y development** — code generation (boilerplate Python para un API endpoint, SQL desde lenguaje natural), automated debugging (analizar logs/snippets, sugerir root causes/fixes), refactoring/traducción entre lenguajes, generación de test cases.
- **General productivity** — document summarization (papers, contratos legales a plain language, action items de meeting transcripts); enterprise search en lenguaje natural (ej. "What were the main concerns raised by European customers in Q4 feedback?" → respuestas sintetizadas de múltiples documentos, no una lista de links); generación de **synthetic data** (perfiles/transacciones artificiales para entrenar fraud detection sin data real sensible, o balancear datasets de ML).

### Verticales (domain-specific)
- **Healthcare** — acelera drug discovery (analizar simulaciones, proponer candidatos); asiste el diagnóstico médico (explicar scores de pacientes high-risk, resumir análisis de imágenes para review del clínico); genera drafts de documentación clínica (discharge summaries) que cumplen estándares y privacidad (HIPAA); treatment plans personalizados con guardrails de política clínica y ética.
- **Finance** — algorithmic trading (interpretar señales, proponer acciones dentro de risk parameters predefinidos); credit risk assessment (sintetizar reportes que interpreten scores de riesgo + data cualitativa, alineados con políticas de lending y fairness); drafting automatizado de reportes de compliance regulatorio (con verificación humana y guardrails); financial advice personalizado que sigue estrictamente regulaciones de suitability como **Reg BI (SEC's Regulation Best Interest)**.
- **Retail** — recomendaciones hiper-personalizadas y *style bundles* curados según historial/browsing; virtual try-ons sobre avatares personalizados; dynamic pricing en tiempo real (demanda, competidores, inventario); promociones targeted automatizadas por segmento.
- **Manufacturing** — optimiza production scheduling y resource allocation en factorías dinámicas (disponibilidad de máquinas, constraints de material, prioridades); quality control vía visual inspection automatizada (detectar defectos sutiles); *generative design* de partes material-efficient según constraints de performance (load-bearing, peso), apto para additive manufacturing.

## Introducción a los sistemas agénticos

Un foco significativo del desarrollo moderno —y del libro— son los **sistemas agénticos**: un paso hacia apps más autónomas y goal-oriented, apalancando las capacidades core de GenAI de forma más integrada y proactiva. Un **AI agent** es un sistema, a menudo potenciado por LLMs, diseñado para percibir su entorno, tomar decisiones y ejecutar acciones para lograr goals específicos. Características clave: **autonomy, reactivity** (responder al entorno), **proactivity** (tomar iniciativa hacia goals) y potencialmente **social ability** (interactuar con otros agentes). Operan en un loop característico de *sensing, reasoning, planning, acting*.

Categorización amplia:
- **Agent-based systems** — a menudo un único agente que ataca tareas, apalancando sus capacidades para interactuar con sistemas o datos.
- **Multi-agent systems** — múltiples agentes, a menudo especializados, que colaboran, coordinan y comunican para resolver problemas más complejos; enfatizan el **control descentralizado** y la interacción dinámica entre agentes.

## La anatomía de la IA agéntica

![[01-fig-1.1.png|730]]
*Figura 1.1 – Agentic anatomy by Dr. Ali Arsanjani*

El diagrama ilustra una arquitectura agéntica con múltiples agentes colaborando dentro de un entorno.

**Componentes core** — los building blocks primarios son los *agentes* mismos y los *entornos* (business o físico) con los que interactúan. Cada agente opera semi-autónomamente (percibe, razona, decide, actúa). Las interacciones ocurren en contextos digitales (data feeds, APIs, databases) y potencialmente físicos (sensores/actuadores). En multi-agente, una **shared memory** o un **communication protocol** suele actuar de hub de coordinación (intercambiar info, planes, goals).

**Anatomía del agente** (estructura interna):
- **Goals** — los objetivos/outcomes deseados, que pueden actualizarse según feedback o contexto cambiante.
- **Sense (perception)** — cómo el agente recolecta info y data de su entorno (digital o físico: APIs, databases, sensores). Es su mecanismo de adquirir contexto (la situational awareness crítica para todo el razonamiento), reforzando "context is king". Un mecanismo popular para estandarizar este acceso al contexto es el **[[MCP]] (Model Context Protocol)** de Anthropic.
- **Reason (thinking models, cognition)** — la unidad de procesamiento core donde la info sensada se analiza; involucra fuertemente LLMs para interpretar data, entender relaciones (entre goals, percepción y acciones) e inferencia compleja.
- **Plan** — diseñar un curso de acción o secuencia de pasos según los insights razonados y los goals actuales.
- **Act (action)** — ejecutar las acciones planeadas sobre el entorno usando las tools disponibles (llamar APIs, controlar elementos robóticos, generar texto).
- **Memory** — almacena el conocimiento individual, experiencias pasadas, estado interno e info aprendida, dando contexto para la decisión.
- **Coordinate** (opcional; solo multi-agente) — interactuar con otros agentes, a menudo vía shared memory o protocolos, para alinear acciones y colaborar hacia goals colectivos (puede involucrar negociación). Se recomienda específicamente vía el protocolo de interoperabilidad **[[A2A]] (agent-to-agent)**.

> [!note] **Function calling** — la capacidad de *actuar* del agente típicamente se habilita vía **function calling**. Al LLM se le da una lista de tools disponibles; cada tool definida con nombre, una descripción clara de su propósito y un schema estructurado de sus parámetros requeridos. Según la tarea en curso, el reasoning core del LLM decide *cuándo* usar una tool, *cuál* es la más apropiada y *qué parámetros* usar. El modelo genera un output estructurado (ej. un objeto JSON) señalando su intención de llamar esa función con los argumentos extraídos; el código del agente lo recibe, ejecuta la función y realimenta el resultado al LLM para continuar el loop operativo.

El agente opera en un **loop continuo**: sensa el entorno → razona sobre la situación usando su memoria y LLM core → planifica la próxima acción hacia sus goals → actúa sobre el entorno → sensa los resultados (feedback loop) para actualizar su memoria e informar los ciclos siguientes. Esto permite adaptación y aprendizaje en el tiempo.

![[01-fig-1.2.png]]
*Figura 1.2 – The agentic loop*

**Data stores y environment context**:
- **Digital business context** — fuentes de data digitales: unstructured data (texto o imágenes), structured data (databases o knowledge graphs) y vector stores (similarity search eficiente sobre embeddings). Los knowledge graphs son útiles para un entendimiento estructurado y semántico de entidades y relaciones.
- **Physical environment context** — para agentes que interactúan con el mundo real: sensores que dan data (cámaras, IoT) y actuadores para manipulación física (brazos robóticos).

Los agentes efectivos suelen necesitar acceso a múltiples data stores e integrar info de contextos diversos.

### Features arquitectónicos clave (Tabla 1.1)

| Feature | Descripción |
|---|---|
| **Modularity** | Los sistemas se pueden diseñar para agregar/quitar agentes sin un rediseño completo, dando flexibilidad. |
| **Scalability** | Las arquitecturas deben manejar potencialmente muchos agentes, fuentes de datos diversas e interacciones complejas eficientemente. |
| **Adaptability** | El loop sense-reason-act con memoria y feedback permite a los agentes aprender y ajustar el comportamiento en el tiempo. |
| **Multimodal interaction** | Los agentes se pueden diseñar para procesar y actuar sobre info de distintas modalidades (texto, imagen o sensor data). |
| **Collaboration (multi-agente)** | Shared memory o communication protocols facilitan la coordinación, habilitando problem-solving colectivo. |

Construir sistemas agénticos robustos es técnicamente demandante: no es ensamblar componentes sino un viaje de desarrollo donde las capacidades se dominan progresivamente. Atención crítica en cada etapa: diseñar para scalability (crecimiento de nº de agentes y complejidad de data), comunicación inter-agente eficiente y low-latency, técnicas sofisticadas de data processing, knowledge representation avanzada (knowledge graphs → razonamiento más profundo), integración óptima del LLM y mecanismos confiables de tool use. Dominar esto refleja una capacidad madurando en la organización — un proceso explicado por el **GenAI Maturity Model**.

## El GenAI Maturity Model: un camino a los sistemas agénticos

Framework estratégico para que las organizaciones evalúen sus capacidades actuales y tracen un curso. Niveles distintos de capacidad y sofisticación, de actividades fundacionales a sistemas agénticos avanzados. Avanzar a los niveles agénticos/multi-agente (Niveles 5 y 6) suele requerir abrazar nuevos estándares de interoperabilidad.

- **Level 0 – Prepare data (data foundation)** — el punto de partida esencial: adquirir, generar (incl. synthetic data), limpiar, curar, preparar y gobernar la data para IA. Aborda calidad, relevancia, licensing y accesibilidad. Sin una base de datos sólida, los niveles superiores son difíciles.
- **Level 1 – Select model(s) and prompt/serve** — interacción entry-level: seleccionar foundation models pre-entrenados adecuados, diseñar prompts efectivos (*prompt engineering*) y desplegar/servir los modelos (a menudo vía APIs) para tareas básicas (generación, Q&A) basadas en el conocimiento inherente del modelo. El tool use vía function calling básico puede empezar acá y luego evolucionar a agents-as-tools o agentes completos.
- **Level 2 – Contextual enhancement (RAG)** — superar limitaciones del modelo dando contexto externo. RAG es central: fetchea dinámicamente info de knowledge sources externas (documentos corporativos, databases) para augmentar el prompt y mejorar accuracy/relevancia. Paso crucial hacia respuestas más factuales y útiles. Ej.: un chatbot usando RAG para traer los detalles de política más recientes de un knowledge base interno antes de responder.
- **Level 3 – Tuning for specificity (agent-ready LLMs)** — adaptar modelos a necesidades específicas vía fine-tuning con data de dominio/propietaria. Técnicas: de **PEFT** (LoRA o adaptor tuning — modificar solo una parte pequeña) a **FFT (full fine-tuning)** — reentrenar partes más sustanciales. El goal: especializar conocimiento, terminología, estilo o comportamiento para roles agénticos. Ej.: tunear sobre data de ventas enterprise para que el agente entienda jargon específico.
- **Level 4 – Grounding and evaluation** — construir confianza y confiabilidad. Mecanismos para *groundear* outputs en facts verificables (a menudo linkeando las respuestas a la source data recuperada vía RAG → citas). Frameworks y métricas de evaluación robustos para monitorear continuamente performance, accuracy, fairness, bias y safety (alineación con responsible AI). Ej.: un financial analysis agent que da summaries con referencias claras a los reportes financieros usados.
- **Level 5 – Single-agent systems** — la emergencia de la IA agéntica real, aplicando la anatomía descrita. Arquitecturas en torno a un único agente coordinado (a menudo orquestado por un LLM) capaz de razonamiento multi-step, planning, interactuar con tools (invocadas vía function calling o descubiertas vía MCP) y ejecutar tareas autónomamente. Prácticas maduras de **LLMOps/AgentOps** (monitoring, logging, gestión del lifecycle) son esenciales. Ej.: un agente autónomo de travel planning interactuando con APIs de vuelos/hoteles.
- **Level 6 – Multi-agent systems** — el frente actual. Múltiples agentes especializados colaborando, coordinando, comunicando (potencialmente vía A2A) y negociando para problemas que exceden a un solo agente. Requiere arquitecturas sofisticadas de comunicación inter-agente, task allocation, conflict resolution y orquestación. Ej.: un sistema de supply chain optimization donde inventory/logistics/forecasting agents colaboran (vía A2A) para responder dinámicamente a disrupciones.

> [!info] Más adelante el libro expande los Niveles 5 y 6 en un **Agentic AI Maturity Model** separado, para capturar mejor los matices y el espectro de sofisticación.

![[01-fig-1.3.png]]
*Figura 1.3 – Maturity Model levels*

### Tabla 1.2 – Niveles de madurez GenAI

| Nivel | Título/Foco | Descripción y actividades clave |
|---|---|---|
| 0 | Prepare data (data foundation) | Adquirir, generar, limpiar, curar y gobernar data. Foco en calidad, relevancia, licensing, accesibilidad. Prerequisito esencial. |
| 1 | Select model and prompt/serve | Seleccionar modelos pre-entrenados, prompt engineering, servir vía APIs para tareas básicas (generación, Q&A). Tool use básico (function calling). |
| 2 | Contextual enhancement (RAG) | Usar RAG para fetchear contexto externo (documentos, databases) y augmentar prompts, mejorando accuracy/relevancia. Ej.: chatbot recuperando info de política. |
| 3 | Tuning for specificity | Fine-tune (PEFT o FFT) con data de dominio para especializar conocimiento/terminología/comportamiento para roles agénticos. Ej.: tunear para sales jargon. |
| 4 | Grounding and evaluation | Grounding (linkear outputs a fuentes, citas) + evaluación robusta (accuracy, bias, safety) para confianza. Ej.: un agente citando fuentes. |
| 5 | Single-agent systems | Arquitecturar en torno a un agente autónomo con tareas multi-step (reasoning, planning, tool use vía function calling/MCP). Requiere LLMOps/AgentOps. Ej.: travel booking agent / automated SAR reporting agent. |
| 6 | Multi-agent systems | Desplegar múltiples agentes especializados que colaboran, coordinan, comunican (vía A2A) y negocian. *Foundational*: workflows top-down estructurados gestionados por un supervisor central (predictibilidad/auditabilidad). *Advanced*: colaboran, negocian y alcanzan consenso en arquitecturas descentralizadas/swarm para problemas muy dinámicos. Ej.: supply chain agents colaborando. |

Lograr IA agéntica sofisticada (Niveles 5 y 6) depende de construir capacidades en los niveles previos (de data foundations a context enhancement y tuning especializado). Mientras el modelo provee la *hoja de ruta*, **los agentic design patterns son el vehículo para avanzar en ella**: el nivel de madurez alcanzado es resultado directo de qué patrones se eligen implementar. Identificando una capacidad objetivo (ej. un sistema totalmente auditable y seguro para transacciones financieras), se pueden **hacer ingeniería inversa** de los requisitos arquitectónicos → concentrar recursos en dominar los 4-5 patrones críticos para ese nivel de confiabilidad, en vez de un intento difuso de "build AI". Así cada inversión técnica se traduce directamente en un estado validado de madurez y valor de negocio.

## El nuevo stack agéntico

A medida que los sistemas evolucionan de prompts standalone a arquitecturas agénticas, la capacidad de modelos y agentes de interactuar confiablemente con tools y entre sí se vuelve primordial. Tres capas del *AI interoperability stack* emergente: **function calling**, **MCP** y el **A2A protocol**.
- **Function calling** — habilita a los LLMs (en el componente de razonamiento) a disparar tools específicas inteligentemente (ej. `book_flight(destination="Tokyo")`, `get_credit_score`, o ejecutar un script Python local).
- **MCP** — forma estandarizada de **describir, descubrir e invocar tools de forma segura** (weather services, calculadoras, vector search, APIs enterprise como `verify_property_appraisal`) como servicios independientes e interoperables → modularidad.
- **A2A** — protocolo para **delegación de tareas estructurada y colaboración entre agentes distintos**, crucial para multi-agente (Nivel 6).

### MCP vs A2A — modelo mental
- **MCP** = cómo un único agente/LLM se conecta a tools, data y sistemas externos. Darle a la IA acceso a todo lo que necesita para su trabajo (search tools, databases, prompts pre-armados). **Integración vertical**: conectar el agente a sus tools.
- **A2A** = cómo distintos agentes de IA se hablan entre sí, sin importar de qué compañía o framework vengan. Un lenguaje compartido para colaborar, delegar y trabajar como equipo. **Integración horizontal**: conectar agentes con otros agentes.

> [!tip] **Modelo mental simple**: **MCP = AI agent connects to tools** · **A2A = AI agents connect to each other**. Están diseñados para trabajar juntos: (1) un orchestrator agent usa **A2A** para delegar tareas a otros agentes; (2) esos agentes usan **MCP** para acceder a las tools/data que necesitan; (3) los resultados vuelven por **A2A**, completando un workflow colaborativo.

![[01-fig-1.4.png]]
*Figura 1.4 – Distributed multi-agent systems using MCP and A2A*

El acceso remoto de Agent A a Agent B lo facilita el **A2A protocol**, que subraya dos componentes para agent registry y discovery:
- **Agent server** — un endpoint que expone la interfaz A2A del agente.
- **Agent card** — un mecanismo de discovery para advertise las capacidades del agente.

**Agent internals** (comunes a A y B): tres componentes core — el **LLM orchestrator** (motor de razonamiento/coordinación: interpreta prompts, planifica, invoca tools/servicios), el módulo **tools and knowledge** (utilidades locales, plugins, funciones de dominio) y la **memory** (contexto persistente o de sesión: interacciones pasadas, preferencias, info recuperada). Accesibles localmente en el runtime del agente y tightly-coupled para respuestas rápidas y context-aware → forman el "cerebro" self-contained de cada agente. Más dos capas remotas:
- **El MCP server** — conecta el agente a tools/databases/servicios externos vía una API **JSON-RPC** estandarizada (protocolo RPC stateless y liviano que usa JSON). Los agentes interactúan como *clientes*, mandando requests para recuperar info o disparar acciones (buscar documentos, consultar sistemas, ejecutar workflows). Permite inyectar dinámicamente data externa real-time en el razonamiento del LLM → mejora accuracy, grounding y relevancia. Ej.: Agent A usa un MCP server para traer un catálogo de productos de un ERP y generar insights para un sales rep.
- **El agent server** — el endpoint que hace al agente *addressable* vía A2A. Permite recibir tareas de peers, responder con resultados o updates intermedios usando **Server-Sent Events (SSE)**, y soportar comunicación multimodal con format negotiation. Complementado por el **agent card** (capa de discovery con metadata estructurada de las capacidades: descripciones, input requirements → selección dinámica del agente correcto). Los agentes pueden delegar tareas, stream-ear progreso y adaptar formatos de output.

El stack da la base técnica, pero tener el blueprint y construir el edificio son cosas distintas: pasar de PoCs experimentales a sistemas production-grade requiere navegar un landscape complejo de desafíos.

## Desafíos que frenan el GenAI production-grade

Pasar de PoCs a sistemas robustos, confiables y escalables presenta hurdles significativos: requiere superar una red interconectada de desafíos técnicos, operativos, legales y éticos.

- **Estratégicos y organizacionales** — graduar PoCs depende de readiness estratégica y organizacional, no solo factibilidad técnica: demostrar **ROI/business value claro** (muchos pilotos se estancan sin beneficios cuantificables), **stakeholder alignment** amplio (business units, IT, legal, compliance) + plan de integración operativa, **problem-solution fit** (matchear capacidades a necesidades, evitar misapplication), **change management** para preparar al workforce y driving de **user adoption** vía sistemas usables y trustworthy.
- **De datos** — **data governance y quality** son fundacionales: asegurar los derechos legales (data ownership/licensing) del training data; la performance depende de data de alta calidad, relevante y unbiased; mala calidad de datos socava todo (sobre todo para agentes que necesitan perception precisa del entorno); romper **data silos** complejos. **Privacy y compliance** no-negociables al manejar data sensible (GDPR, HIPAA vía anonimización/encriptación).
- **De modelo y técnicos** — **robustness y security**: resiliencia ante adversarial attacks (input validation/sanitization), performance consistente en escenarios reales diversos (domain fit, generalization), gestión del **drift** del modelo/agente. Para agentes con sistemas externos: **secure tool use** (API interactions). **Scalability** de producción (infra cloud, hardware TPUs/GPUs, serving low-latency, data pipelines robustos) + monitoring vía LLMOps/AgentOps maduros. **Integración técnica** con legacy y APIs mantenibles (MCP/A2A ayudan). Minimizar inexactitudes/alucinaciones vía context management y grounding.
- **De recursos** — adquirir/retener expertise técnica (data science, ML engineering, software dev, ops) es difícil; los costos/constraints de recursos de construir/entrenar/fine-tunear/correr modelos grandes o sistemas agénticos complejos pueden ser sustanciales.
- **Éticos y responsible AI** — abordar/mitigar bias (en data, modelos y decisiones del agente) para fairness; establecer **transparency, explainability** (entender *por qué* un agente/modelo decidió algo) y governance frameworks robustos para accountability. La adherencia a estándares legales/éticos no es opcional: es fundamental para soluciones sostenibles y trustworthy.

> [!warning] **Caso de fracaso — Returns & Order Status Agent** — Una gran empresa de e-commerce desarrolló un agente para las dos preguntas de customer service más frecuentes ("What is my order status?" y "How do I return an item?"), buscando soporte instantáneo 24/7.
> - **El PoC**: en un entorno de lab controlado fue un éxito rotundo — entrenado sobre un set curado de FAQs y conectado a una copia limpia y *estática* de la order database; respondía perfecto a "How do I return an item?" o "What is the status of order #12345?". Se aprobó y fast-trackeó a producción.
> - **El fracaso en producción** (empezó a fallar espectacularmente en horas):
>   - **Poor context management**: un cliente cuyo paquete se demoró por una huelga de courier preguntó "My order #54321 was supposed to be here yesterday. Where is it?". El agente, solo con acceso a la order database interna, veía status `Shipped` y respondía repetidamente "Your order #54321 has been shipped" — sin el contexto real-world de la disrupción del courier → frustración extrema.
>   - **Hallucination y faulty design**: un cliente preguntó por la política de returns de un ítem promocional *Final Sale*, ausente del RAG knowledge base. En vez de admitir que no sabía, el LLM core **alucinó** generalizando de la política estándar y le dijo con confianza que era elegible para un **full cash refund** — una promesa que la empresa no podía honrar → escalación enojada y financial write-off.
>   - **Entangled workflow failure**: el agente no estaba bien integrado al workflow de soporte humano. Al fallar, su única función era decir "I cannot help with that. Please contact support" — no transfería el chat, ni daba un ticket number, ni pasaba el historial → forzaba a los clientes a empezar todo de nuevo, destruyendo las ganancias de eficiencia.
> - Tras una semana de quejas escalando, menciones negativas en social media y el alto costo de correcciones manuales, retiraron el agente. **La lección: un PoC exitoso sobre data limpia en un lab NO garantiza un sistema production-ready.** No arquitecturar para el contexto real, los edge cases y la integración con los procesos de negocio existentes convirtió un experimento prometedor en un fracaso costoso.

### Tabla 1.3 – Desafíos para llevar GenAI a producción

| Categoría de desafío | Consideraciones / desafíos específicos |
|---|---|
| Strategic y organizational | Graduar PoCs (demostrar ROI, stakeholder alignment, integración operativa), problem-solution fit, change management, driving user adoption. |
| Data-related | Data governance (ownership, licensing), data quality (relevante, unbiased), romper data silos, privacy y compliance (GDPR, HIPAA, user trust). |
| Model y technical | Robustness y security (adversarial attacks, input validation, drift, secure tool use), scalability (infra, serving), integración técnica (legacy, APIs), monitoring y LLMOps/AgentOps, minimizar alucinaciones/asegurar accuracy (grounding, context). |
| Resource-related | Adquirir/retener expertise técnica y gestionar constraints de costo/recursos (compute, development). |
| Ethical y responsible AI | Mitigar bias, asegurar transparency y explainability, establecer governance frameworks, adherir a compliance y estándares éticos. |

Superar este set multifacético requiere un compromiso estratégico a nivel C-level, inversión significativa en infra y talento, prácticas de governance robustas y un roadmap claro que vaya más allá de la experimentación para embeber GenAI y la IA agéntica como capacidades core de valor en la empresa.

## Citas

> "context is king"
> "MCP = AI agent connects to tools"
> "A2A = AI agents connect to each other"
> "a successful PoC that works on clean data in a lab is not a guarantee of a production-ready system."
> "the level of maturity your organization achieves is a direct result of the specific patterns you choose to implement."

## Para aplicar

- **Arquitectar para el contexto, no solo el prompt** — diseñar el sistema para inyectar el contexto correcto en cada vuelta del loop del agente; los prompts iniciales son solo el comienzo. Usar RAG + grounding con citas y knowledge graphs para contexto estructurado.
- **Hacer ingeniería inversa desde la capacidad objetivo** — definir el nivel de madurez/auditabilidad requerido (ej. transacciones financieras totalmente auditables) y concentrar la inversión en los 4-5 patrones que lo habilitan, en vez de "construir IA" sin foco.
- **Diseñar guardrails implícitos en la arquitectura** — input validation/sanitization, enforcement de políticas, y feedback loops / iterative debate que permitan autocorrección antes de actuar.
- **Combinar MCP + A2A** cuando construyas multi-agent: el orquestador delega tareas vía A2A, cada agente accede a sus tools/datos vía MCP, los resultados vuelven por A2A.
- **No declarar production-ready desde el PoC** — validar contra contexto real, edge cases (políticas excepcionales como *Final Sale*, eventos externos como huelgas) e integración con el workflow humano (transferencia con historial, ticket number) antes de desplegar.
- **Ubicarse en el GenAI Maturity Model** (Niveles 0–6) para planear inversión, skill development e implementación; recordar que los niveles altos (5-6) dependen de construir bien los previos (data → RAG → tuning → grounding).

## Conexiones

- [[_Agentic Architectural Patterns for Building Multi-Agent Systems|Mapa del libro]] — el MOC del libro.
- [[02 - Agent-Ready LLMs - Selection, Deployment, and Adaptation]] — cap. 2 (siguiente): el motor (el LLM "agent-ready") cuya selección/deployment/adaptación apuntala el Nivel 1/3/5 de este modelo de madurez.
- [[04 - Agentic AI Architecture - Components and Interactions]] — cap. 4: desarrolla en detalle la anatomía del agente (esta Fig. 1.1 se reaplica al loan processing) y el stack de interoperabilidad.
- [[05 - Multi-Agent Coordination Patterns]] — cap. 5: el *Task Delegation Framework (Supervisor Architecture)* del ejemplo de loan processing es la semilla de la Supervisor Architecture; conflict resolution y negociación (Nivel 6).
- [[09 - Agent-Level Patterns]] — cap. 9: operacionaliza la anatomía (Sense/Reason/Plan/Act) en patrones concretos; RAG como Nivel 2.
- [[06 - Explainability and Compliance Agentic Patterns]] · [[07 - Robustness and Fault Tolerance Patterns]] — caps. 6/7: los desafíos de producción de este capítulo (governance, robustez ante adversarial attacks, alucinaciones) se vuelven patrones.
- [[Orchestrator]] — el patrón supervisor/orquestador del ejemplo de loan processing.
- [[Function Calling]] · [[Tool Calling]] — el mecanismo de acción del agente (Layer 1 del stack).
- [[Hallucinations]] · [[Grounding]] — el corazón de "context is king" y su mitigación (RAG, grounding).
- [[_RAG|RAG]] · [[Chunking Strategies]] · [[Enterprise RAG Assistant]] — Nivel 2 del modelo de madurez; enriquecimiento contextual.
- [[Server-Sent Events]] — el transporte (SSE) que usa el agent server de A2A.
- [[MCP]] · [[A2A]] — el nuevo stack de interoperabilidad (MCP vertical/intra-org, A2A horizontal/inter-org); candidatos a nota propia.
- [[LLM]] — núcleo cognitivo/de razonamiento del agente (candidato a nota propia).
- **JSON-RPC** · **Agent server / Agent card** · **Reg BI** · **PEFT/LoRA/FFT** — conceptos del capítulo; candidatos a nota propia.
