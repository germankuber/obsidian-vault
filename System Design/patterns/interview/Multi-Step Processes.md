---
title: Multi-Step Processes
reading:
  total_words: 1750
  read_words: 1750
  pct: 100
  last_read: 2026-07-02
source: HelloInterview — System Design pattern (investigado con Temporal, InterviewNoodle, DesignGurus, DEV Community)
author: HelloInterview (compilado)
created: 2026-07-02
tags:
  - system-design/orchestration
  - system-design/async
  - type/pattern
  - status/permanent
aliases:
  - Multi-Step Processes
  - Procesos Multi-Paso
  - Workflow Orchestration
updated: 2026-07-02
---

# Multi-Step Processes

> [!note] Definición
> Patrón para coordinar **procesos de negocio que son una secuencia de pasos cruzando servicios**, donde cualquier paso puede fallar y algunos tardan. Como no hay una transacción ACID que abarque todos los pasos en un sistema distribuido, se usa una escalera de coordinación: transacción de base → [[Saga]] (coreografía u orquestación) → durable execution engine. Los invariantes: compensaciones idempotentes definidas de antemano, acciones no compensables al final, e idempotency keys / [[Idempotency|outbox]] para lograr efecto exactly-once.

## The Problem

Muchos procesos de negocio no son una sola operación atómica, sino una **secuencia de pasos que cruzan servicios**, donde cualquiera puede fallar y algunos tardan. Ejemplos: procesar una orden (validar → reservar inventario → cobrar → enviar → notificar), onboarding de usuario, procesamiento de pagos, planear un viaje (reservar vuelo + hotel + auto).

El problema central: en un sistema distribuido **no tenés una transacción ACID que abarque todos los pasos**. No podés hacer rollback de un cobro a una tarjeta externa con un `ROLLBACK` de SQL. Necesitás:
- que el proceso **avance hasta completarse** (ejecución parcial no es deseable),
- que **sobreviva fallos** (crash a mitad de camino, retries, dependencias externas caídas),
- y que si algo falla irrecuperablemente, **deshaga** lo ya hecho de forma consistente.

**Señal para reconocer el patrón:** cualquier workflow con múltiples pasos dependientes que cruzan límites de servicio, especialmente si hay dinero, inventario o efectos externos de por medio.

---

## Solutions

### Single Server Primitives

Si todo el proceso vive en **una sola base de datos y un solo servicio**, la solución más simple es una **transacción de base de datos** — todos los pasos hacen commit juntos o ninguno (atomicidad ACID gratis).

**Cuándo alcanza:** workflows simples, cortos, sin llamadas a servicios externos, donde todo el estado está en una base. No sobre-diseñes: si una transacción resuelve, usá una transacción.

**Cuándo se rompe:** en cuanto un paso llama a un servicio externo (cobrar una tarjeta, mandar un mail, reservar en otro microservicio), la transacción de base ya no cubre todo. Ahí entrás al territorio de saga.

### The Saga Pattern

**Idea central:** modelás el proceso como una secuencia de transacciones locales, cada una en su servicio. Si un paso falla, ejecutás **transacciones compensatorias** que deshacen los pasos previos, en orden inverso. En vez de un rollback atómico global, tenés un "deshacer semántico" paso a paso.

Ejemplo de viaje: reservar vuelo → reservar hotel → reservar auto. Si el auto falla, compensás cancelando el hotel y luego el vuelo.

**Reglas de oro de las compensaciones (caen en la entrevista):**
- Cada paso necesita su compensación definida **de antemano**.
- Las compensaciones deben ser **idempotentes** (podés ejecutarlas varias veces sin efecto extra).
- Las acciones **no compensables** (mandar un mail, que no podés "des-enviar") deben ir **al final** de la saga, cuando ya sabés que todo lo demás salió bien.

**La trampa que hay que mencionar — saga sacrifica el aislamiento (isolation):** transacciones concurrentes pueden leer datos parcialmente actualizados (un estado intermedio de la saga). Se mitiga con semantic locks, version numbers, y diseño cuidadoso de los límites de transacción. Decir esto muestra madurez.

**Frase para la mesa:** "Decir 'uso una saga' no alcanza. Especifico si es coreografía u orquestación, cómo funciona la compensación, y por qué."

### Event-Driven Choreography

**Cómo funciona:** **descentralizado**, sin coordinador central. Cada servicio reacciona a eventos y publica el suyo, disparando el siguiente paso. Servicio A termina, publica `InventoryReserved`; el servicio de pago escucha eso, cobra, publica `PaymentCompleted`; y así. Cada servicio tiene conocimiento local (la analogía es una colonia de hormigas comunicándose por feromonas).

**Pros:** desacoplado, servicios autónomos, sin single point of failure central, bueno para flujos de alto throughput y loosely-coupled.

**Contras / trampa:** **visibilidad y debugging.** No hay un lugar central que sepa el estado del workflow; para debuggear una secuencia fallida hay que rastrear logs a través de múltiples servicios independientes para descubrir dónde se cortó silenciosamente. La lógica condicional (branching) requiere que los servicios publiquen distintos tipos de evento según el estado, y el ruteo se dispersa.

**Cuándo usarlo:** workflows simples y estables, de 3-4 pasos, sin mucho branching.

### Workflow Orchestration

**Cómo funciona:** **centralizado**. Un orquestador (saga orchestrator) maneja el flujo y el estado, comandando a cada servicio en el orden definido. El control flow vive en un solo lugar, como código.

**Pros:** visibilidad clara del estado, fácil de razonar y debuggear, natural para lógica condicional y ejecución paralela, fácil de modificar cuando el workflow cambia seguido.

**Contras:** el orquestador puede volverse un cuello de botella o un single point of failure si no está bien diseñado (los workflow engines modernos lo resuelven).

**Cuándo usarlo:** workflows complejos (5+ pasos), con lógica condicional ("saltear el fraud check para órdenes de menos de $10"), pasos paralelos, o que cambian con frecuencia, y equipos que necesitan visibilidad clara del estado.

**Regla de decisión coreografía vs orquestación (los 4 factores):**

| Factor | Coreografía | Orquestación |
|---|---|---|
| Complejidad | Simple, 3-4 pasos, sin branching | Complejo, 5+ pasos, condicional/paralelo |
| Acoplamiento | Desacoplado, autónomo | Control centralizado |
| Visibilidad | Difícil de debuggear | Estado claro y trazable |
| Cambios | Costoso modificar el flujo | Fácil de modificar |

En la práctica, los sistemas grandes usan **ambos**: coreografía para flujos de alto throughput y loosely-coupled; orquestación para workflows complejos, con estado y críticos para el negocio.

#### Durable Execution Engines

Motores que hacen que un workflow **sobreviva cualquier fallo** y retome exactamente donde quedó, sin importar cuánto dure la caída. Expresás el workflow como código (secuencial, condicional, paralelo) y el motor garantiza la durabilidad, los retries y el manejo de estado.

#### How Durable Execution Works

El mecanismo clave es el **event sourcing / replay**: el motor persiste cada paso completado en un log de eventos. Si el proceso crashea a mitad de camino, al reiniciar **reproduce (replay)** el historial de eventos para reconstruir el estado exacto y continúa desde el punto donde se cortó — sin re-ejecutar los pasos ya completados. Esto exige que las funciones sean **deterministas** (dado el mismo input y el mismo historial, producen el mismo resultado).

En sistemas sin un motor dedicado, el orquestador suele implementarse como una **máquina de estados persistida en una base relacional**, movida por un polling loop o un consumer de eventos. Más trabajo para construirlo bien, pero control total del modelo de ejecución.

#### Managed Workflow Systems

Servicios que dan durabilidad, recuperación de fallos y retry logic "de fábrica", con garantía de ejecución exactly-once y audit trail completo.

#### Implementations

- **Temporal:** expresás el workflow enteramente en código; garantiza que ningún progreso se pierde y retoma exactamente donde quedó, sin código extra más allá de la lógica de negocio. En los límites reales: el Workflow Execution Timeout por default es **∞ (infinito)**, no los ~10 años que se suele repetir (fuente: [Detecting Workflow failures](https://docs.temporal.io/encyclopedia/detecting-workflow-failures)). El payload/blob máximo es **2 MB** en Temporal Cloud (warning a 512 KB) y el gRPC message limit es **4 MB**; otros límites por ejecución son ~**2.000** activities/child-workflows/signals incompletos (recomiendan no pasar de 500), máximo **10.000 signals**, retention default de **30 días** (1-90 configurable) y timer máximo de **100 años** (fuente: [Cloud limits](https://docs.temporal.io/cloud/limits)).
- **AWS Step Functions:** máquinas de estado en AWS, definidas en JSON. La duración máxima de ejecución es **1 año** para Standard Workflow (al superarlo falla con `States.Timeout`) y **5 minutos** para Express Workflow; el payload máximo de input/output es **256 KiB** (UTF-8), una hard quota que aplica a tasks, estados y ejecuciones enteras — y es exactamente la razón por la que conviene guardar referencias (IDs, punteros a blob) en vez del payload completo. La definición de la state machine (ASL) tiene un máximo de **1 MB** (fuente: [Step Functions service quotas](https://docs.aws.amazon.com/step-functions/latest/dg/limits.html)).
- **Azure Durable Functions.**
- Para pasos coreografiados, message brokers: Kafka, RabbitMQ, SQS, Google Pub/Sub.

---

## When to Use in Interviews

### Common interview scenarios

Procesamiento de órdenes de e-commerce, sistemas de pago (Design a Payment System), booking/reservas (Uber ride lifecycle: validación → reserva de fondos → routing → ejecución → confirmación → settlement), onboarding, fulfillment, cualquier proceso con pasos que cruzan servicios.

### When NOT to use it in an interview

- Si todo cabe en **una transacción de base de datos**, usá eso — no metas una saga ni un orquestador.
- Si el proceso es de un solo paso o síncrono, no hay multi-step que coordinar.
- No propongas Temporal/Step Functions para un flujo de 2 pasos que una transacción resuelve. Es sobre-ingeniería.
- **Frase:** "Empiezo con la orquestación más liviana que sirva; solo justifico un workflow engine durable cuando hay múltiples pasos que deben sobrevivir fallos y necesito exactly-once."

---

## Common Deep Dives

### "¿Qué pasa si el proceso que corre tu saga crashea a mitad de camino?"

Este es el corazón del tema. Respuestas:
- **Persistir el estado del workflow** antes de cada paso, para poder retomar. Con un durable execution engine, el replay del event log reconstruye el estado y continúa desde donde quedó — sin re-ejecutar pasos completados.
- **Idempotencia:** cada paso debe ser idempotente, porque tras un crash podés reintentar un paso que quizás ya se ejecutó parcialmente. Usás idempotency keys.
- **Sin engine:** máquina de estados persistida en base + un proceso de recuperación (polling loop) que detecta workflows a medio hacer y los reanuda.
- **Compensaciones idempotentes:** si el crash ocurre durante el rollback, poder re-ejecutar las compensaciones sin daño.

Un caso real de este exacto problema es el de **Airbnb**: durante su migración a microservicios, conexiones caídas y timeouts en el path de pagos distribuido causaban cobros múltiples a guests y payouts múltiples a hosts, porque el handling entre servicios no era idempotente ante el reintento tras una falla a mitad de camino. El fix fue la librería de idempotencia interna **Orpheus** (fases Pre-RPC / RPC / Post-RPC, idempotency key, y lectura solo del master DB para evitar replica lag) — un ejemplo concreto de "idempotency key + persistencia del progreso" aplicado en producción (fuente: [Avoiding Double Payments in a Distributed Payments System](https://medium.com/airbnb-engineering/avoiding-double-payments-in-a-distributed-payments-system-2981f6b070bb)).

Aclaración de honestidad: no hay un postmortem público nombrado de una **compensación de saga** que haya fallado o no haya corrido dejando estado inconsistente (el clásico "orden FAILED pero inventario todavía RESERVED") — ese modo de falla aparece solo en tutoriales genéricos y posts anónimos, sin empresa atribuible.

### "¿Cómo vas a manejar actualizaciones al workflow?"

#### Workflow Versioning

Cuando cambiás la lógica de un workflow, las instancias **en vuelo** fueron iniciadas con la versión vieja. Si al hacer replay usás el código nuevo, el determinismo se rompe (el historial no coincide). Solución: **versionar** los workflows — las instancias viejas siguen con su versión, las nuevas usan la nueva. Los engines exponen APIs de versioning para bifurcar la lógica según la versión de la instancia.

#### Workflow Migrations

Para migrar instancias en vuelo a una nueva versión (cuando no querés arrastrar la vieja para siempre), necesitás una estrategia de migración: drenar las instancias viejas hasta completarlas, o migrar su estado explícitamente. Es delicado por el determinismo del replay.

### "¿Cómo mantenemos el tamaño del estado del workflow bajo control?"

El event log crece indefinidamente en workflows largos. Soluciones:
- **Snapshots / continue-as-new:** periódicamente persistís un snapshot del estado y truncás el historial, arrancando una "nueva" ejecución con el estado comprimido (Temporal tiene `continue-as-new`). Esto no es solo una buena práctica: en Temporal el Event History tiene un warning a **10.240 events / 10 MB** y un hard limit de **51.200 events / 50 MB** (son 10×1024 y 50×1024, por eso no son números redondos), y recomiendan no pasar de "unos pocos miles de events" usando Continue-As-New (fuente: [Workflow Execution limits](https://docs.temporal.io/workflow-execution/limits)). En AWS Step Functions el límite es más duro todavía: el historial de ejecución de un Standard Workflow tiene un máximo de **25.000 events**, y al alcanzarlo la ejecución **falla** (no trunca) — por eso ahí también hay que arrancar nuevas ejecuciones para no chocar la cuota (en Express el historial es ilimitado).
- No guardar payloads gigantes en el estado del workflow; guardar referencias (IDs, punteros a blob storage) en vez del dato completo. Esto no es solo una recomendación de diseño: es lo que te obliga a hacer el límite de payload de 256 KiB de Step Functions o los 2 MB de Temporal Cloud mencionados arriba.

### "¿Cómo lidiamos con eventos externos?"

Workflows que deben esperar señales de afuera (aprobación humana, callback de un pago, webhook):
- **Signals / waits:** el workflow se suspende esperando una señal externa, sin consumir recursos, y se despierta cuando llega. Los engines soportan esto nativamente.
- **Timeouts:** por si el evento externo nunca llega, definís un timeout con una rama de compensación.

### "¿Cómo garantizamos que el paso X corra exactamente una vez?" (exactly-once)

- **Idempotency keys:** cada paso lleva una clave única; si se reintenta, el servicio destino detecta la clave repetida y no re-ejecuta el efecto. El ejemplo canónico de qué pasa cuando esto falla es el caso **WooCommerce/Stripe**: un cliente fue cobrado **3 veces** por una renovación de suscripción porque la idempotency key estaba solo en el API legacy `/charges`, pero el pago realmente iba por `POST /payment_intents` **sin key** — los retries no se deduplicaban. Es el caso de libro de "el paso no era idempotente donde importaba" (fuente: [GitHub issue #2339](https://github.com/woocommerce/woocommerce-gateway-stripe/issues/2339)).
- **Transactional outbox:** en vez de escribir a la base y publicar a la cola por separado (que puede fallar entre medio), escribís el evento a una tabla outbox dentro de la misma transacción de base, y un proceso aparte lo publica — garantizando que el evento se emite si y solo si la transacción committeó. Vale aclarar que esto no es gratis operacionalmente: un reporte de producción de **Yotpo** corriendo outbox con Debezium + Kafka describe conectores fallando, binlogs rotados dejando offsets irrecuperables, y migraciones de host de DB rompiendo referencias de binlog (fuente: [Outbox with Debezium and Kafka: the hidden challenges](https://medium.com/yotpoengineering/outbox-with-debezium-and-kafka-the-hidden-challenges-998c00487ae4)). Aclaración de honestidad: tampoco hay un postmortem público nombrado del outbox pattern causando eventos duplicados que hayan golpeado usuarios — ese modo de falla también aparece solo en posts anónimos, sin empresa atribuible.
- **Durable execution engines** dan exactly-once para los pasos del workflow de fábrica, combinando persistencia del progreso + idempotencia.
- La verdad incómoda a nombrar: "exactly-once delivery" pura no existe en sistemas distribuidos; lo que se logra es "at-least-once delivery + idempotent processing = exactly-once effect". Un caso relacionado, aunque no es estrictamente una falla de compensación de saga sino de inconsistencia de replicación, es el de **GitHub el 21 de octubre de 2018**: una partición de red de 43 segundos disparó un failover Raft del orquestador, y los clusters MySQL East/West aceptaron writes que el otro no tenía (split-brain), dejando ~40 minutos de writes sin replicar que se reconciliaron manualmente desde los binlogs — ilustra el costo real de reconciliar estado distribuido inconsistente cuando las garantías de exactly-once/consistencia se rompen (fuente: [Oct 21 post-incident analysis](https://github.blog/news-insights/company-news/oct21-post-incident-analysis/)).

---

## Conclusion

El patrón se resuelve eligiendo el nivel mínimo de coordinación:
1. **¿Cabe en una transacción de base?** → transacción. Fin.
2. **¿Cruza servicios pero es simple (3-4 pasos, sin branching)?** → saga por coreografía (event-driven).
3. **¿Complejo, condicional, cambia seguido, necesita visibilidad?** → orquestación.
4. **¿Debe sobrevivir fallos arbitrarios con exactly-once?** → durable execution engine (Temporal, Step Functions).

Los invariantes que atraviesan todo: compensaciones idempotentes definidas de antemano, acciones no compensables al final, idempotency keys / outbox para exactly-once, y la conciencia de que la saga sacrifica isolation. La frase que resume la actitud: elegí el enfoque que encaja con tu contexto operativo, no el que se veía mejor en la última charla de conferencia — y nunca traigas un workflow engine para lo que una transacción resuelve.

---

## References

- Fuente base: HelloInterview — pattern "Multi-step Processes" (compilado con Temporal, InterviewNoodle, DesignGurus, DEV Community).
- [AWS Step Functions service quotas](https://docs.aws.amazon.com/step-functions/latest/dg/limits.html)
- [Temporal — Detecting Workflow failures](https://docs.temporal.io/encyclopedia/detecting-workflow-failures)
- [Temporal — Workflow Execution limits](https://docs.temporal.io/workflow-execution/limits)
- [Temporal — Cloud limits](https://docs.temporal.io/cloud/limits)
- [WooCommerce/Stripe gateway — issue #2339 (triple charge)](https://github.com/woocommerce/woocommerce-gateway-stripe/issues/2339)
- [Airbnb — Avoiding Double Payments (Orpheus)](https://medium.com/airbnb-engineering/avoiding-double-payments-in-a-distributed-payments-system-2981f6b070bb)
- [GitHub — Oct 21 2018 post-incident analysis](https://github.blog/news-insights/company-news/oct21-post-incident-analysis/)
- [Yotpo — Outbox with Debezium and Kafka: the hidden challenges](https://medium.com/yotpoengineering/outbox-with-debezium-and-kafka-the-hidden-challenges-998c00487ae4)

## Related

- [[Saga]]
- [[Two-Phase Commit]]
- [[Event Sourcing]]
- [[Event-Driven Architecture]]
- [[Message Queue]]
- [[Dead Letter Queue]]
- [[Idempotency]]
- [[Pub-Sub]]
