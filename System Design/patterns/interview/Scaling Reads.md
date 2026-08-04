---
title: Scaling Reads
reading:
  total_words: 2140
  read_words: 2140
  pct: 100
  last_read: 2026-07-02
source: Scaling Reads — Guía para entrevistas de System Design (estructura del pattern "Scaling Reads" de HelloInterview)
author: Compilado a partir de artículos de system design (Facebook Memcache/USENIX, Box Tech Blog, Layrs, Codelit, AlgoMaster, entre otros)
created: 2026-07-02
tags:
  - system-design/scalability
  - type/pattern
  - status/permanent
aliases:
  - Scaling Reads
  - Escalar Lecturas
updated: 2026-07-02
---

> [!note] Definición
> **Scaling Reads** es el patrón para servir volúmenes enormes de lecturas con baja latencia sin fundir la base de datos. Se aplica a sistemas read-heavy (ratio read:write de 10:1 a 100:1+) y funciona como una escalera: primero optimizás dentro de la base (índices, denormalización), luego escalás horizontalmente (read replicas, sharding) y finalmente agregás capas de cache externas (Redis, CDN). Subís un escalón solo cuando el anterior no alcanza.

## The Problem

En la mayoría de las aplicaciones, el tráfico de lectura crece **mucho más rápido** que el de escritura. La relación read:write típica arranca en 10:1 y llega a 100:1 o más. Ejemplo: en Instagram, abrís la app y ves docenas de fotos que requieren cientos de queries (metadata, info de usuario, engagement), pero quizás posteás una vez por día — una sola escritura.

Las lecturas se vuelven el primer cuello de botella. El problema es servir volúmenes enormes de lecturas con baja latencia sin fundir la base.

> **Regla de oro (progresión natural):** optimizar dentro de la base (índices, denormalización) → escalar horizontalmente (read replicas, sharding) → agregar capas de cache externas (Redis, CDN). Subís un escalón solo cuando el anterior no alcanza. No arranques con un cluster de cache distribuido para un problema que un índice resuelve.

---

## The Solution

### Optimize Within Your Database

#### Indexing

Un índice hace las lecturas rápidas a costa de escrituras un poco más lentas y espacio extra. **B-tree** es el default (soporta exact match y range queries); hash index solo exact match. Para búsqueda full-text → índice invertido (Elasticsearch); para geoespacial → PostGIS o el tipo geo de Redis.

**Regla:** elegí los índices según tus queries reales. Cada índice acelera lecturas pero penaliza cada escritura (hay que actualizarlo). No indexes todo — indexá lo que consultás.

**Frase:** "Antes de escalar horizontalmente, reviso si un índice sobre la columna que filtro resuelve la lentitud. Un full table scan por un `LIKE '%x%'` sin índice es la causa más común de queries lentas."

#### Hardware Upgrades

Scaling vertical: más CPU, RAM, SSD más rápido. El hardware moderno es potente — una sola instancia de Postgres aguanta decenas de miles de TPS y varios TB. Es la opción más simple y a menudo suficiente. **Trampa:** confiar solo en hardware puede enmascarar problemas más profundos de diseño (una query mal indexada). Pero para muchos casos, subir la máquina compra tiempo sin complejidad.

#### Denormalization Strategies

Los esquemas normalizados minimizan duplicación pero requieren joins caros al leer. La **denormalización** cambia complejidad de escritura por velocidad de lectura: guardás el dato pre-joineado o duplicado.

Ejemplo: en vez de joinear `users` y `posts` en cada request de feed, guardás `author_name` y `author_avatar_url` directamente en el documento del post. La lectura se vuelve un fetch de un solo documento — sin joins. Cuando Alice cambia su nombre, actualizás todos sus posts asincrónicamente.

Casos reales: Facebook denormaliza likes y comment counts en el registro del post para render instantáneo del feed; Twitter guarda metadata del tweet junto a engagement en la misma tabla.

**Regla:** denormalizá cuando el dato se lee frecuentemente y se actualiza poco. Automatizá la consistencia con background jobs o materialized views. **Trampa:** una vez que denormalizás, es cuestión de tiempo hasta que diverja de la fuente de verdad — no abuses.

**Materialized views:** resultado de una query precomputado y guardado como tabla, refrescado on-schedule o on-demand. Mueve el trabajo caro fuera del hot path de lectura. Los benchmarks muestran mejoras de hasta **~10x** en tiempo de ejecución al reemplazar una query cara por una materialized view, y a diferencia de las views normales, las materialized views **soportan índices**, lo que amplifica todavía más la ganancia ([Speeding up queries with PostgreSQL materialized views — Aarhusworks](https://aarhusworks.com/postgresql/2024/11/06/speeding-up-db-queries-with-postgres-materialized-view.html), [RisingWave — Mastering Materialized Views in PostgreSQL](https://risingwave.com/blog/mastering-materialized-views-in-postgresql/)). **Gotcha en producción:** el `REFRESH MATERIALIZED VIEW` clásico en Postgres toma un lock exclusivo que **bloquea lecturas durante el refresh**; se evita con `REFRESH ... CONCURRENTLY`, que a su vez **requiere un índice UNIQUE** sobre la view — es el edge case que separa "sé que existen" de "las operé en producción" ([RisingWave — Mastering Materialized Views in PostgreSQL](https://risingwave.com/blog/mastering-materialized-views-in-postgresql/)).

### Scale Your Database Horizontally

#### Read Replicas

Copias de la base optimizadas para lectura. Un nodo **primary** maneja todas las escrituras; una o más **réplicas** sirven las lecturas, absorbiendo la carga del primary. Instagram usa réplicas por todo el mundo.

**La trampa central — replication lag:** las escrituras al primary tardan en propagarse a las réplicas, así que una lectura de réplica puede devolver datos **levemente stale**. Esto genera experiencias inconsistentes (el usuario escribe algo y no lo ve al leer de una réplica). Mitigación: **read-your-own-writes** (leer del primary los datos que el mismo usuario acaba de escribir), o esperar la propagación. Otros costos: complejidad de failover (promover una réplica a primary) y costo de infra.

Para poner un número concreto sobre la mesa: en Aurora, las réplicas comparten el mismo volumen de storage que el primary, así que el lag típico es de **decenas de milisegundos** y "usually much less than 100 ms" ([Amazon Aurora — Replication docs](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.Replication.html), [Amazon Aurora FAQs](https://aws.amazon.com/rds/aurora/faqs/)). Contraste importante: en réplicas clásicas MySQL/Postgres (que replican el binlog/WAL y lo re-aplican) el lag puede ser mucho mayor bajo carga de escritura. El `AuroraReplicaLag` mide el delay entre el commit en storage y cuando el reader aplica el cambio a su buffer cache en memoria; cuando explota casi siempre es por una de cuatro cosas: **reader under-provisioned, queries pesadas comiéndose la CPU, write throughput extremo en el writer, o presión del buffer pool** — útil para responder "¿qué hacés cuando el lag sube?" con causas concretas ([Troubleshoot Aurora Reader Instance Lag — OneUptime](https://oneuptime.com/blog/post/2026-02-12-troubleshoot-aurora-reader-instance-lag/view)).

**Regla:** las réplicas escalan **lecturas**, no escrituras (todas las escrituras siguen yendo al primary). Si el cuello es de escritura, esto no ayuda — necesitás sharding (ver Scaling Writes).

#### Database Sharding

Partís los datos en múltiples bases independientes por una shard key (user_id, video_id). YouTube shardea metadata de videos por video_id, así ninguna máquina maneja todo.

**Trampas a nombrar:** queries cross-shard (búsquedas globales, agregaciones) se vuelven difíciles y lentas; una shard key mal elegida crea "hot shards"; resharding es pesado y propenso a errores; las transacciones cross-shard sacrifican atomicidad o performance.

**Regla:** sharding es la última opción para lecturas — primero réplicas y cache. Shardeás cuando ni las réplicas ni el cache alcanzan, o cuando el dataset no entra en una máquina.

### Add External Caching Layers

#### Application-Level Caching

El patrón que usás el 90% de las veces: **cache-aside con Redis**. Al leer, chequeás el cache; si hay hit lo devolvés (~1ms vs 20-50ms de query a base, speedup de 20-50x); si hay miss, vas a la base, guardás en cache y devolvés.

**Multi-tier:** L1 in-process (nanosegundos, por instancia, para las claves súper-hot) + L2 distribuido (Redis, compartido). Target de hit rate: >90%.

#### CDN and Edge Caching

Servidores en el borde que cachean contenido cerca del usuario. En system design, **el momento más seguro para introducir un CDN es cuando servís media estática a escala** — decí esa razón primero, después expandí. El CDN funciona mejor con contenido inmutable o versionado (cache-busting URLs tipo `/style.abc123.css`); para dinámico, TTLs cortos con `stale-while-revalidate`. Un sitio mayormente estático alcanza fácilmente un **cache hit ratio de 95-99%**, mientras que uno con mucho contenido dinámico cae muy por debajo — el número que conviene citar en la mesa: "para media estática/versionada apunto a 95%+; para dinámico el ratio se desploma y ahí no gana tanto un CDN" ([Cloudflare — What is a cache hit ratio](https://www.cloudflare.com/en-gb/learning/cdn/what-is-a-cache-hit-ratio/)). Netflix empuja videos a caches a nivel ISP con su Open Connect CDN: cerca del **95% del tráfico** de Netflix se sirve directamente desde appliances Open Connect dentro de los ISPs, sin tocar la infra de AWS, con una escala de **19.000+ appliances en 1.500+ ubicaciones de ISP** en más de 100 países, sirviendo el 100% del tráfico de video ([About Netflix — How Netflix Works With ISPs](https://about.netflix.com/en/news/how-netflix-works-with-isps-around-the-globe-to-deliver-a-great-viewing-experience), [Netflix Open Connect Overview (PDF)](https://openconnect.netflix.com/Open-Connect-Overview.pdf)).

---

## When to Use in Interviews

### Common Interview Scenarios

Casi todo lo read-heavy: feeds (Twitter, Instagram, FB News Feed), Bitly (resolver URLs), YouTube (metadata), Yelp, top-K, distributed cache, news aggregator, rate limiter, FB post search. Si el problema tiene ratio read:write alto, este patrón aplica.

### When NOT to Use

- Si el volumen de lectura es bajo, no metas réplicas ni cache. Una base sola alcanza.
- Si el problema es de **escritura**, este patrón no es la respuesta (andá a Scaling Writes).
- **Trampa del cache:** "si podés vivir con la latencia y la carga sin cachear, no lo agregues." El cache introduce problemas de correctitud (invalidación, staleness) difíciles de debuggear. Solo lo agregás cuando el cliente no tolera la latencia o la base no aguanta la carga.

---

## Common Deep Dives

### "¿Qué pasa cuando tus queries empiezan a tardar más a medida que crece el dataset?"

- **Índices:** lo primero. Una query que hacía full scan sobre millones de filas se resuelve con el índice correcto.
- **Denormalización / materialized views:** precomputar joins caros.
- **Réplicas:** distribuir la carga de lectura.
- **Particionar/shardear** si el dataset ya no entra o una tabla es demasiado grande.
- **Paginación por cursor** en vez de offset para no escanear desde el principio en cada página.

### "¿Cómo manejás millones de lecturas concurrentes de la misma data cacheada?" (hot key)

Una hot key es una entrada que recibe muchísimo más tráfico que el resto (ej. `user:taylorswift` en Twitter, millones de req/s). Aunque el hit rate sea alto, esa clave sola puede fundir un nodo de Redis o un shard.

- **Replicar la hot key:** guardar el mismo valor en múltiples nodos y balancear las lecturas.
- **Key-space salting:** sufijo (`taylor-swift-{0..9}`) para dispersar en varios nodos.
- **L1 in-process cache:** para las súper-hot, servirlas desde la memoria del app server, sin siquiera tocar Redis.

### "¿Qué pasa cuando múltiples requests intentan reconstruir una entrada de cache expirada al mismo tiempo?" (thundering herd / cache stampede)

Este es el deep-dive estrella, con números que impresionan. Escenario: un feed cacheado con TTL de 1 hora sirviendo 10.000 req/s. Cuando expira, los próximos 10.000 requests de ese segundo hacen miss y le pegan a la base a la vez; si la query tarda 2s, mandaste 20.000 queries concurrentes. El dato canónico: en Facebook (paper de USENIX 2013 "Scaling Memcache"), una sola clave expirando en mal momento spikeaba las queries de 1.300 a 17.000 por segundo (13x).

**Mitigaciones (nombrá varias):**
- **Leases / per-key locking:** solo un request "gana" el lease y va a la base; los demás esperan (o sirven stale) hasta que se popula. Facebook cortó las queries pico un **92%** solo con leases.
- **Request coalescing (singleflight):** colapsás las requests idénticas en una sola llamada a la base.
- **Probabilistic early expiration:** refrescás la clave un poco **antes** de que expire, de forma probabilística, para que nunca haya un miss masivo simultáneo.
- **TTL jitter:** TTLs escalonados/aleatorizados para que muchas claves no expiren al mismo instante (previene "cache avalanche").
- **Background refresh:** un worker refresca las top-N hot keys antes de que expiren.

### "¿Cómo manejás la invalidación de cache cuando los updates deben ser visibles inmediatamente?"

Cache invalidation es "uno de los dos problemas más difíciles de CS" (Phil Karlton).

- **Invalidar en la escritura:** al actualizar la base, borrás la entrada de cache para que la próxima lectura traiga el dato fresco. "Cuando el usuario actualiza su perfil, borro la entrada de cache."
- **TTL cortos:** dejás que el dato stale viva un ratito si tolerás eventual consistency.
- **Event-driven (CDC/outbox):** un pipeline de invalidación escucha los cambios de la base y borra las claves afectadas.
- **La race condition clave:** entre que un reader lee de la base (dato viejo) y lo escribe al cache, puede llegar una invalidación y borrarlo — dejando el valor viejo cacheado indefinidamente. Solución: la invalidación debe ser más rápida que el replication lag, y/o usar **versioning** (la entrada de cache lleva número de versión; solo invalidás si las versiones coinciden). Así Meta logra consistencia de "10 nueves".
- **Cache failure:** si Redis se cae, fallback a la base con circuit breakers para no fundirla, y quizás un L1 in-process como último recurso.

---

## Conclusion

El patrón es una escalera: índices y denormalización dentro de la base → read replicas → sharding → cache (app + CDN), subiendo solo cuando hace falta. Los ejes: ¿el cuello es de lectura o escritura? (réplicas solo ayudan a lectura), ¿tolerás staleness? (define TTL vs invalidación agresiva), ¿hay hot keys? (replicación/salting).

Los tres deep-dives que casi siempre caen: thundering herd (leases, coalescing, early expiration, jitter), hot keys (replicar/saltear) e invalidación (en-write, versioning, evitar la race). La frase que resume la actitud: si podés vivir sin cache, no lo agregues — pero cuando lo hacés, traé los controles operativos concretos (TTL jitter, request coalescing, circuit breakers) en lugar de solo decir "uso Redis".

---

## References

- [HelloInterview — Scaling Reads pattern](https://www.hellointerview.com/learn/system-design/patterns/scaling-reads) (estructura base del documento fuente)
- Nishtala et al., "Scaling Memcache at Facebook", USENIX NSDI 2013 (dato del spike 1.300 → 17.000 qps y el 92% de reducción con leases)
- [Cloudflare — What is a cache hit ratio](https://www.cloudflare.com/en-gb/learning/cdn/what-is-a-cache-hit-ratio/)
- [About Netflix — How Netflix Works With ISPs](https://about.netflix.com/en/news/how-netflix-works-with-isps-around-the-globe-to-deliver-a-great-viewing-experience)
- [Netflix Open Connect — Overview (PDF)](https://openconnect.netflix.com/Open-Connect-Overview.pdf)
- [Amazon Aurora — Replication docs](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.Replication.html)
- [Amazon Aurora FAQs](https://aws.amazon.com/rds/aurora/faqs/)
- [Troubleshoot Aurora Reader Instance Lag — OneUptime](https://oneuptime.com/blog/post/2026-02-12-troubleshoot-aurora-reader-instance-lag/view)
- [Speeding up queries with PostgreSQL materialized views — Aarhusworks](https://aarhusworks.com/postgresql/2024/11/06/speeding-up-db-queries-with-postgres-materialized-view.html)
- [RisingWave — Mastering Materialized Views in PostgreSQL](https://risingwave.com/blog/mastering-materialized-views-in-postgresql/)

## Related

- [[Primary-Replica]]
- [[Cache-Aside]]
- [[Read-Through]]
- [[Content Delivery Network]]
- [[Sharding]]
- [[CQRS]]
