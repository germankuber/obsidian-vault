---
title: A2 - Appendix B - Topologos User Manual
libro: Building Complex Multi-Agent Systems Using Pattern Prompting
autor: Packt
capitulo: "Apéndice B"
created: 2026-06-22
tags:
  - libros/building-complex-multi-agent-systems-using-pattern-prompting
  - type/reference
  - status/permanent
aliases:
  - Appendix B - Topologos User Manual
  - Appendix B - Topologos
  - A2 - Topologos User Manual
updated: 2026-06-22
---

# Appendix B - Topologos User Manual

> [!info] Apéndice B · *Building Complex Multi-Agent Systems Using Pattern Prompting* — Packt (ISBN 9781806114290)
> Es el **manual de referencia completo de [[Topologos]]**, el sistema que el [[08 - Pattern-Guided Coding|cap. 8]] introdujo y cuyo método se materializó en código en el [[09 - Implementing the ReAct Pattern Over RabbitMQ|cap. 9]]. Topologos = **un senior distributed-systems architect empaquetado como system prompt** (`rabbitmq-agentic-architect-prompt-v5.md`): toma una descripción de un agentic workflow y emite una **topología [[RabbitMQ]] production-ready** justificada con **[[GoF]] + [[Enterprise Integration Patterns|EIP]]**. El apéndice documenta su **two-layer model** ([[Pattern Layer|Pattern]] vs [[Topology Layer|Topology]]), su **protocolo de 4 fases**, sus **26 comandos**, sus **7 artifacts**, los **10 patrones canónicos**, los **5 worked examples** (A–E) y la **persistencia cross-session**. Como es un apéndice, cierra el libro: anterior [[A1 - Appendix A - Pattern Reference|Appendix A]] (el catálogo de patrones que Topologos *aplica*). Navegá: [[_Building Complex Multi-Agent Systems Using Pattern Prompting|Building Complex Multi-Agent Systems]] · anterior [[A1 - Appendix A - Pattern Reference|Appendix A]].

## Resumen

El Apéndice B es el **manual operativo de [[Topologos]]**: no un capítulo narrativo sino una **referencia de usuario** densa, hecha de **14 tablas**, **5 ejemplos trabajados** y un **catálogo de 10 patrones canónicos**. Si el [[08 - Pattern-Guided Coding|cap. 8]] presentó a Topologos como el motor del [[Pattern-Guided Coding|pattern-guided coding]] industrializado (traducir un *agentic workflow pattern* a una topología de colas), y el [[09 - Implementing the ReAct Pattern Over RabbitMQ|cap. 9]] mostró una de sus salidas corriendo de verdad, este apéndice es el documento que enseña a **conducir la herramienta**: cómo cargarla, qué le podés pedir, en qué orden razona, qué te devuelve y cómo llevar su output a un broker real.

La idea central es el **modelo de dos capas**: toda topología agéntica se descompone en una **[[Pattern Layer|capa de patrón]]** (la *agent choreography*: roles, tipos de mensaje, control flow, terminación — el ReAct loop, el Plan-Execute, el debate) y una **[[Topology Layer|capa de topología]]** (la *RabbitMQ shape*: archetypes de exchange/queue, DLQ topologies, routing namespaces, security baselines). Sobre cada capa Topologos ofrece tres operaciones —**Compose** (sumar patrones peer: `/pattern A+B`), **Extend** (agregar un stage a un patrón: `/pattern A with X`) y **Define** (declarar una choreography nueva: `/define pattern`)— y reconcilia ambas capas cuando se combinan (cada queue hereda las políticas de seguridad, cada flow honra el baseline).

Topologos no improvisa: corre un **protocolo de 4 fases** —**Clarify** (entrevista de 5 preguntas o auto-completion), **Pattern Analysis** (5 lentes: pattern+topology decomposition, GoF, EIP, security threat model, failure mode), **Iterative Approval** (decisiones de a una, 8–13 en total, cada una con justificación GoF/EIP, trade-offs, diagrama y un Approve/Modify/Reject) y **Final Output** (7 artifacts: diagrama, manifest, pattern reference card, security manifest, DLQ diagram, operational notes, active definitions)—. Su superficie de comandos son **26 comandos** en 6 categorías (Session, Pattern, Topology, Definition, Operations, Diagram), e incluye operaciones de verificación de calidad (`/validate`, `/simulate`, `/cost`, `/chaos`) y de generación de Infrastructure-as-Code (`/deploy terraform|kubernetes|helm|docker-compose` con modificadores `+ha`/`+tls`/`+oauth`).

El apéndice cierra con los **5 ejemplos trabajados** —A: ReAct conversacional end-to-end (5 lentes, 8 decisiones, manifest, validate/simulate/deploy); B: ReAct + Tool Use por composición; C: extender con `critic-gate` y encadenar extensions; D: `/define pattern debate-adjudicate` completo con su synthetic reference card; E: un topology archetype organization-wide con 4 `/define` + `/use` + `/export`— y la **persistencia cross-session** (pegar el `/export` arriba de `*End of system prompt*`). Es un manual **opinionado**: asume RabbitMQ y solo cubre sistemas agénticos.

> [!note] **Mental model.** Pensá en Topologos como un **senior distributed-systems architect** sentado al lado tuyo: le describís lo que querés construir, te hace preguntas de clarificación, te propone decisiones de diseño **de a una** (con la justificación en patrones GoF/EIP y los trade-offs), las aprobás o las modificás, y al final te entrega los artefactos desplegables. No es un generador de código de un solo disparo; es un **diálogo de diseño guiado**.

## Quick start

Topologos se carga **pegando el system prompt** (`rabbitmq-agentic-architect-prompt-v5.md`) según la superficie (ver Tabla B.1), y la primera respuesta del modelo confirma con **"Topologos online"**. A partir de ahí hay **tres entry points** según por dónde quieras empezar:

- **Conversational** — describís el problema en prosa ("necesito un agente que use tools y a veces escale a un humano") y Topologos arranca la Fase 1 de clarificación.
- **Pattern-first** — invocás directo un patrón: `/pattern react`, y Topologos asume o pregunta lo que falte.
- **Topology-first** — partís de la *shape* de infraestructura: `/topology three-tier-standard + signed-pii-queue + regulated`, y después le encajás el patrón.

Después se **aprueba de a una decisión**: en la Fase 3, Topologos presenta entre **8 y 13 decisiones** y para en cada una esperando un **Approve / Modify / Reject**. Cuando el diseño está cerrado, se obtiene el **artefacto desplegable**: `/validate` para chequear la calidad, y `/deploy terraform +tls` (u otro target) para generar el Infrastructure-as-Code.

> [!tip] El primer mensaje que ves tras cargar el prompt es literalmente **"Topologos online"** — si no aparece, el system prompt no se cargó bien (ver Troubleshooting).

### Tabla B.1 — Cómo cargar Topologos (las 3 superficies)

| Surface | Cómo cargar | Notas |
|---|---|---|
| **Claude.ai / API** | Pegar el contenido de `rabbitmq-agentic-architect-prompt-v5.md` en el **system prompt** | Recomendado **Opus 4.6** o **Sonnet 4.6+**; la primera respuesta del modelo debe ser **"Topologos online"** |
| **Claude Code** | Guardar el prompt como **`.claude/system.md`** en la raíz del proyecto | Se carga automáticamente como system prompt del proyecto |
| **Cowork** | **Drop** del file en la conversación | El archivo se adopta como system prompt |

> [!note] Cualquiera sea la superficie, la señal de que Topologos quedó activo es su saludo **"Topologos online"** como primera respuesta.

## When to use / Skip

Topologos es **opinionado**: asume [[RabbitMQ]] como broker y solo razona sobre **sistemas agénticos**. Usalo cuando estés en alguno de estos **8 escenarios**:

- Diseñar la **topología de colas** de un sistema multi-agente nuevo.
- Traducir un *agentic workflow pattern* (ReAct, Plan-Execute, Reflection…) a infraestructura RabbitMQ.
- Combinar dos patrones (Compose) o extender uno con un stage nuevo (Extend).
- Definir una **choreography custom** que no está en el catálogo (Define).
- Endurecer una topología para **compliance** (PII/PHI → baseline `regulated`).
- Generar **Infrastructure-as-Code** (Terraform/K8s/Helm/Compose) de una arquitectura agéntica.
- **Validar / simular / estimar costo** de una topología antes de desplegarla.
- **Codificar el baseline organizacional** una sola vez y reusarlo (los `/define` + `/use`).

Y **NO** lo uses (los **3 skips**) cuando:

- Querés un **tutorial genérico** de RabbitMQ o de agentes (Topologos diseña, no enseña teoría desde cero).
- Tu broker es **otro** (Kafka / NATS / SQS): Topologos es RabbitMQ-only y sus archetypes no se trasladan.
- El sistema **no es agéntico** (no hay roles de agente, ni razonamiento iterativo, ni tools): no aporta nada.

> [!warning] Topologos **asume RabbitMQ**. No intentes usarlo para Kafka/NATS/SQS: sus queue archetypes, DLQ topologies y security baselines están atados a las primitivas de RabbitMQ (exchanges, DLX, quorum queues, `x-` arguments).

## Core concepts — las dos capas y el protocolo de 4 fases

El corazón conceptual de Topologos son **dos reasoning layers** que razona por separado y luego reconcilia. La **[[Pattern Layer|Pattern layer]]** responde *"¿cuál es la agent choreography?"* (roles, message types, control flow, terminación); la **[[Topology Layer|Topology layer]]** responde *"¿cuál es la RabbitMQ shape?"* (exchange/queue archetypes, DLQ topologies, routing namespaces, security baselines). Pensar en capas separadas es lo que permite, por ejemplo, fijar primero la infraestructura regulada (`/topology three-tier-standard + signed-pii-queue + regulated`) y *después* encajarle el patrón de razonamiento (`/pattern react + tool-use`) sin contradicciones.

### Tabla B.2 — Las 2 reasoning layers

| Layer | Pregunta que responde | Qué define |
|---|---|---|
| **[[Pattern Layer\|Pattern]]** | ¿Cuál es la *agent choreography*? | **Roles** (qué agentes), **message types** (qué mensajes), **control flow** (cómo se coordinan), **termination** (cuándo para) |
| **[[Topology Layer\|Topology]]** | ¿Cuál es la *RabbitMQ shape*? | **Exchange/queue archetypes**, **DLQ topologies**, **routing namespaces**, **security baselines** |

Sobre cada capa Topologos ofrece las **mismas 3 operaciones** —Compose, Extend, Define— dando una grilla 3×2:

### Tabla B.3 — Operations × Layers (Compose / Extend / Define)

| Operación | En la Pattern layer | En la Topology layer |
|---|---|---|
| **Compose** (sumar peers) | `/pattern rag + memory + tool-use` | `/topology three-tier-standard + signed-pii-queue` |
| **Extend** (agregar stage) | `/pattern react with critic-gate` | `/topology-with` (sumar un archetype a una topología) |
| **Define** (declarar nuevo) | `/define pattern debate-adjudicate` | `/define topology our-org-baseline` |

> [!note] **Compose < Extend < Define** en orden de "cuánto inventás". Compose junta patrones que ya existen; Extend inserta un stage en uno existente; Define declara una choreography fundamentalmente nueva. Siempre preferí la operación más liviana que resuelva el caso (ver Tabla B.12).

El segundo pilar es el **protocolo de 4 fases**, que Topologos sigue siempre:

### Tabla B.4 — Las 4 fases del protocolo

| Fase | Nombre | Qué pasa |
|---|---|---|
| **1** | **Clarify** | Entrevista de **5 preguntas** (o **auto-completion** si describiste suficiente): scale, reliability, ordering, infraestructura existente, security/compliance |
| **2** | **Pattern Analysis** | Análisis con **5 lentes**: (A) pattern+topology decomposition, (B) **GoF**, (C) **EIP**, (D) **security threat model**, (E) **failure mode** |
| **3** | **Iterative Approval** | Decisiones **de a una** (8–13): cada una con *proposal* → *justificación GoF/EIP* → *trade-offs* → *diagram* → **Approve / Modify / Reject** |
| **4** | **Final Output** | Emite los **7 artifacts** (ver Tabla B.8) |

> [!tip] La Fase 1 termina con un **acknowledgment** que resume lo que entendió antes de analizar. La Fase 3 **para en cada decisión** — no avanza sola: vos decidís Approve/Modify/Reject decisión por decisión.

**Phase 1 acknowledgment (ejemplo).** Tras la entrevista, Topologos confirma el entendimiento, por ejemplo:

```text
Got it. Designing a ReAct agent:
- Scale: moderate (inferred), single-region greenfield
- Reliability: at-least-once delivery
- Ordering: not required
- Security: internal
- Failure handling: retry-3 (default)
Proceeding to pattern analysis.
```

**Phase 2 (las 5 lentes, ejemplo del Example A).** Topologos descompone el diseño bajo cada lente:

```text
Lens A — Pattern + Topology decomposition:
  Pattern: ReAct loop (Reason → Act → Observe), single agent, tool dispatch.
  Topology: topic exchange for thoughts/actions, work queues per stage.
Lens B — GoF: State (loop state machine), Strategy (tool selection),
  Iterator (observation cycle), Command (action messages).
Lens C — EIP: Process Manager (the loop), Message Channel, Correlation Identifier
  (thread the episode), Dead Letter Channel.
Lens D — Security threat model: prompt-injection in tool inputs, non-JSON poison.
Lens E — Failure mode: tool timeout, agent crash mid-loop, poison message.
```

**Phase 3 decision (formato de una decisión).** Cada decisión se presenta así y **espera tu respuesta**:

```text
Decision 3/8 — Routing namespace
Proposal: single namespace `react.*` (react.thoughts, react.actions, react.observations).
Justification (EIP): Message Channel + Correlation Identifier keep one episode threaded.
Trade-offs: simpler routing, but no per-tool isolation (revisit if multiple tools).
[diagram]
Approve / Modify / Reject?
```

## Command reference

Topologos expone **26 comandos** en **6 categorías**. La tabla siguiente los lista con su categoría y efecto; debajo van las listas enumerables (definition kinds, simulate scenarios, chaos scenarios, diagram views y formats).

### Tabla B.5 — El command set completo (26 comandos)

| Categoría | Comando | Efecto |
|---|---|---|
| **Session** | `/status` | Estado actual del diseño (qué se decidió hasta ahora) |
| **Session** | `/decisions` | Lista las decisiones tomadas y pendientes |
| **Session** | `/manifest` | Emite el RabbitMQ topology manifest (JSON/YAML) |
| **Session** | `/security` | Emite el security manifest |
| **Session** | `/summary` | Resumen del diseño en prosa |
| **Session** | `/restart` | Reinicia el diseño (pide confirmación) |
| **Session** | `/restart-confirm` | Confirma el reinicio |
| **Session** | `/help` | Ayuda de comandos |
| **Pattern** | `/pattern <name>` | Diseña a partir de un patrón canónico (`/pattern react`) |
| **Pattern** | `/<pattern>` | Atajo del anterior |
| **Pattern** | `/pattern-with` | Extiende un patrón con un stage (`/pattern react with critic-gate`) |
| **Topology** | `/topology <archetype>` | Diseña a partir de un topology archetype |
| **Topology** | `/topology-with` | Extiende una topología con un archetype |
| **Definition** | `/define <kind> <name>` | Declara una definición nueva |
| **Definition** | `/define-help` | Ayuda del lenguaje `/define` |
| **Definition** | `/use <name>` | Aplica una definición guardada |
| **Definition** | `/definitions` | Lista las definiciones activas |
| **Definition** | `/export` | Exporta las definiciones (para persistencia cross-session) |
| **Operations** | `/validate` | Valida la topología y devuelve un score (ej. 92/100) |
| **Operations** | `/simulate <scenario>` | Simula un escenario de falla y traza los steps |
| **Operations** | `/cost` | Estima el costo operativo de la topología |
| **Operations** | `/deploy <target> [mods]` | Genera el Infrastructure-as-Code |
| **Operations** | `/chaos <scenario>` | Inyecta un escenario de caos para probar resiliencia |
| **Diagram** | `/diagram <view>` | Renderiza un diagrama de la topología |
| **Diagram** | `/diagram-help` | Ayuda de vistas y formatos de diagrama |

Las listas que acompañan a los comandos:

- **Definition kinds** (`/define <kind>`): `pattern`, `topology`, `queue-archetype`, `dlq-topology`, `security-baseline`, `routing-namespace`.
- **Simulate scenarios** (`/simulate`): `happy-path`, `tool-timeout`, `agent-crash`, `poison-message`, `retry-exhausted`, `human-delay`.
- **Chaos scenarios** (`/chaos`): `consumer-crash`, `retry-storm`, `poison-message`, `network-partition`.
- **Diagram views** (`/diagram`): `full`, `dlq`, `security`, `pattern`, `vhost`, `flow`, `failure-classes`, `definitions`.
- **Diagram formats**: `mermaid` (default), `plantuml`, `graphviz`, `svg`, `ascii`.

> [!tip] `/status`, `/decisions`, `/manifest`, `/security`, `/summary` son tus comandos de inspección durante el diseño; `/validate`, `/simulate`, `/cost`, `/chaos` son los de **verificación de calidad** previos al `/deploy`.

**Output de `/status` (ejemplo).** Muestra el estado del diseño en curso:

```text
Design: ReAct agent (Pattern: react, Topology: single-pattern-default)
Phase: 3 (Iterative Approval) — decision 5/8
Approved: vhost /react-dev, topic exchange react.topic, 3-tier DLQ
Pending: per-tool prefetch, human-review priority queue
Security baseline: internal
```

**Output de `/validate` (ejemplo, score 92/100).** Devuelve un puntaje con desglose:

```text
/validate
Topology score: 92/100
✓ At-least-once delivery (manual ack, publisher confirms recommended)
✓ Dead-letter chain present (3 tiers)
✓ Poison isolation (quarantine queue)
✓ Correlation Identifier threads episodes
⚠ -5 No publisher confirms configured (add for guaranteed delivery)
⚠ -3 Single-region (no HA) — acceptable for dev, revisit for prod
```

**Output de `/simulate tool-timeout` (ejemplo, tabla de steps).** Traza el recorrido del mensaje paso a paso:

| Step | Component | Action | Result |
|---|---|---|---|
| 1 | agent | publishes action → `tools.dispatch` | ok |
| 2 | tool-worker | consumes, calls API, **times out** | no response |
| 3 | tool-worker | `basic_nack(requeue=False)` (transient) | → `dlq.retry1` (TTL 30s) |
| 4 | dlq.retry1 | TTL expires | → `dlq.retry2` (TTL 5min) |
| 5 | dlq.retry2 | TTL expires, still failing | → `dlq.quarantine` |
| 6 | operator | inspects `x-death`, replays or discards | manual |

**Output de `/cost` (ejemplo).** Estima el costo operativo:

```text
/cost
Estimated monthly (moderate load, single node):
- Broker (RabbitMQ, 2 vCPU / 4GB): ~$60/mo
- LLM calls (agent _think, ~10k episodes): dominant variable cost
- Tool API calls: per-tool, see per-tool prefetch
Note: queue/exchange count is negligible; LLM tokens dominate.
```

## Modifiers — sintonizar el diseño con sufijos

Los **modifiers** son palabras que agregás a la invocación para forzar decisiones de diseño sin entrar a la entrevista. Se organizan por categoría, cada una con un **default** (lo que Topologos asume si no decís nada).

### Tabla B.6 — Modifiers por categoría (con default)

| Categoría | Valores | Default |
|---|---|---|
| **Scale** | `low-throughput`, `high-throughput`, `batch`, `real-time` | *inferred* |
| **Reliability** | `at-most-once`, `at-least-once`, `exactly-once` | `at-least-once` |
| **Ordering** | `unordered`, `per-key`, `strict` | *inferred* |
| **Infrastructure** | `greenfield`, `existing-exchanges`, `multi-region`, `federated` | `greenfield` |
| **Security** | `internal`, `internet-facing`, `multi-tenant`, `regulated` | `internal` |
| **Failure** | `retry-3`, `retry-5`, `no-retry` | `retry-3` |
| **Latency** | `low-latency`, `throughput-optimised` | *inferred* |

Qué cambia cada modifier clave:

- **`regulated`** — agrega `signed-pii-queue`, **mTLS**, **OAuth2**, **audit log** y **claim-check** (payloads grandes/sensibles fuera del mensaje); endurece el security baseline.
- **`multi-tenant`** — namespaces y vhosts por tenant, aislamiento de colas y usuarios por tenant; routing namespace `multi-tenant`.
- **`exactly-once`** — Topologos lo **degrada a `at-least-once` + idempotency** (RabbitMQ no garantiza exactly-once nativo; ver FAQ).
- **`high-throughput`** — sube prefetch, usa **competing consumers** y considera **lazy queues**; optimiza para volumen.
- **`retry-5` vs `retry-3`** — cambia la profundidad de la cadena de retry tiers antes de mandar a quarantine.

> [!warning] **`exactly-once` no existe gratis.** Pedirlo hace que Topologos diseñe **at-least-once + idempotency keys** (deduplicación en el consumer). No prometas exactly-once end-to-end: lo correcto es entregas al-menos-una-vez con procesamiento idempotente.

## Auto-derived templates — esqueletos pattern-layer

Cuando componés ciertos patrones, Topologos **deriva automáticamente** un template de arquitectura con nombre y un *spine* (la columna vertebral del flujo). Son **atajos**, no diseños cerrados.

### Tabla B.7 — Auto-derived templates

| Combinación de patrones | Template derivado | Spine |
|---|---|---|
| `rag + memory (+ tool-use)` | **Knowledge Worker** | retrieve → augment → (tool) → respond, con memoria de sesión |
| `plan-execute + tool-use` | **Tool-Augmented Planner** | plan → dispatch tools → collect → replan/finish |
| `orchestrator + multi-agent (+ human-loop)` | **Supervised Swarm** | orchestrator → fan-out subagents → gather → (human gate) |
| `tree-of-thoughts + reflection` | **Self-Critiquing Search** | branch → evaluate → reflect/prune → expand best |

> [!note] Los auto-derived templates son **solo de la Pattern layer**: te dan el esqueleto de choreography, pero seguís decidiendo la Topology layer (queues, DLQ, security) en la Fase 3. Son **skeletons, no shortcuts**: no saltean las decisiones (ver Tips).

## Generating the deployable artifact

El propósito final de Topologos es producir algo desplegable. La Fase 4 emite **7 artifacts**, y `/deploy` los convierte en Infrastructure-as-Code.

### Tabla B.8 — Los 7 artifacts de Phase 4

| # | Artifact | Formato | Para qué sirve |
|---|---|---|---|
| 1 | **Full topology diagram** | Mermaid | Ver la topología completa (exchanges, queues, bindings) |
| 2 | **RabbitMQ topology manifest** | JSON / YAML | La definición declarativa para aplicar a un broker |
| 3 | **Pattern reference card** | Table | Resumen del patrón (roles, messages, flow, GoF/EIP) |
| 4 | **Security manifest** | Structured | Usuarios, permisos, TLS/OAuth, audit, baselines |
| 5 | **DLQ topology diagram** | Mermaid | La cadena de dead-letter (retry/parking/poison) |
| 6 | **Operational notes** | Prose + table | Cómo operar: prefetch, alertas, replay, escalado |
| 7 | **Active definitions** | List | Las definiciones `/define` activas en la sesión |

**`/deploy` — targets y modifiers.** El comando genera el IaC para distintos backends:

### Tabla B.9 — Deploy targets

| Target | Genera |
|---|---|
| **terraform** (default) | `rabbitmq_exchange` / `rabbitmq_queue` / `rabbitmq_binding` / `rabbitmq_user` / `rabbitmq_permissions` |
| **kubernetes** | **RabbitMQ Cluster Operator** + **Topology Operator** (CRDs) |
| **helm** | Valores del **Bitnami chart** |
| **docker-compose** | Broker + plugins + sidecar de **definitions-import** |

### Tabla B.10 — Deploy modifiers

| Modifier | Agrega |
|---|---|
| **+ha** | **Quorum queues**, multi-node, **anti-affinity** |
| **+tls** | **TLS 1.3**, integración con **cert-manager** |
| **+oauth** | Plugin **`rabbitmq_auth_backend_oauth2`** + scope-mapping |

**Workflow end-to-end.** El orden recomendado antes de desplegar:

```text
/validate                 → score + warnings (apuntá a ≥90)
/simulate tool-timeout    → verificá que la DLQ enrute bien
/cost                     → confirmá que el costo cierra
/deploy terraform +tls +oauth   → generá el IaC endurecido
```

**La sesión completa de 8 pasos (end-to-end inline).** Una sesión típica de principio a fin:

```text
1. (cargar el system prompt) → "Topologos online"
2. /pattern react
3. (Fase 1) responder las 5 preguntas → acknowledgment
4. (Fase 2) Topologos corre las 5 lentes
5. (Fase 3) Approve/Modify/Reject las 8 decisiones
6. /validate → 92/100, ajustar warnings
7. /simulate tool-timeout → confirmar la cadena DLQ
8. /deploy terraform +tls → IaC listo
```

**Rendering de diagramas a PNG.** Los diagramas salen como texto (Mermaid/PlantUML/Graphviz) y se rasterizan con la herramienta de cada formato:

```bash
# Mermaid → PNG
mmdc -i topology.mmd -o topology.png

# Graphviz → PNG
dot -Tpng topology.dot -o topology.png

# Mermaid → SVG (vectorial)
mmdc -i topology.mmd -o topology.svg

# PlantUML → PNG
plantuml topology.puml
```

> [!warning] **Topologos no genera PNG nativamente.** Emite el **texto** del diagrama (Mermaid por default) y vos lo renderizás con `mmdc`, `dot`, `plantuml`, etc. El apéndice menciona dos PNGs de ejemplo (`react-topology.png`, `react-tooluse-topology.png`) como *resultado* de rasterizar, no como assets que vengan con el libro.

**Aplicar el manifest a un broker real (3 rutas).** El manifest JSON/YAML se lleva a un RabbitMQ vivo de tres maneras:

1. **`rabbitmqadmin`** — importar el manifest con la CLI (`rabbitmqadmin import definitions.json`); rápido para dev.
2. **Management HTTP API** — `curl` contra `/api/definitions` con el JSON; útil en scripts/CI.
3. **Terraform** (**recomendado para producción**) — el provider de RabbitMQ aplica recursos declarativos con estado y plan/apply.

**Cross-session persistence.** Las definiciones (`/define …`) no sobreviven solas a un nuevo chat. Para conservarlas: `/export` y **pegar el resultado arriba de la línea `*End of system prompt*`** del `rabbitmq-agentic-architect-prompt-v5.md`. Así la próxima sesión arranca con tus archetypes ya cargados.

> [!note] **IaC: design-fidelity, not deployment-tested.** El Infrastructure-as-Code que emite `/deploy` es **fiel al diseño** (refleja exactamente la topología aprobada), pero **no está probado contra un broker real**: validalo/aplicalo en un entorno controlado antes de producción.

## Worked examples (A–E)

Los cinco ejemplos recorren las operaciones de Topologos de la más simple (un patrón canónico) a la más avanzada (un baseline organizacional reusable).

### Example A — ReAct conversacional end-to-end

Arranca conversacional ("quiero un agente ReAct con una tool de búsqueda"), Topologos corre la **Fase 1** (5 preguntas → acknowledgment), la **Fase 2** (las 5 lentes, ya mostradas arriba en *Core concepts*), y la **Fase 3** con **8 decisiones** (vhost, exchange topology, queues, routing namespace, DLQ, security, consumer concurrency, human-review). Emite el **manifest** y se cierra con `/validate` (92/100), `/simulate tool-timeout` y `/deploy`.

**Manifest excerpt (Example A).** Fragmento del RabbitMQ topology manifest emitido:

```json
{
  "vhost": "/react-dev",
  "exchanges": [
    { "name": "react.topic", "type": "topic", "durable": true },
    { "name": "react.dlx.tier1", "type": "fanout", "durable": true }
  ],
  "queues": [
    {
      "name": "agent.react.thoughts",
      "durable": true,
      "arguments": { "x-dead-letter-exchange": "react.dlx.tier1" }
    },
    {
      "name": "agent.react.human-review",
      "durable": true,
      "arguments": { "x-max-priority": 10 }
    }
  ]
}
```

> [!note] El manifest del Example A muestra los dos detalles clave: la cola de thoughts dead-letterea a `react.dlx.tier1` (`x-dead-letter-exchange`), y la cola de revisión humana es una **priority queue** (`x-max-priority: 10`) para que los casos urgentes se atiendan primero.

### Example B — ReAct + Tool Use (composición)

Muestra **Compose** (`/pattern react + tool-use`): se suman dos patrones peer porque cada uno aporta su choreography (ReAct el loop, Tool Use el dispatch a workers). La diferencia con el Example A son las **decisiones que cambian** vs las que quedan igual:

### Tabla B.11 — Changes from canonical ReAct (ReAct + Tool Use)

| Decisión | Canonical ReAct | ReAct + Tool Use |
|---|---|---|
| **1 — Exchange topology** | topic `react.topic` | **+ `tools.dispatch.direct`** (direct exchange para tools) |
| **2 — Queue naming** | una work queue de actions | **3 per-tool queues** (web-search, database, calendar) |
| **3 — Routing** | namespace único `react.*` | **2 namespaces** `react.*` + `tool.*` |
| **4 — DLQ** | 3-tier estándar | **shared 3-tier con tool-name tag** (`x-death` etiquetado por tool) |
| **5 — Security** | usuario único interno | **per-tool worker users** (un usuario por tool) |
| **6 — Reliability** | at-least-once | *unchanged* |
| **7 — Consumer concurrency** | prefetch único | **per-tool prefetch** (web-search=10, database=20, calendar=5) |
| **8 — Human review** | priority queue | *unchanged* |

> [!tip] **¿Compose o Extend?** Acá se usa **Compose** (`react + tool-use`) porque Tool Use es un **patrón peer completo** (tiene su propia choreography de dispatch). Si solo agregaras un *stage* a ReAct sin choreography propia, sería **Extend** (`react with …`). Ver Tabla B.12.

### Example C — Extender con `critic-gate` y encadenar extensions

Muestra **Extend** (`/pattern react with critic-gate`): agrega un stage de crítica al loop ReAct, sumando **3 decisiones nuevas** (9, 10, 11) a las 8 del Example A:

- **Decision 9 — Critic queue & exchange**: dónde vive el critic (cola dedicada, binding desde el agente).
- **Decision 10 — Gate logic**: el critic aprueba/rechaza antes de emitir la `final_answer` (un gate, no un consumidor pasivo).
- **Decision 11 — Critic failure handling**: qué pasa si el critic falla (timeout → permitir o bloquear; clasificación de falla).

Y muestra el **chaining de múltiples extensions**: `/pattern react with critic-gate + output-verifier + retry-budget-guard` hace **crecer el diseño de 8 a ~13 decisiones** (cada extension suma sus propias decisiones de cola/gate/falla).

> [!note] **Extend compone stages sobre un patrón base.** Encadenar extensions (`with A + B + C`) es válido y aditivo: cada stage agrega sus decisiones, llevando un diseño de 8 a 13+ decisiones. Por eso conviene **diagramar temprano** (`/diagram`) para no perder de vista la topología que crece.

### Example D — `/define pattern debate-adjudicate`

Muestra **Define**: declarar una choreography que **no está en el catálogo** (varios agentes debaten y un adjudicador decide). El bloque `/define` completo:

```text
/define pattern debate-adjudicate
roles:
  - debater (N instances): argue a position
  - adjudicator (1): read all arguments, decide
messages:
  - debate.argument (debater → topic)
  - debate.rebuttal (debater → topic)
  - debate.verdict (adjudicator → reply)
flow:
  1. moderator broadcasts the question to all debaters
  2. each debater publishes debate.argument
  3. debaters publish debate.rebuttal (one round)
  4. adjudicator aggregates all arguments + rebuttals
  5. adjudicator publishes debate.verdict
control: round-based, fixed number of rounds
termination: after adjudicator emits debate.verdict (or max-rounds)
primary-failure: debater timeout → adjudicator proceeds with available arguments
derives-from: Multi-Agent Collaboration + Scatter-Gather + Aggregator
```

A partir del `/define`, Topologos genera una **synthetic reference card** que lo cataloga con los mismos patrones GoF/EIP que un patrón nativo:

```text
Synthetic reference card — debate-adjudicate
GoF:  Mediator (adjudicator coordinates), State (round state machine),
      Strategy (adjudication rule), Iterator (rounds)
EIP:  Process Manager (the debate lifecycle), Selective Consumer (adjudicator
      reads only its topic), Aggregator (collect arguments), Correlation
      Identifier (thread the debate)
RabbitMQ shape: topic exchange debate.*, fan-out to N debaters,
      aggregation queue for the adjudicator, fixed-round termination.
```

> [!note] Una vez `/define`-ado, `debate-adjudicate` se trata como un patrón de primera clase: Topologos lo mapea a **GoF (Mediator/State/Strategy/Iterator)** y **EIP (Process Manager/Selective Consumer/Aggregator/Correlation Identifier)**, le da su RabbitMQ shape, y lo podés Compose/Extend como cualquier otro.

### Example E — Topology archetype organization-wide

Muestra cómo **codificar el baseline de una organización** una sola vez con cuatro `/define`, juntarlos en una topología nombrada, y reusarla con un one-liner. Los **4 bloques `/define` completos**:

```text
/define queue-archetype signed-pii-queue
based-on: work-queue
adds:
  - payload signing (HMAC) on publish
  - claim-check for payloads > 256KB (store body in object store, pass reference)
  - x-message-ttl: 3600000 (1h)
purpose: carry PII/PHI safely with integrity + size offloading
```

```text
/define dlq-topology four-tier-with-circuit-breaker
tiers:
  - retry1 (TTL 30s)
  - retry2 (TTL 5min)
  - parking (manual intervention)
  - quarantine (poison, permanent)
circuit-breaker: open the tool route after N consecutive failures, cool-down 60s
```

```text
/define security-baseline regulated-multitenant
extends: regulated
adds:
  - per-tenant vhost isolation
  - per-tenant users + permissions
  - mTLS required
  - OAuth2 scope per tenant
  - audit log on every publish/consume
```

```text
/define topology our-org-baseline
composes:
  - queue-archetype: signed-pii-queue
  - dlq-topology: four-tier-with-circuit-breaker
  - security-baseline: regulated-multitenant
  - routing-namespace: multi-tenant
```

Luego se aplica y se exporta:

```text
/use our-org-baseline
/pattern react + tool-use        # hereda automáticamente el baseline
/export                          # para persistir cross-session
```

> [!tip] **El payoff del one-liner.** Tras codificar `our-org-baseline`, cualquier equipo arranca un sistema **compliant por default** con `/use our-org-baseline` + su patrón — sin volver a discutir PII signing, DLQ tiers, mTLS, OAuth2 ni audit. Codificás el baseline **una vez** y lo reusás siempre.

## El cross-layer example (HIPAA)

Combina las dos capas en un caso de compliance: primero se fija la **Topology layer** con `/topology three-tier-standard + signed-pii-queue + regulated`, y después se le encaja la **Pattern layer** con `/pattern react + tool-use`. Topologos **reconcilia** ambas: cada queue del patrón **hereda las políticas** del baseline (signing, TTL, claim-check), y **cada flow honra el baseline `regulated`** — mTLS en cada conexión, OAuth2 por scope, audit log en cada publish/consume, y claim-check para sacar el PHI del cuerpo del mensaje.

> [!note] La fuerza del two-layer model se ve acá: definís la infraestructura regulada **una vez** y cualquier patrón que le encajes **adopta automáticamente** esas garantías. No hay que re-pedir compliance patrón por patrón; la Topology layer lo impone sobre la Pattern layer.

## Recipes — invocaciones quick-reference

Las invocaciones más usadas, listas para copiar:

```text
# Bootstrap greenfield ReAct
/pattern react

# ReAct con múltiples tools
/pattern react + tool-use

# Agregar un critic stage
/pattern react with critic-gate

# Knowledge worker (auto-derived)
/pattern rag + memory + tool-use high-throughput regulated

# Orchestrated swarm
/pattern orchestrator + multi-agent + human-loop

# Topology-first regulada con PII
/topology three-tier-standard + signed-pii-queue + regulated

# El kitchen-sink (todo a la vez)
/pattern react + tool-use with critic-gate, circuit-breaker high-throughput regulated multi-tenant retry-5

# Org baseline + uso
/use our-org-baseline

# Auditar el diseño
/security

# Simular una falla
/simulate tool-timeout

# Estimar costo
/cost

# Desplegar endurecido
/deploy terraform +tls +oauth

# Diagramas focalizados
/diagram dlq
/diagram security

# Capturar definiciones para reusar
/export

# Reset
/restart
```

> [!tip] Guardá las recipes que más uses; la del kitchen-sink (`/pattern react + tool-use with critic-gate, circuit-breaker high-throughput regulated multi-tenant retry-5`) muestra que **modifiers y extensions se apilan** en una sola línea.

## Tips & best practices

- **Sé específico en la Fase 1** — cuanto más detalle des (scale, ordering, compliance), menos asume Topologos y mejor encaja el diseño.
- **Compose < Extend < Define** — usá siempre la operación más liviana que resuelva el caso; no definas un patrón si componer dos basta.
- **Usá `/pattern` una vez que tenés un workflow claro** — empezá conversacional si dudás; saltá a `/pattern X` cuando ya sabés qué querés.
- **Leé los trade-offs** de cada decisión — la Fase 3 te da el costo de cada elección; no apruebes en piloto automático.
- **Modificá en vez de reiniciar** — `Modify` ajusta una decisión sin perder el resto; `/restart` tira todo.
- **`/validate` antes de `/deploy`** — apuntá a ≥90 y resolvé los warnings primero.
- **`/simulate` + `/chaos` antes de producción** — probá tool-timeout, agent-crash, poison-message y network-partition.
- **Codificá tus defaults una vez** — los `/define` + `/export` (Example E) evitan rediseñar el baseline en cada proyecto.
- **Los templates son skeletons, no shortcuts** — los auto-derived templates te dan el spine; igual decidís la Topology layer.
- **Definition over modification** — para una choreography fundamentalmente distinta, `/define` un patrón limpio en vez de torturar uno existente con extensions.
- **Diagramá temprano** — `/diagram` seguido a medida que crece el diseño (sobre todo al encadenar extensions) para no perder la topología de vista.

## Troubleshooting

| Issue | Fix |
|---|---|
| Salta al diseño **sin preguntar** | El prompt no se cargó completo o describiste de más; pedí explícitamente la Fase 1 de clarificación |
| Las **decisiones no paran** (avanza solo) | Forzá la Fase 3 iterativa; pedí "una decisión a la vez, esperá mi Approve" |
| `/help` sale **abreviado** | Estás con un prompt viejo; usá la **v5** (`rabbitmq-agentic-architect-prompt-v5.md`) |
| Un `/define` es **rechazado** | Te falta un campo obligatorio (roles/messages/flow/termination); completalo |
| `/use` dice **not found** | La definición no está activa; corré `/definitions` y verificá que la `/define`-aste o la pegaste del `/export` |
| La **extension question se repite** | Ambigüedad en el stage; precisá qué hace el stage y dónde se inserta |
| **Patrones no reconocidos** | Topologos defiende sus citas (**GoF 1994**, **Hohpe & Woolf 2003**); si insistís con un nombre inventado, te pide la fuente o lo `/define`-ás |
| **Mermaid no renderiza** | Copiá el bloque a `mmdc`/mermaid.live; revisá que no haya texto extra dentro del fence |
| Se **salteó la Fase 4** | Pedí `/manifest` + `/security` + `/diagram` explícitamente, o reiterá la Fase 4 |
| El **`/deploy` no aplica limpio** | Es design-fidelity, no deployment-tested; validá contra un broker de prueba y ajustá nombres/permisos |
| Pediste **`exactly-once`** | Topologos lo degrada a **at-least-once + idempotency**; diseñá deduplicación en el consumer |
| Una topología **composed double-handlea** una queue | Dos patrones comparten cola; pedí aislamiento (per-pattern namespace) o un binding distinto |
| Un **custom pattern** tiene Fase 2 shallow | El `/define` fue pobre; enriquecé roles/flow/termination para que las 5 lentes tengan material |

> [!warning] Topologos **defiende sus citas**: cuando no reconoce un "patrón", no inventa — apela a **GoF (1994)** y **Hohpe & Woolf (2003)** y te pide la fuente o que lo declares con `/define`. Eso es deliberado: mantiene el vocabulario anclado en patrones probados, no en jerga.

## FAQs

- **¿Cómo modelo múltiples tool calls en un loop?** Tres sub-casos: (1) **secuencial** — un dispatch por iteración del loop ReAct; (2) **paralelo** — fan-out a varias tool queues + Aggregator con correlation window; (3) **per-tool worker** — una cola por tool con su prefetch (Example B).
- **¿Me da código desplegable?** Sí: `/deploy` emite Terraform/K8s/Helm/Compose (design-fidelity), más el manifest aplicable vía `rabbitmqadmin`/HTTP API.
- **¿Genera PNG?** No nativamente: emite Mermaid/PlantUML/Graphviz como texto; rasterizás con `mmdc`/`dot`/`plantuml`.
- **¿Sirve para Kafka/NATS/SQS?** No: Topologos es **RabbitMQ-only**.
- **¿Y si mi patrón no está en el catálogo?** Tres opciones: (1) **Compose** patrones existentes; (2) **Extend** uno con un stage; (3) **`/define`** una choreography nueva (Example D).
- **¿Sobreviven las definiciones entre sesiones?** Solo si `/export`-ás y pegás el resultado arriba de `*End of system prompt*`.
- **¿PII/PHI?** Usá el baseline **`regulated`** (signed-pii-queue, mTLS, OAuth2, audit, claim-check).
- **¿Puedo pararlo solo en plan-mode (sin desplegar)?** Sí: pará tras la **Fase 4** y quedate con los 7 artifacts sin correr `/deploy`.
- **¿Cómo manejo el `reply_to` en el loop ReAct?** Per-correlation reply queue (o `reply_to` con correlation_id) para que la observación vuelva al episodio correcto.
- **¿Y un loop runaway?** Poné un **max-iterations** en la terminación del patrón; al excederlo, termina o escala a humano.
- **¿Cómo migro de ReAct a ReAct + Tool Use sin downtime?** Cinco pasos: (1) declarar las nuevas `tools.dispatch` exchange/queues; (2) desplegar los tool workers en paralelo; (3) bindear las per-tool queues; (4) cambiar el agente a publicar a `tools.dispatch`; (5) drenar y retirar el dispatch viejo.

> [!note] La FAQ de "múltiples tool calls" condensa el patrón clave del cap. 9: el loop **[[Thought-Action-Observation loop|Thought → Action → Observation]]** puede ser secuencial, paralelo (Scatter-Gather + Aggregator) o per-tool, y la elección es una decisión de Topology layer.

## Canonical pattern reference — los 10 patrones canónicos

Topologos reconoce **10 patrones canónicos** (los **8** de la Table 8.1 del [[08 - Pattern-Guided Coding|cap. 8]] **más [[Memory]] y [[Tree of Thoughts]]**). Cada ficha trae **Pattern token + Flow + GoF + EIP + RabbitMQ shape**.

- **[[ReAct]]** — token `react`. **Flow**: Reason → Act → Observe (loop hasta `final_answer`). **GoF**: State, Strategy, Iterator, Command. **EIP**: Process Manager, Message Channel, Correlation Identifier, Dead Letter Channel. **RabbitMQ shape**: topic exchange `react.*`, work queues por stage, 3-tier DLQ.
- **[[Plan-and-Execute]]** — token `plan-execute`. **Flow**: Plan → Execute steps → (replan) → Finish. **GoF**: Command (steps), Strategy (planner), Template Method (step lifecycle). **EIP**: Process Manager, Recipient List, Aggregator. **RabbitMQ shape**: plan exchange + step queues + result aggregation queue.
- **[[Reflection]]** — token `reflection`. **Flow**: Generate → Critique → Revise (iterar). **GoF**: Observer (critic), Strategy (critique rule), State. **EIP**: Process Manager, Message Channel. **RabbitMQ shape**: generation queue + critique queue + revision loop.
- **[[Tool Use]]** — token `tool-use`. **Flow**: Select tool → Dispatch → Collect result. **GoF**: Strategy (tool selection), Command (tool call), Adapter (tool API). **EIP**: Content-Based Router, Message Channel, Correlation Identifier. **RabbitMQ shape**: direct exchange `tools.dispatch`, per-tool queues, per-tool prefetch.
- **[[Multi-Agent Collaboration]]** — token `multi-agent`. **Flow**: agents exchange messages, converge. **GoF**: Mediator, Observer. **EIP**: Publish-Subscribe Channel, Process Manager, Aggregator. **RabbitMQ shape**: topic/fanout exchange, per-agent queues, shared coordination queue.
- **[[RAG]]** — token `rag`. **Flow**: Retrieve → Augment → Generate. **GoF**: Strategy (retriever), Template Method. **EIP**: Content Enricher, Scatter-Gather (hybrid retrieval), Pipes and Filters. **RabbitMQ shape**: retrieval queue → enrichment → generation queue.
- **[[Memory]]** — token `memory`. **Flow**: Read context → use → Write back. **GoF**: Strategy (memory store), Proxy. **EIP**: Content Enricher (inject memory), Claim Check (offload large state). **RabbitMQ shape**: memory enrichment queue + state store reference (claim-check).
- **[[Orchestrator-Subagent]]** — token `orchestrator`. **Flow**: Orchestrator fan-out → subagents → gather. **GoF**: Mediator (orchestrator), Command. **EIP**: Process Manager, Recipient List, Aggregator, Scatter-Gather. **RabbitMQ shape**: orchestrator exchange, per-subagent queues, gather/aggregation queue.
- **[[Human-in-the-Loop]]** — token `human-loop`. **Flow**: Auto → (gate) → Human review → Resume. **GoF**: State (paused/resumed), Strategy (escalation rule). **EIP**: Message Channel, Process Manager, Selective Consumer. **RabbitMQ shape**: **priority queue** para revisión humana (`x-max-priority`), resume binding.
- **[[Tree of Thoughts]]** — token `tree-of-thoughts`. **Flow**: Branch → Evaluate → Prune → Expand best. **GoF**: Composite (the tree), Strategy (evaluation), Iterator. **EIP**: Recipient List (branches), Aggregator, Process Manager. **RabbitMQ shape**: branch exchange, per-branch queues, evaluation/pruning aggregation.

> [!note] **Son 10, no 8.** Topologos amplía los 8 GenAI workflow patterns de la **Table 8.1** del [[08 - Pattern-Guided Coding|cap. 8]] con **[[Memory]]** (estado de sesión, con [[Claim Check]] para offload) y **[[Tree of Thoughts]]** (búsqueda ramificada con poda). Todos comparten el formato **token → flow → GoF → EIP → RabbitMQ shape**.

## Topology-layer primitives — los archetypes canónicos

La capa de topología también tiene su catálogo de **primitivas canónicas** (las piezas que Compose/Extend/Define combinan en la Topology layer).

### Tabla B.14 — Canonical topology-layer primitives

| Familia | Primitivas |
|---|---|
| **Queue archetypes** | `work-queue`, `rpc-reply-queue`, `broadcast-queue`, `priority-queue`, `delayed-queue`, `lazy-queue`, `signed-pii-queue`, `quarantine-queue` |
| **DLQ topologies** | `three-tier-standard`, `exponential-backoff`, `per-failure-class`, `circuit-breaker` |
| **Security baselines** | `internal`, `multi-tenant`, `regulated`, `internet-facing` |
| **Routing namespaces** | `standard`, `versioned`, `multi-tenant` |
| **Topology archetypes** | `single-pattern-default`, `regulated-multitenant`, `federated-multi-region` |

> [!tip] Estas primitivas son el vocabulario de la Topology layer: un `/topology` se arma componiendo un **topology archetype** (ej. `single-pattern-default`) con **queue archetypes** (`signed-pii-queue`), una **DLQ topology** (`circuit-breaker`), un **security baseline** (`regulated`) y un **routing namespace** (`multi-tenant`) — exactamente lo que hace el Example E al definir `our-org-baseline`.

## Glossary

### Tabla B.13 — Glossary completo

| Término | Significado |
|---|---|
| **Agentic pattern** | Una *agent choreography* (roles + mensajes + control flow + terminación); en Topologos, un patrón de la Pattern layer |
| **[[Pattern Layer\|Pattern layer]]** | La capa que responde "¿cuál es la choreography?" (roles, message types, control flow, termination) |
| **[[Topology Layer\|Topology layer]]** | La capa que responde "¿cuál es la RabbitMQ shape?" (archetypes, DLQ, namespaces, security) |
| **Compose** | Sumar patrones/archetypes **peer** (`A + B`) |
| **Extend** | Agregar un **stage** a un patrón existente (`A with X`) |
| **Define** | Declarar una choreography/archetype **nuevo** (`/define`) |
| **Primitive** | Una pieza canónica de la Topology layer (queue archetype, DLQ topology, etc.) |
| **GoF pattern** | Un design pattern del **Gang of Four (1994)** |
| **EIP** | Un **Enterprise Integration Pattern** de **Hohpe & Woolf (2003)** |
| **Vhost** | Un *virtual host* de RabbitMQ (aislamiento lógico de exchanges/queues/users) |
| **DLX** | *Dead-Letter Exchange*: el exchange al que va un mensaje rechazado/expirado |
| **Quorum queue** | Cola replicada con consenso (HA, durable); reemplaza mirrored queues |
| **Lazy queue** | Cola que persiste mensajes a disco para minimizar RAM (alto volumen) |
| **[[Claim Check]]** | EIP: sacar un payload grande del mensaje y pasar una referencia |
| **[[Process Manager]]** | EIP: el componente que dirige un proceso multi-paso (el loop, el plan) |
| **Manual ack** | Acknowledgear/negar explícitamente tras procesar (nunca auto-ack) |
| **Auto-derived template** | Esqueleto pattern-layer que Topologos deriva de ciertas composiciones |
| **x-death** | Header que RabbitMQ agrega al dead-letterear (count, razón, cola origen) |
| **Publisher confirms** | Confirmación del broker de que recibió cada publish (guaranteed delivery) |
| **Routing namespace** | El esquema de routing keys/exchanges (standard / versioned / multi-tenant) |
| **Failure class** | La categoría de una falla (transient/permanent/poison…) que decide su ruta DLQ |
| **Quarantine queue** | Cola terminal para mensajes poison/permanentes (holding area, replay manual) |
| **Synthetic reference card** | La ficha GoF/EIP/shape que Topologos genera para un patrón `/define`-ado |

## Para aplicar

- **Cargá Topologos según tu superficie** (Tabla B.1): pegá `rabbitmq-agentic-architect-prompt-v5.md` en el system prompt (Claude.ai/API), o guardalo como `.claude/system.md` (Claude Code), o dropealo (Cowork); esperá el **"Topologos online"**.
- **Elegí el entry point**: conversacional si dudás, `/pattern X` si ya sabés el workflow, `/topology …` si partís de la infraestructura.
- **Recorré las 4 fases sin atajos**: respondé bien la Fase 1, leé las 5 lentes de la Fase 2, y **aprobá/modificá de a una** las 8–13 decisiones de la Fase 3.
- **Verificá antes de desplegar**: `/validate` (≥90) → `/simulate tool-timeout` + `/chaos` → `/cost` → `/deploy terraform +tls +oauth`.
- **Para compliance, empezá por la Topology layer**: `/topology three-tier-standard + signed-pii-queue + regulated` y *después* encajá el patrón; cada queue hereda el baseline.
- **Codificá tu baseline organizacional una vez** (Example E): `/define` queue-archetype + dlq-topology + security-baseline + topology, `/use our-org-baseline`, `/export` para persistir.
- **Para un patrón que no está en el catálogo**, decidí con la Tabla B.12: dos peers → Compose; un stage nuevo → Extend; choreography distinta → `/define pattern` (como `debate-adjudicate`).
- **Llevá el manifest a un broker real** por una de las 3 rutas (`rabbitmqadmin` para dev, HTTP API para CI, **Terraform para prod**), recordando que el IaC es **design-fidelity, no deployment-tested**.
- **Persistí cross-session**: pegá el `/export` arriba de `*End of system prompt*`.

### Tabla B.12 — When define vs extend vs compose

| Situación | Operación |
|---|---|
| Dos **peer patterns** completos | `/pattern A + B` (**Compose**) |
| Un patrón con un **stage nuevo** | `/pattern A with X` (**Extend**) |
| Una choreography **fundamentalmente distinta** | `/define pattern` (**Define**) |

> [!tip] La pregunta guía: ¿lo que agrego tiene **choreography propia**? Si sí y es peer → Compose. Si es solo un stage dentro del flujo del otro → Extend. Si nada del catálogo encaja → Define.

## Conexiones

- [[_Building Complex Multi-Agent Systems Using Pattern Prompting|Building Complex Multi-Agent Systems]] — el MOC del libro: este apéndice es el **manual de la herramienta** que el libro construyó.
- [[A1 - Appendix A - Pattern Reference|Appendix A - Pattern Reference]] — el **otro apéndice** (anterior): el catálogo de patrones GoF/EIP/Reliability/Microarchitecture que Topologos *aplica*; este apéndice es el manual del sistema que los usa.
- [[08 - Pattern-Guided Coding]] — el capítulo que **introdujo Topologos** y su proceso de 4 fases; este apéndice es su **manual completo** (las 14 tablas, los 26 comandos, los 5 ejemplos amplían lo que el cap. 8 esbozó).
- [[09 - Implementing the ReAct Pattern Over RabbitMQ]] — la **implementación real** de una salida de Topologos: el manifest `react.*` con DLQ de 3 tiers corriendo sobre un broker vivo; el Example A de este apéndice es la versión "de manual" de ese cap.
- [[Topologos]] — el sistema en sí: senior distributed-systems architect empaquetado como system prompt (`rabbitmq-agentic-architect-prompt-v5.md`).
- **Two-layer model** (candidatos a nota propia): [[Pattern Layer]] · [[Topology Layer]].
- Los **10 patrones canónicos**: [[ReAct]] · [[Plan-and-Execute]] · [[Reflection]] · [[Tool Use]] · [[Multi-Agent Collaboration]] · [[RAG]] · [[Memory]] · [[Orchestrator-Subagent]] · [[Human-in-the-Loop]] · [[Tree of Thoughts]].
- **GoF/EIP que Topologos usa para catalogar**: [[Mediator]] · [[Process Manager]] · [[Claim Check]] · [[Correlation Identifier]] · [[Scatter-Gather]] · [[Aggregator]] · [[Content-Based Router]] · [[Dead Letter Channel]] · [[State]] · [[Strategy]] · [[Command]] · [[Iterator]] (candidato a nota propia) · [[Selective Consumer]] (candidato a nota propia).
- **Conceptos de infraestructura RabbitMQ**: [[RabbitMQ]] · [[Dead-Letter Queue]] · [[x-death header]] · [[x-message-ttl]] · [[Publisher confirms]] · [[Competing Consumers]] · [[Quorum Queue]] (candidato a nota propia) · [[Lazy Queue]] (candidato a nota propia) · [[mTLS]] · [[Circuit Breaker]].
- **Cierre del libro**: junto al [[A1 - Appendix A - Pattern Reference|Appendix A]] y el [[10 - The Future and Limitations of LLMs|cap. 10]], este apéndice **cierra la obra entera** — es el último documento del libro.
