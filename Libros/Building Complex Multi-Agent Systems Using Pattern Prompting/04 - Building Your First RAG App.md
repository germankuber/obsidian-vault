---
title: Building Your First RAG App
libro: Building Complex Multi-Agent Systems Using Pattern Prompting
autor: Packt
capitulo: 4
created: 2026-06-20
tags:
  - libros/building-complex-multi-agent-systems-using-pattern-prompting
  - type/case-study
  - status/permanent
aliases:
  - Building Your First RAG App
  - Cap 4 - Building Your First RAG App
updated: 2026-06-22
---
# Building Your First RAG App

> [!info] Capítulo 4 · *Building Complex Multi-Agent Systems Using Pattern Prompting* — Packt (ISBN 9781806114290)
> El capítulo marca el **shift de concepto a implementación**: cierra la brecha entre los **prototipos de [[RAG]]** (los tutoriales de internet) y una app **production-ready** robusta, segura y escalable — brecha que contribuyó a la **alta tasa de fallos de apps GenAI en 2025**. La construye con **[[Pattern-Guided Coding|pattern-guided coding]]** (una forma avanzada de [[Vibe Coding|vibe coding]]): se expresa la arquitectura mediante **[[GoF]]** y **[[Enterprise Integration Patterns|Enterprise Integration Patterns (EIP)]]** (descritos en [[A1 - Appendix A - Pattern Reference|Appendix A]]) y se implementan sobre **[[RabbitMQ]]**, que soporta nativamente los patrones que RAG necesita. RAG es el patrón GenAI simple y muy usado, además del **building block fundacional para arquitecturas agénticas** que el cap. 8 retoma. Navegá: [[_Building Complex Multi-Agent Systems Using Pattern Prompting|Building Complex Multi-Agent Systems]] · anterior [[03 - Building with GenAI Parameters, Tuning, and Project Phases]] · siguiente [[05 - Starting Your Data Migration Project]].

## Resumen

Hay muchísimos tutoriales de RAG en internet, excelentes para arrancar, pero **no están diseñados para enseñar a construir aplicaciones robustas, seguras y escalables**. Se los trata erróneamente como production-ready, y cuando se despliegan en producción **fallan con frecuencia** — esto contribuyó, en parte, a la **alta tasa de fallos de apps GenAI en 2025**. Este capítulo cierra esa brecha entre **apps reales y prototipos** y muestra cómo construir una **app RAG robusta** que aguante las demandas de un entorno de producción real. La técnica central es el **[[Pattern-Guided Coding|pattern-guided coding]]**, una forma avanzada de **AI-assisted ("vibe") coding** que construye aplicaciones —acá una arquitectura RAG— **expresando ideas mediante software & integration patterns**. El enfoque es intuitivo: los patrones son la **lingua franca** de los ingenieros (aparecen en diagramas de arquitectura, documentación y design discussions), así que sirven naturalmente como **vocabulario para comunicar intención al LLM** al generar código.

El capítulo propone un **enfoque estructurado** para arquitecturar soluciones GenAI: primero se descompone el sistema en **[[GoF|Gang of Four (GoF)]]** y **[[Enterprise Integration Patterns|Enterprise Integration Patterns (EIP)]]** familiares (descritos en [[A1 - Appendix A - Pattern Reference|Appendix A]]); después se evalúan concerns clave como **scaling, failover, security, error handling, logging, retries e integration**; y finalmente se examina cómo se implementan en la práctica, específicamente con **[[RabbitMQ]]**, donde el soporte built-in reduce el esfuerzo de desarrollo. RAG sirve como **patrón GenAI simple y muy usado** y a la vez como **building block fundacional para arquitecturas agénticas más avanzadas**.

El arco recorre cinco temas. Primero, **qué significa "production-grade"**: la **Tabla 4.1** lista 12 características que un ingeniero debe atender antes de desplegar, y RabbitMQ —hardened en producción— ayuda a satisfacerlas. Segundo, **las elecciones de tecnología**: por qué NO elegir LangChain/LlamaIndex/n8n/DSPy para enterprise (no llevan tiempo suficiente probado, agregan conveniencia donde ya hay productos maduros, el value del low-code cae con vibe coding), el dato de que en 2025 **menos del 5% de las apps GenAI** dieron valor según MIT, y las ventajas de **queues** (FIFO, producer/consumer, guaranteed delivery, dead-letter) y **topics** (multi-consumer), más un tour de la **Management UI** de RabbitMQ. Tercero, **construir la app RAG** como un request-response augmentado: el **Content Enricher**, los 3 EIP (**Request-Reply**, **Content Enricher**, **Correlation Identifier**), y la evolución a **topics + Scatter-Gather** con LLM routing, dead-letter queues, message history y LLM reliability vía un **exchange**. Cuarto, **el LLM client** como message consumer production-ready: 19 capabilities y los 3 GoF (**Template Method**, **Adapter**, **Strategy**). Quinto, **generar código y deployment**: vibe coding produce el código y un `definitions.json` de RabbitMQ con toda la topología (infrastructure as code). El cap. 5 sigue con **data pipelines / ETL** hacia la vector DB; el cap. 8 retoma esta arquitectura para "agentic patterns".

## What does "production-grade" software mean?

Construir apps de AI puede ser **engañosamente fácil**: escribís un prompt, elegís uno de los muchos "agentic frameworks" y arrancás. Eso es genial y democratiza el desarrollo de software como nunca, pero **es un error asumir que lo construido así puede vivir en producción**. La mayoría de los ingenieros con experiencia sabe lo exigentes que son los entornos de producción. Sin la preparación adecuada, una app probablemente **se quede sin recursos**, se vuelva **intolerablemente lenta** bajo peak load, sea **comprometida** por falta de arquitectura de seguridad, o se convierta en una **isla aislada** de las demás apps porque es demasiado difícil de integrar.

La **[[Production-Grade Software|production-grade software]]** es lo que cierra esa brecha. Para que la app RAG satisfaga estos requisitos estrictos, el libro la construye con **[[RabbitMQ]]**, un producto **hardened en producción** que ayuda a satisfacer las características de la tabla siguiente.

> [!note] **Pattern-guided coding**: forma avanzada de AI-assisted ("vibe") coding que construye aplicaciones expresando ideas mediante **software & integration patterns**. Como los patrones son la lingua franca de los ingenieros, son también un **vocabulario efectivo para comunicar intención al LLM** al generar código.

### Tabla 4.1 — Requirements of production environments

| # | Characteristic | What It Means | Why It Matters/Typical Techniques |
|---|---|---|---|
| 1 | **Secure** | Protects data, logic, and users from malicious actors | Input validation/sanitization, OWASP Top 10 mitigation, least privilege, secret management, WAF, regular security scanning, zero-trust patterns |
| 2 | **Reliable** | Does the correct thing consistently over long periods | High availability (99.9%+), low MTTR, retry/back-off/circuit-breaker patterns, idempotency, exactly-once semantics where needed |
| 3 | **Resilient/Fault-tolerant** | Keeps working (possibly degraded) when things break (dependencies, network, hardware) | Graceful degradation, bulkheads, timeouts, retries, fallback/cached data, chaos engineering |
| 4 | **Observable** | You can quickly understand what is happening inside (and why) | Structured logging, distributed tracing (OpenTelemetry), rich metrics (Prometheus/OTel), meaningful alerts, good error messages |
| 5 | **Maintainable/Evolvable** | Easy and safe to change, extend, and refactor over years without fear | Clean architecture, high test coverage, small modules, clear contracts, good naming, low coupling |
| 6 | **Performant and Scalable** | Handles expected and unexpected load without falling over | Horizontal scaling, efficient algorithms, caching, async processing, capacity planning, auto-scaling |
| 7 | **Correct Under Stress / Input Validation** | Survives invalid, malformed, huge, malicious, or missing inputs gracefully | Strong input validation and sanitization, schema validation, boundary checks, fail-fast |
| 8 | **Recoverable** | Can get back to a healthy state after failure (self-healing) | Automatic restarts, health checks and readiness/liveness probes, blue-green/canary, rollback automation |
| 9 | **Testable** | Can be thoroughly tested at multiple levels (unit → contract → integration → e2e → chaos) | High automation coverage, contract testing, property-based testing, Testcontainers, mutation testing |
| 10 | **Operable/Production-ready** | Easy to deploy, monitor, debug, and operate in real environments | Good CI/CD, infrastructure as code, configuration via environment variables, feature flags, runbooks, SLO/SLI definition |
| 11 | **Secure-by-default and Least Surprise** | Safe and predictable defaults; avoids dangerous shortcuts | Secure headers, no hard-coded credentials, principle of least astonishment in APIs/UI |
| 12 | **Backward and Forward Compatible (when applicable)** | Can evolve APIs/data schemas without breaking existing consumers | Semantic versioning, API versioning, backward-compatible changes, deprecation periods |

> [!tip] La tabla es una **lista parcial** de consideraciones que un ingeniero debe atender **antes de desplegar a producción**. Construir el RAG sobre RabbitMQ hace que la app **herede muchas de estas características** en lugar de implementarlas a mano.

## Making technology choices: advantages of queues and topics

Si investigaste el tooling landscape, sabés que hay **docenas, incluso cientos** de tools que se venden como plataformas para apps agénticas: **LangChain, LlamaIndex, n8n, DSPy**, y más. ¿Por qué no elegir una de esas? La respuesta es directa si construís para el **enterprise**: esas tools **no llevan tiempo suficiente** para demostrar que aguantan las demandas rigurosas de los sistemas de producción. Agregan una **capa de conveniencia** a un espacio donde ya existen productos maduros y probados, y con la llegada del **vibe coding** la value proposition del low-code **se diluye rápidamente**. Es un value-add endeble cuando además sumás los **requisitos extra de aprendizaje** y un **futuro incierto** en un espacio que verá disrupción continua.

Hay un argumento más a favor de usar tools y procesos existentes: las apps GenAI **no viven en el vacío**. Cuanto más valor agregan, mayor prioridad tiene integrarlas con otros componentes de back-end. Es siempre un objetivo primario de IT **reducir la multiplicidad** de tecnologías, lenguajes y tools, porque cada tool extra tiene **costos ocultos**: mantener skills, optimizar hardware y gestionar el riesgo de obsolescencia.

> [!warning] **Dato MIT**: reportes indican que en **2025 menos del 5% de las apps GenAI** se consideraron generadoras de valor para sus organizaciones. Una razón: las organizaciones **eligieron tecnologías que no estaban listas** para implementación más allá de la fase de prototipo.

Contra ese telón de fondo, importa considerar tools **probadas, confiables y bien aptas para enterprise integration**. **[[RabbitMQ]]** destaca: ofrece soporte **maduro y production-ready** de messaging e integration patterns.

### Why RabbitMQ for enterprise-grade GenAI systems

RabbitMQ da respuestas **out-of-the-box** a muchas de las preguntas planteadas, y el resto se satisface configurándolo para los requisitos específicos. Se lo usa para implementar integration patterns con **queues**, así que conviene una discusión breve.

Como implica el nombre, las **queues** implementan un patrón de mensajería **first-in, first-out (FIFO)**. La mayoría de los sistemas de queuing enterprise consideran **dos roles**: **producers** (envían mensajes a la queue) y **consumers** (reciben, o "consumen", mensajes de la queue). Los sistemas de mensajería implementan integration patterns para soportar múltiples producers y consumers, que intercambian mensajes **sync o async y de forma segura**. Features adicionales: **guaranteed delivery** (los mensajes no pueden desaparecer) y **[[Dead-Letter Queue|dead-letter queues]]** (si un consumer no responde, el mensaje se guarda y se puede reintentar luego).

> [!note] En este capítulo: los **producers** son las entidades que quieren invocar un LLM; los **consumers** implementan la lógica de los **LLM clients**. RabbitMQ enruta la respuesta del consumer (LLM client) de vuelta al producer y la **correlaciona** con el mensaje correspondiente que se envió.

![[B34134_4_1.png|526]]
*Figure 4.1 – Queue with multiple producers and consumers*

Similares a las queues son los **topics**. Las queues imponen la regla de que **solo un consumer** puede dequeue un mensaje dado; una vez dequeued, el mensaje **se elimina** de la queue y ningún otro consumer puede dequeue-arlo. Los topics, en cambio, permiten que **múltiples consumers** dequeue y procesen el **mismo mensaje**. Combinaciones de queues y topics se componen en una **topología** que satisface requisitos específicos.

![[B34134_4_2.png|907]]
*Figure 4.2 – Topic with multiple producers and consumers*

> [!tip] Usar queues y topics **desacopla producers y consumers**, logrando varios objetivos del buen diseño: **flexibilidad, adaptabilidad y escalado simplificado**. La mayoría de los bottlenecks se resuelven **agregando más producers, queues o consumers**. La buena noticia: no hay que arquitecturar a nivel de queue — solo se **identifican los patrones EIP** que cumplen los objetivos y se configura RabbitMQ para implementarlos, o se usa pattern-guided coding para prescribir la solución.

### Quick tour of RabbitMQ

Resumiendo lo cubierto, **RabbitMQ** ofrece muchas ventajas:
- Es **open source**.
- Implementa **GoF y EIP patterns out of the box**.
- **Escala muy bien** bajo heavy load.
- **Garantiza** que los mensajes no se pierdan.
- Es confiable y usado por organizaciones como **Uber, Airbnb, NASA y WhatsApp**, lo que **reduce el riesgo de adopción** y lo hace fácil de justificar en entornos enterprise.
- Provee el **balance correcto entre control y abstracción**: configurás soluciones mientras la mecánica subyacente permanece visible y entendible.

A lo largo de años de despliegues, RabbitMQ acumuló features valiosas. Una de las más útiles es el **admin interface**, que permite a los administradores ver métricas de performance, configurar queues e inspeccionar el estado del sistema. Como los sistemas GenAI pueden depender de **third-party systems**, es valioso ir directo a la **root cause** de la latencia u otros problemas; la visibilidad en tiempo real del messaging system es la forma más rápida de hacer **root cause analysis**. Si un componente está sufriendo, a menudo la **primera señal** son **mensajes acumulándose en la queue**.

La **Management UI** de RabbitMQ ofrece un **dashboard web** para monitorear, gestionar y administrar un broker o cluster. Cada tab principal se enfoca en un aspecto, con métricas en tiempo real, listas y acciones de management (según permisos y tags como `management`, `monitoring` o `administrator`):

- **Overview** — landing page para un health check rápido. Muestra estadísticas agregadas de cluster/node: charts de **total queued messages** (ready y unacknowledged), **global message rates** (publish, deliver, acknowledge, etc.), **resource usage** (memory, disk, file descriptors, CPU vía Erlang runtime) y **totales** de connections, channels, queues y exchanges. Ideal para monitoreo at-a-glance y detectar bottlenecks.

![[B34134_4_3.png|967]]
*Figure 4.3 – Overview tab*

- **Connections** — lista todas las **conexiones de cliente activas**. Muestra username, virtual host, IP/port del cliente, node, channel count, SSL status, heartbeat, data transfer rates y state. Sirve para monitorear actividad, identificar conexiones idle o de alto tráfico, y cerrar las problemáticas (con permisos suficientes).

![[B34134_4_4.png|817]]
*Figure 4.4 – Connections tab*

- **Channels** — muestra todos los **AMQP channels abiertos** (sesiones lógicas dentro de las connections). Incluye prefetch settings, unacknowledged messages, confirm mode, publish/acknowledge rates y detalles de connection/queue vinculados. Útil para debug a nivel de channel, flow control o performance.

- **Exchanges** — lista todos los **exchanges** (incluidos defaults como `amq.direct`). Muestra **type** (direct, fanout, topic, headers), durability, message rates (in/out), bindings y status. Sirve para declarar/eliminar exchanges, inspeccionar routing rules vía bindings y monitorear el tráfico.

![[B34134_4_5.png|1141]]
*Figure 4.5 – Exchanges tab*

- **Queues and Streams** — muestra todas las queues con métricas críticas: **message counts** (ready, unacknowledged, total), consumer count/utilization, rates (publish, deliver, ack), memory/disk usage, state y features (durable, auto-delete). Tab primario para queue management: **purge messages, declare o delete queues, peek o get messages** (en versiones soportadas) y troubleshoot de backlogs o consumers lentos.

![[B34134_4_6.png|935]]
*Figure 4.6 – Queues and Streams tab*

- **Admin** — hub administrativo (visible para usuarios con tag `administrator` o similar). Sub-secciones: **Users** (add/delete users, passwords, tags, permissions); **Virtual Hosts** (crear/eliminar vhosts y gestionar acceso); **Policies** (definir y aplicar policies como **HA, TTL y max-length** a queues o exchanges); y **Parameters/Limits/Feature Flags** (runtime settings, resource limits y enable de preview features).

> [!note] La UI es **interactiva**: clickear ítems (p.ej. el nombre de una queue) abre vistas detalladas con sub-tabs de **rates, consumers, bindings** y más. Los **permisos** controlan qué puede ver o modificar cada usuario, desde read-only monitoring hasta full administrator. Es una herramienta esencial para observar el comportamiento en tiempo real, diagnosticar problemas y administrar el día a día **sin llamadas a CLI o API**.

## Building a production-class RAG application using RabbitMQ with integration patterns

El **primer paso** para construir la app RAG es **identificar los patrones** que se necesitan; el **segundo** es **configurar los patrones y la queue topology**. Para los vibe coders, la mayoría de los LLMs **conocen bien estos patrones** y dan implementaciones y configuraciones confiables — aunque **siempre hay que verificar** que el output sea correcto.

Una **arquitectura RAG se parece a un straightforward request-response pattern** con una excepción notable: el **mensaje enviado por el producer se augmenta con datos recuperados en el camino**. El **RAG orchestration component** envía un request al **vector database client**, que devuelve **contexto para mergear** en el mensaje antes de reenviarlo al LLM.

![[B34134_4_7.png]]
*Figure 4.7 – RAG architecture with RabbitMQ*

La Fig 4.8 ilustra un integration pattern llamado **[[Content Enricher]]**: en el camino del mensaje hacia el consumer, se **fetchea data externa** y se usa para **augmentar el mensaje**. Con el esfuerzo relativamente chico de configurar queues, se obtienen **tres patrones importantes**:

- **[[Request-Reply]]** — asegura que el message producer **no se bloquee** esperando una respuesta.
- **[[Content Enricher]]** — enriquece el mensaje **in-flight** para darle al LLM data más relevante.
- **[[Correlation Identifier]]** — **matchea el mensaje enviado con el reply**. Los producers **no consumen threading resources** esperando respuesta; simplemente **se registran para una notificación**.

> [!note] Una descripción de estos patrones está disponible en [[A1 - Appendix A - Pattern Reference|Appendix A]].

Con las decisiones arquitectónicas tomadas hasta acá, la app tiene:
- **Scalability** — se pueden **agregar queues** y subir/bajar el número de producers y consumers.
- **Easy integration** — los sistemas simplemente **encolan un mensaje y son notificados** de la respuesta.
- **Monitoring** — RabbitMQ trae una **admin UI** para ver métricas y configurar notificaciones.

Es una base sólida y para algunas apps se pararía ahí. Pero si el RAG debe dar **alto valor**, debe ser **robusto y flexible**. Inicialmente se asumió que hay **una sola fuente** de enrichment data, pero podría haber **dos, tres o más**. Las empresas tienen muchas fuentes más allá de la vector DB: **content management systems, file stores, relational databases**; varios departamentos corren cada uno **su propia vector database**. Una **orden**, por ejemplo, podría requerir contexto de un servicio **Customer**, uno **Pricing** y uno **Inventory**. Para acomodar **múltiples content-enrichment producers**, se **cambian queues por topics** y se agrega lógica para **agregar las respuestas** — esto es el **[[Scatter-Gather]]**.

![[B34134_4_8.png]]
*Figure 4.8 – Content Enricher – Scatter-Gather with RabbitMQ*

Capacidades adicionales que mejoran flexibilidad y robustez:
- **LLM routing** — querés mandar distintos requests a **distintos LLMs**, por **costo** o porque algunos rinden mejor en ciertas tareas: rutear algunos requests a **LLM A** y otros a **LLM B**. En esta arquitectura, **el producer toma la decisión de routing**.
- **[[Dead-Letter Queue|Dead-letter queueing]]** — ¿qué pasa si un mensaje **no puede entregarse al LLM** (network glitch, technical failure, LLM caído, o mensaje rechazado en el camino)? Descartarlo no es buena opción, ni dejarlo en una **active queue** donde bloquea otros mensajes. Mandar los mensajes fallidos a una **dead-letter queue** es lo mejor: una vez guardado, el mensaje **se recupera luego** o se usa para **root cause analysis**.
- **Maintaining message history** — para **transparencia y debugging**, trackear los componentes por los que pasó un mensaje manteniendo un **message history**. Valioso para troubleshooting y para entender **cómo se distribuye la carga**.
- **Ensuring LLM reliability** — como se discutió en el **[[02 - Embeddings The Language of AI|Capítulo 2]]**, los LLMs se comportan como **web service / API endpoints poco fiables, sin uptime guarantees**. Implementar **retry y fallback robustos** es esencial.

> [!note] Para implementar estas features se configura un **exchange**, que es **como un centro de clasificación de correo** donde las **queues son los buzones**. El exchange **enruta el mensaje a LLM A o LLM B según los message headers**.

![[B34134_4_9.png]]
*Figure 4.9 – Advanced RAG with robust safe-fallback strategy*

## Building the LLM client

Un componente crítico es el que **invoca al LLM**: cumple el rol de **message consumer**. Como la arquitectura misma, debe ser **production-ready** para manejar **miles de transacciones** con fiabilidad y gestionar con elegancia cualquier issue del LLM. La **lista de capabilities/behaviors** que se quieren para un client seguro y confiable (son **19**):

> [!note] Las **19 capabilities** del LLM client: Authentication strategy · Retry policy · Rate-limit handling · Prompt formatting · Response parsing · Tool-calling policy · Streaming policy · Request builder · Response parser · Tokenizer wrapper · Streaming handler · Error mapper · Logging · Metrics · Caching · Tracing · Retries · Cost tracking · Safety filtering.

Es una lista larga, pero para **no perder clientes** ni sufrir downtime por sorpresas desagradables, el componente debe contemplar las 19. ¿Cómo encararlo? **No** se quiere construir un **client separado por cada LLM** (enorme duplicación de código). Algunos behaviors no se necesitan al inicio; otros son requeridos para algunos services y no para otros. La respuesta es usar **patrones, específicamente los [[GoF]]**.

### Tabla 4.A — Los 3 GoF patterns del LLM client

| Patrón | Rol | Para qué |
|---|---|---|
| **[[Template Method]]** | Centraliza el **request lifecycle** en un lugar | Previene lógica duplicada entre providers; la base class es dueña del algoritmo |
| **[[Adapter]]** | **Aísla las API specifics** de cada LLM | Cambiar de LLM requiere a lo sumo construir un **nuevo adapter** |
| **[[Strategy]]** | Vuelve **pluggable** el comportamiento variable | Evita hard-codear behaviors en subclasses y la **subclass explosion** |

### Template Method

Usar una **abstract base class** para definir el workflow común, en **5 pasos ordenados**:
1. **Validate input**
2. **Build a provider request**
3. **Send the request**
4. **Parse the provider response**
5. **Normalize the output**

> [!note] La **base class es dueña del algoritmo**. Las **subclasses llenan los pasos provider-specific**.

### Adapter

Crear un **provider adapter por vendor**, de modo que **todos los providers expongan la misma interfaz interna** aunque sus APIs difieran. Por ejemplo:
- **OpenAI adapter**
- **Anthropic adapter**
- **Gemini adapter**
- **Local vLLM adapter**

El client **habla con el adapter, no directamente con los raw vendor payloads**.

### Strategy

Pasar **objetos intercambiables** al client para los behaviors que **varían**, evitando la **subclass explosion**:
- **Retry policy**
- **Authentication policy**
- **Timeout policy**
- **Prompt formatting**
- **Streaming behavior**
- **Tool-calling behavior**

> [!tip] **Nota de diseño en Python**: a menudo **no se necesita herencia pesada** por provider. Un enfoque más **pythónico**: **un reusable orchestrating client** + **adapter objects** por provider + **strategy objects** para behavior variable. Aunque el Template Method puede implementarse con una abstract base class, **muchos equipos mantienen la herencia shallow** y dejan que la **composición** haga la mayor parte del trabajo.

> [!tip] **Regla práctica fuerte**: usar **Template Method** para el **request lifecycle estable**, **Adapter** para las **diferencias de provider**, y **Strategy** para **policies y variaciones de comportamiento**.

![[B34134_4_10.png]]
*Figure 4.10 – LLM Client Design*

Una descripción de estos patrones está en [[A1 - Appendix A - Pattern Reference|Appendix A]]. Hoy, **aplicar patrones solo requiere pattern-guided coding**: se enciende el mejor code-generating LLM y se construye la app con el proceso de pattern-guided coding.

## Generating code and deployment configuration

El **AI-assisted coding (vibe coding)** permite implementar código y configuration files para **arquitecturas pattern-based** con muy poco o **ningún coding manual**. Se usó pattern-guided coding para generar el código y las configuraciones que matchean la arquitectura de la Fig 4.10.

> [!note] La figura referida como **Figure 4.11** en el texto **no es una imagen embebible** del capítulo (la única imagen `4_11` es un QR code de promo "Subscribe for a free eBook", omitido); se trata como referencia en prosa al deployment artifact.

### Deploying our solution

Los detalles de la arquitectura se capturan en un archivo de RabbitMQ llamado `definitions.json`. **Varios LLMs pueden generar** el `definitions.json` y **todos los archivos y scripts** necesarios para instalar la arquitectura, **portable a Linux, macOS y Windows**. No hace falta escribir ningún archivo a mano — una tarea onerosa en la era pre-AI.

La **folder structure** de los deployable artifacts:

```text
rabbitmq-canonical/
  docker-compose.yml
  .env
  rabbitmq/
    rabbitmq.conf
    enabled_plugins
    definitions.json
```

Esta estructura contiene **deployments declarativos y reproducibles**, y combina **[[Docker Compose]] orchestration, environment secrets, server configuration, plugin activation y la queue/topic topology entera**. Breakdown file-by-file:

- **`docker-compose.yml`** — el config principal de Docker Compose. Define el **service/container** de RabbitMQ; especifica la **imagen** (ej. `rabbitmq:3-management`), **puertos** (`5672` para AMQP, `15672` para la management UI), **volumes**, **environment variables**, **health checks**, etc. Se usa para arrancar el container de forma consistente con `docker compose up`.
- **`.env`** — el archivo de **environment variables**, cargado automáticamente por Docker Compose. Guarda settings **sensibles o configurables** (ej. `RABBITMQ_DEFAULT_USER`, `RABBITMQ_DEFAULT_PASS`). Mantiene **credenciales fuera** de archivos versionados (seguridad y portabilidad). Referenciado en `docker-compose.yml` con la sintaxis `${VARIABLE_NAME}`.
- **`rabbitmq/rabbitmq.conf`** — el **server configuration file** de RabbitMQ (formato moderno **sysctl-style**). Montado en `/etc/rabbitmq/rabbitmq.conf` dentro del container. Configura settings core: **listeners, logging, memory limits, TLS, clustering**, etc. Comúnmente se usa para habilitar **auto-loading de definitions** (ej. `management.load_definitions = /etc/rabbitmq/definitions.json`).
- **`rabbitmq/enabled_plugins`** — un **text file simple** que activa plugins de RabbitMQ al startup. Montado en `/etc/rabbitmq/enabled_plugins`. Contenido típico: `[rabbitmq_management].` (habilita la web UI y la HTTP API). Puede incluir otros plugins como `rabbitmq_prometheus`, `rabbitmq_shovel`, etc. **Sin este archivo** (o la entrada), los plugins quedan **deshabilitados**.
- **`rabbitmq/definitions.json`** — un **JSON** con las **broker topology definitions** (users, vhosts, permissions, exchanges, queues, bindings, policies, etc.). Montado dentro del container y **auto-importado al startup** si está configurado en `rabbitmq.conf`. Habilita **"infrastructure as code"** — setup reproducible entre entornos. Usualmente **exportado** desde una instancia funcionando vía la management UI o la herramienta `rabbitmqadmin`.

> [!tip] Estos archivos están disponibles en el **repositorio de GitHub** del libro, junto con un **README** con instrucciones completas de instalación.

## Citas

> "This has contributed, in part, to the high rate of failures of GenAI applications in 2025."

> "Pattern-guided coding building applications, in our case, a RAG architecture, by expressing ideas through software and integration patterns."

> "patterns are the lingua franca of software engineers, appearing in architecture diagrams, documentation, and design discussions."

> "Reports from MIT indicate that in 2025, fewer than 5% of GenAI applications are deemed to have provided value to their organizations."

> "A RAG architecture resembles a straightforward request-response pattern with one notable exception: the message sent by the producer is augmented with data retrieved along the way."

> "An exchange is like a mail sorting center, where queues are like mailboxes."

> "RAG serves as both a simple and widely used GenAI pattern, as well as a foundational building block for more advanced agentic architectures."

## Para aplicar

- **Tratar la app RAG como production-grade, no como prototipo** — antes de desplegar, recorré la Tabla 4.1 (secure, reliable, resilient, observable, maintainable, scalable, input-validated, recoverable, testable, operable, secure-by-default, backward/forward-compatible) y verificá cómo la cubrís.
- **Usar pattern-guided coding** — expresá la arquitectura mediante **GoF + EIP** y dejá que el LLM genere el código y las configs; **verificá siempre** el output.
- **Elegir tecnología probada para enterprise** — preferí **RabbitMQ** (maduro, production-ready) sobre frameworks nuevos (LangChain/LlamaIndex/n8n/DSPy) que no llevan tiempo suficiente; reducí la multiplicidad de tecnologías (el dato MIT: <5% de apps GenAI dieron valor en 2025).
- **Modelar el RAG como request-response augmentado** — aplicá **Request-Reply** (producer no bloqueante), **Content Enricher** (enriquecer in-flight con la vector DB) y **Correlation Identifier** (matchear reply sin gastar threading); ganás scalability, easy integration y monitoring casi gratis.
- **Anticipar múltiples fuentes y robustez** — cuando haya >1 fuente de enrichment (Customer/Pricing/Inventory), cambiá **queues por topics** y agregá **Scatter-Gather**; sumá **LLM routing** (producer decide LLM A/B vía exchange + headers), **dead-letter queues** (no descartar ni bloquear), **message history** (debug + load) y **retry/fallback** para la falta de fiabilidad del LLM.
- **Construir el LLM client con los 3 GoF** — **Template Method** para el request lifecycle de 5 pasos (validate → build → send → parse → normalize), **Adapter** por provider (OpenAI/Anthropic/Gemini/vLLM), **Strategy** para retry/auth/timeout/prompt/streaming/tool-calling; en Python preferí **composición sobre herencia profunda**.
- **Capturar el deploy como infrastructure-as-code** — generá el árbol `rabbitmq-canonical/` con `docker-compose.yml`, `.env`, `rabbitmq.conf`, `enabled_plugins` y `definitions.json`; arrancá con `docker compose up` y mantené el `definitions.json` como la topología reproducible (exportable vía management UI o `rabbitmqadmin`).

## Conexiones

- [[_Building Complex Multi-Agent Systems Using Pattern Prompting|Building Complex Multi-Agent Systems]] — el MOC del libro.
- [[03 - Building with GenAI Parameters, Tuning, and Project Phases]] — capítulo anterior: prometía construir una **app RAG real** poniendo en práctica los 15 parámetros; este capítulo la construye **production-grade** con RabbitMQ + pattern-guided coding.
- [[05 - Starting Your Data Migration Project]] — capítulo siguiente: arranca el **proyecto de data migration** y construye los **data pipelines / [[ETL]]** para mover datos a la vector database.
- [[08 - Pattern-Guided Coding]] — capítulo posterior: **retoma esta arquitectura** y la industrializa en **Topologos** (pattern-guided coding como método), evolucionando este RAG single-LLM a un **dual-LLM [[Scatter-Gather]]** (GPT-4 + Claude).
- [[02 - Embeddings The Language of AI]] — el LLM como **endpoint poco fiable sin uptime guarantees** se introdujo ahí; acá motiva retry/fallback.
- [[A1 - Appendix A - Pattern Reference|Appendix A - Patterns]] — descripción de los patrones **GoF** y **EIP** usados (placeholder).
- [[RAG]] — microarquitectura construida acá production-ready; building block para arquitecturas agénticas.
- [[RabbitMQ]] — message broker open source, base de la implementación (queues, topics, exchanges, EIP nativos, Management UI).
- **[[Pattern-Guided Coding]]** · **[[Vibe Coding]]** · **[[Production-Grade Software]]** — el método y los objetivos (candidatos a nota propia).
- EIP del capítulo: **[[Request-Reply]]** · **[[Content Enricher]]** · **[[Correlation Identifier]]** · **[[Scatter-Gather]]** (candidatos a nota propia).
- GoF del LLM client: **[[Template Method]]** · **[[Adapter]]** · **[[Strategy]]** (candidatos a nota propia).
- **[[Dead-Letter Queue]]** · **[[Docker Compose]]** — infraestructura de mensajería y deployment (candidatos a nota propia).
- [[Enterprise Integration Patterns]] · [[GoF]] · [[Design Patterns]] — las familias fundacionales de patrones.
