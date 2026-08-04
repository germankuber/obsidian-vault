---
title: Dealing with Contention
reading:
  total_words: 1980
  read_words: 1980
  pct: 100
  last_read: 2026-07-02
source: HelloInterview — System Design pattern (investigado con AlgoMaster, Redis/antirez, Martin Kleppmann, Databricks, System Design Codex)
author: HelloInterview (compilado)
created: 2026-07-02
tags:
  - system-design/consistency
  - system-design/concurrency
  - type/pattern
  - status/permanent
aliases:
  - Dealing with Contention
  - Manejo de Contención
  - Contention
updated: 2026-07-02
---

> [!note] Definición
> La **contención** aparece cuando múltiples clientes intentan modificar el mismo recurso limitado y compartido al mismo tiempo, y el resultado depende del orden (read-modify-write). El pattern es una escalera de decisión —conditional write → optimistic → pessimistic → [[Distributed Lock|distributed lock]]— donde subís un peldaño solo cuando el anterior no cubre tu caso. Los ejes son la tasa de conflictos, el alcance del recurso y la criticidad de la correctitud.

## The Race Condition

La contención aparece cuando **múltiples clientes intentan modificar el mismo recurso al mismo tiempo** y el resultado depende del orden — sin coordinación, pisás datos o corrompés el estado.

El ejemplo canónico: 10.000 usuarios clickean "Comprar" sobre la última entrada de un concierto simultáneamente. Sin control de concurrencia, **sobrevendés**. Otros ejemplos: reservar el último asiento en Ticketmaster, pujar en una subasta, decrementar inventario, transferir plata entre cuentas.

La estructura del problema siempre es la misma: **read-modify-write**. Leés un valor (stock = 1), lo modificás (stock = 0), lo escribís. Si dos transacciones leen el mismo "1" antes de que cualquiera escriba, ambas creen que quedaba stock y las dos venden.

**Señal para reconocer el patrón en la entrevista:** cualquier recurso limitado y compartido bajo demanda concurrente — "el último X", "solo N disponibles", contadores, balances, reservas con ventana de tiempo.

---

## The Solution

Hay un espectro de mecanismos, del más liviano al más pesado. La regla es **empezar por el más simple que resuelva tu caso**, no saltar directo a locks distribuidos.

### Conditional Writes

**Cómo funciona:** hacés la escritura condicional a que el valor no haya cambiado desde que lo leíste, en una sola operación atómica de la base. En SQL: `UPDATE seats SET status='booked' WHERE id=42 AND status='available'`. Si otro ya lo reservó, el `WHERE` no matchea, afecta 0 filas, y sabés que perdiste la carrera. En DynamoDB son las Conditional Expressions; en Redis, operaciones atómicas como `SETNX` o `INCR`.

**Por qué es la primera opción:** es la más simple, no necesita locks explícitos ni infraestructura extra, y la atomicidad la garantiza la base. Para muchísimos casos (decrementar stock, reservar un asiento) alcanza con esto.

Este mismo patrón de version-check no es exclusivo de bases SQL ni de Redis: Google Cloud Storage lo aplica a nivel de object storage con sus **generation preconditions** — el número de generación del objeto funciona como precondición, y una escritura que colisiona falla con **HTTP 412 Precondition Failed**, forzando un read-modify-write nuevo. Es compare-and-swap por versión, la misma idea de la escritura condicional pero sobre blobs en vez de filas ([GCS — Generations and Preconditions](https://download.huihoo.com/google/gdgdevkit/DVD1/developers.google.com/storage/docs/generations-preconditions.html)).

**Frase para la mesa:** "Antes de meter locks, uso una escritura condicional atómica — un `UPDATE ... WHERE status='available'`. Si afecta cero filas, el recurso ya se tomó y devuelvo error al cliente."

### Pessimistic Locking

**Filosofía:** asume que los conflictos **van a pasar** y los previene adquiriendo un lock **antes** de tocar el dato. Mientras una transacción tiene el lock, las demás esperan en fila. En SQL: `SELECT ... FOR UPDATE`.

**Comportamiento:** crea contención en **tiempo de lectura** — se acumulan threads bloqueados esperando el lock. Con las 10.000 entradas: 9.999 requests esperan en fila mientras una completa.

**Cuándo usarlo:** alta contención (conflictos frecuentes), donde el retry es caro, y donde necesitás garantía fuerte de exclusividad — sistemas bancarios, transferencias.

**Contras / trampas:** los locks reducen concurrencia y throughput. Una transacción que crashea sosteniendo un lock puede **deadlockear** el sistema hasta que expire el timeout (típicamente 30-60s). El lock timeout es un parámetro de tuning crítico: muy corto abortás transacciones legítimas largas; muy largo, procesos crasheados retienen recursos de rehén. Regla práctica: mantené los locks el menor tiempo posible y lockéa lo mínimo (preferí row-level sobre table-level).

#### Common Failure Modes

- **Deadlock:** dos transacciones esperan un recurso que la otra tiene. Se previene adquiriendo los locks **siempre en el mismo orden determinístico** (ej. por ID ascendente), con lock timeouts, y detección de ciclos por parte de la base.
- **Lock holder crash:** el que tiene el lock muere y nadie lo libera. Se mitiga con TTL/timeout en el lock.
- **Long-held locks:** transacciones largas que estrangulan la concurrencia. No debés mantener un lock de base abierto durante una operación larga (ej. una ventana de reserva de 5 minutos) — eso no es para lo que están los locks de base.

### Optimistic Concurrency Control (OCC)

**Filosofía:** asume que los conflictos son **raros** y solo chequea al momento de commit. Deja que todas las transacciones procedan en paralelo sin lockear, y valida antes de escribir.

**Cómo funciona — version numbers:** cada fila tiene una columna `version`. Leés el dato con su versión (v=5), hacés tu cálculo, y al escribir: `UPDATE ... SET value=X, version=6 WHERE id=42 AND version=5`. Si otro escribió primero, la versión ya es 6, tu `WHERE` no matchea, y tu update falla → reintentás con el dato fresco.

**Comportamiento:** descubre conflictos en **tiempo de escritura** — se acumulan intentos fallidos y retries, no threads bloqueados. Con las 10.000 entradas: las 10.000 corren libres, una gana sin bloquear a las demás, 9.999 fallan y reintentan.

**Cuándo usarlo:** aplicaciones **read-heavy con baja tasa de conflictos** (feeds, dashboards, CMS donde muchos leen y pocos escriben), y sistemas distribuidos donde lockear es caro o impráctico. Es el default para el caso 95% lecturas / 5% escrituras.

**La trampa a nivel escala — retry storms:** bajo **alta** contención, OCC degenera. Si 100 transacciones colisionan, 99 fallan y reintentan simultáneamente, creando otra ola de colisión. Sin **exponential backoff + jitter**, convertís un problema de concurrencia en un thundering herd. Por eso OCC es malo para alta contención: ahí conviene pessimistic.

Un motor real que muestra esta degeneración sin intermediarios es MongoDB con WiredTiger: es **OCC puro y sin lock-wait**, y ante un snapshot desactualizado no reintenta la operación en fila — devuelve `WT_ROLLBACK` y **reinicia toda la transacción** con un snapshot fresco. Es un restart completo, no un retry incremental, y es exactamente la degeneración de OCC bajo contención materializada en un motor de producción ([MongoDB Manual — WiredTiger Storage Engine](https://www.mongodb.com/docs/manual/core/wiredtiger/)).

**Regla de decisión pessimistic vs optimistic:**

| | Pessimistic | Optimistic (OCC) |
|---|---|---|
| Asume | Conflictos frecuentes | Conflictos raros |
| Cuándo detecta | Al leer (bloquea) | Al escribir (falla y retry) |
| Acumula | Threads bloqueados | Intentos fallidos |
| Mejor para | Alta contención, writes frecuentes al mismo dato | Read-heavy, baja contención, distribuido |
| Costo | Menor throughput, riesgo de deadlock | Retry storms si hay mucha contención |

Muchos sistemas de producción combinan: MVCC como default, pessimistic locks sobre las filas hot conocidas, y retry logic a nivel aplicación para OCC.

### Isolation Levels

Los niveles de aislamiento de las transacciones ACID definen qué anomalías de concurrencia permitís, del más débil al más fuerte:

- **Read Uncommitted:** podés leer datos no committeados (dirty reads). Casi nunca se usa.
- **Read Committed:** solo leés datos committeados. Default de Postgres. Suficiente para la mayoría de las apps web y catálogos (bien servido con MVCC). Vale la pena notar que esto no es solo el nivel de aislamiento default: PostgreSQL usa **MVCC optimista por default** en general — las lecturas nunca bloquean escrituras, y los locks pessimistic (row-level o advisory) hay que pedirlos explícitamente. El framework de "elegir entre OCC y pessimistic" es una decisión de diseño real, pero en Postgres el punto de partida ya es optimista ([PostgreSQL Docs — Ch. 13 Concurrency Control](https://www.postgresql.org/docs/current/mvcc.html)).
- **Repeatable Read:** una fila leída dos veces en la misma transacción da lo mismo.
- **Serializable:** el más fuerte; las transacciones se comportan como si corrieran en serie. Necesario para requisitos estrictos (financieros, compliance), típicamente con 2PL (two-phase locking).

**Regla para la mesa:** más aislamiento = más correctitud pero menos concurrencia/performance. Elegí el más débil que tolere tu caso. No todo el sistema necesita el mismo nivel.

### Distributed Locks

**Cuándo aparecen:** cuando el recurso a proteger **no vive en una sola base** que te dé atomicidad — coordinás entre servicios, o el lock debe durar más que una transacción de base (ej. la ventana de reserva de 5 minutos de Ticketmaster). Ahí usás un lock externo, típicamente Redis. (Ver [[Distributed Lock]].)

**Cómo funciona (Redis single-instance):** `SET lock_key random_value NX PX 30000` — setea la clave solo si no existe (`NX`), con expiración de 30s (`PX`). El `random_value` identifica al dueño; para liberar, borrás solo si el valor coincide (con un script Lua atómico: check-and-delete). El TTL garantiza que si el cliente crashea, el lock se libera solo.

**Datos y debate importantes (caen en deep dives):**
- El lock single-instance de Redis es un **single point of failure**: si ese nodo cae o particiona, la seguridad se rompe (la replicación de Redis es asíncrona, así que un failover puede dejar dos masters creyendo que tienen el lock).
- **Redlock** es el algoritmo multi-master de Redis: N nodos independientes (típicamente **5**), adquirís el lock en la mayoría (3 de 5), tolera 2 caídas. Los nodos deben ser totalmente independientes (sin replicación, idealmente en distintas AZ).
- **La crítica de Martin Kleppmann (hay que conocerla):** Redlock hace suposiciones peligrosas sobre timing y clocks, y **no genera fencing tokens**. Su recomendación: si necesitás locks solo por eficiencia (best-effort), usá el single-node simple; si los necesitás para **correctitud**, no uses Redlock — usá un sistema de consenso (etcd, ZooKeeper) que dé linealizabilidad.
- **La réplica de antirez ("Is Redlock safe?") también vale conocerla, porque el debate no quedó cerrado del lado de Kleppmann.** Dos contra-argumentos concretos: (a) Redlock **re-chequea el tiempo transcurrido después** de adquirir la mayoría y antes de otorgar el lock, así que una GC pause durante la adquisición queda cubierta por diseño; (b) el fix de fencing tokens de Kleppmann ya asume que el recurso protegido puede imponer tokens monotónicos crecientes — pero si puede hacer eso, el token random único de Redlock cumple el mismo rol. antirez concede la preocupación por el clock monotónico, pero pide testing estilo Jepsen en vez de descartar Redlock de plano ([antirez.com — Is Redlock safe?](https://antirez.com/news/101)).
- **Fencing token:** un número monotónicamente creciente que el lock entrega en cada adquisición. El recurso protegido rechaza operaciones con un token menor al último que vio. Esto protege contra el caso en que un cliente pausa (GC pause) más allá del TTL, pierde el lock sin saberlo, y otro ya lo tomó — sin fencing, ambos escribirían. El mecanismo tiene un ancestro directo con nombre propio: el **sequencer** de Chubby (2006), el lock service de Google — un token compuesto por nombre del lock, modo y generation number que el cliente reenvía al servicio protegido, el cual rechaza generation numbers viejos. Es el precursor real de los fencing tokens de Kleppmann y del `zxid` de ZooKeeper ([Google Research — The Chubby lock service, OSDI'06](https://research.google.com/archive/chubby-osdi06.pdf); [Kleppmann — How to do distributed locking](https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html)).
- **Auto-extension (lease + heartbeat):** para tareas largas, un thread de fondo extiende el TTL periódicamente (a la mitad del intervalo) mientras el cliente esté sano.

---

## Choosing the Right Approach

El orden mental, del más simple al más complejo:

1. **¿Alcanza con una escritura condicional atómica en la base?** → Conditional write. (Casi siempre sí para un solo recurso.)
2. **¿Read-heavy con conflictos raros?** → Optimistic (version numbers).
3. **¿Alta contención sobre el mismo dato, writes frecuentes?** → Pessimistic (`SELECT FOR UPDATE`).
4. **¿El lock cruza servicios o dura más que una transacción de base?** → Distributed lock (Redis), con fencing tokens si la correctitud es crítica (o etcd/ZooKeeper).

---

## When to Use in Interviews

### Recognition Signals

- "El último asiento / la última entrada / stock limitado."
- Contadores que múltiples clientes incrementan.
- Balances de cuenta, transferencias.
- Reservas con ventana de tiempo (hold de 5-10 min).
- Cualquier "solo uno puede ganar" concurrente.

### Common Interview Scenarios

Ticketmaster (reservar asiento sin doble-venta), Uber (no asignar el mismo driver a dos rides), subastas online (puja más alta gana), sistemas de pago, inventario de e-commerce, rate limiter (contador compartido).

### When NOT to overcomplicate

- Si una escritura condicional simple resuelve el caso, **no metas locks distribuidos**. Es la sobre-ingeniería más común y penalizada acá.
- Si el recurso no es realmente compartido o la contención es teórica, no gastes complejidad.
- No propongas Redlock (5 nodos) cuando un `SELECT FOR UPDATE` o un conditional write alcanzan. El interviewer quiere ver que elegís el nivel justo.

---

## Common Deep Dives

### "¿Cómo prevenís deadlocks con pessimistic locking?"

- **Ordenar la adquisición de locks:** siempre adquirí múltiples locks en el mismo orden determinístico (ej. por ID ascendente). Si toda transacción que necesita las filas A y B las lockea en orden A→B, nunca hay un ciclo. Este es el mecanismo principal.
- **Lock timeouts:** si no conseguís el lock en X tiempo, abortás y reintentás, rompiendo la espera indefinida.
- **Detección de deadlocks de la base:** Postgres/MySQL detectan ciclos y matan una de las transacciones (la víctima), que debe reintentar.
- **Minimizar el scope y la duración:** lockéa lo mínimo (row-level, no table-level) el menor tiempo posible.

### "¿Cómo manejás el problema ABA con optimistic concurrency?"

El problema ABA: leés un valor A, otro lo cambia a B y lo vuelve a A antes de que valides; tu versión "coincide" pero el dato pasó por estados intermedios que te perdiste, y podés corromper el estado.

- **Version numbers monotónicos:** en vez de comparar el **valor**, comparás un número de versión que **siempre incrementa**. Aunque el valor vuelva a A, la versión ya avanzó (v5 → v7), así que detectás que hubo cambios. Esto es lo que hace robusto al OCC frente a ABA.
- **Timestamps o contadores globales** cumplen el mismo rol.
- La clave: nunca bases la validación en el valor observado, sino en un token que nunca retrocede.

### "¿Qué pasa con la performance cuando todos quieren el mismo recurso?" (hot resource)

Cuando la contención se concentra en un solo recurso (el asiento de Taylor Swift, el contador global), ninguna estrategia de locking escala bien porque todo serializa sobre ese punto. El caso real que le puso números a esto fue la presale del Eras Tour de Taylor Swift en noviembre de 2022: Ticketmaster esperaba ~3.5M de fans verificados pero entraron **14M+ usuarios**, generando **~3.5 mil millones de requests** en total (~4x el pico previo del sitio) contra apenas **2.4M tickets** disponibles — terminó en una audiencia del Senado de EE.UU. en enero de 2023 ([Educative.io — Taylor Swift Ticketmaster Meltdown](https://www.educative.io/blog/taylor-swift-ticketmaster-meltdown)).

- **Optimistic degenera** en retry storms (mitigar con backoff + jitter, pero igual desperdiciás cómputo).
- **Pessimistic degenera** en una fila de espera enorme con latencia creciente.
- **Soluciones:**
  - **Particionar el recurso:** en vez de un contador único, N sub-contadores (`counter-{0..9}`) que sumás al leer. Esto es key-space salting aplicado a contención — distribuís la carga.
  - **Cola / serialización:** poné las requests en una cola FIFO y procesalas de a una (Ticketmaster usa una cola virtual). Convertís la contención en throughput controlado.
  - **Request collapsing:** colapsás requests idénticas para no pegarle N veces al recurso.
  - **Aceptar aproximación:** para rate limiting o contadores no críticos, correctitud aproximada alcanza y te ahorra la coordinación estricta.

---

## Conclusion

El patrón se domina con una escalera de decisión: conditional write → optimistic → pessimistic → distributed lock, subiendo un peldaño solo cuando el anterior no cubre tu caso. Los ejes que definen la elección son la **tasa de conflictos** (baja → optimistic; alta → pessimistic), el **alcance del recurso** (una base → locking de base; cruza servicios → lock distribuido) y la **criticidad de la correctitud** (crítica → fencing tokens o consenso; best-effort → Redis simple).

La frase que resume la actitud correcta: la mayoría de los problemas empiezan con soluciones de una sola base antes de escalar a coordinación distribuida — y el error más penalizado es saltar a Redlock cuando un `UPDATE ... WHERE` atómico alcanzaba.

---

## References

- HelloInterview — System Design pattern "Dealing with Contention" (base del documento).
- [PostgreSQL Docs — Concurrency Control (MVCC)](https://www.postgresql.org/docs/current/mvcc.html)
- [MongoDB Manual — WiredTiger Storage Engine](https://www.mongodb.com/docs/manual/core/wiredtiger/)
- [Google Cloud Storage — Generations and Preconditions](https://download.huihoo.com/google/gdgdevkit/DVD1/developers.google.com/storage/docs/generations-preconditions.html)
- [antirez — Is Redlock safe?](https://antirez.com/news/101)
- [Martin Kleppmann — How to do distributed locking](https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html)
- [Google Research — The Chubby lock service (OSDI'06)](https://research.google.com/archive/chubby-osdi06.pdf)
- [Educative.io — Taylor Swift Ticketmaster Meltdown](https://www.educative.io/blog/taylor-swift-ticketmaster-meltdown)

## Related

- [[Distributed Lock]]
- [[Quorum]]
- [[Two-Phase Commit]]
- [[Idempotency]]
- [[Race Condition]]
- [[Retry with Backoff]]
