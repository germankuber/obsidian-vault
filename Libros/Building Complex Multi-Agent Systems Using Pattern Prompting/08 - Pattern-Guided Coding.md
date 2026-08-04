---
title: Pattern-Guided Coding
libro: Building Complex Multi-Agent Systems Using Pattern Prompting
autor: Packt
capitulo: 8
created: 2026-06-22
tags:
  - libros/building-complex-multi-agent-systems-using-pattern-prompting
  - type/case-study
  - status/permanent
aliases:
  - Pattern-Guided Coding
  - Cap 8 - Pattern-Guided Coding
updated: 2026-06-22
---
# Pattern-Guided Coding

> [!info] Capítulo 8 · *Building Complex Multi-Agent Systems Using Pattern Prompting* — Packt (ISBN 9781806114290)
> El capítulo **eleva el [[Pattern-Guided Coding|pattern-guided coding]] del [[04 - Building Your First RAG App|cap. 4]] a un método industrializado**: en vez de generar una app puntual, define un sistema (**Topologos**) que toma cualquier **GenAI workflow pattern** (ReAct, Plan-and-Execute, Reflection, Multi-Agent…) y lo traduce a una **topología de colas [[RabbitMQ]] production-ready** describiéndolo con los patrones consolidados de **[[GoF]]** y **[[Enterprise Integration Patterns|Enterprise Integration Patterns (EIP)]]**. La tesis operativa: **"GoF and EIP patterns are the patterns. Everything else is the problem to which they are applied."** Recorre la necesidad del pattern-guided coding, su proceso de 4 fases, los GoF de producers/consumers, la estrategia [[Dead-Letter Queue|DLQ]] multi-tier, cómo generar cualquier patrón con un comando, y un **case study** que migra el RAG single-LLM del [[04 - Building Your First RAG App|cap. 4]] a un **dual-LLM [[Scatter-Gather]]** (GPT-4 + Claude) con [[Aggregator|aggregator]] y merge strategies. El [[A2 - Appendix B - Topologos User Manual|Appendix B]] es el manual completo de Topologos; el próximo capítulo conecta los manifests generados a un broker en vivo. Navegá: [[_Building Complex Multi-Agent Systems Using Pattern Prompting|Building Complex Multi-Agent Systems]] · anterior [[07 - Tips and Best Practices]].

## Resumen

El capítulo cierra el arco práctico del libro convirtiendo el **[[Pattern-Guided Coding|pattern-guided coding]]** —introducido en el [[04 - Building Your First RAG App|cap. 4]] como técnica para construir una app RAG— en un **proceso reproducible y auditable** capaz de materializar cualquier microarquitectura GenAI. La idea central es que los llamados "GenAI workflow patterns" o "agentic patterns" (ReAct, Plan-and-Execute, Reflection, Tool Use, Multi-Agent Collaboration, RAG, Orchestrator-Subagent, Human-in-the-Loop) **no son patrones nuevos**: son **problemas** que se resuelven combinando los patrones de software ya probados de **[[GoF]]** (Gang of Four) y **[[Enterprise Integration Patterns|EIP]]** (Hohpe & Woolf). De ahí la frase que ancla el capítulo: *"GoF and EIP patterns are the patterns. Everything else is the problem to which they are applied."*

El capítulo justifica la **necesidad del pattern-guided coding** con tres razones (los LLMs alucinan arquitectura, el código generado ad-hoc no es production-ready, y la falta de un vocabulario común hace irreproducible el resultado) y fija **diez requisitos** que un sistema GenAI production-ready debe cumplir. Sobre esa base presenta **Topologos**, un sistema/agente que **traduce un patrón GenAI a una topología [[RabbitMQ]]** —colas, exchanges, bindings, [[Dead-Letter Queue|DLQ]]— mediante un **proceso de 4 fases**: (1) seleccionar el GenAI workflow pattern, (2) analizarlo bajo cuatro lentes A/B/C/D, (3) mapearlo a EIP + topología de colas, (4) emitir un manifest production-ready. La **Table 8.1** da el mapping canónico de los 8 patrones GenAI → topología RabbitMQ → GoF → EIP.

Luego baja al detalle de implementación: los **producers y consumers** se construyen con GoF concretos (**[[Strategy]]** para el routing, **[[Command]]** para los mensajes, **[[Template Method]]** para el lifecycle, **[[Channel Adapter]]** para hablar con el broker, **[[Competing Consumers]]** para escalar) bajo la **regla de manual ACK/NACK** (nunca auto-ack). La **resiliencia** se modela con una **estrategia [[Dead-Letter Queue|DLQ]] de tres tiers** (retry / parking / poison) gobernada por el **x-death header** y clasificando las fallas en seis clases (Table 8.2). El capítulo muestra cómo **generar cualquier patrón con un solo comando** (`/pattern …`) más comandos auxiliares (`/manifest`, `/security`, `/summary`), cómo **customizarlo**, un **iterative approval loop** y **auditing**. Cierra con un **case study** que evoluciona el RAG single-LLM del [[04 - Building Your First RAG App|cap. 4]] a un **dual-LLM Scatter-Gather** (GPT-4 + Claude) con **[[Aggregator|aggregator]]**, **[[Correlation Identifier|correlation window]]** y **tres merge strategies** (best-of, ensemble, fallback), resumido en la **Table 8.3**. El **[[A2 - Appendix B - Topologos User Manual|Appendix B]]** es el manual de referencia de Topologos; el **capítulo siguiente** conecta estos manifests a un broker live.

## La necesidad del pattern-guided coding

El capítulo arranca preguntando por qué hace falta un método disciplinado si ya se puede pedirle código a un LLM. La respuesta es que el **AI-assisted coding ingenuo no produce sistemas production-ready**, y lo argumenta con **tres reasons**:

> [!warning] **Los 3 reasons que justifican el pattern-guided coding**
> 1. **Los LLMs alucinan arquitectura** — pedirle a un modelo "construime un sistema ReAct" produce código plausible pero arquitectónicamente inconsistente, sin garantías de resiliencia, observabilidad ni escalado. El LLM rellena huecos con invenciones.
> 2. **El código ad-hoc no es production-grade** — lo generado sin un esqueleto de patrones carece de las características de la Tabla 4.1 del [[04 - Building Your First RAG App|cap. 4]] (secure, reliable, resilient, observable, etc.); funciona en el demo y falla en producción.
> 3. **Falta un vocabulario común → irreproducibilidad** — sin nombrar patrones (la **lingua franca** de los ingenieros) el resultado depende del prompt exacto y no es auditable ni repetible. Dos prompts equivalentes dan dos arquitecturas distintas.

La solución es **anclar la generación en patrones nombrados**: si describís la arquitectura con **[[GoF]] + [[Enterprise Integration Patterns|EIP]]**, el LLM tiene un esqueleto preciso, verificable y reproducible sobre el cual generar, y el output hereda las propiedades production-grade de esos patrones.

> [!note] **La tesis operativa del capítulo (verbatim).** *"GoF and EIP patterns are the patterns. Everything else is the problem to which they are applied."* Los "agentic patterns" no son patrones de software: son **problemas** a los que se aplican los patrones reales de GoF y EIP.

### ¿Por qué no "Agentic Patterns"?

El capítulo incluye un recuadro que explica la elección deliberada de terminología, en línea con la tesis del libro de no inventar vocabulario nuevo donde ya existe uno consolidado ([[01 - Introduction Patterns, Abstractions, and the GenAI Landscape|cap. 1]], donde se prefirió **microarchitecture** sobre "agentic pattern").

> [!quote] **Why not "Agentic Patterns"?** — los llamados "agentic patterns" describen *problemas* o *workflows* (qué quiere lograr el sistema), no *soluciones* reusables de diseño. Llamarlos "patterns" confunde el problema con su solución. El término correcto para la solución sigue siendo **GoF + EIP**; "agentic pattern" nombra apenas el requirement.

### Los 10 requisitos de un sistema GenAI production-ready

Antes de generar nada, el capítulo fija el **checklist de diez requisitos** que todo sistema GenAI production-ready debe cumplir (refinan los 12 de la Tabla 4.1 al dominio multi-agente):

1. **Reliable message delivery** — ningún mensaje se pierde (guaranteed delivery, manual ACK).
2. **Resilience / fault tolerance** — retries, fallbacks, [[Dead-Letter Queue|DLQ]]; el sistema degrada con gracia ante fallas de LLM/red.
3. **Horizontal scalability** — escalar agregando consumers ([[Competing Consumers]]), no reescribiendo.
4. **Observability** — tracing, métricas y logs estructurados en cada hop del mensaje.
5. **Loose coupling** — producers y consumers desacoplados vía colas/exchanges; reemplazables sin tocar el resto.
6. **Idempotency / exactly-once semantics** — reprocesar un mensaje no duplica efectos (correlation IDs, dedup).
7. **Security** — autenticación, autorización, validación de input, PII shielding, multi-tenant isolation.
8. **Backpressure & rate limiting** — controlar el throughput para no saturar LLMs ni presupuesto de tokens.
9. **Auditability / traceability** — cada decisión y cada mensaje es trazable de punta a punta (message history).
10. **Reproducibility** — la topología se captura como **infrastructure-as-code** (manifest), reproducible entre entornos.

> [!tip] El pattern-guided coding **no es opcional cosmético**: es el mecanismo por el cual estos 10 requisitos se satisfacen "por construcción" en vez de a mano, porque cada GoF/EIP los aporta de fábrica.

## Cómo funciona: Topologos y su proceso de 4 fases

El motor del capítulo es **Topologos**, un sistema/agente de pattern-guided coding que **toma un GenAI workflow pattern y emite una topología [[RabbitMQ]] production-ready** (colas, exchanges, bindings, DLQs) más el código de producers/consumers. Su manual de referencia completo vive en el **[[A2 - Appendix B - Topologos User Manual|Appendix B]]**. La topología simple objetivo (un producer, una cola, un consumer) es la base sobre la que se construye todo:

![[B34134_8_1.png]]
*Figure 8.1 – Simple queue topology*

Topologos opera con un **proceso de 4 fases**:

![[B34134_8_2.png]]
*Figure 8.2 – Four-step process*

> [!note] **Las 4 fases del proceso Topologos**
> - **Fase 1 — Select the GenAI workflow pattern.** Elegir el patrón objetivo (ReAct, Plan-and-Execute, Reflection, Tool Use, Multi-Agent Collaboration, RAG, Orchestrator-Subagent, Human-in-the-Loop). Es la declaración del *problema*.
> - **Fase 2 — Analyze the pattern through four lenses.** Descomponer el patrón bajo las **4 lentes A/B/C/D** (ver abajo) para extraer su estructura de mensajería real.
> - **Fase 3 — Map to EIP + queue topology.** Traducir la estructura a **[[Enterprise Integration Patterns|EIP]]** concretos y a la topología de colas/exchanges/bindings/DLQ que los implementa.
> - **Fase 4 — Emit a production-ready manifest.** Generar el **manifest** (infrastructure-as-code: definición de colas, exchanges, bindings, policies, DLQ) más el código de producers/consumers con sus GoF.

**Fase 1** establece el problema a resolver:

![[B34134_8_3.png]]
*Figure 8.3 – Phase 1 of Topologos process*

**Fase 2** lo analiza bajo las cuatro lentes:

![[B34134_8_4.png]]
*Figure 8.4 – Phase 2 of Topologos process*

**Fase 3** lo mapea a EIP + topología de colas:

![[B34134_8_5.png]]
*Figure 8.5 – Phase 3 of Topologos process*

**Fase 4** emite el manifest production-ready:

![[B34134_8_6.png]]
*Figure 8.6 – Phase 4 of Topologos process*

### Las 4 lenses de la Fase 2 (A/B/C/D)

El corazón analítico de Topologos es mirar cada GenAI workflow pattern a través de **cuatro lentes** que revelan su naturaleza de mensajería:

- **Lens A — Message flow / topology.** ¿Cómo fluyen los mensajes? ¿Es request-reply lineal, scatter-gather, loop iterativo, pipeline? Define la forma de la topología de colas.
- **Lens B — Coordination / control.** ¿Quién decide el siguiente paso? ¿Hay un orchestrator central ([[Process Manager|process manager]]) o coordinación distribuida (choreography)? Define dónde vive la lógica de control.
- **Lens C — State / context.** ¿Qué estado se mantiene entre pasos? ¿Hay un context que se enriquece (Content Enricher), un historial, una ventana de correlación? Define cómo se persiste/propaga el contexto.
- **Lens D — Failure / resilience.** ¿Qué puede fallar y cómo se recupera? ¿Retries, timeouts, DLQ, fallback LLM? Define la estrategia de resiliencia y el DLQ tiering.

> [!tip] Las 4 lentes A/B/C/D mapean casi 1:1 a los EIP: A→ routing patterns (Scatter-Gather, Pipes-and-Filters), B→ [[Process Manager]] vs choreography, C→ [[Content Enricher]] / [[Correlation Identifier]] / [[Aggregator]], D→ [[Dead-Letter Queue|Dead-Letter Channel]] + retry.

### Tabla 8.1 — GenAI workflow pattern → RabbitMQ topology → GoF → EIP

El mapping canónico que produce Topologos para los 8 patrones GenAI principales:

| GenAI workflow pattern | RabbitMQ topology | GoF pattern(s) | EIP pattern(s) |
|---|---|---|---|
| **ReAct** (reason + act loop) | Single work queue con loop de re-encolado; DLQ | **State**, **Command** | **Message Channel**, **Return Address**, **Dead-Letter Channel** |
| **Plan-and-Execute** | Planner queue → exchange fan-out a step queues; aggregate replies | **Command**, **Template Method** | **Process Manager**, **Splitter**, **Aggregator** |
| **Reflection** (critic gate) | Generate queue → critic queue → loop back hasta aprobar | **Strategy**, **Chain of Responsibility** | **Message Filter**, **Recipient List** |
| **Tool Use** | Topic exchange ruteando a una queue por tool; reply correlacionado | **Command**, **Adapter** | **Channel Adapter**, **Correlation Identifier**, **Request-Reply** |
| **Multi-Agent Collaboration** | Topics + competing consumers por rol; bus de mensajes | **Mediator**, **Observer** | **Publish-Subscribe**, **Competing Consumers**, **Message Bus** |
| **RAG** | Request → Content Enricher (vector DB) → LLM → reply | **Strategy**, **Template Method** | **Content Enricher**, **Scatter-Gather**, **Correlation Identifier** |
| **Orchestrator-Subagent** | Orchestrator queue → fan-out a subagent queues → aggregate | **Command**, **Composite** | **Process Manager**, **Scatter-Gather**, **Aggregator** |
| **Human-in-the-Loop** | Auto queue → human-review queue (manual ACK) → resume | **State**, **Command** | **Message Filter**, **Control Bus**, **Dead-Letter Channel** |

> [!note] La tabla deja clara **la tesis del capítulo en forma tabular**: cada "agentic pattern" de la izquierda (el *problema*) se resuelve enteramente con combinaciones de **GoF + EIP** (las *soluciones*). No hay ninguna columna de "patrón nuevo".

## Producers y consumers con patrones GoF

Una vez fijada la topología, Topologos genera los **producers y consumers** con un conjunto fijo de **GoF patterns**, cada uno resolviendo una responsabilidad concreta:

- **[[Strategy]]** — el **routing pattern**: el producer decide a qué cola/exchange/LLM mandar el mensaje según una `RoutingStrategy` intercambiable en runtime (mismo Strategy del [[04 - Building Your First RAG App|cap. 4]] y del [[06 - Ingesting Data Using Airbyte and Pinecone|cap. 6]]). Evita hard-codear el routing.
- **[[Command]]** — cada **mensaje es un Command**: un objeto/serialización autocontenida que encapsula la acción a ejecutar y su payload, desacoplando emisor de receptor.
- **[[Template Method]]** — el **lifecycle del consumer** (receive → validate → process → ack/nack) vive en una base class; las subclasses llenan solo el `process`.
- **[[Channel Adapter]]** — el adaptador que **traduce entre la app y el broker [[RabbitMQ]]** (publish/consume, serialización, headers), aislando la lógica de negocio del protocolo AMQP (mismo EIP que los inbound/outbound adapters del [[06 - Ingesting Data Using Airbyte and Pinecone|cap. 6]]).
- **[[Competing Consumers]]** — **múltiples consumers sobre la misma cola** procesan mensajes en paralelo; escalar throughput = agregar consumers. Es el mecanismo de horizontal scalability.

> [!warning] **Regla de oro: manual ACK/NACK, nunca auto-ack.** El consumer debe **acknowledgear manualmente** un mensaje *después* de procesarlo con éxito, y hacer **NACK** (con o sin requeue) si falla. El auto-ack borra el mensaje al entregarlo, así que un crash a mitad de procesamiento **pierde el mensaje** silenciosamente — viola "reliable message delivery". El manual ACK es lo que habilita retries y DLQ.

### RoutingStrategy + AgentProducer (Strategy)

El producer expone una interfaz `RoutingStrategy` y un `AgentProducer` que la usa, de modo que cambiar la lógica de ruteo (round-robin, por tenant, por costo, por capacidad del LLM) **no toca el producer**:

```javascript
// Strategy: la familia de algoritmos de routing es intercambiable
interface RoutingStrategy {
  route(message): string; // devuelve el routing key / target queue
}

class CostBasedRouting implements RoutingStrategy {
  route(message) {
    return message.priority === "high" ? "llm.gpt4" : "llm.cheap";
  }
}

class RoundRobinRouting implements RoutingStrategy {
  route(message) { /* alterna entre targets */ }
}

// El producer depende de la abstracción, no de una estrategia concreta
class AgentProducer {
  constructor(channel, routingStrategy) {
    this.channel = channel;
    this.strategy = routingStrategy; // inyectada
  }
  publish(message) {
    const routingKey = this.strategy.route(message);
    this.channel.publish("agent.exchange", routingKey, serialize(message), {
      headers: { correlationId: message.correlationId },
      persistent: true, // guaranteed delivery
    });
  }
}
```

### El mensaje como Command (JSON)

Cada mensaje viaja como un **[[Command]]** serializado a JSON: autocontenido, versionado y correlacionable.

```json
{
  "command": "INVOKE_LLM",
  "version": "1.0",
  "correlationId": "req-7f3a-2025",
  "replyTo": "reply.queue.client-42",
  "payload": {
    "prompt": "Summarize the IFRS 17 manual section 4.2",
    "context": ["chunk-001", "chunk-018"],
    "model": "gpt-4",
    "temperature": 0.2
  },
  "metadata": {
    "tenant": "zzz-insurance",
    "issuedAt": "2026-06-22T10:15:00Z",
    "retryCount": 0
  }
}
```

> [!note] El `command` nombra la acción, `correlationId` + `replyTo` implementan **[[Correlation Identifier]]** + **Return Address**, `metadata.retryCount` habilita la lógica de retry/DLQ, y `tenant` soporta el aislamiento **multi-tenant**.

## Estrategia DLQ multi-tier (resiliencia)

La lente D (failure/resilience) se materializa en una **estrategia [[Dead-Letter Queue|DLQ]] de tres tiers**, en vez de una única cola de descartes. La decisión de a qué tier va un mensaje fallido se gobierna con el **`x-death` header** que [[RabbitMQ]] agrega automáticamente cada vez que un mensaje es dead-lettered (cuenta de rechazos, razón, cola de origen).

> [!note] **Los 3 tiers de DLQ**
> - **Tier 1 — Retry DLQ (transient).** Fallas transitorias (timeout de red, LLM momentáneamente caído). El mensaje se re-encola con **back-off** y un límite de reintentos leído del `x-death` count. Si supera el límite → baja al siguiente tier.
> - **Tier 2 — Parking DLQ (needs intervention).** Fallas que excedieron los retries o requieren revisión humana/operativa. El mensaje queda "parqueado" para inspección, root-cause analysis o reprocesamiento manual (no se descarta, no bloquea la cola activa).
> - **Tier 3 — Poison DLQ (permanent / malformed).** Mensajes que **nunca** podrán procesarse (payload corrupto, schema inválido, contenido rechazado). Se aíslan permanentemente para evitar loops infinitos de reproceso ("poison messages").

El **`x-death` header** es la fuente de verdad: contiene el número de veces que el mensaje fue rechazado y desde qué cola, lo que permite decidir programáticamente *retry → park → poison* sin estado externo.

> [!tip] **DLQ vs [[Dead-Letter Queue|DLX]]**: el **DLX (dead-letter exchange)** es el exchange al que RabbitMQ enruta un mensaje cuando es rechazado/expira/excede length; la **DLQ (dead-letter queue)** es la cola bindeada a ese DLX donde el mensaje aterriza. El tiering se logra con DLXs encadenados (retry-DLX → parking-DLX → poison-DLX).

### Tabla 8.2 — Failure class → DLQ tier → EIP

Las seis clases de falla y a qué tier/EIP corresponden:

| Failure class | Descripción | DLQ tier | EIP |
|---|---|---|---|
| **Transient** | Falla momentánea recuperable (timeout de red puntual) | Tier 1 — Retry DLQ (reintento inmediato/back-off corto) | **Dead-Letter Channel** + retry |
| **Transient persistent** | Falla transitoria que persiste varios reintentos (LLM caído un rato) | Tier 1 → Tier 2 (retry con back-off creciente; si agota, parking) | **Dead-Letter Channel**, **Delayer** |
| **Permanent** | Falla irrecuperable por lógica (dependencia ausente, config inválida) | Tier 2 — Parking DLQ (intervención) | **Dead-Letter Channel**, **Control Bus** |
| **Poison** | Mensaje malformado que rompe el consumer cada vez | Tier 3 — Poison DLQ (aislamiento permanente) | **Invalid Message Channel** |
| **Expired** | Mensaje que superó su TTL antes de procesarse | Tier 2/3 según política (TTL → DLX) | **Message Expiration**, **Dead-Letter Channel** |
| **Rejected** | Mensaje explícitamente NACK-eado por validación/seguridad | Tier 3 — Poison DLQ (no reintentar) | **Message Filter**, **Invalid Message Channel** |

> [!tip] La tabla deja claro que **no toda falla se reintenta**: distinguir *transient* (reintentar) de *poison/rejected* (aislar ya) evita los retry-storms y los poison-message loops que tumban sistemas de colas mal diseñados.

## Generar cualquier patrón con un comando

El payoff práctico de Topologos: **un solo comando genera la topología + el código** de cualquier GenAI workflow pattern, componiendo modificadores. El comando `/pattern` toma el patrón base más una serie de **flags componibles** (gate de crítico, circuit-breaker, throughput, compliance, multi-tenancy, retries):

```text
/pattern react + tool-use with critic-gate, circuit-breaker high-throughput regulated multi-tenant retry-5
```

Ese comando le pide a Topologos un sistema **ReAct + Tool Use**, con un **critic-gate** (Reflection), un **circuit-breaker**, optimizado **high-throughput**, **regulated** (compliance), **multi-tenant** y con **retry-5** (5 reintentos antes de parking). Topologos resuelve los EIP/GoF, la topología de colas y emite el manifest.

Comandos auxiliares que completan el flujo:

- **`/manifest`** — emite el **manifest de infrastructure-as-code** (definición de exchanges, queues, bindings, DLX/DLQ tiers, policies) listo para cargar en RabbitMQ (el equivalente al `definitions.json` del [[04 - Building Your First RAG App|cap. 4]]).
- **`/security`** — genera/audita la **capa de seguridad**: validación de input, PII shielding, autenticación, autorización y aislamiento multi-tenant del manifest.
- **`/summary`** — produce un **resumen legible** de la topología generada (qué colas, qué patrones, qué resiliencia), para revisión humana y documentación.

> [!tip] El valor está en la **composabilidad**: en vez de elegir un "agentic pattern" enlatado, declarás el problema combinando patrones base + modificadores cross-cutting (resiliencia, seguridad, throughput) y Topologos resuelve la arquitectura production-ready con GoF+EIP.

### Customizar, iterative approval loop y auditing

Topologos no es un generador de una sola pasada: incorpora **customización**, un **bucle de aprobación iterativo** y **auditoría**.

- **Customizing** — el output (manifest + código) es editable: se puede ajustar la topología, cambiar estrategias de routing, afinar los tiers de DLQ o las policies de seguridad, y re-generar. Los patrones permanecen como esqueleto estable mientras se customizan los detalles.
- **Iterative approval loop** — la generación pasa por un **loop de revisión y aprobación humana**: Topologos propone, el ingeniero revisa (con `/summary`), pide cambios, y itera hasta aprobar. Esto encarna el principio de **"siempre verificá el output del LLM"** del [[04 - Building Your First RAG App|cap. 4]] y mantiene al humano en el control de la arquitectura.
- **Auditing** — cada generación queda **trazada y auditable**: qué patrón se pidió, qué EIP/GoF se aplicaron, qué decisiones tomó Topologos. Satisface el requisito #9 (auditability) y permite reproducir o justificar la arquitectura ante revisión.

## Case study: del RAG single-LLM al dual-LLM Scatter-Gather

El capítulo demuestra Topologos **evolucionando el RAG del [[04 - Building Your First RAG App|cap. 4]]**: pasa de un **single-LLM RAG** a un **dual-LLM Scatter-Gather** que consulta **GPT-4 y Claude en paralelo** y fusiona ambas respuestas. El objetivo: mayor calidad/robustez aprovechando que distintos LLMs rinden mejor en distintas tareas.

- **Antes (Fig 8.7) — RAG single-LLM.** La arquitectura del cap. 4: request → Content Enricher (vector DB) → **un solo LLM** → reply. Un único punto de generación.

![[B34134_8_7.png]]
*Figure 8.7 – Before: Single-LLM RAG queue topology*

- **Después (Fig 8.8) — dual-LLM Scatter-Gather.** El request se **dispersa (scatter)** vía un exchange a **dos colas de LLM** (una para **GPT-4**, otra para **Claude**); ambos procesan en paralelo; un **[[Aggregator|aggregator]]** **reúne (gather)** las dos respuestas dentro de una **[[Correlation Identifier|correlation window]]** y las fusiona con una **merge strategy**.

![[B34134_8_8.png]]
*Figure 8.8 – After: Dual-LLM RAG with Scatter-Gather, Aggregator, and Merge Strategy*

El patrón EIP central es el **[[Scatter-Gather]]** (ya usado en el cap. 4 para múltiples fuentes de enrichment, ahora para múltiples LLMs), apoyado en el **[[Aggregator]]** (reúne N respuestas correlacionadas en una) y el **[[Correlation Identifier]]** (la **correlation window** matchea las respuestas que pertenecen al mismo request original y espera hasta tener ambas o hasta el timeout).

> [!note] **Tres merge strategies del Aggregator** (cómo fusionar las respuestas de GPT-4 y Claude):
> - **Best-of** — elegir la **mejor** de las dos respuestas según un criterio (scoring, confidence, un critic LLM). Una gana, la otra se descarta.
> - **Ensemble** — **combinar** ambas respuestas en una sola (síntesis, voto, promedio de embeddings), aprovechando lo mejor de cada modelo.
> - **Fallback** — usar la respuesta del **LLM primario** y recurrir al secundario **solo si el primario falla** o no llega dentro de la correlation window (resiliencia: dual-LLM como redundancia).

### Aggregator con correlation window (pseudocódigo)

El Aggregator mantiene un buffer por `correlationId`, junta las respuestas que llegan y emite el resultado fusionado cuando tiene todas las esperadas — o cuando vence el timeout de la **correlation window** (entonces fusiona lo que haya, típicamente vía fallback).

```javascript
class DualLLMAggregator {
  constructor(mergeStrategy, windowMs) {
    this.pending = new Map();      // correlationId -> { responses, timer }
    this.merge = mergeStrategy;    // best-of | ensemble | fallback
    this.windowMs = windowMs;      // correlation window timeout
  }

  onMessage(msg) {
    const id = msg.correlationId;
    if (!this.pending.has(id)) {
      // primera respuesta de este request: abrir la ventana de correlación
      this.pending.set(id, { responses: [], expected: 2,
        timer: setTimeout(() => this.handleTimeout(id), this.windowMs) });
    }
    const entry = this.pending.get(id);
    entry.responses.push({ model: msg.model, body: msg.body });

    if (entry.responses.length === entry.expected) {
      clearTimeout(entry.timer);
      const result = this.merge(entry.responses); // best-of / ensemble / fallback
      this.emit(id, result);
      this.pending.delete(id);
    }
  }

  handleTimeout(id) {
    // venció la correlation window: fusionar lo recibido (p. ej. fallback al que llegó)
    const entry = this.pending.get(id);
    const result = this.merge(entry.responses); // con < expected respuestas
    this.emit(id, result);
    this.pending.delete(id);
  }
}
```

> [!tip] La **correlation window** es la pieza que vuelve robusto al Scatter-Gather: sin un timeout, una respuesta perdida de Claude colgaría el request para siempre; con la ventana, el Aggregator degrada con gracia (fallback al LLM que sí respondió).

### El pipeline de generación de proyecto (6 pasos)

El case study se construye con un **pipeline de generación de proyecto de 6 pasos** que Topologos sigue para emitir el proyecto completo, no solo un snippet:

1. **Select pattern** — declarar el patrón objetivo (acá: RAG → dual-LLM Scatter-Gather).
2. **Analyze (4 lenses)** — descomponerlo con A/B/C/D (flujo scatter-gather, control vía aggregator, contexto vía correlation window, resiliencia vía fallback + DLQ).
3. **Map to EIP + topology** — Scatter-Gather + Aggregator + Correlation Identifier + Content Enricher; exchange + dos LLM queues + reply queue + DLQ tiers.
4. **Generate manifest** — emitir el infrastructure-as-code (`/manifest`) con exchanges, queues, bindings y DLX/DLQ.
5. **Generate code** — producers/consumers con Strategy + Command + Template Method + Channel Adapter + el DualLLMAggregator.
6. **Review & approve** — pasar por el iterative approval loop (`/summary`, `/security`) hasta aprobar y dejar el resultado auditado.

### Tabla 8.3 — Cambios del case study (new / changed / unchanged)

Qué se agrega, qué cambia y qué se conserva al migrar el RAG del cap. 4 al dual-LLM Scatter-Gather:

| Elemento | Estado | GoF / EIP |
|---|---|---|
| **Vector DB Content Enricher** | Unchanged | **Content Enricher** |
| **Producer / RAG orchestrator** | Changed (ahora hace scatter a 2 LLMs) | **Strategy** (routing), **Command** |
| **Routing exchange** | Changed (fan-out a 2 LLM queues) | **Recipient List**, **Publish-Subscribe** |
| **GPT-4 LLM consumer** | Changed (era el único LLM) | **Template Method**, **Adapter**, **Channel Adapter** |
| **Claude LLM consumer** | New | **Template Method**, **Adapter**, **Channel Adapter** |
| **Aggregator (correlation window)** | New | **Aggregator**, **Correlation Identifier** |
| **Merge strategy (best-of/ensemble/fallback)** | New | **Strategy** |
| **Reply queue + Correlation Identifier** | Unchanged | **Correlation Identifier**, **Return Address** |
| **DLQ tiers (retry/parking/poison)** | Changed (extendido a ambos LLMs) | **Dead-Letter Channel**, **Invalid Message Channel** |
| **Competing consumers (scale per LLM)** | New | **Competing Consumers** |
| **Manual ACK/NACK** | Unchanged | (regla de delivery) |

> [!note] La tabla muestra el **valor del pattern-guided coding**: la mayor parte de la arquitectura del cap. 4 se **conserva o se extiende** (no se reescribe), y lo *new* (segundo LLM, Aggregator, merge strategy, competing consumers) se expresa enteramente con GoF/EIP existentes. La evolución de single→dual LLM es una recomposición de patrones, no un rediseño.

## Citas

> "GoF and EIP patterns are the patterns. Everything else is the problem to which they are applied."

> "Why not 'Agentic Patterns'?"

## Para aplicar

- **No le pidas arquitectura suelta a un LLM** — anclá toda generación en **patrones nombrados (GoF + EIP)**: dan un esqueleto verificable, reproducible y production-grade, y evitan las tres fallas (alucinación arquitectónica, código no-production, irreproducibilidad).
- **Tratá el "agentic pattern" como problema, no como solución** — describí *qué* querés (ReAct, Reflection, Multi-Agent…) y dejá que los **GoF+EIP** sean el *cómo*; usá la Table 8.1 como mapping de partida.
- **Seguí el proceso de 4 fases** — (1) seleccioná el patrón, (2) analizalo con las **4 lentes A/B/C/D** (message flow, coordination, state, failure), (3) mapealo a EIP + topología de colas, (4) emití un manifest infrastructure-as-code.
- **Construí producers/consumers con los GoF fijos** — **Strategy** (routing), **Command** (mensajes), **Template Method** (lifecycle), **Channel Adapter** (broker), **Competing Consumers** (escalar); y **siempre manual ACK/NACK**, nunca auto-ack.
- **Diseñá la resiliencia con DLQ de 3 tiers** — clasificá la falla (Table 8.2: transient / transient-persistent / permanent / poison / expired / rejected) y enrutala a **retry / parking / poison** usando el **`x-death` header**; no reintentes mensajes poison/rejected.
- **Generá con un comando componible** — `/pattern <base> + <mods> with <cross-cutting>` (ej. `/pattern react + tool-use with critic-gate, circuit-breaker high-throughput regulated multi-tenant retry-5`), y completá con `/manifest`, `/security` y `/summary`; iterá en el **approval loop** y dejá todo **auditado**.
- **Para multi-LLM, usá Scatter-Gather + Aggregator + correlation window** — dispersá el request a varias LLM queues, reuní con un Aggregator dentro de una **correlation window** con timeout, y elegí la **merge strategy** según el objetivo: **best-of** (calidad), **ensemble** (combinar), **fallback** (resiliencia/redundancia).

## Conexiones

- [[_Building Complex Multi-Agent Systems Using Pattern Prompting|Building Complex Multi-Agent Systems]] — el MOC del libro.
- [[07 - Tips and Best Practices]] — capítulo anterior: las best practices (incluyendo "descubrir tu propia microarquitectura" con las 4 acciones) que este capítulo industrializa en Topologos.
- [[04 - Building Your First RAG App]] — la base directa: el **pattern-guided coding**, los EIP (**[[Request-Reply]]**, **[[Content Enricher]]**, **[[Correlation Identifier]]**, **[[Scatter-Gather]]**), los GoF del LLM client (**[[Template Method]]**, **[[Adapter]]**, **[[Strategy]]**), la **[[Dead-Letter Queue|DLQ]]** y el `definitions.json`; el case study de este capítulo **evoluciona ese RAG** a dual-LLM.
- [[02 - Embeddings The Language of AI]] — el LLM como **endpoint poco fiable**: motiva la estrategia DLQ multi-tier y el fallback merge.
- [[06 - Ingesting Data Using Airbyte and Pinecone]] — el **[[Channel Adapter]]** y el **[[Strategy]]** pattern reaparecen como mecanismo de adaptación al broker y de routing intercambiable.
- [[01 - Introduction Patterns, Abstractions, and the GenAI Landscape]] — la tesis de **patrones sobre tools** y **microarchitecture (no "agentic pattern")**: este capítulo la lleva a su conclusión ("GoF and EIP are the patterns; everything else is the problem").
- [[A2 - Appendix B - Topologos User Manual|Appendix B]] — el **manual de referencia completo de Topologos** (placeholder).
- [[A1 - Appendix A - Pattern Reference|Appendix A - Patterns]] — descripción de los patrones **GoF** y **EIP** referidos (placeholder).
- **[[Pattern-Guided Coding]]** · **[[RabbitMQ]]** · **[[RAG]]** · **[[Scatter-Gather]]** — el método, el broker y los patrones centrales.
- GoF del capítulo: **[[Strategy]]** · **[[Command]]** · **[[Template Method]]** · **[[Channel Adapter]]** · **[[Competing Consumers]]** · **[[State]]** · **[[Mediator]]** · **[[Process Manager]]** (candidatos a nota propia).
- EIP del capítulo: **[[Aggregator]]** · **[[Correlation Identifier]]** · **[[Content Enricher]]** · **[[Dead-Letter Queue]]** (Dead-Letter Channel / DLX / DLQ) · **[[Invalid Message Channel]]** · **[[Recipient List]]** · **[[Publish-Subscribe]]** (candidatos a nota propia).
- Conceptos sembrados: **[[Topologos]]** · **[[DLQ tiering]]** (retry/parking/poison) · **[[x-death header]]** · **[[Correlation window]]** · **[[Merge strategy (best-of, ensemble, fallback)]]** · **[[Manual ACK-NACK]]** (candidatos a nota propia).
- GenAI workflow patterns mapeados (Table 8.1): **[[ReAct]]** · **[[Plan-and-Execute]]** · **[[Reflection]]** · **[[Tool Use]]** · **[[Multi-Agent Collaboration]]** · **[[Orchestrator-Subagent]]** · **[[Human-in-the-Loop]]** (candidatos a nota propia).
