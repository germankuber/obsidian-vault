---
title: A1 - Appendix A - Pattern Reference: GoF, EIP, Reliability, GenAI
libro: Building Complex Multi-Agent Systems Using Pattern Prompting
autor: Packt
capitulo: "Apéndice A"
created: 2026-06-22
tags:
  - libros/building-complex-multi-agent-systems-using-pattern-prompting
  - type/reference
  - status/permanent
aliases:
  - Appendix A - Pattern Reference
  - Appendix A - Patterns
  - A1 - Pattern Reference
updated: 2026-06-22
---

# A1 - Appendix A - Pattern Reference: GoF, EIP, Reliability, GenAI

> [!info] Apéndice A · *Building Complex Multi-Agent Systems Using Pattern Prompting* — Packt (ISBN 9781806114290)
> Es la **sección de referencia / glosario** que **consolida todo el vocabulario de patrones** usado a lo largo del libro: los **[[GoF|GoF design patterns]]**, los **[[Enterprise Integration Patterns|Enterprise Integration Patterns (EIP)]]** de Hohpe & Woolf, los **reliability patterns** ([[Circuit Breaker]], [[Retry with Exponential Backoff]]) y los **microarchitecture patterns** GenAI ([[Orchestration]], [[Choreography]], [[RAG Microarchitecture]]). Cada ficha sigue el formato **Intent → Structure (prosa + diagrama) → Used in this book** (los usos concretos), y la **Table A.1** mapea cada patrón a los capítulos donde aparece. La tesis del apéndice: los patrones **no son invenciones de la era AI**, son **soluciones de ingeniería durables** que ahora dan un lenguaje preciso para **construir, explicar y evaluar** sistemas agénticos — la novedad aparente del stack GenAI descansa sobre fundamentos familiares (queues, adapters, routers, enrichers, retry policies, orchestration/choreography). Como es un apéndice, no hay "próximo capítulo": cierra el libro junto al [[10 - The Future and Limitations of LLMs|cap. 10]] (y el [[A2 - Appendix B - Topologos User Manual|Appendix B]], el manual de Topologos). Navegá: [[_Building Complex Multi-Agent Systems Using Pattern Prompting|Building Complex Multi-Agent Systems]] · anterior [[10 - The Future and Limitations of LLMs]] · siguiente [[A2 - Appendix B - Topologos User Manual|Appendix B]].

## Resumen

El Apéndice A es la **referencia consolidada de patrones** del libro: reúne en un solo lugar todo el vocabulario arquitectónico que los capítulos fueron sembrando, para que sirva a la vez como **glosario** (buscar qué significa un patrón) y como **guía de estudio compacta** (repasar el catálogo entero antes de diseñar). La premisa que lo organiza —idéntica a la tesis del [[01 - Introduction Patterns, Abstractions, and the GenAI Landscape|cap. 1]] y la conclusión del [[08 - Pattern-Guided Coding|cap. 8]]— es que estos patrones **no son invenciones de la era AI**: son **soluciones durables y probadas de ingeniería de software** que, aplicadas a GenAI, dan un **lenguaje preciso** para construir, explicar y evaluar sistemas agénticos. Nombrar el patrón antes de codear es lo que vuelve el resultado verificable y reproducible (el [[Pattern-Guided Coding|pattern-guided coding]] de los caps. 4 y 8).

El apéndice se organiza en **cuatro partes** que recorren las cuatro familias de patrones que el libro usa, más un cierre sobre la arquitectura RAG de producción. **Part 1 — GoF Design Patterns** (A.1 [[Strategy]], A.2 [[Adapter]], A.3 [[Template Method]]): los patrones de objetos del Gang of Four que estructuran el LLM client y los producers/consumers. **Part 2 — Enterprise Integration Patterns** (A.4–A.14): los once EIP de Hohpe & Woolf que modelan la mensajería sobre [[RabbitMQ]] — desde el [[Message Channel]] básico hasta el [[Scatter-Gather]] y el [[Content-Based Router]], pasando por los de transformación, ruteo, correlación y resiliencia de mensajes. **Part 3 — Reliability Patterns** (A.15 [[Circuit Breaker]], A.16 [[Retry with Exponential Backoff]]): los patrones que blindan las llamadas a dependencias poco fiables (los LLMs y las APIs). **Part 4 — Microarchitecture Patterns** (A.17 [[Orchestration]], A.18 [[Choreography]], A.19 [[RAG Microarchitecture]]): los patrones de coordinación de componentes y la microarquitectura GenAI por excelencia, que el apéndice cierra desglosando en cuatro sub-diagramas de **RAG de producción** (hybrid retrieval, chunking strategies, embedding flow, production query flow).

Cada una de las **19 fichas** (A.1–A.19) sigue la misma estructura: un **Intent** (qué resuelve el patrón), una **Structure** (cómo se compone, en prosa + un diagrama), y un bloque **Used in this book** con los tres usos concretos donde el patrón apareció a lo largo de los capítulos. La **Table A.1** funciona como índice maestro: lista los 19 patrones con su **ID**, **nombre**, **categoría** (GoF Behavioral/Structural, EIP por sub-familia, Reliability, Microarchitecture) y los **capítulos** donde se usan. El párrafo de cierre del apéndice consolida la idea: la aparente novedad de la arquitectura GenAI descansa sobre fundamentos de ingeniería familiares, y **RAG no reemplaza la arquitectura clásica — la operacionaliza**.

> [!note] **La premisa del apéndice.** Los patrones no son invenciones de la era AI: son soluciones de ingeniería **durables** que ahora proveen un **lenguaje preciso** para construir, explicar y evaluar sistemas agénticos. Por eso este apéndice es a la vez glosario **y** guía de estudio: aprender el catálogo es aprender el idioma del libro entero.

### Tabla A.1 — Índice de patrones

El índice maestro del apéndice: los 19 patrones con su categoría y los capítulos donde el libro los aplica.

| ID | Pattern | Category | Chapters |
|---|---|---|---|
| A.1 | **[[Strategy]]** | GoF — Behavioral | [[04 - Building Your First RAG App\|4]], [[06 - Ingesting Data Using Airbyte and Pinecone\|6]] |
| A.2 | **[[Adapter]]** | GoF — Structural | [[04 - Building Your First RAG App\|4]], [[06 - Ingesting Data Using Airbyte and Pinecone\|6]] |
| A.3 | **[[Template Method]]** | GoF — Behavioral | [[04 - Building Your First RAG App\|4]] |
| A.4 | **[[Message Channel]]** | EIP — Messaging Infrastructure | [[04 - Building Your First RAG App\|4]], [[06 - Ingesting Data Using Airbyte and Pinecone\|6]] |
| A.5 | **[[Channel Adapter]]** | EIP — Messaging Infrastructure | [[04 - Building Your First RAG App\|4]], [[06 - Ingesting Data Using Airbyte and Pinecone\|6]] |
| A.6 | **[[Publish-Subscribe Channel]]** | EIP — Messaging | [[04 - Building Your First RAG App\|4]] |
| A.7 | **[[Content Enricher]]** | EIP — Message Transformation | [[03 - Building with GenAI Parameters, Tuning, and Project Phases\|3]], [[04 - Building Your First RAG App\|4]] |
| A.8 | **[[Request-Reply]]** | EIP — Messaging | [[04 - Building Your First RAG App\|4]] |
| A.9 | **[[Correlation Identifier]]** | EIP — Messaging | [[04 - Building Your First RAG App\|4]] |
| A.10 | **[[Scatter-Gather]]** | EIP — Message Routing | [[04 - Building Your First RAG App\|4]] |
| A.11 | **[[Dead Letter Channel]]** | EIP — Messaging | [[04 - Building Your First RAG App\|4]] |
| A.12 | **[[Message History]]** | EIP — System Management | [[04 - Building Your First RAG App\|4]] |
| A.13 | **[[Content-Based Router]]** | EIP — Message Routing | [[04 - Building Your First RAG App\|4]] |
| A.14 | **[[Pipes and Filters]]** | EIP — Message Routing | [[05 - Starting Your Data Migration Project\|5]], [[06 - Ingesting Data Using Airbyte and Pinecone\|6]] |
| A.15 | **[[Circuit Breaker]]** | Reliability Pattern | [[04 - Building Your First RAG App\|4]] |
| A.16 | **[[Retry with Exponential Backoff]]** | Reliability Pattern | [[04 - Building Your First RAG App\|4]], [[07 - Tips and Best Practices\|7]] |
| A.17 | **[[Orchestration]]** | Microarchitecture Pattern | [[01 - Introduction Patterns, Abstractions, and the GenAI Landscape\|1]], [[04 - Building Your First RAG App\|4]], [[07 - Tips and Best Practices\|7]] |
| A.18 | **[[Choreography]]** | Microarchitecture Pattern | [[01 - Introduction Patterns, Abstractions, and the GenAI Landscape\|1]], [[07 - Tips and Best Practices\|7]] |
| A.19 | **[[RAG Microarchitecture]]** | GenAI Microarchitecture | [[02 - Embeddings The Language of AI\|2]], [[03 - Building with GenAI Parameters, Tuning, and Project Phases\|3]], [[04 - Building Your First RAG App\|4]], [[05 - Starting Your Data Migration Project\|5]] |

> [!tip] La tabla deja clara la **arqueología del stack GenAI**: cada pieza "nueva" (RAG, multi-LLM, agentes) se reduce a combinaciones de GoF + EIP + reliability patterns ya conocidos. La columna *Category* clasifica el origen del patrón; la columna *Chapters* es el índice inverso (de patrón a dónde se usa).

## Part 1 — GoF Design Patterns

Los patrones de objetos del **Gang of Four** (Gamma, Helm, Johnson, Vlissides). En este libro estructuran el **LLM client** y los **producers/consumers** de mensajes: encapsulan el comportamiento variable (Strategy), aíslan APIs heterogéneas (Adapter) y fijan el esqueleto de un algoritmo (Template Method).

### A.1 Strategy

*GoF — Behavioral | Chapters 4, 6*

**Intent.** Definir una **familia de algoritmos**, encapsular cada uno y hacerlos **intercambiables en runtime**, de modo que el algoritmo varíe independientemente de los clientes que lo usan. Evita los condicionales hard-codeados y la explosión de subclases.

**Structure.** Un `Context` mantiene una referencia a un objeto `Strategy` (una interfaz); las `ConcreteStrategy` implementan cada algoritmo; el cliente inyecta la estrategia concreta y el contexto delega en ella sin conocer cuál es. Cambiar de comportamiento = cambiar el objeto inyectado, no el contexto.

![[B34134_Appendix_A_1.png]]
*Figure A.1 – Strategy pattern*

> [!note] **Used in this book**
> - **LLM client (cap. 4)** — los *behaviors pluggables* (p. ej. la lógica de generación por provider) se modelan como Strategy para evitar la explosión de subclases, prefiriendo composición sobre herencia profunda.
> - **Routing del producer (cap. 8 / pattern-guided coding)** — la `RoutingStrategy` decide a qué cola/exchange/LLM mandar cada mensaje (round-robin, por tenant, por costo) sin tocar el `AgentProducer`.
> - **No-code adapter swap de Airbyte (cap. 6)** — el Strategy permite intercambiar **source/destination adapters en runtime** (S3 → SharePoint, Pinecone → Qdrant) aportando solo connection info al context, sin reescribir la arquitectura.

### A.2 Adapter

*GoF — Structural | Chapters 4, 6*

**Intent.** **Convertir la interfaz** de una clase en otra que el cliente espera, permitiendo que colaboren clases con interfaces incompatibles. Aísla al cliente de las particularidades de cada implementación externa.

**Structure.** Un `Adapter` implementa la interfaz `Target` que el cliente usa y, por dentro, traduce las llamadas al `Adaptee` (la clase con interfaz incompatible). El cliente habla siempre el mismo idioma; el adapter absorbe las diferencias.

![[B34134_Appendix_A_2.png]]
*Figure A.2 – Adapter pattern*

> [!note] **Used in this book**
> - **LLM client por provider (cap. 4)** — un Adapter por cada LLM (OpenAI, Claude…) aísla las *API specifics* de cada provider detrás de una interfaz uniforme; cambiar de modelo no toca el resto del client.
> - **Channel Adapter al broker (caps. 4, 6, 8)** — la variante EIP del Adapter (ver A.5) traduce entre la app y [[RabbitMQ]] / una HTTP API.
> - **Inbound/outbound adapters de Airbyte (cap. 6)** — los connectors de Airbyte son adapters que conectan fuentes (S3) y destinos (vector DBs) traduciendo la data al formato del pipeline.

### A.3 Template Method

*GoF — Behavioral | Chapter 4*

**Intent.** Definir el **esqueleto de un algoritmo** en una operación de la clase base, **delegando algunos pasos a las subclases**, de modo que estas redefinan ciertos pasos sin cambiar la estructura general del algoritmo.

**Structure.** Una clase abstracta define un `templateMethod()` que invoca, en orden fijo, una serie de pasos; algunos están implementados en la base y otros son abstractos (*hook/primitive operations*) que las subclases concretan. El flujo lo controla la base; el detalle, la subclase.

![[B34134_Appendix_A_3.png]]
*Figure A.3 – Template Method pattern*

> [!note] **Used in this book**
> - **Request lifecycle del LLM client (cap. 4)** — el ciclo de la request (los 5 pasos) vive en la base class; cada provider llena solo los pasos específicos.
> - **Consumer lifecycle (cap. 8)** — el lifecycle del consumer (receive → validate → process → ack/nack) está en la base; las subclasses implementan solo el `process`.
> - **Pipelines de procesamiento reproducibles** — el esqueleto fijo garantiza que validación, procesamiento y acknowledgment ocurran siempre en el mismo orden, sin que cada consumer lo reimplemente.

## Part 2 — Enterprise Integration Patterns

Los **EIP** de Gregor Hohpe y Bobby Woolf: el catálogo de patrones de **mensajería asíncrona** sobre el que se construye toda integración enterprise. [[RabbitMQ]] los implementa *out of the box*. Se agrupan por sub-familia: *Messaging Infrastructure* (el canal y su adapter), *Messaging* (el estilo de conversación), *Message Transformation* (enriquecer el mensaje), *Message Routing* (decidir su destino) y *System Management* (observabilidad).

### A.4 Message Channel

*EIP — Messaging Infrastructure | Chapters 4, 6*

**Intent.** Conectar dos componentes que quieren comunicarse mediante un **canal lógico de mensajes**, **desacoplando** producer y consumer: ninguno conoce al otro, solo al canal. Habilita que escalen y fallen de forma independiente.

**Structure.** Un `Sender` publica mensajes en un `Message Channel` (en RabbitMQ, una *queue* o un *exchange*); uno o más `Receiver` los consumen del canal. El canal es la pieza de infraestructura que media; producer y consumer no se referencian directamente.

![[B34134_Appendix_A_4.png]]
*Figure A.4 – Message Channel pattern*

> [!note] **Used in this book**
> - **Base del desacople producer/consumer (cap. 4)** — queues (FIFO) y topics desacoplan el orquestador RAG de los LLM consumers.
> - **Pipeline ETL de Airbyte (cap. 6)** — el Message Channel desacopla el producer (source) del consumer (destination), permitiendo que escalen independientemente.
> - **Loose coupling como requisito de producción** — es el mecanismo concreto que satisface el requisito de *loose coupling* de un sistema GenAI production-ready.

### A.5 Channel Adapter

*EIP — Messaging Infrastructure | Chapters 4, 6*

**Intent.** Conectar una aplicación (o un sistema externo) a un **messaging channel** sin que la lógica de negocio conozca el protocolo de mensajería. Es el Adapter (A.2) especializado en el límite entre la app y el broker.

**Structure.** Un `Channel Adapter` se sitúa entre la app y el `Message Channel`: publica/consume mensajes, serializa/deserializa, gestiona headers y traduce el protocolo (AMQP, HTTP). La lógica de negocio invoca métodos de dominio; el adapter los convierte en operaciones del broker.

![[B34134_Appendix_A_5.png]]
*Figure A.5 – Channel Adapter pattern*

> [!note] **Used in this book**
> - **Conexión al broker RabbitMQ (caps. 4, 8)** — aísla la lógica de negocio del protocolo AMQP (publish/consume, serialización, headers).
> - **Inbound channel adapters de Airbyte (cap. 6)** — message-driven o polling-based, traen data desde la fuente externa hacia el pipeline.
> - **Outbound channel adapters de Airbyte (cap. 6)** — sacan data hacia file/database/API (ej. hacia vector DBs Qdrant/[[Pinecone]]/Milvus).

### A.6 Publish–Subscribe Channel

*EIP — Messaging | Chapter 4*

**Intent.** Entregar **una copia de cada mensaje a cada consumidor interesado** (en vez de a un único consumer), de modo que múltiples receivers reaccionen al mismo evento. Es el canal de *broadcast* del messaging.

**Structure.** Un `Publisher` envía a un `Publish-Subscribe Channel` (en RabbitMQ, un *fanout/topic exchange*); el canal entrega una copia a **cada** `Subscriber` bindeado. Agregar un nuevo subscriber no afecta al publisher ni a los demás.

![[B34134_Appendix_A_6.png]]
*Figure A.6 – Publish–Subscribe Channel pattern*

> [!note] **Used in this book**
> - **Fan-out a múltiples consumers (cap. 4)** — el exchange entrega el mismo evento a varios consumers interesados.
> - **Multi-Agent Collaboration (cap. 8, Table 8.1)** — el bus de mensajes pub-sub permite que múltiples agentes/roles reaccionen al mismo mensaje.
> - **Scatter de un request a varias LLM queues** — base del fan-out que precede al [[Scatter-Gather]] dual-LLM.

### A.7 Content Enricher

*EIP — Message Transformation | Chapters 3, 4*

**Intent.** **Aumentar un mensaje con datos faltantes** que el receptor necesita pero que el emisor no tenía, consultando una fuente externa (una DB, un servicio) e inyectando el resultado en el mensaje antes de pasarlo adelante.

**Structure.** Un `Content Enricher` recibe el mensaje original, consulta un `Resource` externo (p. ej. una vector DB), y emite un mensaje **enriquecido** con la información agregada. El núcleo del RAG: enriquecer el prompt con los chunks recuperados.

![[B34134_Appendix_A_7.png]]
*Figure A.7 – Content Enricher pattern*

> [!note] **Used in this book**
> - **RAG = Content Enricher (caps. 3, 4)** — el patrón que **define** la retrieval-augmented generation: el request se enriquece con los chunks relevantes de la vector DB antes de llegar al LLM.
> - **Enrichment desde la vector DB (cap. 4)** — la consulta semántica que inyecta contexto grounded en el prompt.
> - **Pieza *unchanged* del case study dual-LLM (cap. 8)** — el Content Enricher (vector DB) se conserva intacto al migrar de single a dual LLM (Table 8.3).

### A.8 Request–Reply

*EIP — Messaging | Chapter 4*

**Intent.** Permitir una **conversación bidireccional** sobre mensajería asíncrona: el requestor envía un *request message* y espera un *reply message* correlacionado, replicando la semántica request-response sobre canales desacoplados.

**Structure.** Un `Requestor` publica en un *request channel* con un **Return Address** (la cola de reply) y un **Correlation Identifier**; el `Replier` procesa y publica la respuesta en esa reply queue con el mismo correlation ID, para que el requestor matchee el reply a su request original.

![[B34134_Appendix_A_8.png]]
*Figure A.8 – Request–Reply pattern*

> [!note] **Used in this book**
> - **Modelado del RAG como request-response augmentado (cap. 4)** — uno de los tres EIP que RabbitMQ provee casi gratis al configurar queues.
> - **Tool Use (cap. 8, Table 8.1)** — la reply correlacionada del tool al agente.
> - **El loop del agente** — cada invocación al LLM/tool es un request-reply sobre el broker.

### A.9 Correlation Identifier

*EIP — Messaging | Chapter 4*

**Intent.** Permitir que un requestor **matchee cada reply con su request original** cuando hay múltiples conversaciones en vuelo simultáneamente, marcando cada mensaje con un identificador único que el reply preserva.

**Structure.** El `Requestor` añade un `Correlation Identifier` al request; el `Replier` lo **copia** en el reply; el requestor mantiene un mapa de IDs pendientes y, al llegar una respuesta, la asocia a la request correcta por su ID.

![[B34134_Appendix_A_9.png]]
*Figure A.9 – Correlation Identifier pattern*

> [!note] **Used in this book**
> - **Matchear replies del LLM (cap. 4)** — el tercer EIP "gratis" del RAG sobre RabbitMQ.
> - **Correlation window del Aggregator (cap. 8)** — el `correlationId` permite reunir las respuestas de GPT-4 y Claude que pertenecen al mismo request original dentro de una ventana de tiempo.
> - **Threading del episodio ReAct (cap. 9)** — el `correlation_id` hila cada Thought → Action → Observation y aparece en cada log line.

### A.10 Scatter–Gather

*EIP — Message Routing | Chapter 4*

**Intent.** **Difundir un request a múltiples recipientes** que lo procesan en paralelo y **reunir (gather) sus respuestas** en un único resultado agregado. Combina un broadcast (scatter) con una agregación correlacionada (gather).

**Structure.** Un `Scatter` (Recipient List o Pub-Sub) envía el request a N recipientes; cada uno responde; un `Aggregator` (A re-usar) junta las N respuestas correlacionadas por su Correlation Identifier, espera hasta tenerlas todas o hasta un timeout, y emite el resultado combinado.

![[B34134_Appendix_A_10.png]]
*Figure A.10 – Scatter–Gather pattern*

> [!note] **Used in this book**
> - **Múltiples fuentes de enrichment (cap. 4)** — dispersar la consulta a varias fuentes y reunir el contexto.
> - **Dual-LLM del case study (cap. 8)** — el request se dispersa a una cola de GPT-4 y una de Claude; un **Aggregator** reúne ambas respuestas dentro de una **correlation window** y las fusiona con una merge strategy (best-of / ensemble / fallback).
> - **Orchestrator-Subagent (cap. 8, Table 8.1)** — fan-out a subagent queues y agregación de resultados.

### A.11 Dead Letter Channel

*EIP — Messaging | Chapter 4*

**Intent.** Dar un **destino para los mensajes que el sistema no puede entregar o procesar** (rechazados, expirados, que exceden length), de modo que no se pierdan ni bloqueen la cola activa, quedando disponibles para retry, inspección o root-cause analysis.

**Structure.** Cuando un mensaje es rechazado/expira, el broker lo enruta a un *dead-letter exchange (DLX)* y de ahí a una *dead-letter queue (DLQ)* bindeada. Encadenando DLXs se logra el *tiering* retry → parking → poison, gobernado por el `x-death` header que cuenta los rechazos.

![[B34134_Appendix_A_11.png]]
*Figure A.11 – Dead Letter Channel pattern*

> [!note] **Used in this book**
> - **DLQ del RAG (cap. 4)** — guardar mensajes no entregables al LLM sin bloquear la cola activa.
> - **DLQ de 3 tiers (cap. 8)** — retry / parking / poison según la failure class (Table 8.2), gobernada por el [[x-death header]].
> - **Cadena concreta en código (cap. 9)** — `dlq.retry1` (30s) → `dlq.retry2` (5min) → `dlq.quarantine`, con `x-message-ttl` encadenado.

### A.12 Message History

*EIP — System Management | Chapter 4*

**Intent.** Proveer **trazabilidad de la ruta que recorrió un mensaje** a través del sistema, adjuntándole una lista de los componentes por los que pasó, para depurar y auditar flujos distribuidos asíncronos.

**Structure.** Cada componente que procesa el mensaje **agrega su entrada** (timestamp, nombre del componente) a un `Message History` que viaja con el mensaje. El historial permite reconstruir el camino completo de punta a punta.

![[B34134_Appendix_A_12.png]]
*Figure A.12 – Message History pattern*

> [!note] **Used in this book**
> - **Observability del pipeline RAG (cap. 4)** — tracing del mensaje en cada hop.
> - **Auditability como requisito (cap. 8)** — cada decisión y cada mensaje trazable de punta a punta (requisito #9 production-ready).
> - **Logs hilados por correlation_id (cap. 9)** — el historial efectivo del episodio ReAct, visible en cada log line.

### A.13 Content-Based Router

*EIP — Message Routing | Chapter 4*

**Intent.** **Enrutar un mensaje a distintos destinos según su contenido** (un campo, un header, un tipo), centralizando la decisión de ruteo en un componente en vez de dispersarla por los consumers.

**Structure.** Un `Content-Based Router` inspecciona el mensaje y, según una regla sobre su contenido/headers, lo publica en el canal de destino correcto. En RabbitMQ se implementa con un *exchange* que rutea por *routing key* o *headers*.

![[B34134_Appendix_A_13.png]]
*Figure A.13 – Content-Based Router pattern*

> [!note] **Used in this book**
> - **Exchange que rutea por headers (cap. 4)** — el LLM routing del RAG según el contenido del mensaje.
> - **RoutingStrategy del producer (cap. 8)** — la decisión de a qué LLM/cola va el mensaje según prioridad/costo/tenant.
> - **Tool Use: una queue por tool (cap. 8, Table 8.1)** — topic exchange ruteando cada comando a la tool correcta.

### A.14 Pipes and Filters

*EIP — Message Routing | Chapters 5, 6*

**Intent.** Dividir un **procesamiento complejo en una secuencia de pasos independientes** (*filters*) conectados por canales (*pipes*), de modo que cada filtro haga una sola transformación y se puedan recomponer, reordenar o escalar individualmente.

**Structure.** Una cadena `Filter → Pipe → Filter → Pipe → …`: cada `Filter` consume del pipe de entrada, transforma, y publica en el pipe de salida; los `Pipe` son message channels. El ETL (Extract → Transform → Load) es el ejemplo canónico.

![[B34134_Appendix_A_14.png]]
*Figure A.14 – Pipes and Filters pattern*

> [!note] **Used in this book**
> - **El pipeline ETL (cap. 5)** — Extract → Embed → Load como filtros concurrentes conectados por canales.
> - **Airbyte: Load Data → Create Embeddings → Ingest Data (cap. 6)** — la arquitectura lineal S3 → Airbyte → Pinecone es un pipes-and-filters.
> - **Stages concurrentes** — cada stage escala independientemente al ser un filtro desacoplado.

## Part 3 — Reliability Patterns

Los patrones que **blindan las llamadas a dependencias poco fiables** — y en GenAI el LLM es el endpoint poco fiable por excelencia. Protegen al sistema de fallas transitorias y de cascadas de fallo.

### A.15 Circuit Breaker

*Reliability Pattern | Chapter 4*

**Intent.** **Evitar que un sistema reintente sin parar una operación que probablemente va a fallar**, cortando el flujo hacia una dependencia caída para darle tiempo a recuperarse y para no agotar recursos en llamadas condenadas (cascading failures).

**Structure.** Un `Circuit Breaker` envuelve la llamada y tiene tres estados: **Closed** (pasa todo, cuenta fallas), **Open** (rechaza inmediatamente sin llamar, tras superar un umbral de fallas) y **Half-Open** (deja pasar algunas llamadas de prueba; si pasan, vuelve a Closed; si fallan, vuelve a Open).

![[B34134_Appendix_A_15.png]]
*Figure A.15 – Circuit Breaker pattern*

> [!note] **Used in this book**
> - **Protección de llamadas al LLM (cap. 4)** — cortar el flujo a un LLM/dependencia que falla repetidamente.
> - **Modificador componible de Topologos (cap. 8)** — el flag `circuit-breaker` se agrega a la generación del patrón.
> - **Mejora de producción de los tool workers (cap. 9)** — los API clients reales se envuelven en circuit breakers.

### A.16 Retry with Exponential Backoff

*Reliability Pattern | Chapters 4, 7*

**Intent.** **Reintentar una operación fallida esperando intervalos crecientes** entre intentos, para superar fallas transitorias sin saturar la dependencia ni provocar *retry storms* cuando muchos clientes reintentan a la vez.

**Structure.** Tras una falla, el cliente espera un *back-off* que **crece exponencialmente** (1s, 2s, 4s, 8s…), opcionalmente con *jitter*, hasta un límite de reintentos; superado el límite, escala a otra estrategia (DLQ, parking). El conteo de intentos se lee del estado del mensaje (p. ej. el `x-death` count).

![[B34134_Appendix_A_16.png]]
*Figure A.16 – Retry with Exponential Backoff pattern*

> [!note] **Used in this book**
> - **Retry/fallback ante el LLM poco fiable (cap. 4)** — reintentar la llamada con back-off antes de dar por fallida la request.
> - **Dynamic throttling y back-off (caps. 5, 7)** — el back-off exponencial (1/2/4/8 s) como control de costo y de rate-limiting frente a las APIs de embedding/LLM.
> - **Tier 1 (retry) de la DLQ (caps. 8, 9)** — el re-encolado con back-off creciente del primer tier antes de bajar a parking/poison.

## Part 4 — Microarchitecture Patterns

Los patrones de **coordinación de componentes** (orchestration vs choreography) y la **microarquitectura GenAI** por excelencia (RAG). Son los patrones de más alto nivel: definen **quién decide el siguiente paso** y cómo se ensambla un sistema agéntico completo.

### A.17 Orchestration

*Microarchitecture Pattern | Chapters 1, 4, 7*

**Intent.** Coordinar múltiples componentes mediante un **controlador central** (*orchestrator* / process manager) que decide explícitamente el siguiente paso, invoca a cada componente y mantiene el estado del flujo. La lógica de control vive en **un solo lugar**.

**Structure.** Un `Orchestrator` central conoce el workflow completo: llama al componente A, recibe su resultado, decide y llama al B, y así sucesivamente. Los componentes no se conocen entre sí; solo conocen al orquestador, que centraliza la coordinación y la visibilidad.

![[B34134_Appendix_A_17.png]]
*Figure A.17 – Orchestration pattern*

> [!note] **Used in this book**
> - **Uno de los dos paradigmas de coordinación (cap. 1)** — soportado nativamente por [[RabbitMQ]] y los ESBs.
> - **El RAG orchestrator (cap. 4)** — el componente que coordina enrichment → LLM → reply.
> - **Plan-and-Execute / Orchestrator-Subagent (cap. 8, Table 8.1)** — el **Process Manager** EIP como orquestador que dispara y agrega los subagentes.

### A.18 Choreography

*Microarchitecture Pattern | Chapters 1, 7*

**Intent.** Coordinar componentes **sin un controlador central**: cada componente reacciona a eventos y emite los suyos, de modo que el flujo emerge de las reacciones locales (*event-driven*). No hay un único punto que conozca el workflow completo.

**Structure.** Los componentes se comunican por **eventos** sobre un canal pub-sub: cada uno escucha los eventos que le importan, hace su parte y publica nuevos eventos que disparan a otros. La coordinación es **distribuida**; no hay orquestador. Más resiliente y desacoplado, pero más difícil de razonar/monitorear globalmente.

![[B34134_Appendix_A_18.png]]
*Figure A.18 – Choreography pattern*

> [!note] **Used in this book**
> - **El otro paradigma de coordinación (cap. 1)** — contraparte distribuida de la orchestration.
> - **Multi-Agent Collaboration (cap. 8, Table 8.1)** — agentes que reaccionan a mensajes en un bus pub-sub sin un orquestador central.
> - **Lens B del análisis de Topologos (cap. 8)** — "¿hay un orchestrator central o coordinación distribuida (choreography)?" es una de las 4 lentes que clasifican el patrón.

### A.19 RAG Microarchitecture

*GenAI Microarchitecture | Chapters 2, 3, 4, 5*

**Intent.** El **patrón de despliegue GenAI por excelencia**: aumentar la generación de un LLM con **información recuperada** de una base de conocimiento (vector DB), de modo que el modelo responda **grounded** en datos verificables en vez de solo desde sus parámetros. Constriñe al LLM a un dominio acotado y verificable — lo trata como **interfaz de lenguaje, no como oráculo**.

**Structure.** Un pipeline en dos tiempos: **(1) ingestion** (documentos → chunks → embeddings → knowledge store) y **(2) query** (pregunta → retrieval de chunks relevantes → enrichment del prompt → generación del LLM → respuesta). Combina varios EIP: **[[Content Enricher]]** (el corazón), **[[Request-Reply]]**, **[[Correlation Identifier]]** y, al escalar, **[[Scatter-Gather]]**.

![[B34134_Appendix_A_19.png]]
*Figure A.19 – RAG Microarchitecture pattern*

> [!note] **Used in this book**
> - **El hilo conductor del libro (caps. 2, 3, 4, 5)** — de la intuición de embeddings (cap. 2) al ejemplo de Harry Potter (cap. 3) a la app production-grade (cap. 4) al proyecto de data migration (cap. 5).
> - **Building block de las arquitecturas agénticas (cap. 8)** — el RAG es el ladrillo que se extiende a dual-LLM y multi-agente.
> - **El deployment pattern más maduro (cap. 10)** — porque constriñe al LLM a un dominio grounded y verificable.

### A.20–A.23 — RAG en producción (4 sub-diagramas)

El apéndice cierra desglosando la **arquitectura RAG de producción** en cuatro sub-diagramas que detallan las decisiones reales de un sistema RAG operativo: cómo se recupera, cómo se chunkea, cómo fluyen los embeddings y cómo se sirve una query en producción.

**A.20 — Hybrid retrieval.** Combina **retrieval léxico** ([[BM25]], que captura exact matches y términos raros) con **retrieval vectorial** (que captura similitud semántica), y **fusiona ambos rankings** antes del ensamblado del contexto, cubriendo tanto las coincidencias exactas como la variación semántica que cada método por separado se pierde.

![[B34134_Appendix_A_20.png]]
*Figure A.20 – Hybrid retrieval*

**A.21 — Chunking strategies.** Las estrategias para dividir documentos antes de embeber: **fixed-size** (cortar cada N tokens, simple pero rompe la semántica), **semantic** (cortar en límites de significado) y **structure-aware** (respetar la estructura del documento: secciones, párrafos, headings). La elección condiciona qué chunks se recuperan y la calidad del retrieval.

![[B34134_Appendix_A_21.png]]
*Figure A.21 – Chunking strategies*

**A.22 — Embedding flow.** El flujo que convierte los **chunks en vectores** y los carga en un **knowledge store indexable**: cada chunk pasa por el embedding model, produce un vector, y se almacena con su metadata en la vector DB lista para búsqueda por similitud.

![[B34134_Appendix_A_22.png]]
*Figure A.22 – Embedding flow*

**A.23 — Production query flow.** El flujo completo de una query en producción, en cuatro etapas: **rewrite** (reformular/expandir la pregunta del usuario) → **retrieval** (recuperar los chunks candidatos, típicamente con hybrid retrieval) → **reranking** (reordenar los candidatos por relevancia fina) → **generation** (el LLM genera la respuesta grounded en los chunks reordenados).

![[B34134_Appendix_A_23.png]]
*Figure A.23 – Production query flow*

> [!note] **El párrafo de cierre del apéndice.** La aparente novedad de la arquitectura GenAI descansa sobre **fundamentos de ingeniería familiares**: queues, adapters, routers, enrichers, retry policies, orchestration y choreography. **RAG no reemplaza la arquitectura clásica — la operacionaliza.** Por eso dominar este catálogo de patrones es, literalmente, dominar el lenguaje con el que se construyen, explican y evalúan los sistemas agénticos.

## Para aplicar

- **Nombrá el patrón antes de codear** — el [[Pattern-Guided Coding|pattern-guided coding]] de los caps. 4 y 8 funciona porque anclás la generación en patrones nombrados (GoF + EIP), que dan un esqueleto verificable y reproducible; usá este apéndice como el diccionario de esos nombres.
- **Mapeá problema → GoF/EIP/Reliability/Microarchitecture** — ante un requerimiento (routing, correlación, resiliencia, coordinación), buscá en la Table A.1 la categoría correspondiente: comportamiento variable → [[Strategy]]; interfaces incompatibles → [[Adapter]]; conversación bidireccional → [[Request-Reply]]; reunir N respuestas → [[Scatter-Gather]] + [[Aggregator]]; dependencia caída → [[Circuit Breaker]]; falla transitoria → [[Retry with Exponential Backoff]]; ¿quién decide? → [[Orchestration]] vs [[Choreography]].
- **Usá las fichas como checklist de estudio** — cada ficha es Intent → Structure → usos; repasarlas es repasar el vocabulario completo del libro, suficiente para reconstruir su argumento sin volver a los capítulos.
- **Para RAG de producción, decidí las 4 dimensiones** — hybrid retrieval (léxico + vectorial fusionados), chunking strategy (fixed / semantic / structure-aware), embedding flow (chunk → vector → store) y query flow (rewrite → retrieval → rerank → generation): el apéndice las separa porque cada una es una decisión de diseño explícita.
- **Tratá los EIP como ya implementados** — [[RabbitMQ]] los provee out of the box; al diseñar, asumí Message Channel / Channel Adapter / Content Enricher / Correlation Identifier / Dead Letter Channel disponibles y construí sobre ellos en vez de reinventarlos.

## Conexiones

- [[_Building Complex Multi-Agent Systems Using Pattern Prompting|Building Complex Multi-Agent Systems]] — el MOC del libro: este apéndice es su **glosario de patrones**.
- [[10 - The Future and Limitations of LLMs]] — capítulo (conceptual) anterior; cierra el cuerpo del libro y este apéndice es la referencia que lo acompaña.
- [[A2 - Appendix B - Topologos User Manual|Appendix B]] — el otro apéndice: el manual de referencia del sistema **[[Topologos]]** que *aplica* todos estos patrones.
- [[08 - Pattern-Guided Coding]] — el capítulo que **usa este vocabulario** de forma más intensiva: su tesis *"GoF and EIP patterns are the patterns; everything else is the problem"* es la justificación de por qué este apéndice existe; la **Table 8.1** mapea cada GenAI workflow pattern a estos GoF/EIP. (Su placeholder `[[Appendix A - Patterns]]` resuelve a esta nota.)
- [[04 - Building Your First RAG App]] — el capítulo que **introduce la mayoría** de estos patrones (GoF del LLM client + EIP del RAG + reliability + DLQ).
- **GoF** (Part 1): [[Strategy]] · [[Adapter]] · [[Template Method]] — y los GoF adicionales del cap. 8 que este catálogo contextualiza: [[Command]] · [[State]] · [[Mediator]] · [[Composite]] · [[Observer]] · [[Chain of Responsibility]].
- **EIP** (Part 2): [[Message Channel]] · [[Channel Adapter]] · [[Publish-Subscribe Channel]] · [[Content Enricher]] · [[Request-Reply]] · [[Correlation Identifier]] · [[Scatter-Gather]] · [[Dead Letter Channel]] · [[Message History]] · [[Content-Based Router]] · [[Pipes and Filters]] · y los relacionados [[Aggregator]] · [[Process Manager]] · [[Recipient List]] · [[Invalid Message Channel]].
- **Reliability** (Part 3): [[Circuit Breaker]] · [[Retry with Exponential Backoff]].
- **Microarchitecture** (Part 4): [[Orchestration]] · [[Choreography]] · [[RAG Microarchitecture]].
- **RAG de producción**: [[Hybrid Search]] · [[BM25]] · [[Chunking]] · [[Embeddings]] · [[Vector Database]] · [[Reranking]] (candidato a nota propia) — las decisiones de los sub-diagramas A.20–A.23.
- Patrones fundacionales del vault: [[GoF]] · [[Enterprise Integration Patterns]] · [[Design Patterns]] · [[Reliability Patterns]] (candidato a nota propia) · [[Microarchitecture]].
