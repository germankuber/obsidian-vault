---
title: "04 - Agentic AI Architecture - Components and Interactions"
libro: Agentic Architectural Patterns for Building Multi-Agent Systems
autor: Ali Arsanjani
capitulo: 4
created: 2026-06-11
tags:
  - libros/agentic-architectural-patterns
  - type/case-study
  - status/permanent
aliases:
  - Agentic AI Architecture - Components and Interactions
---

# 04 - Agentic AI Architecture - Components and Interactions

> [!info] Capítulo 4 · *Agentic Architectural Patterns for Building Multi-Agent Systems* — Ali Arsanjani
> El blueprint arquitectónico del agente: qué lo distingue de un LLM y de un workflow, su anatomía (Goals·Sense·Reason·Plan·Act·Memory·Coordinate) como bloques funcionales, los data stores/contexto que necesita, y los modelos de interacción + stack emergente (function calling → tool protocols → A2A). Cierra la Parte 1 y abre la Parte 2. Navegá: [[_Agentic Architectural Patterns for Building Multi-Agent Systems|Mapa del libro]] · anterior [[03 - The Spectrum of LLM Adaptation for Agents - RAG to Fine-tuning]].

## Resumen

Tras la Parte 1 (landscape de GenAI, preparar el LLM como motor, espectro de adaptación), este capítulo hace el **shift de la teoría a la estructura**: provee el blueprint arquitectónico práctico para sistemas agénticos robustos. El punto pivotal: **un LLM, por capaz que sea, no es el producto final sino un componente crítico dentro de un constructo operativo más amplio y distribuido** — un paradigm shift *de sistemas monolíticos a ecosistemas dinámicos de agentes inteligentes role-based*. El capítulo (1) define qué hace "agéntico" a un sistema y lo distingue de LLMs y workflows, (2) disecciona la **anatomía del agente** como bloques funcionales de un loop operativo continuo, (3) explora los **data stores y contexto** del que dependen, (4) introduce los **modelos de interacción y features** + el stack tecnológico emergente, y (5) mapea las **consideraciones técnicas** a los componentes que impactan. Entender esta arquitectura es prerequisito para los design patterns de la Parte 2.

![[04-fig-4.1.png]]
*Figura 4.1 – The hierarchy of autonomy*

## Definir un agente: conceptos core y capacidades

Un **AI agent** es un sistema, a menudo potenciado por LLMs, diseñado para percibir su entorno, tomar decisiones y ejecutar acciones para lograr goals específicos — una entidad más autónoma con objetivos en curso que opera más allá de responder a un prompt. Cuatro características clave lo distinguen de interacciones LLM básicas:
- **Autonomy** — grado de auto-gobierno; opera independientemente sin intervención humana directa constante. Fijado un goal, determina los pasos para alcanzarlo.
- **Reactivity (o sensing)** — percibe su entorno operativo y responde a cambios/eventos. El "sensing" es clave del comportamiento agéntico.
- **Proactivity (o goal-orientation)** — no solo reacciona; toma iniciativa de forma goal-directed, con planning y ejecución complejos en el tiempo.
- **Social ability** (opcional pero común en multi-agente) — interactúa y se comunica con otros agentes y humanos usando lenguajes/protocolos acordados para colaborar, negociar o coordinar.

**Jerarquía de autonomía** (LLM → automated workflow → AI agent):
- **LLMs** — el reasoning core / "cerebro" (understanding, planning, content generation). Los **MMMs (multi-modal models)**, ej. Google Gemini 3, son una evolución: entrenados en texto, código, imágenes, audio y video → proyectan distintas modalidades a un espacio semántico compartido, permitiendo razonamiento nativo cross-input (ej. analizar un screenshot de un dashboard para diagnosticar un error, o escuchar un audio para extraer sentiment, sin modelos de traducción separados). **Pero un MMM NO es un agente**: sigue siendo inherentemente *stateless y pasivo* — procesa input y predice output, no inicia acciones, no mantiene memoria de largo plazo por voluntad propia ni persigue goals independientemente; solo responde cuando se le hace prompt. Es la arquitectura del agente alrededor la que transforma ese potencial pasivo en comportamiento activo y goal-directed.
- **Automated workflows** (AI orchestration) — secuencia de tareas, algunas predeterminadas y deterministas, otras no-deterministas (intent interpretation + function calling). Como un script de pasos `if-this-then-that`: ejecuta acciones pero es determinista y rígido, sigue un camino fijo; carece de la capacidad de razonar, planificar dinámicamente o adaptarse a lo imprevisto.
- **AI agents** — el más sofisticado: sistema completo que usa un LLM como "cerebro" para lograr un goal autónomamente. Es *goal-oriented, stateful y adaptive*: razona sobre su entorno, crea un plan dinámico, usa tools para ejecutarlo y aprende de los outcomes. (Ej.: un LLM draftea un email al pedírselo; un workflow manda un email pre-escrito cada lunes 9 AM; un AI agent gestiona toda la inbox, categoriza proactivamente, agenda meetings según el contenido y alerta de urgencias, aprendiendo las preferencias del usuario.)

### Comparación LLM vs workflow vs agente (Tabla 4.1)

| Característica | LLM | Automated workflow | AI agent |
|---|---|---|---|
| **Core function** | Genera texto, responde, sintetiza info según un prompt. | Ejecuta una secuencia estática predefinida de tareas. | Logra autónomamente un goal específico. |
| **Decision-making** | Pattern matching y generación probabilística. No toma decisiones. | Lógica hard-coded `if-this-then-that`. No razona. | Razona, planifica y toma decisiones dinámicas en problemas complejos. |
| **State and memory** | Stateless: cada interacción es nueva (aunque se puede pasar contexto). | Generalmente stateless; no recuerda ejecuciones pasadas. | Stateful: mantiene memoria de corto y largo plazo para aprender y adaptarse. |
| **Adaptability** | No se adapta salvo fine-tuning o nuevas técnicas de prompting. | Rígido: cualquier cambio requiere re-programar el workflow. | Altamente adaptive; aprende de la experiencia y reacciona a cambios del entorno. |
| **Interaction model** | Recibe un prompt, devuelve una respuesta. | Disparado por un evento, ejecuta un script fijo. | Opera en un loop continuo *sense-reason-plan-act* persiguiendo sus goals. |
| **Failure handling** | Pasivo/Ninguno: puede generar info plausible pero incorrecta (alucinación) sin conciencia del fallo. | Rígido/Frágil: falla duro o dispara un exception path pre-seteado (ej. "stop and alert admin") al romperse reglas. | Resiliente/Self-correcting: detecta errores, reflexiona sobre la causa y reintenta autónomamente con otra estrategia o tool. |

Un agente **integra y eleva** las capacidades de los otros dos: aprovecha el LLM para razonar pero lo coloca en una estructura que permite ejecución autónoma y goal-driven que un workflow estático no puede.

## La anatomía de un agente (Tabla 4.2)

Los componentes core como **bloques funcionales** de un loop operativo continuo (perceive → reason → act):

| Componente | Función core (recap) | Rol arquitectónico e implementación |
|---|---|---|
| **Goals** (especificados vía Instructions) | Los objetivos o outcomes deseados. | Define la *objective function* del agente y dirige su planning de alto nivel. Implementado como parámetros de config o un estado dinámico actualizable. |
| **Sense (Perception)** | Recolecta info y data del entorno (digital o físico). | La *input layer*. Implementada vía API listeners, data stream processors o protocolos estandarizados como [[MCP]] (Model Context Protocol). |
| **Reason (Cognition)** | La unidad de procesamiento core donde la info sensada se analiza e interpreta. | El *cognitive core* donde se integra el agent-ready LLM. Interpreta inputs, los evalúa contra los goals y formula estrategias de alto nivel. |
| **Plan** | Diseña una secuencia de acciones según los insights razonados. | La *tactical layer*, también potenciada por el LLM. Descompone la estrategia del Reason en una secuencia concreta y ordenada de pasos ejecutables / tool calls. |
| **Act (Action)** | Ejecuta las acciones planeadas sobre el entorno usando tools disponibles. | La *output layer*. Implementada invocando tools: APIs externas, ejecutar código, comandos a actuadores, generar respuestas. |
| **Memory** | Almacena conocimiento, experiencias y estado para dar contexto a las decisiones. | Gestiona el *state*. Implementada con variables de corto plazo (tarea actual) y stores persistentes de largo plazo (vector DBs para RAG, preferencias de usuario). |
| **Coordinate** | Interactúa con otros agentes para alinear acciones y colaborar (principalmente multi-agente). | La *inter-agent communication layer*. Gestiona la vida completa de una tarea, trackeando estados estandarizados [[A2A]] (`submitted`, `working`, `input-required`, `completed`). Implementada vía protocolos como **Agent2Agent (A2A)**, permitiendo delegar trabajo y sincronizar estado deterministicamente a través de fronteras distribuidas. |

Los bloques operan en un **ciclo continuo** (sense → reason sobre la info nueva en el contexto de goals y memory → plan → act); los resultados se sensan en la siguiente iteración, creando un **feedback loop** que permite aprender y adaptarse. Ese loop operativo dinámico es el fundamento de los comportamientos y patrones más complejos del resto del libro.

### Case study 1 — Travel Planning Agent (Tabla 4.3)

Single-agent autónomo cuyo goal primario es reservar un itinerario de viaje completo desde un request en lenguaje natural, interactuando con sistemas externos.

| Componente | Ejemplo Travel Planning Agent |
|---|---|
| **Goals** | Cumplir: *"Book a round-trip flight to Paris and a 4-star hotel for 5 nights next month, keeping the total cost under $2500."* Dinámico, actualizable si el usuario cambia de idea o agrega constraints. |
| **Sense (Perception)** | Procesa el request en lenguaje natural; sigue "sensando" al recibir data de sistemas externos (lista de vuelos de una airline API, pricing de un hotel booking service). |
| **Reason (Cognition)** | El LLM core analiza preferencias explícitas ("Paris", "4-star", "5 nights") e intención implícita; interpreta la data estructurada de las búsquedas, compara opciones contra el budget y hace inferencia compleja para el itinerario óptimo. |
| **Plan** | (1) descomponer el request en sub-tareas (flight/hotel booking); (2) ejecutar flight search; (3) ejecutar hotel search; (4) analizar resultados para una combinación válida que cumpla constraints; (5) presentar el itinerario final para confirmación; (6) ejecutar las acciones de booking finales. |
| **Act (Action)** | Usa tools: `search_flights(destination="CDG", month="next")`, `find_hotels(city="Paris", rating=4, nights=5)`. Tras confirmación, llama funciones de booking, genera el mensaje de confirmación y actualiza el profile de largo plazo del usuario con las nuevas preferencias. |
| **Memory** | Corto plazo: preferencias del usuario, opciones de vuelo/hotel encontradas, historial de la conversación. Largo plazo: recordar la airline/hotel chain preferida de interacciones pasadas para personalizar. |
| **Coordinate** | Si fuera multi-agente, podría coordinar con otros: tras reservar el viaje, delegar a un "Excursion Agent" ("Find and book a ticket for the Louvre Museum"), colaborando hacia el goal colectivo. |

### Case study 2 — Agentic Loan Processing System (Tabla 4.4)

Aplicación de la *agentic anatomy* (Fig. 1.1) a un workflow multi-agente full-stack de aprobación de préstamos. Una institución financiera despliega un sistema multi-agente que va de la interacción inicial con el cliente al disbursement final (verificación de datos, credit assessment, risk scoring, compliance, aprobación). Cada agente especializa una fase, usando el **A2A protocol** para comunicar/coordinar y accediendo a una **shared memory layer** para contexto consistente.

![[04-fig-4.2.png]]
*Figura 4.2 – Use case example: Agentic loan processing*

| Componente | Ejemplo Loan Processing |
|---|---|
| **Goals** | Goal especializado por rol: `Intake Agent` (recolecta data completa del aplicante), `Credit Check Agent` (valida historial crediticio y flaggea anomalías), `Risk Assessment Agent` (puntúa el risk profile), `Compliance Agent` (verifica adherencia regulatoria), `Approval Agent` (decisión final holística). |
| **Sense (Perception)** | Recolecta data vía intake forms (lenguaje natural, structured data), API calls a credit bureaus, DBs internas de CRM y KYC, document OCR, real-time market feeds (préstamos de tasa variable). Cada agente usa **MCP** para acceder a data estructurada/no estructurada contextualmente. |
| **Reason (Cognition)** | Potenciado por LLM, razona sobre: semántica de documentos financieros (pay stubs, bank statements), reglas de política y guidelines de riesgo, intención inferida de las queries del aplicante, contradicciones en la data (ej. income mismatch). El cognitive core evalúa inputs contra los goals. |
| **Plan** | Agent-specific: el Intake planea follow-ups por info faltante; el Credit Check planea qué bureaus consultar (Equifax, TransUnion) y en qué orden + estrategia ante fallos de API/discrepancias; el Risk elige un scoring model según el tipo de préstamo; el Compliance selecciona los checks según jurisdicción; el Approval construye un decision tree integrando outputs de los demás. |
| **Act (Action)** | Manda verification emails, actualiza el loan status en CRM, llama APIs para leer/escribir data, genera compliance audit trails, dispara agentes downstream al completar tareas. |
| **Memory** | Corto plazo: session memory específica del aplicante. Largo plazo: patrones de fraude agregados, decisiones históricas, cambios regulatorios. Permite mejora basada en aprendizaje (ajustar por nuevos indicadores de riesgo). |
| **Coordinate** | Vía el **A2A Protocol** (Google's Agent2Agent Interoperability Protocol) como capa de comunicación: canal seguro y estándar para delegar tareas e intercambiar info; reporte de status (`working`, `completed`, `failed`) al agente delegante; interoperabilidad (agentes de distintas plataformas colaboran sin compartir su memoria/lógica interna). |

**Flujo ilustrativo (lifecycle)**: (1) el cliente envía la aplicación → `Intake Agent` parsea y guarda en shared memory; (2) se dispara el `Credit Check Agent` → recupera FICO e historial vía API; (3) el `Risk Agent` razona sobre credit+income+loan amount → asigna risk score; (4) el `Compliance Agent` chequea KYC, AML, GDPR y reglas bancarias locales; (5) el `Approval Agent` integra inputs, razona sobre conflictos y decide; (6) si se rechaza, el `Appeals Agent` puede ofrecer productos alternativos (ej. secured loan); (7) el `Disbursement Agent` ejecuta el release de fondos y actualiza los backends.

**Coordinación multi-agente vía A2A**: los agentes colaboran mandándose tareas estructuradas con un protocolo definido como A2A — *comunicación directa*, donde un agente delega explícitamente un trabajo a un agente receptor específico. En vez de publicar un score a un bus general, el orquestador primero manda una tarea al `Risk Agent`; al recibir el resultado, manda una *nueva* tarea al `Approval Agent` incluyendo el risk score en el payload → cadena de delegación clara y auditable. Disputas o edge cases (ej. credit scores borderline) pueden invocar un *deliberation loop*: una secuencia de mensajes A2A donde los agentes intercambian ofertas, contraofertas, o escalan a otro agente o a un humano.

**Beneficios**: *responsiveness* (reaccionan a cambios en data real-time, ej. fluctuación de credit score), *compliance by design* (reglas modulares embebidas + capas de política compartidas), *transparency* (memory logs y reasoning traces para auditorías), *adaptability* (los agentes evolucionan independientemente; ej. reemplazar el Risk Agent por un LLM más nuevo).

## Data stores y contexto del entorno para agentes

*Context is king* (cap. 1) vale especialmente para agentes que deben tomar decisiones informadas y actuar apropiadamente; dependen mucho de data stores y contexto del entorno para percibir, razonar y actuar.

![[04-fig-4.3.png]]
*Figura 4.3 – Contextual inputs versus grounding failures*

(La Fig. 4.3 contrasta un workflow saludable y *grounded* —contexto rico → razonamiento preciso— contra failure modes donde memoria stale o retrieval fallido lleva a acciones incorrectas.)

- **Digital business context** — todas las fuentes de data digitales del entorno enterprise/online:
  - **Unstructured data** — texto (reports, emails, articles), imágenes, audio, video. Los LLMs son particularmente buenos procesando texto no estructurado.
  - **Vector stores** — DBs especializadas para *similarity search* eficiente sobre embeddings (retrieval por significado, no keywords); componente core de muchos sistemas RAG. Dos categorías: librerías/DBs open source (**FAISS, Weaviate, Chroma**) para control local/self-hosted, y managed services comerciales (**Pinecone, Google Vertex AI Vector Search**) por escalabilidad y facilidad de gestión.
  - **Structured data** — data organizada de DBs relacionales o spreadsheets; data points bien definidos para tareas específicas (customer records, product info).
  - **Knowledge graphs** — info como red de entidades y sus relaciones → entendimiento estructurado y semántico de un dominio; útiles para razonamiento complejo sobre conceptos interconectados.
- **Physical environment context** — para agentes que interactúan con el mundo real (robots, IoT):
  - **Sensors** — cámaras, micrófonos, sensores de temperatura, GPS → data del entorno físico.
  - **Actuators** — brazos robóticos, motores, switches → acciones físicas o manipular objetos.

Los agentes efectivos suelen necesitar **integrar info de múltiples tipos de data stores** y sintetizar desde contextos diversos para un entendimiento comprehensivo.

### Ejemplo — Supply Chain Management Agent (Tabla 4.5)

Agente que monitorea y optimiza inventario y logística en tiempo real:

| Context / Data store | Ejemplo Supply Chain Management Agent |
|---|---|
| **Digital business context** | El mundo digital donde opera, toda la data operativa de la empresa. |
| Unstructured data | Procesa shipping manifests (PDFs), notificaciones de delays de proveedores (emails), y noticias sobre port strikes o eventos climáticos para entender disrupciones potenciales. |
| Vector stores | Ante un nuevo evento geopolítico, usa un vector store para una búsqueda semántica de whitepapers internos sobre cómo eventos pasados similares impactaron las shipping lanes, recuperando el contexto histórico relevante. |
| Structured data | Consulta una DB relacional para inventory levels de SKUs específicos, fechas de entrega esperadas y capacidad de warehouse. |
| Knowledge graphs | Usa un knowledge graph para entender las relaciones multi-nivel complejas entre proveedores, plantas, centros de distribución y destinos retail → razonar que un delay de un único supplier de componentes impactará tres product lines específicas. |
| **Physical environment context** | Los elementos del mundo real que el agente sensa y sobre los que actúa, vía IoT. |
| Sensors | Stream continuo de GPS trackers en camiones, sensores de temperatura en contenedores refrigerados (integridad del producto), y RFID scanners en loading bays confirmando la llegada de mercadería. |
| Actuators | Si predice una disrupción mayor en una ruta primaria, puede disparar un actuador en el smart warehouse (conveyor belt, brazo robótico) para desviar los pallets afectados a otro loading bay para una ruta alternativa. |

## Modelos de interacción y features clave

### Features arquitectónicos (Tabla 4.6)

Un buen diseño agéntico da lugar a features potentes (ilustrados con el Supply Chain Management Agent):

| Feature arquitectónico | Ejemplo en el supply chain system |
|---|---|
| **Modularity** | Para sumar fraud detection de invoices, en vez de reconstruir el `InventoryAgent` existente, se desarrolla y agrega un nuevo `InvoiceFraudAgent` especializado. Si el modelo de fraude se actualiza, solo ese agente se ve afectado. |
| **Scalability** | En peak de holiday shipping, maneja el aumento masivo de data GPS desplegando dinámicamente múltiples instancias del `LogisticsAgent` en paralelo; un load balancer distribuye las tareas de route optimization, manteniendo el sistema responsivo. |
| **Adaptability** | Un `SupplierCommsAgent` observa que emails de "Supplier-X" con la frase "production delay" consistentemente preceden shortages críticos de inventario; con esa correlación aprendida, adapta su comportamiento para escalar automáticamente futuros emails con esos keywords a "urgent" y alertar al `InventoryAgent`. |
| **Multimodal interaction** | Un `ReceivingDockAgent` procesa primero un shipping manifest de texto (PDF), luego usa una cámara para inspeccionar visualmente el pallet recibido por cajas dañadas (image data); actúa sobre la info combinada, actualizando el inventario solo si la inspección visual matchea la descripción del manifest. |
| **Collaboration** | Un `DisruptionMonitoringAgent` detecta un cierre de puerto de un news feed y actualiza un status en una DB compartida; el `InventoryAgent` sensa el cambio y manda un mensaje directo al `LogisticsAgent` pidiendo rutas alternativas, iniciando una resolución colaborativa. |

### Modelos de interacción entre agentes

Dos modelos fundamentales, que difieren en cómo intercambian info:
- **Direct communication** — los agentes se mandan mensajes explícitamente usando un lenguaje y protocolo común. *Ejemplo*: el `InventoryAgent` detecta stock bajo y manda un mensaje directo `{"task": "reorder_part", "part_id": "XYZ-123", "quantity": 500}` al `ProcurementAgent`.
- **Indirect communication (Stigmergy)** — los agentes interactúan indirectamente observando y modificando un entorno compartido (ej. una DB). *Ejemplo*: el `ManufacturingAgent` actualiza un registro en una DB compartida a `{"status": "complete"}`; el `LogisticsAgent`, que monitorea esa DB, ve el cambio de status e inicia el shipping.

![[04-fig-4.4.png]]
*Figura 4.4 – Agent interaction models*

### El stack tecnológico emergente (3 capas)

La implementación práctica de esos modelos se apoya en un stack emergente de capas complementarias:
- **Layer 1: Function calling** — la capa de interacción fundamental donde el LLM del agente dispara una **tool local**. Permite identificar inteligentemente qué tool usar, cuándo y con qué parámetros, todo dentro de un único application runtime. Fue el primer gran paso para que los LLMs fueran más que generadores de texto. *Limitación primaria*: el developer es responsable de hostear, correr y asegurar la tool en el mismo entorno. *Ejemplo*: el LLM del `LogisticsAgent` determina que necesita el path de entrega más eficiente y genera una call a su función interna `calculate_optimal_route()`.
- **Layer 2: Tool protocols** — forma estandarizada de que los agentes descubran y usen **tools externas como servicios interoperables**. Protocolos como el **MCP de Anthropic** desacoplan el hosting/ejecución de la tool del agente mismo. Evolución significativa respecto de soluciones framework-specific (ej. el `ToolExecutor` de LangChain o los routers de LangGraph): mientras esas tools OSS gestionan la ejecución dentro del runtime de una app específica (a menudo requiriendo dependencias compartidas), MCP permite definir tools vía schemas y hostearlas independientemente → cualquier sistema compliant puede descubrirlas e invocarlas a través de fronteras de proceso o red, volviendo las tools servicios portables como REST APIs. *Ejemplo*: para considerar el clima, el `LogisticsAgent` usa un tool protocol para descubrir y conectarse a una weather service API de terceros, trayendo data de tormentas real-time a su cálculo de ruta sin acoplarse fuertemente a esa API.
- **Layer 3: Agent-to-agent protocols** — el nivel más alto, enfocado en colaboración entre **agentes independientes** que pueden correr en distintos frameworks o empresas. Protocolos como **A2A** se enfocan en estándares universales: delegar tareas estructuradas, gestionar workflows asíncronos, habilitar coordinación multi-agente compleja — actuando como un traductor universal o **"SMTP for AI agents"**. *Ejemplo*: tras calcular una ruta, el `LogisticsAgent` necesita reservar sea freight; usa un A2A protocol para mandar una tarea estructurada a un `FreightForwarderAgent` completamente separado, operado por una logística de terceros.

Muchos sistemas sofisticados usan un **enfoque híbrido**: function calling para acciones internas y protocolos de nivel más alto para colaboración externa.

![[04-fig-4.5.png]]
*Figura 4.5 – Emerging agentic stack*

## Consideraciones técnicas para arquitecturas agénticas (Tabla 4.7)

Construir sistemas agénticos robustos es técnicamente demandante. Mapeo de las consideraciones técnicas a los componentes arquitectónicos que más impactan:

| Consideración técnica | Componente(s) arquitectónico(s) afectado(s) |
|---|---|
| Data processing e integración | Sense, Memory |
| Knowledge representation | Reason, Memory |
| LLM integration y orchestration | Reason, Plan, Coordinate |
| Reliable tool use mechanisms | Act |
| State management y memory | Memory |
| Scalability de poblaciones de agentes | Coordinate, arquitectura general del sistema |
| Eficiencia de comunicación inter-agente | Coordinate |
| Security y governance | Reason (prompt injection), Act (sandboxing), Memory (privacy), Coordinate (AuthN/AuthZ) |

Abordar estas consideraciones es un viaje de refinamiento progresivo de capacidades, alineado con las etapas del GenAI Maturity Model. Esta exploración (anatomía, dependencia de data, modelos de interacción) prepara el terreno para los design patterns de la Parte 2.

## Citas

> "an LLM, no matter how capable, is not the end product but a critical component within a broader, distributed operational construct."
> "from monolithic systems to dynamic ecosystems of intelligent, role-based AI agents"
> "However, an MMM is not an agent."
> "Agents are more than LLMs"
> A2A como "SMTP for AI agents"

## Para aplicar

- **Distinguir agente de LLM y de workflow** — un agente es goal-oriented, stateful, adaptive y self-correcting; un LLM es stateless/pasivo, un workflow rígido/frágil. No confundir un MMM (potente pero pasivo) con un agente.
- **Diseñar con la anatomía completa** — Goals (objective function) · Sense (input layer, MCP) · Reason (cognitive core/LLM) · Plan (tactical layer) · Act (output layer/tools) · Memory (corto + largo plazo, vector DB) · Coordinate (A2A con estados submitted/working/input-required/completed). Pensar el loop sense→reason→plan→act con feedback.
- **Nutrir el contexto desde múltiples data stores** — unstructured + vector stores (FAISS/Weaviate/Chroma o Pinecone/Vertex) + structured + knowledge graphs; para físico, sensores + actuadores. Integrar y sintetizar a través de contextos.
- **Elegir el modelo de interacción** — directa (mensajes explícitos, cadena de delegación auditable) o indirecta/stigmergy (entorno compartido/DB). Para edge cases, deliberation loops vía A2A.
- **Aplicar el stack por capas / híbrido** — function calling para tools/acciones internas, tool protocols (MCP) para tools externas como servicios portables, A2A para colaboración entre agentes independientes/cross-enterprise.
- **Mapear los desafíos técnicos a componentes** (Tabla 4.7) — saber dónde se manifiesta cada hurdle: prompt injection→Reason, sandboxing→Act, privacy→Memory, AuthN/AuthZ→Coordinate, etc.

## Conexiones

- [[_Agentic Architectural Patterns for Building Multi-Agent Systems|Mapa del libro]] — el MOC del libro.
- [[03 - The Spectrum of LLM Adaptation for Agents - RAG to Fine-tuning]] — cap. 3: la arquitectura jerárquica (orquestador/sub-agents/AgentTools) que aquí se generaliza; RAG y grounding (Fig. 4.3 contextual inputs vs grounding failures).
- [[01 - GenAI in the Enterprise - Landscape, Maturity, and Agent Focus]] — cap. 1: la *agentic anatomy* (Fig. 1.1) que este capítulo reaplica al loan processing; *context is king*; el Task Delegation Framework.
- [[02 - Agent-Ready LLMs - Selection, Deployment, and Adaptation]] — cap. 2: el agent-ready LLM que se integra en el componente Reason; function calling como Layer 1 del stack.
- [[09 - Agent-Level Patterns]] — cap. 9: operacionaliza estos componentes en patrones (Sense→Multimodal, Memory→Memory/RAG, Reason→Structured Reasoning, Act→Single Agent Baseline).
- [[06 - Explainability and Compliance Agentic Patterns]] · [[07 - Robustness and Fault Tolerance Patterns]] — caps. 6/7: la Shared Epistemic Memory y los patrones de robustez/seguridad (prompt injection, sandboxing) que la Tabla 4.7 anticipa.
- [[08 - Human-Agent Interaction Patterns]] — cap. 8: el stack de interacción (function calling/A2A) y el Agent Delegates to Agent se apoyan en esta arquitectura.
- [[Orchestrator]] — el orquestador que delega tareas (loan processing, supply chain).
- [[Function Calling]] · [[Tool Calling]] — Layer 1 del stack emergente.
- [[MCP]] · [[A2A]] — Layer 2 (tool protocols, MCP) y Layer 3 (agent-to-agent, A2A, "SMTP for AI agents"); candidatos a nota propia.
- [[_RAG|RAG]] · [[Chunking Strategies]] · [[Hybrid Search]] · [[Grounding]] · [[Hallucinations]] — vector stores y grounding del componente Memory/Sense.
- [[LangGraph]] · [[LangChain|AI Framework]] — los frameworks (ToolExecutor/routers) que MCP supera como tool protocol; ya en el vault (AI Agents).
- [[LLM]] — el cognitive core integrado en Reason (candidato a nota propia).
- **MMM (multi-modal model)** · **Stigmergy** · **Knowledge graph** · **Vector store** · **Hierarchy of autonomy** — conceptos del capítulo; candidatos a nota propia.
