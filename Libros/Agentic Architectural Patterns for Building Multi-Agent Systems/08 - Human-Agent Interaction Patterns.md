---
title: 08 - Human-Agent Interaction Patterns
libro: Agentic Architectural Patterns for Building Multi-Agent Systems
autor: Ali Arsanjani
capitulo: 8
created: 2026-06-11
tags:
  - libros/agentic-architectural-patterns
  - type/case-study
  - status/permanent
aliases:
  - Human-Agent Interaction Patterns
updated: 2026-06-11
---

# 08 - Human-Agent Interaction Patterns

> [!info] Capítulo 8 · *Agentic Architectural Patterns for Building Multi-Agent Systems* — Ali Arsanjani
> Los patrones que gobiernan la interfaz humano↔agente: cinco patrones (Agent Calls Human, Human Delegates to Agent, Human Calls Agent, Agent Delegates to Agent, Agent Calls Proxy Agent) ordenados como un maturity model de 5 niveles, integrados en 4 tiers arquitectónicos, encadenados en un ejemplo de booking corporativo y medidos con métricas por patrón. Navegá: [[_Agentic Architectural Patterns for Building Multi-Agent Systems|Mapa del libro]] · siguiente [[09 - Agent-Level Patterns]].

## Resumen

Tras los capítulos que establecieron cómo hacer los sistemas agénticos coordinados, compliant y robustos, este se dedica a **la relación más crítica de todas: la del agente con su usuario humano**. Para que los sistemas agénticos pasen de automatización de backend a *augmentar* genuinamente a los knowledge workers, sus interacciones con personas deben ser intuitivas, confiables y efectivas. El enfoque vuelve a ser "map before the journey": antes de detallar cada patrón, primero una guía estratégica de implementación con un **maturity model** que ordena los patrones en un roadmap progresivo, de bots transaccionales simples a partners colaborativos y proactivos.

El capítulo cubre cinco patrones de interacción que abarcan todo el espectro de colaboración humano-agente: desde comandos directos transaccionales hasta delegaciones complejas y de larga duración, más dos patrones internos (agente↔agente, agente↔proxy) que hacen posibles esas experiencias sin cargar al usuario con la complejidad interna. Consistente con el enfoque del [[07 - Robustness and Fault Tolerance Patterns|Capítulo 7]], incluye secciones dedicadas de **pattern chaining** (cómo se encadenan en un workflow real) y de **evaluation metrics** (cómo transformar la "buena UX" subjetiva en datos objetivos y medibles). Mientras los Capítulos 5 y 6 se enfocaron en la lógica interna de coordinación y observabilidad, los Capítulos 7 y 8 se enfocan en los puntos críticos de contacto donde el sistema se encuentra con el mundo real.

La tesis del capítulo: construir confianza es el objetivo último. Las interacciones son diversas (no monolíticas); la escalación a humano es un *feature core, no un fallo*; la colaboración entre agentes debe ser invisible y fluida para el usuario; y todos los patrones, de distintas formas, existen para construir y mantener la confianza del usuario asegurando que el agente entienda la intención, actúe alineado con los goals, dé transparencia cuando hace falta y opere de forma segura en nombre del usuario. La conclusión: aplicando estos patrones se pasa de crear meras herramientas a diseñar verdaderos *colaboradores de IA*.

## Maturity model de interacción humano-agente (Tabla 8.1)

La adopción correcta es **progresiva**: no hace falta (ni se recomienda) implementar todos los patrones de golpe. Se construye una base de interacciones simples y confiables y se agregan capas de autonomía y colaboración a medida que el sistema crece en complejidad y responsabilidad. Cinco niveles, de bots transaccionales a asistentes digitales colaborativos y proactivos:

| Nivel | Capacidades | Patrones habilitados | Resumen |
|---|---|---|---|
| **1. Transactional systems** | Interacciones directas de un solo turno | Human Calls Agent | El sistema actúa como herramienta responsiva: maneja comandos y queries simples y bien definidos con velocidad y precisión. |
| **2. Assisted automation** | Delegación y escalación básicas | Human Delegates to Agent · Agent Calls Human | El sistema toma tareas simples multi-step pero conoce sus límites: escala confiablemente a un humano ante ambigüedad o para aprobación. |
| **3. Collaborative systems** | Workflows multi-agente internos | Agent Delegates to Agent | El sistema resuelve problemas complejos orquestando un equipo de agentes especialistas internos, ocultos al usuario. |
| **4. Secure and interoperable ecosystems** | Interacciones externas seguras | Agent Calls Proxy Agent | El sistema interactúa de forma segura y confiable con sistemas de terceros, habilitando colaboración cross-enterprise. |
| **5. Proactive and personalized partners** | Colaboración anticipatoria, context-aware | Todos los patrones + memoria de largo plazo | El sistema evoluciona de herramienta a *partner*: aprende preferencias del usuario y asiste proactivamente con goals complejos. |

El maturity model da el **cuándo** (la secuencia de adopción). La arquitectura de integración da el **dónde** (cómo encajan en las capas funcionales del sistema).

## Arquitectura de integración: los 4 tiers

Cómo se organizan los patrones en tiers funcionales dentro de una aplicación completa, con clara separación de concerns desde la UI hasta el manejo seguro de comunicaciones externas:

| Tier | Rol | Patrones habilitados |
|---|---|---|
| **UI tier** | Punto de contacto directo con el humano; todas las interacciones se inician acá. | Human Calls Agent (comandos directos) · Human Delegates to Agent (goals complejos) |
| **Orchestration / primary agent tier** | Aloja el agente primario con el que interactúa el usuario (ej. `TravelPlannerAgent`, `ResearchAssistantAgent`); planificación de alto nivel, descomposición de goals, manejo del workflow global. | Recibe tareas delegadas e inicia Agent Delegates to Agent hacia el tier especialista; maneja escalaciones vía Agent Calls Human. |
| **Specialist / worker agent tier** | Agentes funcionales fine-grained que ejecutan la lógica de negocio core (ej. `FlightBookingAgent`, `NewsSentimentAgent`). | Ejecuta tareas recibidas vía Agent Delegates to Agent; cuando hace falta inicia requests externos a través del security tier. |
| **Security / proxy tier** | Capa especializada y aislada, responsable de TODA comunicación externa; el único tier con credenciales y lógica para interactuar con el mundo exterior. | Aloja los proxy agents que actúan de gateway seguro a APIs de terceros vía Agent Calls Proxy Agent. |

## Pattern chaining en la práctica: booking de viaje corporativo

Ejemplo que ilustra cómo los patrones se encadenan en un workflow real para cumplir un goal complejo, escalando a humano cuando el sistema llega a sus límites operativos. Flujo del booking:

1. **Human Delegates to Agent** — el usuario le dice al `TravelOrchestratorAgent`: *"Book me a trip to the New York office next week for the Q3 planning meeting. Get me a refundable flight and a room at our corporate preferred hotel."*
2. **Agent Delegates to Agent** — el `TravelOrchestratorAgent` descompone el goal y delega sub-tareas: manda *"Find refundable flights to JFK for next week"* a su `FlightBookingAgent` especialista, y *"Find rooms at corporate hotel in NYC for next week"* a su `HotelBookingAgent`.
3. **Agent Calls Proxy Agent** — el `FlightBookingAgent` debe usar el portal seguro de viajes de la empresa (Concur). Llama a un `ConcurProxyAgent` pasándole los criterios de vuelo; el proxy es el único agente con las API keys para interactuar de forma segura con el servicio Concur.
4. **Agent Calls Human** — el `HotelBookingAgent` descubre que el hotel preferido está sold out. Identifica dos hoteles aprobados alternativos pero no puede decidir solo. Dispara escalación: *"The corporate hotel is unavailable. Would you prefer Hotel A (closer to the office) or Hotel B (better amenities)?"* El orquestador pausa el workflow.
5. **Human Calls Agent** — el usuario recibe la notificación y responde: *"Book Hotel A."* Es un comando directo y transaccional que resuelve la ambigüedad.
6. **Cierre** — el `TravelOrchestratorAgent` recibe la decisión del humano, le indica al `HotelBookingAgent` que proceda, junta las confirmaciones finales de todos los especialistas y presenta un itinerario completo al usuario.

## Agent Calls Human (Human-in-the-Loop Escalation)

- **Qué es** — mecanismo estructurado para que un agente, por diseño o necesidad, pause su operación y pida ayuda humana: cuando su confianza cae bajo un umbral crítico, ante datos altamente ambiguos, o en decisiones high-stakes que la política de la empresa exige que apruebe un humano. Define un *human-in-the-loop checkpoint*.
- **Contexto** — un agente autónomo, durante su operación, encuentra una situación que no puede resolver solo. El sistema necesita maximizar automatización para ser eficiente, pero también permitir supervisión humana para garantizar seguridad y manejar edge cases fuera de las capacidades del agente.
- **Problema** — ¿cómo puede un agente pausar con elegancia su operación autónoma y escalar a un humano? Una escalación mal manejada puede ser disruptiva, dar contexto insuficiente para decidir, o no capturar bien la respuesta del humano, socavando todo el workflow.
- **Solución** — cuando el agente identifica una situación que requiere intervención humana, **empaqueta el estado actual y todo el contexto relevante en un request estructurado**. Ese request se rutea a un operador humano vía una UI dedicada o una task queue. El workflow del agente queda **pausado** hasta que el humano provee una decisión, que se realimenta al sistema permitiendo al agente retomar la tarea con la nueva información humano-validada.
- **Ejemplo (loan application ambiguity)** — `LoanApprovalAgent` con goal de procesar aplicaciones autónomamente salvo que la confianza caiga **bajo 95%**; underwriter humano revisa casos ambiguos escalados. Flujo: (1) **Trigger** — el agente verifica OK income y credit score, pero al analizar el property appraisal detecta una discrepancia significativa entre los pies cuadrados del appraisal (**1.800 sq. ft**) y los del tax record (**1.500 sq. ft**), y su confianza cae. (2) **Package context** — crea un "review package" con applicant ID, links a los documentos en conflicto, y el summary *Discrepancy found in property square footage between appraisal and tax records. Human review required to validate property value.* (3) **Escalate** — llama a un `HumanReviewTool` que empuja el package al dashboard del underwriter. (4) **Pause** — el workflow de esa aplicación queda pausado. (5) **Human decision** — el underwriter revisa, determina que fue un error clerical en el tax record, y decide vía UI: *"VALIDATE_APPRAISAL"*. (6) **Resume** — el agente recibe la decisión estructurada, actualiza su estado interno con el override humano y sigue con el siguiente paso de aprobación. (En código: `CONFIDENCE_THRESHOLD = 0.95`, un `HumanReviewSystem.request_decision(package)` que en un sistema real haría push a una UI y esperaría un callback.)
- **Consecuencias** — *Pros*: **Safety y trust** (fundamental para sistemas seguros y confiables; vetea decisiones críticas o ambiguas con un humano, reduciendo el riesgo de errores automatizados costosos) y **manejo de edge cases** (mecanismo robusto para situaciones novedosas no entrenadas). *Con*: **bottleneck** — el humano en el loop puede volverse cuello de botella de performance; la velocidad global queda limitada por la disponibilidad y responsiveness de los operadores humanos.
- **Guía** — diseñar la interfaz human-facing con cuidado: que presente el contexto de forma concisa y dé una forma **estructurada** de input (botones, forms) para minimizar ambigüedad. Asegurar un mecanismo robusto para gestionar el estado "pausado" del agente, incluyendo **timeouts** y **default actions** si el humano no responde dentro de cierto período.

![[08-fig-8.1.png]]
*Figura 8.1 – Agent Calls Human escalation flow*

![[08-fig-8.2.png]]
*Figura 8.2 – Agent Calls Human escalation flow (cont.)*

## Human Delegates to Agent

- **Qué es** — captura la esencia de la IA agéntica: en vez de dar instrucciones paso a paso, el usuario **delega un objetivo de alto nivel, a menudo ambiguo**, al agente. Marca un shift fundamental: de command-and-control directo a una *partnership* donde el humano fija la dirección estratégica y el agente maneja la ejecución táctica. Es el patrón que más se parece a la visión popular de un "personal assistant" / "copilot" / "co-scientist".
- **Nota — los dos lados de la ambigüedad (delegación vs escalación)** — Human Delegates to Agent y Agent Calls Human **no se contradicen, se complementan**: *Human Delegates to Agent (ambigüedad estratégica)* — el humano da un goal "fuzzy" de alto nivel (ej. *Generate a competitor report*) y el trabajo primario del agente es **resolver esa ambigüedad** creando un plan concreto paso a paso. *Agent Calls Human (ambigüedad táctica)* — es la red de seguridad; se dispara cuando el agente, ejecutando su plan, encuentra un problema táctico nuevo e imprevisto que no puede o no debe resolver solo (ej. *los datos de pricing del competidor están protegidos con password, ¿los salteo o espero?*). En corto: **un agente capaz resuelve la ambigüedad estratégica inicial; un agente seguro sabe cuándo escalar la ambigüedad táctica nueva.**
- **Contexto** — un usuario tiene un goal de alto nivel o una tarea compleja multi-step que no quiere o no puede ejecutar manualmente; quiere descargar todo el proceso a un sistema autónomo capaz.
- **Problema** — ¿cómo entregar efectivamente un goal complejo a un agente? El sistema debe capturar con precisión la intención del usuario desde una instrucción de alto nivel, operar autónomamente por un período extendido, y mantenerse alineado con el goal original sin guía humana constante.
- **Solución** — estructura la interacción como un **handoff claro**: el usuario provee un objetivo de alto nivel; el agente entra en un loop autónomo donde primero **crea un plan detallado** descomponiéndolo en sub-tareas ejecutables, y luego **lo ejecuta** usando sus tools y razonamiento. Puede dar updates periódicos o pedir clarificación ante una ambigüedad irrecuperable, pero por lo demás opera independientemente hasta lograr el goal.
- **Ejemplo (market research)** — un marketing manager delega a un `MarketAnalysisAgent`. **Goal delegado**: *"Generate a report on the top three competitors for our new 'ProWidget X' in the European market. Focus on their pricing, key features, and recent customer sentiment. I need a draft by tomorrow."* **Plan generado por el agente**: (1) *Identify competitors* — web search de top competidores de 'ProWidget X' en Europa; (2) *Get pricing* — para cada competidor, financial data API para pricing; (3) *Analyze sentiment* — product review aggregator para resumir sentiment de los últimos 6 meses; (4) *Synthesize report* — consolidar todo en un documento estructurado; (5) *Deliver* — mandar el reporte final por email. **Ejecución autónoma**: ejecuta cada sub-tarea secuencialmente, guardando los hallazgos intermedios en su memoria sin input humano adicional. **Completion**: una vez drafteado, envía email al manager con el documento adjunto. (En código, el agente actúa de orquestador: `_llm_create_plan(goal)` genera un plan estructurado, luego itera invocando dinámicamente los tools — `WebSearchTool`, `ReviewAggregatorTool` — y cierra con `_llm_synthesize_report` + `send_email`.)
- **Consecuencias** — *Pros*: **eficiencia** (extremadamente potente para productividad; descarga tareas complejas y time-consuming a un sistema autónomo) y **capacidad** (resuelve problemas demasiado grandes para una interacción single-prompt). *Con*: **riesgo de misalignment** — el riesgo primario es que el agente malinterprete el goal inicial de alto nivel y gaste recursos significativos ejecutando un plan no alineado con la verdadera intención del usuario.
- **Guía** — para mitigar el misalignment, implementar un **Plan Confirmation step**: tras generar el plan inicial (paso 2 del ejemplo), el agente lo presenta al usuario para una aprobación rápida "go/no-go" antes de empezar la ejecución autónoma. Ese pequeño checkpoint asegura que la interpretación del goal es correcta sin obligar al usuario a supervisar cada paso.

![[08-fig-8.3.png]]
*Figura 8.3 – Human Delegates to Agent workflow*

## Human Calls Agent

- **Qué es** — estructura el ciclo request-response fundamental para queries **transaccionales y directas**: el usuario necesita una pieza específica de información o quiere una sola acción bien definida ejecutada de inmediato, y espera una respuesta rápida, precisa y concisa. Es la base para chatbots, voice assistants y otras herramientas de interacción directa, donde el rol primario del agente es ser una interfaz directa a una tool o pieza de información.
- **Contexto** — interacción transaccional; el usuario espera una respuesta rápida y precisa sin pasos conversacionales innecesarios.
- **Problema** — ¿cómo dar una respuesta directa, responsiva y precisa a una query transaccional? El agente debe entender rápido el comando directo, usar la tool apropiada para obtener la info o ejecutar la acción, y devolver el resultado de forma concisa.
- **Solución** — trata la query del usuario como una **invocación directa**. La lógica primaria del agente: clasificar la intención del usuario, seleccionar la *única mejor tool* para esa intención, ejecutarla con los parámetros extraídos de la query, y devolver el resultado, a menudo con mínimo filler conversacional.
- **Ejemplo (order status)** — un usuario interactúa con un retail chatbot. (1) **Comando directo**: *"Where is my order #ABC-123?"* (2) **Intent classification & tool selection**: el LLM core reconoce esto como una "order status query" e identifica `getOrderStatusTool` como la tool apropiada. (3) **Parameter extraction**: extrae el order ID `ABC-123` como parámetro necesario. (4) **Tool execution**: llama `getOrderStatusTool(order_id="ABC-123")`; la tool consulta la shipping database y devuelve `{"status": "In Transit", "location": "Denver, CO", "estimated_delivery": "2025-09-22"}`. (5) **Response generation**: el LLM recibe el output y lo formatea en una respuesta clara y legible. (6) **Return to user**: *"Your order #ABC-123 is currently in transit in Denver, CO, with an estimated delivery date of September 22, 2025."* (En código, el `RetailBotAgent` usa el LLM principalmente como **router inteligente**: identifica intención, extrae parámetros e invoca la `OrderStatusTool` para traer datos real-time del backend.)
- **Consecuencias** — *Pros*: **velocidad y eficiencia** (optimizado para velocidad; ideal para asistentes transaccionales muy responsivos) y **simplicidad** (lógica directa; uno de los patrones agénticos más fáciles de implementar y debuggear). *Con*: **scope limitado** — no apto para tareas complejas multi-step ni goals long-running; brilla en interacciones single-purpose bien definidas.
- **Guía** — la clave es la **definición robusta de tools**: bien documentadas con descripciones claras y parámetros **strongly typed** (cada parámetro define explícitamente su tipo: string, integer, Boolean). Esa metadata es lo que permite al LLM seleccionar la tool correcta y extraer los argumentos necesarios desde una query conversacional. Conecta con [[Function Calling]] / [[Tool Calling]].

![[08-fig-8.4.png]]
*Figura 8.4 – Human Calls Agent sequence*

## Agent Delegates to Agent

- **Qué es** — implementa una estructura jerárquica o colaborativa: un agente primario "supervisor" actúa de project manager, descompone el goal del usuario en sub-tareas y delega cada una al "worker" especializado apropiado. Permite combinar las skills de múltiples expertos para resolver problemas mucho más complejos que los que cualquier agente único podría manejar solo. (Orquestar un equipo de agentes internos = un *agent mesh*.)
- **Contexto** — un agente primario (ej. un [[Orchestrator|orquestador]]) recibió del usuario una tarea compleja que requiere múltiples skills o dominios de conocimiento especializados.
- **Problema** — ¿cómo cumplir un request complejo sin sobrecargar a un único agente "generalista" ni obligar al usuario a interactuar directamente con múltiples agentes especializados?
- **Solución** — estructura **supervisor (orchestrator)**: el usuario interactúa con un único agente primario que analiza el request y actúa de "project manager", descomponiendo el goal global en sub-tareas. Delega cada una al "worker" especializado con las tools y conocimiento específicos. El primario **junta los resultados** de los workers y sintetiza la respuesta final, haciendo la colaboración interna **invisible** al usuario.
- **Ejemplo (comprehensive financial analysis)** — un analista delega a su `ResearchAssistant`. **Goal**: *"Give me a full workup on CompanyCorp. I need a summary of their last earnings call, a technical analysis of their stock chart, and a check for any recent negative news."* **Plan de descomposición**: sub-tarea 1 — resumir el transcript del Q1 earnings call; sub-tarea 2 — analizar el stock chart de 3 meses por support/resistance; sub-tarea 3 — escanear news de los últimos 7 días por sentiment negativo. **Delegación agente-a-agente**: manda la summarization al `EarningsCallSummarizerAgent`, el chart analysis al `TechnicalChartAgent`, y el news scan al `NewsSentimentAgent`. **Ejecución especialista**: cada uno hace su tarea con sus tools dedicadas y devuelve el resultado al orquestador. **Síntesis y respuesta**: el `ResearchAssistant` junta las piezas, las sintetiza en un único reporte coherente y lo presenta al analista. (En código, setup multi-agente jerárquico con `asyncio`: el `ResearchAssistantAgent` delega y usa `await asyncio.gather(...)` para correr a los 3 especialistas **en paralelo** y luego `_llm_synthesize` para el reporte final.)
- **Consecuencias** — *Pros*: **modularidad y especialización** (cada agente es experto en su dominio y se desarrolla, testea y actualiza independientemente) y **enhanced capability** (combinando skills se resuelven problemas mucho más complejos que con un único agente). *Con*: **orchestration overhead** — la performance y confiabilidad dependen fuertemente del orquestador; diseñar la lógica de descomposición, delegación y síntesis suma complejidad y puede introducir latencia.
- **Guía** — la **capacidad de planning del orquestador es el componente más crítico**. Para workflows simples y predecibles, el plan de descomposición puede ser una secuencia estática predefinida de llamadas a agentes. Para tareas complejas y dinámicas, el orquestador mismo puede necesitar usar un LLM para generar un plan multi-step (como en el ejemplo).

![[08-fig-8.5.png]]
*Figura 8.5 – Agent Delegates to Agent architecture*

## Agent Calls Proxy Agent

- **Qué es** — introduce un intermediario especializado, el **proxy**, que actúa de gateway seguro y estandarizado a un sistema externo (API de un partner third-party, base de datos sensible). Desacopla la lógica core del agente de las complejidades de las integraciones externas y **centraliza la enforcement de seguridad**. Dar al agente primario credenciales directas a cada sistema externo es un riesgo de seguridad mayor y una pesadilla de integración.
- **Contexto** — un agente, cumpliendo un request, necesita interactuar con un sistema externo. El acceso directo desde el primario no está permitido por políticas de seguridad, network boundaries, o el deseo de abstraer la complejidad del sistema externo.
- **Problema** — ¿cómo interactúa un agente con un sistema externo de forma **segura y confiable**, sin acoplarse fuertemente a él y sin comprometer la seguridad?
- **Solución** — el primario **no llama al sistema externo directamente**; manda un request interno simple al proxy agent. El proxy es el único componente con las credenciales y la lógica para comunicarse con el sistema externo: **traduce** el request interno al formato específico del sistema externo, maneja la interacción segura, y **traduce de vuelta** la respuesta (a menudo compleja) a un formato simple y estandarizado para el primario.
- **Ejemplo (cross-enterprise loyalty program)** — un usuario reserva un vuelo con un `AirlineBookingAgent` y quiere aplicar un descuento del programa de fidelidad de una cadena hotelera partner. Roles: **primario** `AirlineBookingAgent` (maneja el workflow de booking), **proxy** `HotelBonanzaProxyAgent` (maneja seguramente toda comunicación con la HotelBonanza API), **sistema externo** la HotelBonanza partner API. Flujo: (1) **User request**: *"Book the flight to London and apply my 'HotelBonanza' loyalty discount."* (2) **Acción del primario**: el `AirlineBookingAgent` sabe que NO tiene permitido acceder al sistema HotelBonanza directamente. (3) **Call to proxy**: llama al `HotelBonanzaProxyAgent` con un request interno simple `{"action": "validate_discount", "user_id": "user123", "loyalty_code": "HB-XYZ"}`. (4) **Proxy execution**: el proxy —único con las secret API keys— formatea la API call específica requerida por el sistema externo y la manda seguramente. (5) **External response**: la HotelBonanza API devuelve un objeto JSON complejo confirmando el descuento. (6) **Proxy translation**: el proxy parsea la respuesta, extrae la info esencial (el discount code de un solo uso) y devuelve al primario una respuesta simple estandarizada `{"status": "success", "discount_code": "APPLIED123"}`. (7) **Task completion**: el `AirlineBookingAgent` recibe esa respuesta simple y aplica el código al booking, completando el request **sin haber manejado nunca credenciales externas ni lógica de API compleja**. (En código, el `HotelBonanzaProxyAgent` reside en un contexto seguro y aislado, es el único que carga la API key, y traduce internal request ↔ external API format vía `http_post` con header `Authorization: Bearer`.)
- **Consecuencias** — *Pros*: **seguridad** (centraliza y aísla el acceso a sistemas externos; los primarios nunca manejan credenciales sensibles, reduciendo la attack surface) y **decoupling/maintainability** (si la API del partner cambia, solo se actualiza el proxy; los primarios quedan intactos). *Con*: **latencia** — introduce un "hop" extra en la cadena de comunicación; menos apto para interacciones que requieren respuestas de latencia extremadamente baja.
- **Guía** — usar este patrón para **enforcement de una frontera de seguridad clara** entre el sistema agéntico interno y el mundo exterior. El proxy debe ser el único componente con el acceso de red y las credenciales para alcanzar un servicio externo específico. Asegurar que el protocolo de comunicación interno entre primario y proxy sea simple y estandarizado, creando efectivamente una **API interna** que abstrae la complejidad externa.

![[08-fig-8.6.png]]
*Figura 8.6 – Agent Calls Proxy Agent pattern for secure interaction*

## Métricas de evaluación por patrón (Tabla 8.2)

En un sistema production-grade la UX no puede ser cuestión de opinión: hay que medirla para cuantificar el valor, diagnosticar debilidades y justificar la inversión continua. Métricas de muestra e instrumentación por patrón:

| Patrón | Métrica | Instrumentación |
|---|---|---|
| Agent Calls Human | Escalation rate / resolution time | Loguear cada evento de escalación. Medir el tiempo desde la escalación hasta la respuesta humana y la reanudación posterior de la tarea. |
| Human Delegates to Agent | Task success rate / user satisfaction | Trackear el completion rate end-to-end de los goals delegados. Seguir con una encuesta simple al usuario (CSAT/NPS). |
| Human Calls Agent | First-contact resolution rate / average response time | Medir el porcentaje de queries resueltas en un solo turno. Trackear la latencia end-to-end desde el input del usuario hasta la respuesta final. |
| Agent Delegates to Agent | Orchestration overhead / sub-task failure rate | Loguear timestamps de cada delegación inter-agente y su respuesta para medir la latencia agregada. Trackear errores devueltos por los agentes especialistas. |
| Agent Calls Proxy Agent | External API error rate / security incidents | Monitorear logs del proxy por API calls fallidas o con timeout. Implementar security monitoring en el entorno aislado del proxy. |

## Citas

> "for agentic systems to move beyond backend automation and truly augment human knowledge workers, their interactions with people must be intuitive, trustworthy, and effective."
> "A capable agent can resolve the user's initial strategic ambiguity. A safe agent knows when to escalate new tactical ambiguity."
> "Escalation is a core feature, not a failure"
> "we move beyond creating mere tools and begin to design true AI collaborators."

## Para aplicar

- **Adoptar los patrones por niveles, no todos de golpe** — seguir el maturity model: Nivel 1 transaccional (Human Calls Agent) → Nivel 2 delegación+escalación (Human Delegates to Agent · Agent Calls Human) → Nivel 3 colaboración interna (Agent Delegates to Agent) → Nivel 4 ecosistemas seguros (Agent Calls Proxy Agent) → Nivel 5 partner proactivo (todos + memoria de largo plazo).
- **Organizar el sistema en 4 tiers** — UI tier (Human Calls/Delegates), orchestration/primary tier (descomposición + escalación), specialist/worker tier (lógica de negocio), security/proxy tier (única capa con credenciales externas). Separación de concerns desde la UI hasta lo externo.
- **Tratar la escalación a humano como feature core** — diseñar el human-in-the-loop con interfaz concisa, input estructurado (botones/forms), y manejo robusto del estado "pausado" con **timeouts** y **default actions**. No es un fallo; es la red de seguridad.
- **Agregar un Plan Confirmation step a la delegación** — tras generar el plan, presentarlo al usuario para un "go/no-go" rápido antes de ejecutar autónomamente, mitigando el riesgo de misalignment sin supervisión paso a paso.
- **Definir tools robustas para las interacciones transaccionales** — descripciones claras y parámetros strongly typed, para que el LLM seleccione la tool correcta y extraiga argumentos de la query del usuario.
- **Invertir en la capacidad de planning del orquestador** — plan estático para workflows predecibles; plan generado por LLM (multi-step) para tareas complejas y dinámicas. Correr a los especialistas en paralelo cuando sea posible.
- **Aislar toda comunicación externa tras un proxy** — el proxy como única frontera de seguridad con las credenciales y el acceso de red; protocolo interno simple/estandarizado (una API interna) que abstrae la complejidad externa. Si el partner cambia su API, solo se toca el proxy.
- **Instrumentar métricas por patrón** — escalation rate/resolution time, task success/CSAT, first-contact resolution/latency, orchestration overhead/sub-task failure, external API error rate/security incidents. Transformar la "buena UX" en datos objetivos (ver Tabla 8.2).

## Conexiones

- [[_Agentic Architectural Patterns for Building Multi-Agent Systems|Mapa del libro]] — el MOC del libro.
- [[07 - Robustness and Fault Tolerance Patterns]] — cap. 7: introduce el enfoque de pattern chaining + evaluation metrics que este capítulo replica.
- [[09 - Agent-Level Patterns]] — cap. siguiente: los patrones internos del agente individual (Sense/Reason/Plan/Act). El cap. 8 arquitecta la relación humano-agente; el 9 hace zoom en el agente mismo.
- [[01 - GenAI in the Enterprise - Landscape, Maturity, and Agent Focus]] — cap. 1: el *Task Delegation Framework (Supervisor Architecture)* del ejemplo de loan processing es la semilla del patrón Agent Delegates to Agent.
- [[Orchestrator]] — el patrón supervisor/orquestador que descompone y delega (Agent Delegates to Agent).
- [[Function Calling]] · [[Tool Calling]] — mecanismo detrás de Human Calls Agent (intent classification, parameter extraction, tool execution).
- [[Generator-Evaluator Pattern]] — el loop generate→critique→correct emparenta con el self-review; acá en clave humano-agente, el humano es el evaluador en Agent Calls Human.
- [[A2A]] — Agent-to-Agent protocol: la comunicación agente↔agente (Agent Delegates to Agent) y cross-organizacional con OAuth (relevante al proxy); candidato a nota propia.
- [[MCP]] — Model Context Protocol: integración agente↔tools dentro de la organización; candidato a nota propia.
- [[LLM]] — núcleo de razonamiento del agente (router en transaccional, planner en delegación); candidato a nota propia.
- [[Hallucinations]] · [[Grounding]] — el human-in-the-loop como mitigación adicional ante decisiones high-stakes inciertas.
