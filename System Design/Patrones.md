---
title: Patrones de System Design
created: 2026-06-29
updated: 2026-07-05
tags:
  - system-design
  - type/case-study
  - status/permanent
aliases:
  - Patrones
  - Patrones de System Design
  - Hoja de estudio de patrones
---
# Patrones de System Design — Hoja de estudio detallada

> Compilado de 20 problemas. De Hello Interview (gratuitos): **Bitly, Dropbox, GoPuff, Ticketmaster, FB News Feed, Tinder, Rate Limiter, WhatsApp, Top-K, Uber, Web Crawler, Ad Click, YouTube**. De fuentes públicas (breakdowns premium, marcados): **Google Docs, Payment, Job Scheduler, Distributed Cache, Metrics Monitoring, Online Auction, ChatGPT**. Cada patrón tiene: **qué problema resuelve · cómo funciona por dentro · el trade-off · el gotcha que el entrevistador sondea**. Organizado por eje. Lista de frecuencia (cuántos de los 20 problemas usa cada patrón, ordenada de más usado a más específico) al final.

---

## Cómo usar esta hoja

Ante un problema nuevo: (1) aplica el **eje común** — casi siempre acierta; (2) identifica el/los **eje(s) específico(s)** según qué tiene el problema (recurso disputado, blobs, fan-out, geo, tiempo real, stream, etc.); (3) eso te dice qué deep dives atacar.

---

## EJE COMÚN — Read-heavy / escala (aplica a casi todo)

### Identificar el ratio read/write primero

**Problema:** la arquitectura entera depende de si lees mucho más de lo que escribes, y diseñar sin saberlo te lleva por el camino equivocado. **Mecanismo:** antes de dibujar nada, estima lecturas por escritura. Si el ratio es alto (Bitly 1000:1, GoPuff 20k qps de lectura, Ticketmaster con miles refrescando el mismo evento), optimizas para lectura desde el inicio: cache, réplicas, precomputación. Si es write-heavy (Ad Click 10k clicks/s, Tinder 4B swipes/día), priorizas ingestión: colas, batching, LSM stores. **Trade-off:** cada decisión posterior (qué DB, dónde cachear, cómo shardear) cae de este número. **Gotcha:** decirlo al inicio sin que te lo pidan es señal de seniority; saltártelo y luego "descubrir" que necesitabas cache es señal de junior.

### In-memory cache entre app y DB

**Problema:** la DB es el cuello de botella en lectura; ir a disco por cada request no escala. **Mecanismo:** Redis/Memcached delante de la DB. La jerarquía de latencia manda: memoria ~100ns, SSD ~0.1ms, HDD ~10ms → memoria es ~1000× más rápida que SSD. Flujo: cache hit responde directo; miss va a DB y rellena la entrada (cache-aside). **Trade-off:** ganas latencia y descargas la DB, pero introduces el problema de invalidación (datos potencialmente stale) y una dependencia más que puede caer. **Gotcha:** te van a preguntar la estrategia de invalidación y qué pasa en un cache miss masivo (thundering herd) — tener respuesta separa niveles. _Bitly, GoPuff, Ticketmaster, FB, YouTube (metadata)._

### Cache con TTL ajustado al perfil del dato

**Problema:** un TTL único para todo o cachea demasiado poco (poco beneficio) o sirve datos peligrosamente stale. **Mecanismo:** calibras el TTL por volatilidad del dato. Lo volátil (inventory de GoPuff que cambia con cada compra) → TTL corto, ~1 min. Lo casi inmutable (URL destino de Bitly, contenido de un post) → TTL largo, horas/días. Complementas con invalidación activa en la escritura cuando puedes. **Trade-off:** TTL corto = más frescura pero más misses (más carga a DB); TTL largo = menos carga pero más staleness. **Gotcha:** la regla mental es "read-heavy + cambia poco = cachear agresivo"; cachear con TTL largo algo volátil es el error clásico.

### Separar Read Service de Write Service

**Problema:** lecturas y escrituras tienen perfiles de carga opuestos, y un servicio único te obliga a escalar ambas juntas aunque solo una sea el cuello. **Mecanismo:** partes el servicio en dos despliegues independientes que comparten datos pero escalan por separado. Bitly separa redirects (lectura masiva) de creación de links; GoPuff separa availability de orders; Tinder separa el swipe service (4B writes/día) del profile service. **Trade-off:** ganas escalado independiente y aislamiento de fallos, a cambio de más complejidad operacional y coordinación de datos compartidos. **Gotcha:** justifica la separación con números (la asimetría de carga), no "porque sí" — si lecturas y escrituras son parejas, separar es over-engineering.

### Read replicas + leader para writes

**Problema:** un solo nodo no aguanta la lectura, pero necesitas un punto consistente para escribir. **Mecanismo:** el leader recibe todas las escrituras; las réplicas se sincronizan async y sirven lecturas. Reparte la carga de lectura entre N réplicas. Uber lo combina con geo-sharding (réplicas por región); GoPuff y Bitly lo usan directo. **Trade-off:** escalas lectura linealmente, pero las réplicas tienen replication lag → lecturas potencialmente stale (consistencia eventual). **Gotcha:** si una lectura necesita ver su propia escritura inmediata (read-your-writes), no puede ir a réplica — hay que rutear esa al leader o usar otra técnica.

### Estimación back-of-envelope para localizar el bottleneck

**Problema:** sin números no sabés qué componente se rompe primero ni qué arquitecturas están descartadas. **Mecanismo:** cuenta rápida en el momento que la necesitas (no toda al inicio). Traduces escala a una restricción concreta: Bitly 600k reads/s; Dropbox 50GB/100Mbps=1.1h de subida; GoPuff 20k qps; Uber 2M location updates/s = $200k/día en DynamoDB; Top-K 700k tps; Crawler 8 máquinas para 10B páginas en <5 días. **Trade-off:** ninguno real — es puro criterio; lo importante es la _implicación_ (qué descarta este número), no la precisión decimal. **Gotcha:** fudgea los números para que la mate sea fácil (100k seg/día en vez de 86400); el entrevistador evalúa el razonamiento, no tu aritmética mental. Traducir a dólares por usuario suma criterio de negocio.

### Horizontal scaling de servicios stateless

**Problema:** un servicio en una sola máquina tiene techo de CPU/memoria y es punto único de fallo. **Mecanismo:** si el servicio no guarda estado local (todo el estado vive en DB/cache externos), pones N instancias detrás de un load balancer y agregas más bajo demanda. Bitly, Uber (ride service), Ad Click (click processor), YouTube (video service). **Trade-off:** escalado casi infinito y tolerancia a fallos, pero exige que el servicio sea realmente stateless — cualquier estado en memoria local rompe el modelo. **Gotcha:** lo que parece stateless a veces no lo es (sesiones en memoria, conexiones WebSocket pegadas a un host) — ahí necesitas sticky sessions o mover el estado afuera.

---

## EJE: CONTENCIÓN — Recurso finito disputado

### Locking para evitar double-booking

**Problema:** dos usuarios pelean por el mismo recurso finito (asiento, último item, driver) y sin coordinación ambos creen que lo obtuvieron. **Mecanismo:** serializas el acceso con un lock. Dos sabores principales: lock distribuido con TTL en Redis (Ticketmaster reserva un asiento, Uber bloquea un driver) o transacción ACID que hace el check-and-set atómico (GoPuff). El que adquiere el lock procede; el resto espera o falla. **Trade-off:** garantizas consistencia (no double-booking) a costa de throughput (serializas lo que era paralelo) y de manejar locks colgados. **Gotcha:** qué pasa si el que tiene el lock se cae sin liberarlo — por eso los locks llevan TTL o las transacciones hacen rollback automático. _Ticketmaster, GoPuff, Uber, Tinder._

### Transacción ACID en una DB vs lock distribuido (trade-off)

**Problema:** necesitas atomicidad entre varios datos relacionados, y hay dos formas con consecuencias muy distintas. **Mecanismo:** opción A, todo en una Postgres con aislamiento `SERIALIZABLE` — la DB te da atomicidad gratis (GoPuff junta orders+inventory en una transacción). Opción B, datos en dos stores distintos coordinados por un lock distribuido — eliges la mejor DB para cada dato. **Trade-off:** A es más simple y sin estado inconsistente posible, pero acopla el escalado de ambos datos a una sola DB. B desacopla y deja optimizar cada store, pero abre crashes con estado a medias (hay que barrer) y deadlocks cuando los recursos se cruzan. **Gotcha:** la regla es "si necesitas atomicidad, colocar los datos en un mismo ACID store es lo más simple" — saltar a lock distribuido sin necesidad es complejidad gratis.

### Distributed lock con TTL (Redis)

**Problema:** necesitas un lock que sobreviva entre servicios distribuidos y que se libere solo si el dueño desaparece. **Mecanismo:** `SET key value NX EX 10` es atómico — pone la clave solo si no existe (NX) con expiración (EX). Ticketmaster mapea ticketId→userId con TTL 10min; Uber bloquea driverId con TTL 10s (la ventana de aceptación). El TTL garantiza que un crash no deja el recurso bloqueado para siempre. **Trade-off:** expiración nativa y simple, pero partes el read path (otra fuente de verdad además de la DB) y dependes de la disponibilidad de Redis. **Gotcha:** el clásico es el TTL que expira _durante_ el pago — el lock se suelta, otro reserva, y el primer pago entra tarde; se maneja con refund automático o extendiendo el lock. _Ticketmaster, Uber._

### Status implícito (alternativa al lock distribuido)

**Problema:** el lock distribuido parte tu fuente de verdad en dos (Redis + DB); a veces no lo necesitas. **Mecanismo:** modelas el estado del recurso como "available OR (reserved pero el timestamp de reserva ya expiró)". Una transacción corta lee el estado, confirma disponibilidad y reserva, todo en la misma DB. El recurso se considera libre por lógica de tiempo, no por una clave externa. **Trade-off:** una sola fuente de verdad (Postgres), más limpio, el sweep job que borra reservas viejas es solo cosmético; pero Redis gana en concurrencia extrema donde la contención sobre la fila sería brutal. **Gotcha:** funciona bien cuando la contención por recurso individual es moderada; si miles pelean por el _mismo_ asiento, el lock dedicado escala mejor. _Ticketmaster._

### Single-partition transaction (colocar datos relacionados en la misma partición)

**Problema:** en NoSQL no hay transacciones multi-partición, pero necesitas atomicidad entre dos registros relacionados. **Mecanismo:** fuerzas que ambos caigan en la misma partición construyendo una clave compartida. Tinder ordena los dos user IDs y forma `user_pair = min:max`, así el swipe A→B y el B→A viven en la misma partición Cassandra, y puedes chequear el match atómicamente con un batch o LWT single-partition. **Trade-off:** obtienes atomicidad sin coordinación distribuida, pero el diseño de la partition key queda atado a ese patrón de acceso, y partições con pares muy activos pueden crecer / volverse hot. **Gotcha:** las "lightweight transactions" de Cassandra (Paxos) solo dan linearizabilidad _dentro de una partición_ — por eso el truco de colocar todo junto es lo que las hace usables. _Tinder._

### Item vs Inventory (clase vs instancia)

**Problema:** confundir el tipo abstracto con el ejemplar físico lleva a un modelo de datos que no puede rastrear disponibilidad. **Mecanismo:** separas el catálogo (Item: "Cheetos", lo que el usuario busca) del stock físico (Inventory: cada paquete concreto en un DC específico, lo que el sistema decrementa al vender). El usuario consulta Items; el sistema reserva/descuenta Inventory para no vender la misma unidad dos veces. **Trade-off:** más entidades y joins, pero es la única forma de modelar disponibilidad real. **Gotcha:** la heurística es empezar por entidades concretas (el paquete) y subir a las abstractas (el producto) — al revés te perdés la granularidad que necesitas para el locking. _GoPuff, e-commerce, ticketing._

### Consistencia eventual como margen de ingeniería

**Problema:** la consistencia fuerte es cara (bloquea réplicas, async, cache); si el requisito no la exige, la estás pagando de gratis. **Mecanismo:** negocias explícitamente cuánta staleness tolera el producto y usas ese margen como palanca: habilita read replicas, cache con TTL, procesamiento async. Dropbox tolera segundos de desfase entre dispositivos; FB tolera ~1 min en el feed; Rate Limiter acepta que los nodos discrepen un poco. **Trade-off:** desbloqueas soluciones mucho más simples y escalables a cambio de que distintos lectores puedan ver estados ligeramente distintos por un rato. **Gotcha:** preguntá al entrevistador el nivel aceptable — asumir consistencia fuerte cuando no hace falta te complica el diseño; asumir eventual cuando se necesitaba fuerte (matching de Tinder, pago) es un error grave.

### Idempotencia en la API

**Problema:** con retries y entrega at-least-once, la misma operación puede llegar dos veces y duplicar efectos (doble cobro, doble click contado). **Mecanismo:** diseñas la operación para que ejecutarla N veces tenga el mismo efecto que una. Usas una idempotency key (la genera el cliente o el sistema) y verificas si ya la procesaste antes de actuar. `PUT` para acciones binarias (follow/unfollow) es naturalmente idempotente. **Trade-off:** necesitas almacenar/consultar las keys ya vistas (algo de estado y latencia extra), a cambio de seguridad ante duplicados. **Gotcha:** dónde dedup importa — en Ad Click tiene que ser _antes_ del stream, porque deduplicar dentro de la ventana de Flink no cruza el boundary del minuto y contás el duplicado dos veces. _Bitly (retry en colisión), Ticketmaster (webhook Stripe), FB (PUT follow), Ad Click (impression ID)._

---

## EJE: RATE LIMITING — Throttling y cuotas

### Algoritmos de rate limiting

**Problema:** distintos algoritmos cambian precisión, memoria y comportamiento ante bursts. **Mecanismo:** cuatro opciones. _Token bucket_ (el preferido, lo usa Stripe): un balde se rellena a ritmo constante, cada request consume un token, permite bursts hasta la capacidad del balde y rate sostenido por el refill. _Fixed window_: cuentas requests por ventana fija de 1 min — simple pero sufre boundary effects (200 requests legítimos en 2 segundos si caen a ambos lados de la frontera). _Sliding window log_: guardas timestamp de cada request, preciso pero caro en memoria. _Sliding window counter_: híbrido que pondera la ventana anterior y la actual, aproximado pero barato (dos contadores). **Trade-off:** precisión vs memoria vs complejidad. **Gotcha:** token bucket gana porque modela bien el tráfico real (bursty) con poca memoria; saber por qué descartás los otros es lo que te piden. _Rate Limiter, Web Crawler (sliding window por dominio)._

### Ubicación del rate limiter (edge vs servicio vs in-process)

**Problema:** dónde corre el limiter define qué información ve y cuánta latencia agrega. **Mecanismo:** tres ubicaciones. _In-process_ (en cada app server): rapidísimo pero cada server solo ve su fracción del tráfico → con N servers el límite global se multiplica por N, roto. _Servicio dedicado_: estado global preciso, pero agrega un round trip de red a cada request. _API Gateway / edge_ (el más común): el limiter está en la puerta, rechaza antes de que el request toque tus servers (como un bouncer), pero solo ve lo que hay en el HTTP request (headers, IP, path). **Trade-off:** precisión global vs latencia vs riqueza de contexto. **Gotcha:** el edge necesita estado externo rápido (Redis) para el contador compartido; sin eso vuelve al problema del in-process. _Rate Limiter._

### Atomicidad con Lua script en Redis

**Problema:** el contador de rate limit es un read-modify-write, y si dos requests lo hacen a la vez, ambos leen el mismo valor y dejan pasar de más. **Mecanismo:** `MULTI/EXEC` no basta porque el read (HMGET) suele estar _fuera_ de la transacción. La solución es mover toda la secuencia leer-calcular-actualizar a un script Lua, que Redis ejecuta atómicamente como una unidad indivisible. Expandes el límite atómico para abarcar el read-modify-write completo. **Trade-off:** atomicidad garantizada sin locks externos, a cambio de meter lógica en Lua (menos legible, harder to test). **Gotcha:** es el mismo principio que el "dealing with contention" — el bug clásico es creer que `MULTI/EXEC` solo ya resuelve la race; no, si el read quedó afuera, la race sigue. _Rate Limiter, Tinder (swipe match atómico)._

### Fail-open vs fail-closed

**Problema:** cuando Redis (la dependencia del limiter) cae, hay que decidir qué hacer con los requests que no podés verificar. **Mecanismo:** _fail-open_ deja pasar todo (la API sigue viva pero sin protección); _fail-closed_ rechaza todo (proteges el backend pero tu API queda offline). **Trade-off:** depende de qué es peor en tu caso. Social media en pico viral → fail-closed, porque el fallo del limiter suele coincidir con la avalancha de tráfico que más necesita protección, y fail-open ahí causaría cascada total. Pagos → fail-closed por seguridad. **Gotcha:** la respuesta correcta no es universal — articular _por qué_ tu sistema elige uno (y que la verdadera solución es prevenir el fallo con master-replica) es lo que demuestra criterio. _Rate Limiter._

### Config dinámica: poll vs push

**Problema:** querés cambiar reglas de rate limit sin redeployar código. **Mecanismo:** _poll_ — los gateways consultan una DB/config service cada ~30s y cachean local; simple. _Push_ — un sistema como [[ZooKeeper]] o Redis pub/sub notifica a todos los gateways al instante cuando cambia una regla. **Trade-off:** poll es trivial de implementar pero tiene lag (hasta 30s para propagar, problemático en una emergencia de seguridad); push propaga en segundos pero agrega complejidad (manejar conexiones caídas, fallos parciales donde unos gateways actualizan y otros no). **Gotcha:** push solo se justifica cuando los updates son urgentes (incidente de seguridad, trading); para la mayoría de casos operativos el lag de poll es aceptable. _Rate Limiter._

### Connection pooling

**Problema:** abrir una conexión TCP nueva a Redis por cada check de rate limit agrega el costo del handshake a cada request. **Mecanismo:** los gateways mantienen un pool de conexiones persistentes a Redis y las reusan entre requests, eliminando el handshake TCP (que cuesta 20-50ms según distancia de red). **Trade-off:** latencia mucho menor a cambio de gestionar el tamaño del pool (muy chico = contención por conexiones; muy grande = recursos desperdiciados). **Gotcha:** es la optimización de latencia _más_ importante en hot paths a Redis, y la mayoría de clients la hacen automática — pero hay que tunear el tamaño según volumen. _Rate Limiter, casi cualquier hot path a un store externo._

### Distribución geográfica para latencia

**Problema:** un usuario en Tokio hablando con infra en Virginia paga la latencia del viaje transatlántico en cada request. **Mecanismo:** desplegás la infra (gateways, Redis, réplicas) en varias regiones cerca de los usuarios, y cada uno habla con la copia más próxima. **Trade-off:** latencia mucho menor a cambio de complejidad de consistencia entre regiones — para rate limiting aceptás consistencia eventual entre regiones (un usuario podría exceder ligeramente el límite global saltando de región). **Gotcha:** funciona porque rate limiting tolera ese desfase; para datos que exigen consistencia fuerte global, la distribución geográfica abre el problema mucho más duro de coordinación cross-region. _Rate Limiter, Uber, CDN en general._

---

## EJE: BLOBS — Archivos grandes

### Presigned URL (subir/bajar directo al blob)

**Problema:** pasar archivos grandes por tu backend desperdicia su ancho de banda y lo convierte en cuello de botella. **Mecanismo:** el cliente sube/baja directo a S3 sin tocar tu server. El server solo firma con sus credenciales una URL temporal con permiso acotado (PUT para subir, GET para bajar) a una key específica y con expiración. El tráfico pesado va cliente↔S3 directo. En descarga, el server valida permisos _antes_ de firmar la URL. **Trade-off:** descargas tu backend del tráfico pesado, a cambio de ceder control directo sobre la transferencia (el cliente habla con S3). **Gotcha:** equivalentes por nube (SAS token en Azure, signed URL en GCS); la URL firmada es un bearer token — quien la tenga accede, por eso la expiración corta importa. _Dropbox, YouTube, WhatsApp (attachments)._

### CDN para servir contenido cerca del usuario

**Problema:** servir contenido pesado desde un origen central es lento para usuarios lejanos. **Mecanismo:** red de edge servers que cachean cerca del usuario. Modelo _pull_: el edge se llena bajo demanda copiando del origen en el primer miss (subes al origen, no al CDN). Controlás caché con headers (`Cache-Control`, `stale-while-revalidate`). Invalidás por versionado de URL (preferido, instantáneo) o purge vía API (lento, con coste). Range requests (HTTP 206) permiten servir trozos de un video. **Trade-off:** latencia drásticamente menor para contenido popular, a cambio de que solo sirve contenido no personalizado y la invalidación es el punto doloroso. **Gotcha:** si todo (segmentos + manifest) está en el CDN, el cliente nunca toca tu backend para seguir streameando — pero versionar URLs es mejor que purgar porque purgar es lento y caro. _Dropbox, YouTube, Bitly (edge redirects), Ticketmaster._

### Chunking + multipart upload

**Problema:** subir un archivo de GBs en un solo request choca con timeouts, límites de payload del gateway e interrupciones que tiran todo el progreso. **Mecanismo:** partís el archivo en piezas de 5-10MB y subís cada una por separado (en paralelo o secuencial). Resuelve timeouts (cada pieza es chica), límites del gateway, interrupciones (reanudás solo las faltantes) y UX (mostrás progreso). El chunking ocurre en el **cliente**. **Trade-off:** subida robusta y reanudable a cambio de coordinar el ensamblaje de piezas y rastrear cuáles llegaron. **Gotcha:** el cliente hace el trabajo (otra instancia del "cliente es parte del sistema"); S3 multipart upload lo implementa nativo, pero te van a pedir explicar el mecanismo por dentro. _Dropbox, YouTube._

### Resumable uploads con estado de chunks

**Problema:** si una subida larga se corta, el usuario no debería re-subir lo ya enviado. **Mecanismo:** la metadata guarda el estado de cada chunk (`not-uploaded`/`uploading`/`uploaded`). Al reconectar, el cliente consulta la metadata, ve qué chunks ya están y sube solo los faltantes. Aplicás "trust but verify": aceptás los updates de progreso del cliente, pero verificás server-side (ETags vía S3 ListParts) antes de marcar el archivo completo. **Trade-off:** experiencia resiliente a cortes a cambio de mantener y verificar estado por chunk. **Gotcha:** no confíes ciegamente en el cliente cuando dice "ya subí este chunk" — un cliente malicioso o con bug podría mentir; la verificación contra S3 es la red de seguridad. _Dropbox, YouTube._

### Fingerprinting (hash del contenido)

**Problema:** identificar un archivo por nombre colisiona entre usuarios y no detecta duplicados reales. **Mecanismo:** identificás el archivo por un hash de su _contenido_ (SHA-256), no por su nombre. Dos usuarios subiendo el mismo archivo producen el mismo fingerprint → podés deduplicar (guardar una sola copia) y dar resumability (sabés si ya existe). **Trade-off:** dedup y detección de duplicados a cambio del costo de hashear (CPU, y hay que leer todo el contenido). **Gotcha:** el nombre del archivo es una mala clave porque colisiona entre usuarios y no dice nada del contenido; el fingerprint es estable e identifica el dato real. _Dropbox, YouTube, Web Crawler (content dedup)._

### Content-Defined Chunking (CDC) para delta sync

**Problema:** con chunks de tamaño fijo, insertar 1 byte al inicio del archivo desplaza todas las fronteras, cambia el fingerprint de _todos_ los chunks y hace inútil el delta sync. **Mecanismo:** CDC define las fronteras de los chunks según el _contenido_ usando un rolling hash (Rabin): cuando el hash de una ventana deslizante cumple cierta condición, ahí cortás. Así una edición pequeña solo afecta a los chunks vecinos a la edición, no a todos. **Trade-off:** delta sync eficiente (solo re-subís lo que cambió) a cambio de chunks de tamaño variable y más complejidad que el corte fijo. **Gotcha:** es exactamente lo que hace rsync/Dropbox para sincronizar cambios pequeños en archivos grandes sin re-transmitir todo. _Dropbox._

### Comprimir antes de cifrar

**Problema:** cifrar primero destruye la posibilidad de comprimir. **Mecanismo:** el cifrado introduce aleatoriedad estadística (el texto cifrado parece ruido), y la compresión depende de encontrar patrones/redundancia — así que si ciframos primero, no queda nada que comprimir. Por eso comprimís y _después_ ciframos. Y lo hacés selectivamente: vale mucho en texto (5GB→1GB), nada en media ya comprimido (JPEG, MP4). La decisión va en el cliente según tipo/tamaño/red. **Trade-off:** ahorro de ancho de banda y storage a cambio de CPU de compresión (desperdiciada si el archivo ya está comprimido). **Gotcha:** el orden importa y es contra-intuitivo; algoritmos según caso (Gzip universal, Brotli para texto, Zstandard balance). _Dropbox._

### Pasar URLs entre workers, no archivos

**Problema:** mover blobs pesados entre nodos de un pipeline satura la red interna. **Mecanismo:** los datos temporales del pipeline (segmentos de video, audio extraído) se guardan en S3, y los workers se pasan _URLs_ que apuntan a esos archivos, no los archivos mismos. **Trade-off:** evitás transferir blobs entre nodos a cambio de round-trips a S3 (que escala bien y es barato). **Gotcha:** es el patrón natural en pipelines de media processing — el orquestador coordina pasando referencias, no payloads. _YouTube (DAG de procesamiento)._

### No guardar payloads grandes en la cola

**Problema:** meter el blob pesado dentro del mensaje de la cola es caro y las colas no están hechas para eso. **Mecanismo:** el mensaje de la cola lleva solo un ID/referencia; el payload pesado (el HTML crawleado, el archivo) vive en blob storage. El consumer lee el ID de la cola y va a buscar el blob. **Trade-off:** cola liviana y barata a cambio de un lookup extra al blob store. **Gotcha:** es un anti-pattern guardar HTML/archivos en la cola — las colas optimizan throughput de mensajes chicos, no almacenamiento de blobs. _Web Crawler, WhatsApp (attachments), Ad Click._

---

## EJE: FAN-OUT — Distribución social / feeds

### Fan-out on write vs fan-out on read

**Problema:** hay dos momentos para armar un dato derivado (feed, timeline, agregado) y la elección define la performance. **Mecanismo:** _on read_ — ensamblás el feed cuando el usuario lo pide, juntando posts de todos los que sigue en el momento (flexible, sin storage extra, pero lento si seguís a muchos). _On write_ — cuando alguien postea, empujás el post a los feeds precomputados de todos sus seguidores (lectura es un lookup instantáneo, pero la escritura es cara: un post de alguien con 90M seguidores dispara 90M escrituras). **Trade-off:** on read optimiza escritura y gasta en lectura; on write al revés. La elección sale del ratio read/write. **Gotcha:** ninguno gana solo a escala real — lleva al híbrido. _FB, timelines, notificaciones._

### Precomputación de resultados

**Problema:** hacer trabajo caro en el read path (frecuente, sensible a latencia) no escala. **Mecanismo:** movés el cómputo al write path (raro) y guardás el resultado ya armado, así la lectura es un lookup simple. FB mantiene un `PrecomputedFeed` con los IDs de los posts (no los posts completos, para ahorrar espacio), limitado a ~200 por usuario. **Trade-off:** lecturas rapidísimas a cambio de escrituras más caras y storage para el resultado precomputado. **Gotcha:** guardás referencias (IDs), no objetos completos — hidratás los detalles en lectura desde cache; precomputar el objeto entero explota el storage. _FB, Tinder (feeds precomputados), materialized views._

### Enfoque híbrido / decisión por-entidad

**Problema:** una estrategia global única falla en los extremos de la distribución. **Mecanismo:** elegís la estrategia _por entidad_ según su perfil. FB precomputa el feed para cuentas normales (fan-out on write), pero resuelve en lectura las megacuentas (Bieber, 90M seguidores) porque empujar a 90M feeds es inviable; el feed final del usuario es un _merge_ de su parte precomputada más los posts on-read de las megacuentas que sigue. **Trade-off:** óptimo en ambos extremos a cambio de complejidad (dos caminos + merge). **Gotcha:** funciona porque la distribución es asimétrica (poquísimas megacuentas, muchísimas normales) — el merge solo paga el costo on-read para la minoría que lo justifica. _FB. Cualquier distribución de cola larga._

### Async workers + cola

**Problema:** trabajo pesado en el path síncrono hace esperar al usuario y no absorbe picos. **Mecanismo:** encolás el trabajo tolerante a retraso (SQS, Kafka) y workers lo procesan a su ritmo. Suaviza picos (la cola amortigua), reparte carga pareja y aísla fallos (si un worker cae, otro toma el mensaje). **Trade-off:** desacoplás y ganás resiliencia a cambio de consistencia eventual (el resultado tarda) y la necesidad de at-least-once + idempotencia. **Gotcha:** cuidado con trabajo muy variable por mensaje (un fan-out de 1k vs 90M followers en el mismo topic) — conviene partir las tareas para que un mensaje gigante no bloquee a los chicos. _FB, Ticketmaster (admission worker), Uber (matching queue), Web Crawler._

### Hot key problem

**Problema:** un KV store escala asumiendo carga pareja por el keyspace, pero una clave concreta (post viral, ad viral) recibe tráfico desproporcionado y satura su partición física. **Mecanismo:** identificás que el cuello no es el volumen total sino la _concentración_ — todas las requests pegan a la misma partición/nodo. **Trade-off:** no se arregla agregando nodos al cluster (la clave sigue en un solo nodo); requiere replicar esa clave o repartir su carga. **Gotcha:** distinguir hot key de "necesito más capacidad" es clave — más shards no ayudan si el problema es una sola clave caliente. _FB (posts), Ad Click (ad viral), YouTube (video hot), Tinder (usuarios activos)._

### Sharding vs replicación (trade-off)

**Problema:** dos formas de distribuir datos resuelven problemas opuestos. **Mecanismo:** _sharding_ pone cada dato en un nodo → maximiza capacidad total de almacenamiento, pero es vulnerable a hot keys (una clave caliente satura su shard). _Replicación_ pone cada dato en varios nodos → maximiza throughput por clave (cualquier réplica responde), pero gasta más memoria/storage. **Trade-off:** capacidad vs throughput por clave. **Gotcha:** elegís según el dolor — si el problema es volumen total que no entra, shardeás; si es una clave con tráfico brutal, replicás. A escala real, combinás ambos. _Caches, read replicas, CDN, casi todos los sistemas grandes._

### Cache redundante/replicada (para hot keys en lectura)

**Problema:** shardear el cache por key recrea el hot key (la clave viral sigue en un shard). **Mecanismo:** en vez de shardear, replicás el item caliente en varias instancias de cache, y un load balancer reparte las lecturas entre ellas. El tráfico viral se divide entre N réplicas → N× throughput, sin coordinación entre ellas. **Trade-off:** absorbe hot keys de lectura a cambio de más misses iniciales (cada réplica se calienta sola) y menos items totales cacheables (cada uno ocupa N veces). **Gotcha:** esto es para hot keys de _lectura_; para hot keys de _escritura_ se usa el suffix de partition key (ver Ad Click), que es el problema espejo. _FB._

### Hot shard mitigation con partition key suffix (para hot keys en escritura)

**Problema:** un AdId viral concentra todas sus escrituras en un solo shard del stream y lo satura. **Mecanismo:** agregás un sufijo random al partition key (`AdId:0` … `AdId:N`) solo para los ads populares (detectados por ad spend o volumen previo), repartiendo sus escrituras entre N sub-particiones. Al escribir al destino final, quitás el sufijo y agregás por el AdId original (los upserts con SUM combinan bien los conteos parciales). **Trade-off:** distribuís la carga de escritura a cambio de tener que re-agregar las sub-particiones después. **Gotcha:** es el espejo del cache redundante — aquel reparte _lecturas_ calientes, este reparte _escrituras_ calientes; aplicarlo solo a los keys realmente calientes evita explotar la cardinalidad. _Ad Click._

---

## EJE: IDs — Generación única a escala

### Hashing + base62 + manejo de colisión

**Problema:** generar un short code único, corto y URL-safe a partir de una URL larga. **Mecanismo:** hasheás la URL (SHA-256), codificás en base62 (a-z, A-Z, 0-9 — sin `+/` que rompen URLs) y tomás los primeros N caracteres. Las colisiones crecen con n/|S| (paradoja del cumpleaños): las manejás con un UNIQUE constraint en la DB y reintento con un salt (máx 3-5 intentos). **Trade-off:** triple tensión entre unicidad (más chars), brevedad (menos chars) y eficiencia (menos reintentos). **Gotcha:** base62 sobre base64 es por los caracteres problemáticos en URLs; y el manejo de colisión con reintento+salt es lo que te van a pedir detallar. _Bitly._

### Counter atómico centralizado + batching

**Problema:** garantizar IDs únicos sin colisiones entre múltiples servicios de escritura distribuidos. **Mecanismo:** un counter incremental global → base62. Redis es ideal (single-threaded, `INCR` atómico). El problema distribuido es que todos los write services necesitan una única fuente de verdad. El _batching_ lo resuelve: cada instancia pide 1000 valores de golpe y los consume local, reduciendo round-trips drásticamente. Multi-región: das rangos disjuntos (región A: 0-1B, B: 1B-2B). **Trade-off:** unicidad garantizada sin colisiones a cambio de un punto central (mitigado con batching) y de IDs predecibles. **Gotcha:** los codes secuenciales son enumerables (alguien recorre /1, /2, /3 y scrapea todo) — se ofusca con XOR contra un secreto. _Bitly, WhatsApp (sequence numbers vía Redis INCR)._

---

## EJE: MODELADO DE DATOS

### GSI / índice invertido / relación inversa

**Problema:** una tabla solo se consulta eficiente por su clave primaria; consultar por otro campo fuerza full scan. **Mecanismo:** creás un índice secundario (Global Secondary Index en DynamoDB) — una copia de los datos reindexada por otra clave, que la DB mantiene sincronizada automáticamente. El caso típico es invertir una relación: tenés "a quién sigo" y querés "quién me sigue". **Trade-off:** lecturas rápidas por el nuevo campo a cambio de storage extra y escrituras más lentas (hay que actualizar el índice). **Gotcha:** el GSI es eventualmente consistente con la tabla base — una lectura inmediata tras escribir podría no ver el cambio aún. _FB (followers), Bitly, Dropbox (SharedFiles), WhatsApp (chats por user)._

### Modelar un grafo sobre un KV store

**Problema:** las relaciones N-a-N son grafos, y el instinto es reachear por una graph DB que quizás no necesitás. **Mecanismo:** si tu acceso son solo lookups directos (¿A sigue a B?) y no traversals multi-salto (amigos-de-amigos-de-amigos), modelás las relaciones como filas en un KV store (userId_origen, userId_destino) y te ahorrás operar Neo4j. **Trade-off:** simplicidad operacional a cambio de no poder hacer traversals profundos eficientes. **Gotcha:** reservá la graph DB para cuando los traversals multi-salto sean el _core_ del producto; para follows simples, un KV store sobra. _FB (follows). Permisos, amistades._

### Normalizar a tabla de relación vs lista embebida

**Problema:** modelar N-a-N como lista embebida en cada entidad hace lentísimas las queries inversas. **Mecanismo:** creás una tabla de relación dedicada (userId, fileId) en vez de embeber una lista. Dropbox usa una tabla `SharedFiles` en vez de un campo `sharelist` por archivo, porque para responder "archivos compartidos conmigo" con listas embebidas tendrías que escanear _cada_ archivo. **Trade-off:** queries eficientes en ambas direcciones a cambio de un join y más normalización. **Gotcha:** la lista embebida parece más simple pero te mata la query inversa — la tabla de relación con índices en ambas columnas es lo que escala. _Dropbox, FB (Follow). Shares, permisos._

### Cursor-based pagination

**Problema:** la paginación por offset se vuelve lenta en profundidad y se rompe si entran datos nuevos mientras paginás. **Mecanismo:** usás un cursor (un puntero a tu posición, típicamente un timestamp o ID) en vez de un offset numérico. La siguiente página arranca _después_ del cursor, saltando directo con el índice sin descartar las filas anteriores. **Trade-off:** rápido a cualquier profundidad y estable ante inserciones (sin duplicados ni saltos) a cambio de no poder saltar a "página 47" arbitraria. **Gotcha:** offset N hace al motor leer y descartar N filas (lento en profundidad) y si insertan algo arriba, todo se corre y ves duplicados; el cursor evita ambos. _FB (feed), Bitly. Listados, scroll infinito._

### Elegir DB por patrón de acceso

**Problema:** elegir DB "por costumbre" en vez de por cómo accedés a los datos lleva a cuellos de botella. **Mecanismo:** la DB correcta sale del data flow. Write-heavy → Cassandra (LSM trees: commit log + memtable + SSTable, optimizado para escritura). Analytics multi-dimensional → OLAP columnar. Point lookups por clave → KV. Atomicidad entre datos → ACID relacional. Time-range con baja cardinalidad → TSDB. **Trade-off:** cada DB optimiza un patrón y penaliza otros (Cassandra es genial escribiendo, mala en range queries/agregaciones). **Gotcha:** justificá con el patrón de acceso concreto — Tinder usa Cassandra para swipes por el volumen de escritura, YouTube por videoId porque solo hace point lookups. _Tinder, Ad Click, YouTube, GoPuff, WhatsApp._

---

## EJE: GEO — Proximidad

### Índice geoespacial (geohash / quadtree)

**Problema:** un B-tree no sirve para datos 2D (lat/long) → buscar por proximidad fuerza full scan calculando distancia a cada punto. **Mecanismo:** estructuras especializadas. _Quadtree_ particiona el espacio en cuadrantes recursivamente, así buscar "cerca de aquí" recorre solo las ramas relevantes. _Geohash_ codifica lat/long en un entero/string donde prefijos comunes = cercanía geográfica, permitiendo búsqueda por rango. Redis lo trae nativo (`GEOADD`/`GEOSEARCH`), o PostGIS sobre Postgres. **Trade-off:** proximity search eficiente a cambio de una estructura/extensión especializada. **Gotcha:** el punto que te piden entender es _por qué_ el índice tradicional falla en 2D — la distancia euclidiana no mapea a un orden lineal único. _Uber, Tinder, proximity search._

### In-memory geospatial store para writes masivos + queries rápidas

**Problema:** 2M location updates/s de drivers matarían una DB tradicional ($200k/día en DynamoDB) y las queries de proximidad serían lentas. **Mecanismo:** Redis geoespacial maneja el volumen en memoria; cada `GEOADD` sobreescribe la posición previa del driver (así siempre tenés la última, sin acumular histórico). Las queries de proximidad usan el índice geohash interno de Redis. **Trade-off:** throughput y latencia excelentes a cambio de durabilidad (es memoria) — mitigada porque los datos se reconstruyen en ~5s (los drivers re-pingean) más RDB/AOF y Redis Sentinel para failover. **Gotcha:** la baja durabilidad requerida es lo que _habilita_ usar memoria — si perder 5s de posiciones fuera inaceptable, no podrías; acá sí. _Uber._

### Pre-filtrar barato antes del cálculo caro

**Problema:** correr la operación cara sobre todo el dataset es inviable. **Mecanismo:** reducís candidatos con un filtro barato primero, y solo sobre esos pocos corrés el cálculo caro. GoPuff filtra los DCs dentro de un radio de 60 millas con matemática simple (bounding box) _antes_ de llamar al travel-time service (caro, tercero) solo sobre ese puñado. **Trade-off:** muchísimo menos cómputo caro a cambio de una pasada extra barata. **Gotcha:** el bounding box rectangular es una sobre-aproximación (incluye esquinas fuera del radio real), pero está bien porque el filtro caro posterior los descarta — el barato solo tiene que no dejar fuera verdaderos positivos. _GoPuff. Proximity search, matching._

### Partición / sharding por geografía

**Problema:** datos geográficos esparcidos por todos los shards hacen que cada query toque muchas particiones. **Mecanismo:** agrupás los datos por región para que una query pegue en una o dos particiones. GoPuff deriva un region ID de los 3 primeros dígitos del zip → la query toca 1-2 particiones. Uber hace geo-sharding y solo necesita scatter-gather (consultar varios shards) cuando la búsqueda cae en un boundary entre regiones. **Trade-off:** queries localizadas y latencia menor (cliente cerca del shard) a cambio del problema de boundaries y posible desbalance (regiones densas vs vacías). **Gotcha:** el scatter-gather en los bordes es el caso que te van a sondear — una búsqueda cerca del límite entre dos regiones tiene que consultar ambas. _GoPuff, Uber. Delivery local, geo-search._

### Adaptive update intervals (lógica en el cliente)

**Problema:** drivers pingeando posición cada 5s generan carga masiva, mucha de ella redundante. **Mecanismo:** el cliente ajusta dinámicamente la frecuencia de pings según contexto usando sensores on-device: parado o lento → menos updates; moviéndose rápido o cambiando de dirección → más. **Trade-off:** menos carga al backend manteniendo precisión a cambio de complejidad en el cliente (algoritmos de cuándo pingear). **Gotcha:** otra instancia de "el cliente es parte del sistema" — el entrevistador valora que no trates al cliente como una caja tonta que solo dibuja. _Uber._

---

## EJE: REAL-TIME / SYNC / conexiones persistentes

### WebSocket / conexión bidireccional persistente

**Problema:** REST (request-response) es ineficiente para alta frecuencia de mensajes en ambas direcciones. **Mecanismo:** abrís un WebSocket (sobre TLS) que queda abierto y permite al server empujar mensajes al cliente sin que este pregunte, y viceversa. Para chat un L4 load balancer alcanza (no necesitás capacidades L7 de ruteo por path/header). **Trade-off:** latencia baja y push real a cambio de conexiones stateful (cada socket vive en un server concreto) que complican el escalado horizontal. **Gotcha:** L4 sobre L7 acá porque no necesitás inspeccionar el contenido HTTP — y el estado de las conexiones es lo que lleva al patrón pub/sub para rutear entre servers. _WhatsApp, chat, colaboración, gaming._

### Push + poll híbrido (con safety net)

**Problema:** el push puro pierde eventos si la conexión se cae silenciosamente; el poll puro tiene latencia. **Mecanismo:** combinás ambos — WebSocket/SSE empuja cambios al instante, _más_ un polling periódico (cada minutos) como red de seguridad por si la conexión cayó sin avisar. Dropbox usa esto: push para latencia baja, poll para garantizar que no perdés un cambio. **Trade-off:** latencia baja + garantía de entrega a cambio de la carga del polling de respaldo. **Gotcha:** el poll no es redundante — cubre el caso de conexión muerta no detectada, que el push solo no maneja. _Dropbox, WhatsApp._

### SSE/WebSocket + Pub/Sub para broadcast a sockets distribuidos

**Problema:** las conexiones viven en servers concretos, pero un evento es global y tenés que entregarlo al server correcto sin saber dónde está el socket del destinatario. **Mecanismo:** el estado vive en Redis y cada server se suscribe a un canal Pub/Sub. Cuando ocurre un evento, un worker publica al canal; _todos_ los servers lo reciben (fan-out), pero solo el que tiene el socket del usuario destino actúa y lo empuja. Esto elimina sticky sessions porque el estado no vive en el proceso sino en Redis. Redis Pub/Sub (at-most-once, ligero, sin persistencia) vs Kafka (no sirve para millones de topics — ~50kb overhead cada uno). **Trade-off:** ruteo a sockets arbitrarios sin coordinación a cambio de un broadcast que toca todos los servers. **Gotcha:** Pub/Sub es at-most-once (si no hay suscriptor o Redis falla, el mensaje se pierde) — por eso la durabilidad va a un inbox aparte, no al Pub/Sub. _Ticketmaster (virtual queue), WhatsApp (routing entre chat servers), FB Live Comments._

### Inbox / outbox para entrega offline

**Problema:** un mensaje debe entregarse aunque el destinatario esté offline, y el Pub/Sub no garantiza entrega. **Mecanismo:** escribís el mensaje a un inbox durable por destinatario _antes_ de intentar la entrega en tiempo real. Si está offline, el mensaje espera ahí; al reconectar, el cliente sincroniza su inbox. El ACK del cliente borra el mensaje del inbox. La durabilidad vive en el inbox, no en el canal de tiempo real. **Trade-off:** garantía de entrega a cambio de storage por destinatario y la sincronización en reconexión. **Gotcha:** el orden importa — escribís al inbox (durable) _antes_ de publicar al Pub/Sub (best-effort), así si el Pub/Sub falla el mensaje no se pierde, se recupera al reconectar. _WhatsApp._

### ACK del cliente + heartbeats para detectar conexiones muertas

**Problema:** TCP tarda minutos en detectar una conexión muerta, demasiado para chat en tiempo real. **Mecanismo:** dos piezas. _Heartbeats_ — el server manda ping cada 10-30s y el cliente responde pong; si no llega el pong en el timeout, la conexión se considera muerta y se detecta en segundos, no minutos. _ACK_ — el cliente confirma recepción de cada mensaje, así sabés que llegó de verdad hasta el cliente (no solo que salió del server). **Trade-off:** detección rápida de caídas y confirmación de entrega a cambio del overhead de los heartbeats (con 200M conexiones y heartbeat cada 10s son 20M ping/pong por segundo, manejable porque son mensajes diminutos). **Gotcha:** heartbeat interval + timeout te da la cota superior de tiempo de detección — un número concreto que podés citar. _WhatsApp._

### Sequence numbers para detectar gaps

**Problema:** un cliente no sabe si perdió un mensaje cuando el canal es best-effort. **Mecanismo:** cada mensaje lleva un número monótono creciente (por chat o por usuario, generado con Redis INCR). Si el cliente recibe el #5 habiendo visto el #3, sabe que perdió el #4 y pide re-sync. Podés piggybackear el número actual en los heartbeats para detección rápida con carga mínima. **Trade-off:** detección de mensajes perdidos a cambio de mantener un contador atómico (otra dependencia). **Gotcha:** el gap solo se detecta cuando llega _otro_ mensaje — si el chat queda en silencio justo después del mensaje perdido, no te enterás hasta el próximo; por eso el polling de respaldo sigue siendo necesario. _WhatsApp._

### No ordenar, estampar con timestamp del servidor (NTP)

**Problema:** garantizar orden perfecto de mensajes en un sistema distribuido es caro (requiere delays para esperar rezagados + un mecanismo de reordenamiento). **Mecanismo:** en vez de garantizar orden de procesamiento, los chat servers sincronizan su reloj por NTP y estampan cada mensaje con la hora de recepción; el cliente ordena por ese timestamp al mostrar. **Trade-off:** simplicidad enorme a cambio de que ocasionalmente un mensaje "aparezca arriba" de otro enviado después. **Gotcha:** la clave es de producto, no técnica — los usuarios _prefieren_ ver mensajes nuevos rápido antes que garantizar orden estricto, así que el trade-off es aceptable (contraste con Flink, que sí reordena con watermarks cuando el orden importa). _WhatsApp._

### Estado efímero vía conexiones activas (presence / last seen)

**Problema:** escribir a la DB en cada heartbeat para mantener "última vez en línea" genera write amplification masiva (millones de writes/s de valor mínimo). **Mecanismo:** explotás dos hechos: cuando un usuario está conectado su server _lo sabe_ (puede reportarlo en vivo vía pub/sub), y cuando se desconecta también lo sabés. Así solo escribís a la DB el último _disconnect_; si alguien pregunta "last seen", o el usuario está conectado (su server responde ONLINE) o devolvés el disconnect guardado. **Trade-off:** mínimas escrituras a cambio de coordinar la respuesta entre el dato en DB y el estado en vivo. **Gotcha:** un conditional write (actualizar solo si el nuevo timestamp es mayor) evita que dos servers se pisen el disconnect. _WhatsApp._

### Partición por user vs por chat (depende del workload)

**Problema:** elegir mal la unidad de partición del pub/sub desperdicia suscripciones o publicaciones. **Mecanismo:** la elección depende de (a) cuántos chats tiene un usuario y (b) qué tan grandes son. Chats 1:1 dominantes → partición por user (cada server se suscribe a 1 canal por usuario conectado, publica a 1). Pocos chats enormes → partición por chat (1 canal compartido por chat, 1 publish llega a todos). Solución adaptativa: los chats grandes (>25 participantes) tienen canal propio _además_ del de usuario. **Trade-off:** optimizás suscripciones o publicaciones según el patrón, pero el adaptativo agrega complejidad de manejar ambos. **Gotcha:** WhatsApp es 1:1 dominante → por user; el caso de chats grandes es un "celebrity problem" (edge case raro que impacta desproporcionado) que se resuelve adaptativamente. _WhatsApp._

### Push notifications nativas (APNS / FCM)

**Problema:** notificar a un usuario que no tiene la app abierta o está en background. **Mecanismo:** usás los servicios push nativos del dispositivo (Apple Push Notification Service, Firebase Cloud Messaging) en vez de intentar mantener una conexión propia. Tu backend le entrega el mensaje a APNS/FCM y ellos lo empujan al dispositivo. **Trade-off:** alcance a usuarios offline/background sin mantener conexiones a cambio de depender de servicios de terceros y sus límites. **Gotcha:** es la pieza para el caso "el destinatario no está activo" — distinto del WebSocket que es para usuarios con la app abierta. _Tinder (match notification), Uber (ride request al driver)._

---

## EJE: BÚSQUEDA — Full-text

### Full-text search: Postgres GIN vs Elasticsearch

**Problema:** `LIKE '%texto%'` no puede usar índices → full table scan, lentísimo. **Mecanismo:** dos opciones. _Postgres con índice GIN sobre `tsvector`_ — full-text search nativo, sin infra nueva, suficiente para búsqueda simple. _Elasticsearch_ — índice invertido (mapea cada término a los documentos que lo contienen) + fuzzy search tolerante a typos + ranking por relevancia + escala horizontal. **Trade-off:** GIN evita operar otro sistema; ES da capacidades mucho más ricas pero es otra pieza que sincronizar y mantener. **Gotcha:** el argumento más fuerte para ES no es la velocidad sino el _fuzzy search_ (typos, variantes) — si solo necesitás match exacto, GIN puede alcanzar. _Ticketmaster, Tinder (feed con filtros), FB Post Search._

### CDC (Change Data Capture) para sincronizar índice de búsqueda

**Problema:** mantener Elasticsearch sincronizado con Postgres sin perder cambios ni hacer dual writes frágiles. **Mecanismo:** leés el WAL (write-ahead log, el log de transacciones de Postgres) con una herramienta como Debezium, que publica cada cambio a Kafka, y un consumer indexa en ES. No es polling ni escritura doble: si el cambio llegó al WAL (o sea, se commiteó), CDC lo va a ver. **Trade-off:** sincronización confiable y desacoplada a cambio de consistencia eventual (el índice va detrás de la DB) y más piezas. **Gotcha:** como ES es read-heavy y no le gustan los writes uno por uno, conviene batchear los cambios antes de indexar. _Ticketmaster, Tinder, FB Post Search._

### Jerarquía de caché para búsqueda repetida

**Problema:** muchas búsquedas se repiten y pegar a ES cada vez es desperdicio. **Mecanismo:** capas de caché de cerca a lejos del usuario, resolviendo lo más arriba posible: CDN (queries idénticas no personalizadas, en el edge) → Redis (frecuentes, con TTL o invalidación por tags) → cache nativa de ES (filtros, agregaciones) → ES como último recurso. **Trade-off:** latencia mínima para búsquedas populares a cambio de la complejidad de coordinar múltiples capas y, sobre todo, de invalidarlas. **Gotcha:** el costo real del caching no es agregar capas sino la _invalidación_ — cada capa más es otra cosa que puede servir datos stale. _Ticketmaster._

---

## EJE: STREAM / DATA PROCESSING — Agregación de eventos

### Stream processing con Kafka + Flink

**Problema:** procesar un firehose de eventos en tiempo real con agregaciones por ventana, sin perder datos ante fallos. **Mecanismo:** Kafka transporta el stream (particionado, retenido); Flink consume y agrega. Flink te da: agregación por ventana con _event-time_ (usa cuándo ocurrió el evento, no cuándo llegó), _watermarks_ (saben cuándo es seguro cerrar una ventana esperando rezagados), exactly-once processing, y fault tolerance con recuperación de estado (rewind al offset de Kafka). **Trade-off:** capacidades potentes a cambio de operar Flink y de que el entrevistador (y vos) entiendan sus internals. **Gotcha:** podrías hacer agregación con consumers de Kafka crudos + contador en memoria, y para mid-level está bien; Flink se justifica por event-time, watermarks y exactly-once, que reimplementar a mano es error-prone. _Top-K, Ad Click. Analytics en tiempo real._

### Batch processing con Spark (MapReduce)

**Problema:** procesar volúmenes grandes que no caben en una máquina, o computar una fuente de verdad exacta. **Mecanismo:** Spark distribuye el trabajo con MapReduce — lee chunks en paralelo, agrega cada uno (map), combina los parciales (reduce), escribe el resultado. **Trade-off:** escala a volúmenes enormes a cambio de latencia (es batch, corre cada X) y overhead de levantar el job. **Gotcha:** criterio de seniority — 300MB cabe en una máquina y no _necesitás_ Spark; lo usás por el framework de distribución si crece y por el paradigma MapReduce fácil de razonar, no porque sí. _Ad Click (batch layer), Top-K (windows grandes)._

### Lambda architecture (speed layer + batch layer)

**Problema:** querés baja latencia _y_ exactitud, que suelen estar en tensión. **Mecanismo:** corrés dos caminos en paralelo. _Speed layer_ (Flink) da resultados rápidos pero potencialmente imprecisos. _Batch layer_ (Spark sobre un data lake con los eventos raw) es la fuente de verdad lenta pero exacta. Una reconciliación periódica usa el batch para corregir las inexactitudes del speed. **Trade-off:** lo mejor de ambos a cambio de mantener dos pipelines que computan lo mismo. **Gotcha:** se usa donde un error de datos cuesta dinero (billing, ad clicks) — el speed layer sirve el dato "ya", el batch lo corrige "bien" después. _Ad Click._

### Reconciliación periódica

**Problema:** errores transitorios (fallos de Flink, eventos fuera de orden, bugs) introducen inexactitudes que el stream solo no detecta. **Mecanismo:** volcás los eventos raw a un data lake (S3) en paralelo al stream; un job batch periódico (cada hora/día) re-agrega desde el raw y compara contra lo que produjo el stream. Si discrepan, investigás y corregís el dato en la DB de destino. **Trade-off:** garantía de exactitud eventual a cambio del costo de almacenar raw y correr el job de reconciliación. **Gotcha:** es la pieza que hace confiable al speed layer — sin reconciliación, los errores del stream se acumulan silenciosamente. _Ad Click._

### Tumbling vs sliding windows

**Problema:** "última hora" puede significar dos cosas distintas, con costos muy diferentes. **Mecanismo:** _tumbling_ — ventanas fijas alineadas a fronteras (9:00-10:00, 10:00-11:00); fáciles porque solo acumulás. _Sliding_ — exactamente las últimas 60 min desde ahora (9:06-10:06 si son las 10:06); caras porque al avanzar tenés que sumar lo nuevo y restar lo que salió de la ventana. **Trade-off:** tumbling es barato pero menos preciso temporalmente; sliding es exacto pero un slide de 1 min multiplica la memoria 43.200×. **Gotcha:** el truco práctico es sliding solo para "última hora" (donde la frescura importa) y tumbling para día/mes (donde no cambia rápido y nadie nota la diferencia). _Top-K._

### Batching de escrituras (agregar antes de persistir)

**Problema:** un write a la DB por cada evento individual no escala (700k tps en Top-K). **Mecanismo:** agregás en memoria (Flink) durante una ventana y flusheás las sumas periódicamente, en vez de escribir cada evento. Aprovechás que pocos items (videos/ads virales) concentran la mayoría de los eventos, así muchos eventos colapsan en un write. Bajó los shards de Top-K de 70 a 5-10. **Trade-off:** muchísimas menos escrituras (y las DBs manejan bulk mejor que writes sueltos) a cambio de latencia (el dato aparece al flushear) y de perder lo no-flusheado si el procesador cae sin checkpoint. **Gotcha:** el batching convierte un stream constante en lumps periódicos — bien si se reparten entre shards. _Top-K, Uber (batch de location updates), Ad Click._

### Precompute + cron warming de cache

**Problema:** una cache con TTL, al expirar, genera un flood de requests a la DB y rompe el SLA justo en los boundaries. **Mecanismo:** en vez de esperar a que expire, un cron precomputa el resultado e inyecta ("warmea") la cache _antes_ de que la entrada vieja caduque. Así las requests siempre leen de cache y nunca tocan la DB. Retenés las entradas unas horas como respaldo por si el cron se atrasa (servís dato algo viejo en vez de nada). **Trade-off:** SLA estable sin floods a cambio de complejidad operacional (¿y si el cron falla? monitoreo). **Gotcha:** resuelve el problema (b) del TTL puro — el cron "se adelanta" a la expiración, evitando el pico de misses sincronizado en el boundary. _Top-K, Ad Click (pre-agregación nocturna)._

### Aggregate por shard y merge

**Problema:** al shardear los datos, el top-K global no está en ningún shard individual. **Mecanismo:** tomás el top-K de _cada_ shard y los mergeás — matemáticamente, el top-K global está garantizado dentro de la unión de los top-K locales (un item del top global tiene que estar en el top de _su_ shard). **Trade-off:** consultas paralelas a N shards + un merge barato, a cambio de no tener la respuesta en un solo lugar. **Gotcha:** la garantía matemática es la clave — no necesitás traer _todo_ de cada shard, solo su top-K, porque nada fuera del top-K local puede estar en el top-K global. _Top-K. Cualquier ranking sobre datos particionados._

### Aggregates a granularidad creciente (rollups)

**Problema:** sumar miles de filas hora-por-hora para responder "último mes" es lentísimo. **Mecanismo:** mantenés agregados pre-computados a granularidad creciente — por hora, por día, por mes, en tablas separadas. Para una ventana grande sumás los agregados diarios (30 filas) en vez de los horarios (720). **Trade-off:** queries rápidas sobre ventanas grandes a cambio de mover complejidad al write (mantener varias tablas de rollup) y storage extra. **Gotcha:** es una forma de precomputación jerárquica — cada nivel de rollup ahorra trabajo de agregación a cambio de escrituras adicionales. _Top-K, Ad Click. OLAP, time-series._

### Count-Min Sketch (aproximación probabilística)

**Problema:** mantener conteos exactos de billones de items requiere cientos de GB de memoria. **Mecanismo:** CMS estima conteos con memoria sublineal (cientos de MB) usando varias funciones de hash que mapean cada item a un array 2D de contadores; al consultar, tomás el mínimo de los contadores hasheados (de ahí "min"). _Olvida_ los IDs (no podés listar), pero recuerda cuántas veces vio cada uno. Lo combinás con un sorted list/heap para resolver top-K. Soporta remove (para sliding windows) si solo decrementás lo que incrementaste. **Trade-off:** memoria masivamente menor a cambio de aproximación (sobreestima por colisiones de hash, nunca subestima). **Gotcha:** funciona porque para top-K los líderes suelen diferir por miles de views, no por decenas — el error de aproximación no cambia el ranking; se usa cuando "trends" basta y no necesitás precisión exacta. _Top-K. Conteo aproximado a escala._

### Request coalescing

**Problema:** cuando una cache key caliente expira, mil requests simultáneos pegan todos a la DB (thundering herd). **Mecanismo:** ante un miss, dejás que _una_ request por server vaya a la DB; las demás esperan esa respuesta y la comparten. **Trade-off:** evitás el flood a cambio de que las requests en espera tengan algo más de latencia. **Gotcha:** mitiga el problema (a) del cache miss masivo, pero no el (b) de romper el SLA — para eso necesitás precompute+warming; coalescing y warming atacan partes distintas del mismo problema. _Top-K._

### OLAP / columnar para analytics multi-dimensional

**Problema:** agregar (COUNT/SUM) sobre millones de filas con alta cardinalidad y filtrar por varias dimensiones es lento en una DB row-based. **Mecanismo:** bases columnar (Redshift, Snowflake, BigQuery, ClickHouse) guardan por columna, así una agregación lee solo las columnas que necesita y las recorre velozmente. Manejan alta cardinalidad (millones de ad IDs) y slicing multi-dimensional (device, geo, campaña). **Trade-off:** agregaciones analíticas rapidísimas a cambio de no ser buenas para writes transaccionales o lookups de filas individuales. **Gotcha:** la distinción con TSDB es la pregunta — una TSDB sirve "métrica X sobre tiempo" con baja cardinalidad; la OLAP gana cuando hay alta cardinalidad y querés cortar por múltiples dimensiones. _Ad Click, Top-K._

### Bases especializadas: time-series vs OLAP (criterio)

**Problema:** elegir entre TSDB y OLAP sin entender por qué una falla. **Mecanismo:** el factor decisivo es la cardinalidad. Con billones de IDs distintos, una TSDB (InfluxDB, Prometheus) trata cada ID como una serie y un top() termina escaneando todo → muere; pero el mismo problema se resuelve con precompute + cache. TimescaleDB (Postgres + hypertables + continuous aggregates) y los OLAP real-time (Druid/Pinot/ClickHouse, que pre-agregan en ingest) sí manejan alta cardinalidad. **Trade-off:** cada uno optimiza un perfil; usar el equivocado garantiza scans. **Gotcha:** el consejo es entender los _primitivos_ (alta cardinalidad → precompute + cache) en vez de memorizar "X DB no sirve para Y" — así razonás cualquier caso. _Top-K, Ad Click._

### Dedup antes del stream (no en la ventana)

**Problema:** deduplicar dentro de la ventana de agregación cuenta duplicados que caen en boundaries distintos. **Mecanismo:** hacés el dedup _antes_ de poner el evento en el stream — chequeás el impression ID contra una cache; si ya existe, lo descartás; si no, lo metés al stream y agregás el ID a la cache. **Trade-off:** dedup correcto cruzando ventanas a cambio de una cache de IDs vistos en el hot path. **Gotcha:** la razón es precisa — un duplicado a ambos lados de la frontera de un minuto se contaría dos veces si deduplicás dentro de Flink, porque cada ventana lo ve una vez; tiene que ser antes del stream. _Ad Click._

---

## EJE: PIPELINES / FAULT TOLERANCE — Data processing

### Pipeline multi-etapa para fault tolerance

**Problema:** un servicio monolítico que hace todo (fetch + extract + store) pierde _todo_ el progreso si falla cualquier sub-tarea. **Mecanismo:** partís el servicio en etapas pipelined conectadas por colas (Fetch → Extract → Store). Si una etapa falla, reintentás solo esa sin perder el resto; escalás cada una independiente (el fetch es I/O-bound, el transcode es CPU-bound); y sos robusto a cambios (cambiar la lógica de extracción no obliga a re-hacer el fetch caro). **Trade-off:** resiliencia y escalado granular a cambio de más componentes y estado intermedio que coordinar. **Gotcha:** ante cualquier problema de data processing, el primer instinto debería ser pipelinizar — aísla fallos y permite reprocesar una etapa sin tocar las demás. _Web Crawler, YouTube (DAG), Ad Click._

### Exponential backoff con visibility timeout (SQS) + DLQ

**Problema:** reintentar inmediatamente o con timers en memoria es frágil (los timers se pierden si el worker cae). **Mecanismo:** SQS no tiene backoff nativo pero da las primitivas. El _visibility timeout_ esconde un mensaje recibido pero no borrado; al expirar, reaparece para otro consumer. Ajustás ese timeout dinámicamente con `ChangeMessageVisibility` según `ApproximateReceiveCount` (cuántas veces se recibió) para lograr backoff exponencial. Tras N fallos, el redrive policy manda el mensaje a una _dead-letter queue_ automáticamente. **Trade-off:** reintentos robustos que sobreviven crashes a cambio de configurar bien los timeouts y la DLQ. **Gotcha:** los timers en memoria son el anti-pattern — si el worker muere, el timer muere; el visibility timeout vive en SQS, no en el worker. _Web Crawler._

### At-least-once vía no-ack hasta completar

**Problema:** si un worker toma un mensaje y se cae antes de terminar, ese trabajo se pierde. **Mecanismo:** el mensaje queda "en vuelo" pero no se borra/commitea hasta confirmar que el trabajo se completó (resultado en blob storage). Si el worker cae, el mensaje reaparece y otro lo retoma. Kafka: el offset no avanza hasta procesar con éxito. SQS: el visibility timeout re-expone el mensaje al expirar sin delete. **Trade-off:** garantía de no perder trabajo a cambio de posibles duplicados (si el worker terminó pero murió antes de confirmar) → exige idempotencia. **Gotcha:** at-least-once implica que el mismo trabajo puede correr dos veces — el procesamiento tiene que ser idempotente o vas a duplicar efectos. _Web Crawler, Uber (Kafka commit tras match), Ad Click._

### Per-resource lock + jitter contra thundering herd

**Problema:** N crawlers podrían pegarle al mismo dominio a la vez (violando politeness de 1 req/s), y si todos esperan y reintentan al resetear la ventana, vuelven a chocar sincronizados. **Mecanismo:** un lock atómico por dominio (Redis `SET NX` con TTL igual al crawl delay) serializa el acceso; quien no consigue el lock difiere su mensaje. Para evitar que todos reintenten exactamente al mismo tiempo, agregás _jitter_ — un delay aleatorio pequeño que desincroniza los reintentos. **Trade-off:** respeta el rate por recurso y evita el efecto manada a cambio de un lock distribuido y algo de aleatoriedad. **Gotcha:** sin jitter, varios procesos esperando la misma ventana reintentan simultáneamente, uno gana y el resto rebota — el jitter rompe esa sincronización. _Web Crawler._

### Deduplicación por contenido (content hash) vs por clave

**Problema:** URLs distintas pueden servir contenido idéntico (`example.com` vs `www.example.com`, mirrors entre dominios), y dedup por URL no los pilla. **Mecanismo:** primera línea es dedup por URL (¿ya crawleé esta URL?); segunda línea es dedup por _contenido_ — hasheás la página después de fetchearla y comparás el hash contra los ya vistos (con índice en la DB o un bloom filter). Si el contenido ya existe, saltás el parsing caro. **Trade-off:** evitás trabajo redundante real a cambio de tener que fetchear antes de poder deduplicar por contenido (la URL no basta). **Gotcha:** dedup por URL es necesario pero insuficiente — el contenido idéntico bajo URLs distintas es común en la web y solo el content hash lo detecta. _Web Crawler._

### Max depth contra ciclos/traps

**Problema:** páginas diseñadas para atrapar crawlers (se auto-enlazan infinitamente) te dejan crawleando el mismo sitio para siempre. **Mecanismo:** llevás un campo de profundidad (cantidad de _hops_ de link desde el seed, no segmentos del path): seed = 0, página enlazada desde el seed = 1, etc. Si la profundidad supera un umbral (15-20), parás de seguir esa rama. **Trade-off:** evitás traps y ciclos a cambio de potencialmente no llegar a páginas legítimas muy profundas. **Gotcha:** "depth" es hops de link, no profundidad de URL — confundirlos te haría cortar sitios legítimos con paths largos o no atrapar traps con URLs cortas. _Web Crawler. Traversal de grafos._

### DNS como bottleneck oculto

**Problema:** a miles de req/s sobre millones de dominios distintos, la resolución DNS se vuelve el cuello (puede ser el 70% del tiempo por thread según el paper de Mercator). **Mecanismo:** cacheás los lookups por dominio (todas las URLs de un mismo dominio reusan la resolución) y hacés round-robin entre múltiples proveedores DNS para repartir la carga y no chocar con rate limits de uno solo. **Trade-off:** menos latencia y menos riesgo de rate limit a cambio de gestionar cache de DNS y varios proveedores. **Gotcha:** es un bottleneck que casi nadie menciona (curl resuelve DNS solo, así que se invisibiliza) — sacarlo a la luz es señal de haber pensado el problema a escala real. _Web Crawler._

### Procesamiento como DAG con orquestador

**Problema:** trabajo con dependencias complejas (unos pasos dependen de otros, algunos paralelizables) no es un pipeline lineal. **Mecanismo:** modelás el trabajo como un grafo dirigido acíclico — el post-procesamiento de video es split → transcode (paralelo por segmento, sin dependencias entre ellos) → generar manifest → marcar completo. Un orquestador (Temporal) construye el grafo y asigna trabajo a workers en el momento correcto según las dependencias; los datos temporales van a S3 y se pasan URLs. **Trade-off:** paralelismo masivo donde se puede + orden donde hay dependencias, a cambio de operar un orquestador. **Gotcha:** la parte CPU-bound cara (transcodificar) se paraleliza al máximo porque los segmentos son independientes — identificar qué partes del DAG paralelizan es la clave. _YouTube (video post-processing)._

---

## EJE: WORKFLOWS DURABLES — Multi-paso con timeouts/humanos

### Durable execution (Temporal / Step Functions)

**Problema:** un workflow multi-paso con esperas humanas (driver acepta/rechaza/no responde) debe sobrevivir crashes del servicio sin perder su estado. **Mecanismo:** un framework de ejecución durable (Temporal, AWS Step Functions) modela el flujo completo como un workflow que persiste su estado en cada paso. Maneja timeouts, retries y fallbacks built-in; si el servicio cae, el workflow se reanuda exactamente donde quedó. Uber lo usa para el matching (manda al driver 1, espera 10s, si rechaza/timeout pasa al siguiente, repite). **Trade-off:** garantía de ejecución y fault-tolerance "gratis" a cambio de introducir y operar un sistema de orquestación nuevo. **Gotcha:** dato de color — Uber creó Cadence (origen de Temporal) precisamente para este tipo de workflow con humano en el loop; los timeouts en memoria son justamente lo que esto reemplaza. _Uber (matching workflow), YouTube (orquestación del DAG)._

### Delay queue para timeouts

**Problema:** querés actuar si algo no ocurre en cierto tiempo (el driver no responde en 10s), sin un timer en memoria que se pierda en un crash. **Mecanismo:** al mandar la request al driver, programás en paralelo un mensaje retrasado (SQS delay queue) que se procesará tras el timeout. Cuando se procesa, chequeás si la ride sigue sin asignar; si sí, pasás al siguiente driver y programás otro delayed message. **Trade-off:** timeouts durables sin estado en memoria a cambio de coordinar la cancelación del mensaje retrasado si el driver _sí_ acepta antes (race a manejar). **Gotcha:** el caso peliagudo es que el driver acepte justo después de programado el delayed message pero antes de procesarlo — hay que manejarlo para no reasignar una ride ya aceptada. _Uber._

### Queue + dynamic scaling para no perder requests en picos

**Problema:** en picos de demanda el servicio se satura y tira requests, y si una instancia cae pierde las que estaba procesando. **Mecanismo:** metés una cola (Kafka) entre el request y el matching service; los workers consumen a su ritmo y la cola absorbe el pico, escalando workers según el largo de la cola. El offset se commitea solo _tras_ un match exitoso, así un crash no pierde la request (otro worker la retoma). Particionás por región. **Trade-off:** no perdés requests y absorbés picos a cambio de operar la cola y de latencia (esperan en cola). **Gotcha:** una FIFO pura sufre head-of-line blocking (una request lenta bloquea las de atrás) — se mitiga con priority queue (proximidad, rating). _Uber._

### Stream retention + checkpointing para no perder datos

**Problema:** si el procesador de stream cae, ¿perdés los datos en vuelo? **Mecanismo:** dos capas. Kafka/Kinesis replican y _retienen_ los mensajes (ej. 7 días), así si el procesador cae podés re-leer (replay) desde un timestamp. Flink además hace _checkpointing_ — periódicamente vuelca su estado a S3, así reanuda desde el último checkpoint sin reprocesar todo. **Trade-off:** garantía de no perder datos a cambio de storage de retención y overhead de checkpoints. **Gotcha:** señal de seniority es saber cuándo _no_ hace falta — con ventanas de agregación chicas (1 min), si Flink cae perdés a lo sumo 1 min y lo recuperás con replay del stream, así que el checkpointing puede ser innecesario; proponerlo reflexivamente sin pensar la ventana es de junior. _Ad Click, Top-K._

---

## EJE: MEDIA — Video / streaming

### Almacenar en segmentos + múltiples formatos

**Problema:** guardar el video crudo no sirve — distintos devices necesitan distintos formatos, y no se puede descargar "parte" de un archivo monolítico. **Mecanismo:** el post-procesamiento parte el video en segmentos de pocos segundos y transcodifica cada uno a varias combinaciones de codec+container (formatos) playables en distintos devices. **Trade-off:** habilita streaming incremental y adaptación de calidad a cambio de un pipeline de procesamiento complejo y más storage (N formatos × M segmentos). **Gotcha:** segmentar es lo que _habilita_ todo lo demás (download incremental, adaptive bitrate) — sin segmentos estás atado a descargar el archivo entero. _YouTube. Audio/video._

### Adaptive bitrate streaming + manifest file

**Problema:** servir una calidad fija o buffereás (red lenta) o desperdiciás ancho de banda (red rápida con video de baja calidad). **Mecanismo:** durante el upload se genera un _manifest file_ — un índice de todos los segmentos en todos los formatos disponibles. El cliente descarga el manifest, elige el formato segmento a segmento según las condiciones de red del momento, y si la red empeora baja a segmentos más comprimidos sin cortar la reproducción. **Trade-off:** reproducción fluida adaptándose a la red a cambio de un cliente más complejo (participa activamente eligiendo) y de generar/almacenar todos los formatos. **Gotcha:** el cliente es protagonista — mide su propia red y decide; el server solo provee el menú (manifest) y los segmentos. _YouTube._

### Download incremental de segmentos

**Problema:** descargar el video entero antes de reproducir hace esperar minutos (10GB = 13 min a 100Mbps) y un corte de red tira todo. **Mecanismo:** el cliente descarga el primer segmento (pocos segundos), empieza a reproducir, y carga los siguientes en background mientras se ve el actual. Es streaming real, no download+playback. **Trade-off:** reproducción casi inmediata y resiliente a cortes a cambio de depender de que el video esté segmentado y de coordinar la carga anticipada. **Gotcha:** la diferencia con descargar en chunks es sutil pero real — los chunks de un archivo no siempre son reproducibles por sí solos; los segmentos sí son unidades playables, por eso esto funciona y el chunking crudo no garantiza reproducción. _YouTube._

### Server-side redirect para tracking garantizado

**Problema:** un redirect client-side se puede bypassear (un usuario avanzado lee la URL destino y va directo, sin que registres el click). **Mecanismo:** el click pega primero a tu server, que registra el click y responde con un 302 (redirect) hacia el destino. Así trackeás _cada_ click y podés agregar parámetros de tracking a la URL. **Trade-off:** tracking confiable a cambio de un hop extra por tu server (algo más de latencia y carga). **Gotcha:** el client-side parece más simple pero deja un agujero — alguien construye una extensión que salta tu tracking; el server-side lo cierra. _Ad Click._

### Impression ID firmado con HMAC (idempotencia + anti-fraude)

**Problema:** deduplicar clicks por userId rompe el retargeting (mostrar el mismo ad al mismo user a propósito), y dejar al cliente generar el ID permite fraude. **Mecanismo:** el Ad Placement Service genera un _impression ID_ único por cada instancia de ad mostrada (1000 users viendo el mismo ad = 1000 IDs distintos), así contás 1 click por user por _instancia_, no por user-ad. Lo firmás con HMAC (hash con clave secreta) antes de mandarlo al browser; al volver el click, verificás la firma para que un atacante no pueda falsificar IDs válidos. Dedup contra cache antes del stream. **Trade-off:** idempotencia correcta + anti-fraude a cambio de generar/firmar/verificar IDs (HMAC verifica en microsegundos, negligible). **Gotcha:** la sutileza es "por instancia, no por user" — dedup por userId contaría un solo click aunque le mostraras el ad 10 veces legítimamente. _Ad Click._

---

## EJE: SEGURIDAD / AISLAMIENTO

### Sandbox para código/contenido no confiable

**Problema:** ejecutar código de usuario arbitrario en tu infra es un riesgo de seguridad enorme. **Mecanismo:** corrés el código en contenedores efímeros aislados — sin acceso a red, con límites de CPU/memoria y timeout de ejecución, descartados tras correr. **Trade-off:** ejecución segura de código hostil a cambio del overhead de levantar/tirar contenedores aislados. **Gotcha:** multi-tenancy hostil (LeetCode corre código de miles de usuarios) exige aislamiento fuerte — un proceso compartido sería trivialmente explotable. _LeetCode._

### Encryption in transit + at rest + signed URLs

**Problema:** datos sensibles expuestos en tránsito, en reposo, o vía acceso no autorizado. **Mecanismo:** HTTPS cifra en tránsito; cifrado server-side en S3 (con clave gestionada aparte) protege en reposo; signed URLs de expiración corta (bearer tokens) controlan el acceso a blobs. Podés sumar IP binding o auth cookies. **Trade-off:** protección en capas a cambio de gestión de claves y la naturaleza bearer de las URLs firmadas. **Gotcha:** la signed URL es un bearer token — quien la tenga accede mientras no expire, por eso la expiración corta importa. _Dropbox, YouTube._

### Nunca confiar en datos del cliente

**Problema:** datos sensibles enviados por el cliente (userId, precio, timestamp) son trivialmente manipulables. **Mecanismo:** userId va en la sesión/JWT (no en el body/query donde el cliente lo edita); los timestamps los genera el server; valores como el precio/fareEstimate se recuperan de la DB, nunca se aceptan del cliente. **Trade-off:** seguridad a cambio de no poder shortcutear pasando datos cómodos desde el cliente. **Gotcha:** ver un userId o un precio en el body de una request es un red flag inmediato — esos datos van en el token o se derivan server-side. _Uber, Tinder, casi todos._

---

## EJE: COLABORACIÓN / CONSENSO / TRANSACCIONES DISTRIBUIDAS

> _Patrones de Google Docs, Payment System, Job Scheduler, Online Auction. Procedencia: fuentes públicas (no Hello Interview), porque estos breakdowns son premium._

### Operational Transformation (OT) vs CRDT — edición colaborativa

**Problema:** dos usuarios editan el mismo punto de un documento a la vez; si aplicás las operaciones en orden de llegada, cada cliente ve un documento distinto (divergen). **Mecanismo:** dos familias. _OT_ (lo que usa Google Docs, algoritmo Jupiter): un servidor central _transforma_ cada operación contra las concurrentes antes de aplicarla — si Alice inserta en pos 1 y Bob también, la de Bob se ajusta para preservar la intención de ambos. _CRDT_: diseñás las operaciones para que conmuten por construcción (cada caracter lleva un ID único e inmutable), así se mergean en cualquier orden sin coordinación central. **Trade-off:** OT mantiene el documento compacto (sin metadata por caracter) pero las funciones de transformación tienen cientos de edge cases y exigen servidor central; CRDT funciona offline/peer-to-peer y es más simple de razonar pero carga metadata/tombstones por caracter y necesita compactación para no crecer indefinidamente. **Gotcha:** Google eligió OT porque el server _igual_ ve toda operación (para permisos, render, storage), así que el costo de transformar centralmente es mínimo (<5ms); CRDT gana cuando offline/multi-master es requisito de primera clase. No elijas CRDT "porque suena más moderno" — la respuesta arranca del requisito de producto. _Google Docs, Figma, Notion. Fuente: System Design Sandbox, arxiv (Eg-walker, Kleppmann)._

### WebSocket + Pub/Sub para fan-out de edits (ya documentado, reforzado)

**Problema:** los clientes editando un doc están conectados a gateways distintos; un edit tiene que llegar a todos. **Mecanismo:** WebSocket para el canal bidireccional; tras validar y persistir el edit, se publica a un message queue/pub-sub que hace fan-out a todos los gateways suscritos a ese documento, y cada gateway lo empuja a sus clientes. **Trade-off:** desacopla gateways del scaling a cambio de latencia del hop por el broker (mitigable aplicando el edit en memoria en el gateway y persistiendo async). **Gotcha:** "usar WebSockets" resuelve el _transporte_, no los edits concurrentes — eso lo resuelve OT/CRDT; confundirlos es el error clásico. _Google Docs. (Mismo patrón que Ticketmaster/WhatsApp.)_

### Idempotency key con request hash (anti doble-cobro)

**Problema:** un cliente reintenta un pago tras un timeout y lo cobra dos veces. **Mecanismo:** cada request lleva una idempotency key única generada por el cliente. El server guarda `(merchant_id, idempotency_key) → (request_hash, response, status)`. Misma key + mismo payload → devuelve la respuesta original sin re-ejecutar; misma key + payload distinto → falla con 409 Conflict. El payment_order_id se reusa como dedup key en el PSP de terceros. **Trade-off:** exactly-once efectivo (at-least-once delivery + writes idempotentes) a cambio de almacenar y consultar las keys. **Gotcha:** exactly-once no existe a nivel de red — se _construye_ con at-least-once + idempotencia; decir "uso exactly-once networking" es señal de no entenderlo. El hash del payload es clave: detecta reuso incorrecto de la misma key. _Payment, Stripe, Adyen. Fuente: Pragmatic Engineer, Stripe docs._

### Double-entry ledger inmutable (append-only)

**Problema:** un sistema financiero no puede perder ni corromper un registro de dinero, y necesita auditabilidad total. **Mecanismo:** contabilidad de doble entrada — cada transacción escribe dos asientos (un débito y un crédito) que suman cero, en una misma transacción atómica (ambos o ninguno). El ledger es inmutable/append-only: nunca actualizás "monto de 10 a 8", agregás un asiento correctivo o una reversión. El balance se deriva sumando asientos. **Trade-off:** integridad y auditabilidad perfectas (podés reconstruir el balance en cualquier punto del pasado) a cambio de más escrituras y storage que un simple "campo balance". **Gotcha:** los montos se guardan como string/decimal, no float — los errores de redondeo de double son inaceptables con dinero; y nunca permitir que una pierna del doble asiento commitee sin la otra. _Payment. Fuente: Formance, Pragmatic Engineer, systemdesignhandbook._

### 2PC vs Saga (transacción distribuida)

**Problema:** una operación cruza varios servicios (orden + inventario + pago + envío) y necesitás atomicidad sin una sola DB. **Mecanismo:** _2PC_ (two-phase commit) — un coordinador pregunta a todos "¿pueden commitear?" (prepare) y si todos dicen sí, ordena commit; mantiene locks en todos los participantes hasta el final. _Saga_ — descomponés en una secuencia de transacciones locales, cada una commitea y libera locks inmediatamente; si el paso N falla, ejecutás transacciones _compensatorias_ (C1…Cn) que deshacen los pasos previos en reversa. Saga tiene dos sabores: orquestación (un cerebro central dirige) o coreografía (cada servicio reacciona a eventos). **Trade-off:** 2PC da consistencia fuerte pero es lento, bloqueante y frágil (el coordinador es punto único); Saga da alta disponibilidad y encaja con microservicios pero es eventualmente consistente y exige diseñar compensaciones correctas (a veces imposibles — no podés "des-enviar" un email). **Gotcha:** para pagos suele usarse híbrido — un core ledger fuertemente consistente (single-DB transaction) envuelto en una Saga para el resto; las compensaciones imposibles se manejan con best-effort ("email de disculpa") o previniendo (enviar el email solo en el último paso). _Payment, e-commerce. Fuente: techinterview.org, designgurus._

### Outbox pattern (publicación confiable de eventos)

**Problema:** tenés que actualizar la DB _y_ publicar un evento (a Kafka), pero si la DB commitea y el publish falla, perdés el evento (dual-write problem). **Mecanismo:** en la misma transacción de DB que cambia el estado, escribís el evento a una tabla "outbox". Un proceso aparte lee la outbox y publica a Kafka, marcando como enviado. Así el evento se publica si y solo si la transacción commiteó. **Trade-off:** publicación de eventos exactamente alineada con el estado a cambio de un proceso relay extra y latencia (el evento sale un instante después). **Gotcha:** es esencial siempre que una transacción de DB deba disparar un evento — el dual-write directo (commit DB, luego publish) pierde eventos ante crash entre ambos. _Payment, event-driven systems. Fuente: techinterview.org._

### Optimistic Concurrency Control (OCC) con version/CAS

**Problema:** dos bids/updates llegan casi a la vez sobre el mismo registro; sin control, ambos pasan el check ("$105 > $100") y ambos se aceptan (lost update). **Mecanismo:** cada registro lleva una columna `version`. El update es condicional: `UPDATE ... SET version=version+1 WHERE id=? AND version=?`. Si otra escritura ganó, 0 filas afectadas → le decís al usuario "outbid, reintentá". En auctions de alta tasa, un Redis/Valkey CAS vía Lua hace el check-and-set atómico en sub-milisegundo. **Trade-off:** evita locks de fila caros (que no escalan con cientos de bids concurrentes) a cambio de que los perdedores reintenten. **Gotcha:** OCC brilla cuando los conflictos son _raros_; si todos pelean por la misma fila constantemente (hot row), los reintentos se disparan y conviene serializar con un contador atómico en Redis. En auctions, el truco extra es no escribir `current_price` en cada bid (hot row, MVCC bloat) sino tratar la tabla de bids como append-only y derivar el máximo. _Online Auction, Robinhood. Fuente: techinterview.org, crackingwalnuts._

### Proxy bidding (lógica de negocio en el server)

**Problema:** un usuario no quiere estar pendiente del auction; quiere poner su máximo y que el sistema puje por él lo mínimo necesario. **Mecanismo:** guardás el `proxy_max` del usuario (cifrado, nunca visible para otros). Cuando llega un bid nuevo, comparás contra el proxy max actual: si el nuevo supera el proxy max, el nuevo postor gana y notificás al anterior que fue outbid; si no, el sistema sube automáticamente al proxy bidder lo mínimo. Toda la lógica corre dentro de una transacción de DB. **Trade-off:** mejor UX y más engagement a cambio de lógica de negocio sensible que hay que proteger (el max nunca se expone). **Gotcha:** es el diferenciador que muestra que entendés el _producto_, no solo la infra — y el max cifrado at-rest es la sutileza de seguridad. _Online Auction (eBay). Fuente: techinterview.org._

### Leader election para evitar doble-scheduling

**Problema:** corrés N instancias del scheduler para alta disponibilidad, pero si todas ejecutan el cron "enviar reporte diario", el usuario recibe N emails (split-brain). **Mecanismo:** un protocolo de consenso (Raft/Paxos vía etcd/[[ZooKeeper]]) elige un único leader que toma las decisiones autoritativas de scheduling; los followers esperan. Si el leader cae, se elige uno nuevo en segundos y reconstruye el estado desde storage durable. **Trade-off:** elimina el doble-scheduling a cambio de menor disponibilidad durante la transición de leader (ventana de failover) y la complejidad del consenso. **Gotcha:** la elección debe converger rápido (Google SRE: <1 min para no perder un launch de cron); y al promoverse, el nuevo leader debe concluir los launches a medio terminar del anterior (partial failures), no solo empezar limpio. _Job Scheduler, distributed cron. Fuente: Google SRE book, educative._

### At-least-once + idempotencia (en vez de exactly-once)

**Problema:** un worker toma un job y se cae antes de confirmar; ¿lo perdés o lo ejecutás dos veces? Exactly-once es imposible de garantizar (Two Generals' Problem). **Mecanismo:** asumís at-least-once — el job se reintenta hasta confirmar éxito, aceptando posibles duplicados — y hacés la lógica del job _idempotente_ para que ejecutar dos veces sea inofensivo. Un visibility timeout esconde el job mientras un worker lo procesa; si el worker no confirma, reaparece. Para jobs largos, checkpointing (guardar progreso y reanudar). **Trade-off:** simplicidad y robustez a cambio de exigir idempotencia en la lógica del job. **Gotcha:** decir "at-least-once" con confianza y explicar por qué exactly-once es ilusorio es señal de madurez en distribuidos; el visibility timeout debe ser mayor que la duración esperada del job más larga, o re-entregás jobs que aún corren. _Job Scheduler, Web Crawler, Uber. Fuente: educative, Google SRE._

### Priority starvation: aging + weighted fair queuing

**Problema:** una priority queue pura deja jobs de baja prioridad sin ejecutar nunca si siguen llegando de alta prioridad. **Mecanismo:** dos mitigaciones. _Aging_ — un job que esperó más de un umbral sube su prioridad efectiva (uno de baja que esperó una hora se trata como normal). _Weighted fair queuing_ — los workers no siempre toman de la cola más alta; toman 70% de alta, 20% de media, 10% de baja, garantizando progreso a todos. **Trade-off:** evita starvation a cambio de que jobs de alta prioridad esperen ocasionalmente. **Gotcha:** es el follow-up típico tras proponer priority queue — si no tenés respuesta para la starvation, se nota. _Job Scheduler. Fuente: mockingly.ai._

### Anti-sniping / soft close (extensión automática de tiempo)

**Problema:** un bidder puja en el último segundo (snipe) para que nadie alcance a contrarrestar, ganando por timing en vez de por valor. **Mecanismo:** si llega un bid dentro de una ventana al cierre (ej. últimos 30-60s), extendés automáticamente el `current_end_time` (ej. +2-5 min). Se repite mientras lleguen bids en la ventana, hasta que cesa el interés. Se trackea un `effective_end_time` separado del original, actualizado atómicamente junto con la aceptación del bid (mismo Lua CAS que valida el bid extiende el tiempo). **Trade-off:** fairness (todos pueden responder) a cambio de un cierre no determinista y lógica de close más compleja. **Gotcha:** necesitás salvaguardas contra extensión infinita — un `max_extensions` (ej. 20) y un `absolute_end_time` (ej. original +30 min) como techo duro; pasado eso se aceptan bids pero el auction cierra. Y al extender, hay que re-armar el timer (Flink keyed-timer consume el evento `end_time_changed` y reprograma). Preguntá si está en scope antes de asumir end time fijo. _Online Auction (eBay no lo usa, muchas plataformas sí). Fuente: systemdesignhandbook, crackingwalnuts._

---

## EJE: CONSISTENT HASHING / PARTICIONAMIENTO (Distributed Cache)

> _Patrones de Distributed Cache. Procedencia: fuentes públicas (breakdown premium)._

### Consistent hashing (vs mod hashing)

**Problema:** con `hash(key) % N` nodos, agregar o quitar un nodo cambia N y _remapea casi todas las keys_ → invalidación masiva de cache. **Mecanismo:** mapeás nodos y keys a un anillo de hash (0 a 2³²); una key pertenece al primer nodo en sentido horario. Al agregar un nodo, solo las keys entre él y su predecesor se remapean — el resto no se mueve. Virtual nodes (cada nodo físico aparece muchas veces en el anillo) reparten la carga más parejo y suavizan el rebalanceo. **Trade-off:** rebalanceo mínimo al cambiar topología a cambio de una estructura de anillo y búsqueda O(log N) por binary search. **Gotcha:** el mod hashing es el anti-pattern — funciona hasta que escalás, y ahí remapea todo; consistent hashing es el estándar precisamente por minimizar el movimiento de datos. _Distributed Cache, WhatsApp (chat servers), Rate Limiter (shards), Uber. Fuente: systemdesignhandbook, múltiples._

### Políticas de evicción: LRU vs LFU

**Problema:** la cache tiene memoria finita; ¿qué desalojás cuando se llena? **Mecanismo:** _LRU_ (Least Recently Used) desaloja lo no accedido hace más tiempo, asumiendo localidad temporal (lo reciente se vuelve a acceder); implementación clásica: doubly-linked list (orden de acceso) + hash map (lookup O(1)), cada acceso mueve la entrada a la cabeza, se desaloja la cola. _LFU_ (Least Frequently Used) desaloja lo menos accedido en total, mejor para popularidad estable. **Trade-off:** LRU se adapta a cambios de popularidad pero sufre "scan pollution" (un barrido grande ensucia la cache); LFU mantiene lo popular pero se aferra a viejos hits y no reacciona a popularidad cambiante. **Gotcha:** algoritmos como 2Q o TinyLFU+S-LRU combinan ambos para evitar la contaminación por scan — mencionar la admission policy (TinyLFU) sobre heavy skew es señal de profundidad. _Distributed Cache, Redis (volatile-lru, allkeys-lru). Fuente: systemdesignhandbook, dev.to._

### Estrategias de escritura: cache-aside vs write-through vs write-back

**Problema:** ¿cómo mantenés la cache sincronizada con la DB fuente de verdad? **Mecanismo:** _Cache-aside_ (lazy) — la app chequea cache, si miss lee DB y rellena; resiliente a fallo de cache, simple, pero ventana de staleness hasta que expire el TTL. _Write-through_ — la app escribe a cache y la cache escribe a DB sincrónicamente; consistencia fuerte pero latencia de escritura doble. _Write-back_ (write-behind) — la app escribe a cache, la cache escribe a DB async después; escrituras ultrarrápidas pero riesgo de pérdida si la cache cae antes de sincronizar. **Trade-off:** consistencia vs latencia de escritura vs riesgo de pérdida. **Gotcha:** para interview, cache-aside + TTL es el default recomendado para el 90% de lecturas; write-back solo cuando la velocidad de escritura es crítica y tolerás pérdida. _Distributed Cache, casi todo sistema con cache. Fuente: adevguide, dev.to._

### Quorum N/R/W (consistencia ajustable)

**Problema:** querés balancear consistencia, disponibilidad y latencia sin elegir un extremo fijo. **Mecanismo:** cada dato se replica en N nodos; una escritura necesita W confirmaciones, una lectura R respuestas. Si R+W > N, garantizás que toda lectura ve la última escritura (consistencia fuerte). Bajando W (ej. W=1) priorizás disponibilidad de escritura; bajando R, lecturas rápidas. El coordinador escribe local y hace fan-out a los N-1, pero responde al cliente apenas junta W ACKs (no espera los N). **Trade-off:** son tres perillas (N, R, W) que mueven el punto en el espectro consistencia/disponibilidad/latencia. **Gotcha:** R+W>N da "read-your-writes" pero no linearizabilidad total; y W=1 maximiza disponibilidad (la escritura se acepta si un solo nodo la persiste) a costa de durabilidad. _Distributed Cache, Dynamo/Cassandra. Fuente: Dynamo paper (Amazon)._

### Sloppy quorum + hinted handoff (always-writeable)

**Problema:** si uno de los N nodos réplica está caído, ¿rechazás la escritura? Eso mata disponibilidad. **Mecanismo:** _sloppy quorum_ — la escritura aterriza en el siguiente nodo sano del anillo en vez del caído. Ese nodo guarda el dato con un _hint_ (marca de quién era el destinatario real) en un store local separado; periódicamente chequea si el nodo original revivió y le entrega el dato (hinted handoff), luego lo borra. **Trade-off:** "always writeable" (la escritura solo falla si _todos_ los nodos están caídos) a cambio de una ventana donde el dato vive en el nodo equivocado. **Gotcha:** funciona si las fallas son transitorias y el churn es bajo; si el nodo con hints muere antes de entregar, el dato queda en riesgo — por eso hace falta también anti-entropy (Merkle). _Distributed Cache, Dynamo. Fuente: Dynamo paper._

### Anti-entropy con Merkle trees

**Problema:** las réplicas divergen con el tiempo (nodos que cayeron y volvieron, hints perdidos) y comparar millones de keys una a una para re-sincronizar es carísimo. **Mecanismo:** cada nodo construye un Merkle tree sobre sus datos por rango de keys — las hojas son hashes de los valores, los nodos internos hashes de sus hijos, la raíz un hash de todo. Para comparar dos réplicas, comparás raíces: si coinciden, están idénticas (cero transferencia). Si difieren, bajás por las ramas que no matchean hasta las hojas divergentes y sincronizás solo esas. **Trade-off:** detección de inconsistencias en O(log N) comparaciones en vez de O(N) (un nodo con 1M keys: ~20 comparaciones vs 1M, ~50.000× menos) a cambio de mantener el árbol. **Gotcha:** es un proceso de background, NO está en el hot path de read/write (esos usan quorum + vector clocks); en un cluster sano las raíces casi siempre coinciden y no se transfiere nada. _Distributed Cache, Dynamo, Cassandra. Fuente: Dynamo paper._

### Gossip protocol para membership y failure detection

**Problema:** ¿cómo sabe cada nodo qué otros nodos existen y cuáles están vivos, sin un registro central que sea punto único de fallo? **Mecanismo:** cada nodo intercambia periódicamente su vista del cluster (membership, liveness) con peers aleatorios; la información se propaga epidémicamente (como un rumor) hasta que todos convergen. No hay master ni registro central. Usa un detector de fallos por acumulación (Phi Accrual) en vez de un timeout binario. **Trade-off:** descentralización total y sin punto único de fallo a cambio de convergencia eventual (toma algunas rondas que todos se enteren de un cambio) y tráfico de fondo constante. **Gotcha:** es lo que hace a Dynamo simétrico (todo nodo es igual) — combinado con consistent hashing, agregar/quitar un nodo solo afecta a sus vecinos en el anillo. _Distributed Cache, Dynamo, Cassandra. Fuente: Dynamo paper._

### Vector clocks para detectar escrituras concurrentes

**Problema:** con escrituras concurrentes a la misma key en distintas réplicas (sin master que serialice), ¿cuál gana? Los timestamps de reloj de pared no sirven (los relojes difieren entre máquinas). **Mecanismo:** cada versión de un objeto lleva un vector clock — una lista de pares (nodo, contador). Cuando un nodo actualiza, incrementa su propio contador. La versión V1 precede causalmente a V2 si todos los contadores de V1 son ≤ los de V2; ahí V2 reemplaza a V1 automáticamente. Si ninguno domina al otro, las versiones son _concurrentes_ — un conflicto real. **Trade-off:** detectás conflictos genuinos vs orden causal sin reloj global, a cambio de que los conflictos reales los tenga que reconciliar la aplicación (más contexto semántico) en el próximo read. **Gotcha:** el vector clock puede crecer si muchos nodos tocan la key — Dynamo lo capa y poda los más viejos (rompe la garantía causal en casos rarísimos, medido como "extremadamente raro" en producción). La detección de conflicto siempre ocurre en el read. _Distributed Cache, Dynamo. Fuente: Dynamo paper._

---

## EJE: TIME-SERIES / OBSERVABILIDAD (Metrics Monitoring)

> _Patrones de Metrics Monitoring. Procedencia: fuentes públicas (breakdown premium)._

### Modelo de datos time-series (name + labels → serie)

**Problema:** las métricas son time-stamped a volumen brutal y un modelo relacional no escala. **Mecanismo:** una serie se identifica por nombre + conjunto de labels (`http_requests{method=GET, status=200}`); cada combinación de labels es una serie única. El tipo de métrica (counter, gauge, histogram) determina qué operaciones son válidas. Los labels son lo que da poder de slicing pero también lo que dispara la cardinalidad. **Trade-off:** consultas flexibles por dimensión a cambio de que cada label nuevo multiplica las series. **Gotcha:** los labels manejan la cardinalidad — meter un label de alta cardinalidad (user_id, request_id) explota el número de series y degrada el TSDB; es el error #1 que los interviewers de Datadog quieren que levantes sin que te pregunten. _Metrics Monitoring, Prometheus. Fuente: mockingly.ai, techinterview.org._

### Pull vs Push para recolección de métricas

**Problema:** ¿el sistema scrape-a los targets o los targets le envían? **Mecanismo:** _Pull_ (Prometheus) — el sistema hace GET periódico a un endpoint `/metrics` de cada target, descubriéndolos vía service discovery. _Push_ — los targets envían métricas al collector. **Trade-off:** pull gana en debugging (sabés qué target falla), health check (si no responde, está caído) y autenticidad; push gana en jobs efímeros/serverless (no viven para ser scrapeados), redes con firewall complejo, y performance (UDP fire-and-forget). **Gotcha:** no es either/or — producción suele combinar (pull para servicios estables, push para jobs cortos); y para métricas como counters, pérdida ocasional es aceptable (fire-and-forget está bien). _Metrics Monitoring. Fuente: mockingly.ai, double-pointer._

### Tiering hot/warm/cold + downsampling

**Problema:** retener métricas a resolución completa por 13 meses es económicamente inviable. **Mecanismo:** datos recientes a resolución raw (por segundo) en almacenamiento rápido (hot); a medida que envejecen, se _downsamplean_ a granularidad creciente (1m, 5m, 1h) y migran a almacenamiento más barato (warm, cold). El downsampling guarda min/max/sum/count por bucket, no solo el promedio, para preservar la capacidad de análisis. **Trade-off:** retención larga económica a cambio de perder resolución fina en datos viejos (aceptable: nadie consulta CPU por-segundo de hace un año). **Gotcha:** explicar downsampling con el detalle min/max/sum/count en vez de solo "agrego" es lo que separa candidatos; y Prometheus solo no retiene largo plazo — necesitás Thanos/Cortex/Mimir con remote_write a S3. _Metrics Monitoring. Fuente: mockingly.ai, wasilzafar._

### Kafka como buffer de ingesta (desacoplar de la TSDB)

**Problema:** si la TSDB cae o se satura, perdés métricas. **Mecanismo:** los collectors escriben a Kafka, que absorbe spikes y desacopla la recolección del almacenamiento; stream processors (Flink) leen de Kafka y escriben a la TSDB. Si la TSDB no está disponible, los datos esperan en Kafka. Particionás por nombre de métrica. **Trade-off:** no perder métricas ante fallo de la TSDB a cambio de la infra de Kafka. **Gotcha:** mismo patrón que Ad Click — el stream buffer protege contra pérdida cuando el storage está bajo estrés; es reutilización del eje de stream processing. _Metrics Monitoring. Fuente: mockingly.ai, Medium (Goutam Gupta)._

---

## EJE: LLM SERVING / INFERENCIA (ChatGPT)

> _Patrones de ChatGPT. Procedencia: fuentes públicas (breakdown premium)._

### Streaming token-by-token vía SSE

**Problema:** generar la respuesta completa antes de mostrarla hace esperar segundos al usuario. **Mecanismo:** el LLM genera un token a la vez (autoregresivo: cada token depende de los anteriores), y el inference server emite cada token al instante vía Server-Sent Events (SSE) o HTTP chunked, en vez de esperar la respuesta completa. El frontend lo renderiza creando el efecto "typing". Se activa con `stream=true`. **Trade-off:** latencia percibida mucho menor (ves el primer token rápido) a cambio de una conexión de streaming abierta durante toda la generación. **Gotcha:** SSE (unidireccional server→client) basta acá — no necesitás WebSocket bidireccional porque el cliente ya mandó todo su prompt; la generación es inherentemente secuencial (no podés generar el token 5 sin el 4). _ChatGPT. Fuente: systemdesignschool, atharvanaik._

### Continuous batching en GPU

**Problema:** las GPUs son carísimas y procesar requests de a uno las deja infrautilizadas; pero el batching naive sufre head-of-line blocking (un request corto espera a uno largo). **Mecanismo:** un scheduler agrupa requests en batches para saturar el cómputo de la GPU. _Continuous batching_ permite que requests nuevos se unan al batch en los boundaries de token a medida que otros terminan, en vez de esperar a que todo el batch acabe. **Trade-off:** maximiza utilización de GPU y throughput a cambio de un scheduler más complejo; balancea latencia individual contra throughput agregado. **Gotcha:** el head-of-line blocking es la razón de ser del continuous batching — sin él, un prompt corto batcheado con uno largo paga la cola del largo; mencionar P95/P99 y tail latency es señal de profundidad. _ChatGPT, vLLM, TensorRT-LLM. Fuente: systemdesignhandbook, arxiv (BatchLLM)._

### Inyección de contexto (modelos stateless)

**Problema:** el LLM es stateless — no recuerda nada entre requests, solo procesa el prompt actual. **Mecanismo:** para simular una conversación continua, el sistema inyecta el historial completo (system message + historial de la conversación + input nuevo) en _cada_ request. Una capa de context management arma ese prompt. **Trade-off:** conversación coherente a cambio de que el prompt crezca con cada turno (más tokens = más costo y latencia, y tope de context window). **Gotcha:** la naturaleza stateless es el desafío central — el "recuerdo" es una ilusión construida re-enviando todo el historial; a escala se optimiza con KV-cache y prefix sharing (reusar el cómputo del prefijo común). _ChatGPT. Fuente: systemdesignhandbook, atharvanaik._

### Model routing (modelo según complejidad)

**Problema:** correr el modelo flagship (caro) para toda query desperdicia dinero en las simples. **Mecanismo:** un router clasifica la query y la manda al modelo apropiado — una pregunta simple va a un modelo chico y rápido, una tarea de coding compleja al flagship. Optimiza costo dinámicamente. **Trade-off:** ahorro de costo significativo a cambio de la complejidad del router y el riesgo de rutear mal (mandar algo complejo al modelo chico). **Gotcha:** es la palanca de costo principal en GenAI — el costo de inferencia domina la arquitectura (un día de 1.2B tokens cuesta decenas de miles de dólares), así que rutear por complejidad es criterio de negocio, no solo técnico. _ChatGPT. Fuente: systemdesignhandbook._

### Speculative decoding

**Problema:** generar token por token con el modelo grande es lento (cada token = un forward pass completo del modelo pesado). **Mecanismo:** un modelo "draft" chico y rápido genera varios tokens por adelantado especulativamente; el modelo "target" grande los verifica en un solo pass paralelo. El grande verifica múltiples tokens más rápido de lo que los generaría uno a uno. **Trade-off:** menor latencia a cambio de correr dos modelos y desperdiciar el draft cuando el target rechaza su especulación. **Gotcha:** funciona porque verificar en paralelo es más barato que generar secuencialmente; lo soportan vLLM, TensorRT-LLM nativamente. Es optimización avanzada — mencionarla muestra que conocés el stack de inferencia real. _ChatGPT. Fuente: systemdesignhandbook, AI with Aish._

### KV-cache y PagedAttention (gestión de memoria GPU)

**Problema:** en generación autoregresiva, cada token nuevo necesita las claves/valores (KV) de todos los tokens anteriores; recalcularlos cada vez es carísimo, así que se cachean — pero los sistemas naive reservan memoria contigua por la longitud máxima posible de salida, desperdiciando el 60-80% de la VRAM por fragmentación (no sabés cuántos tokens generará cada request). **Mecanismo:** _PagedAttention_ (vLLM, inspirado en la memoria virtual de los SO) parte el KV-cache en bloques de tamaño fijo que pueden vivir en memoria física no contigua; cada secuencia direcciona sus bloques lógicos vía una block table que mapea a bloques físicos dispersos. La memoria se asigna on-demand a medida que se generan tokens. **Trade-off:** desperdicio de memoria bajo ~4% (solo el último bloque parcial) → caben más requests concurrentes por GPU y contextos más largos, a cambio de kernels GPU custom para los nuevos patrones de acceso. **Gotcha:** habilita prefix sharing — cuando varias requests comparten prompt (o un prompt genera múltiples completions), comparten los mismos bloques KV con copy-on-write; es lo que hace viable el batching a escala. Resuelve la fragmentación; el continuous batching resuelve la utilización — necesitás ambos. _ChatGPT, vLLM, TensorRT-LLM, HF TGI. Fuente: paper PagedAttention (Kwon et al, SOSP 2023), Red Hat, RunPod._

### Vector search / ANN para retrieval semántico (RAG)

**Problema:** buscar por significado (no por keyword exacto) requiere comparar embeddings — vectores densos de 768-1536 dimensiones — y el nearest-neighbor exacto sobre millones de vectores es inviable (curse of dimensionality); un B-tree no sirve para alta dimensionalidad. **Mecanismo:** _Approximate Nearest Neighbor_ (ANN) sacrifica precisión mínima por velocidad de órdenes de magnitud. Familias de índice: _HNSW_ (grafo jerárquico navegable, el default para baja latencia), _IVF_ (particiona el espacio en clusters, buscás solo los cercanos), _PQ_ (product quantization, comprime vectores para escala billón con poca memoria). La similitud se mide con cosine (texto, insensible a magnitud), dot product o L2. En RAG: chunkeás documentos, generás embeddings, los guardás con metadata; ante una query, la embebés, corrés ANN para el top-K, e inyectás esos chunks en el prompt del LLM. **Trade-off:** búsqueda semántica a escala a cambio de aproximación (recall <100%) y el tuning recall-vs-latencia. **Gotcha:** pre-filtrar metadata _antes_ del ANN reduce el conjunto candidato (más rápido); y no siempre necesitás vector DB — si los datos son estructurados y las queries exactas, un índice relacional gana. Hybrid search (BM25 keyword + vector) cubre lo que cada uno se pierde. _ChatGPT/RAG, recomendaciones, búsqueda semántica. Fuente: arxiv (HNSW Malkov & Yashunin), infrasketch, MachineLearningMastery, pgvector docs._

---

## EJE: RESILIENCIA / AISLAMIENTO DE FALLOS (transversal)

> _Patrones de fault isolation a gran escala. Procedencia: fuentes públicas (AWS re:Invent, builder docs)._

### Cell-based architecture (bulkhead)

**Problema:** en un sistema monolítico a escala, un fallo (deploy malo, tenant tóxico, bug) afecta a _toda_ la flota de clientes. **Mecanismo:** dividís el sistema en celdas — réplicas independientes y autocontenidas del sistema completo, cada una la unidad mínima de escala (puede haber cientos). Una capa de routing fina (lo más delgada y robusta posible) asigna cada cliente/tenant a una celda. Un fallo queda contenido en una celda: afecta a esa fracción de clientes, no a todos. Viene del patrón bulkhead de la construcción naval (compartimentos estancos: una brecha inunda una sección, no el barco). **Trade-off:** blast radius predecible + performance y escalabilidad predecibles a cambio de costo (recursos de costo fijo se replican por celda — mitigable agrupando N celdas en una VPC) y complejidad operacional. **Gotcha:** el anti-pattern silencioso es que las celdas compartan estado mutable casualmente — ahí vuelve a ser un monolito sin que nadie lo note; las celdas no deben compartir estado, y los cross-cell calls se tratan como dependencias externas (timeouts, fallbacks). La capa de routing es el punto crítico: hacela la más delgada y testeada. _AWS, multi-tenant a escala. Fuente: AWS re:Invent 2018 (ARC338), AWS Builder Center, Well-Architected._

### Shuffle sharding (sobre cell-based)

**Problema:** si asignás cada cliente a una sola celda y esa cae, ese 100% de clientes queda afuera; y un "noisy neighbor" tóxico tumba a todos los de su celda. **Mecanismo:** en vez de asignar cada cliente a una celda fija, lo asignás a un _subconjunto aleatorio pequeño_ de un pool mucho mayor (como repartir cartas de un mazo barajado: dos clientes rara vez comparten el mismo subconjunto completo). Si un cliente tóxico tumba sus nodos, otro cliente solo comparte parte del subconjunto y sigue sirviéndose por el resto. Con overlap limitado (ej. máximo 1 nodo en común entre dos clientes), el blast radius se reduce drásticamente (de 25% a fracciones de 1%). **Trade-off:** aislamiento mucho mayor del noisy-neighbor a cambio de que funciona mejor sobre cosas stateless/replicadas/seguras de servir desde varios targets (DNS, request handlers, workers), no sobre estado mutable compartido. **Gotcha:** shuffle sharding NO es lo mismo que cell-based — es una técnica de asignación que se _aplica sobre_ arquitecturas con frontend stateless; combinarlas es lo potente. _AWS (Route 53, S3), multi-tenant. Fuente: AWS Builders' Library, re:Invent 2019 (DOP328)._

### Aislamiento por AZ / región (active-active vs active-passive)

**Problema:** un fallo de una availability zone o región entera tumba todo si no hay redundancia geográfica. **Mecanismo:** replicás la infra en varias AZs/regiones. _Active-active_ — todas reciben tráfico, si una cae el resto absorbe (sin downtime). _Active-passive_ — una primaria sirve, una standby espera y el equipo redirige tráfico cuando hace falta. Las AZs sirven como frontera natural de celda. **Trade-off:** active-active da disponibilidad máxima pero duplica costo de infra; active-passive ahorra pero el failover toma tiempo (redirigir tráfico). **Gotcha:** las celdas y el aislamiento por AZ se combinan — una celda puede vivir dentro de una AZ, dándote fault isolation en dos niveles (la celda y la zona). _Transversal, AWS. Fuente: AWS Architecture Blog, re:Invent._

---

## META-PATRÓN — El arco de razonamiento

Lo que demuestra dominio no es llegar a la solución final, sino el **razonamiento iterativo**: cada optimización resuelve un cuello de botella y revela el siguiente.

Ejemplo (FB News Feed):

1. Naive (fan-out on read) → muere con usuarios que siguen a muchos.
2. Precompute (fan-out on write) → arregla lectura, muere con usuarios seguidos por muchos.
3. Híbrido (read o write por-cuenta) → resuelve ambos, deja hot keys en lectura.
4. Cache redundante → resuelve los hot keys.

No es "encontré la respuesta correcta"; es "cada solución reveló el siguiente bottleneck, y la final reconoce que no hay respuesta única — depende del perfil de cada entidad".

**Otros principios transversales:**

- **El cliente es parte del sistema** — chunking/compresión (Dropbox), cache de swipes (Tinder), adaptive intervals (Uber), ABR (YouTube).
- **A veces la mejor solución no es técnica** — virtual queue (Ticketmaster), max friends 5000 (FB), retargeting como decisión de producto (Ad Click).
- **Parámetros tuneables sin cambiar la lógica** — TTL, tamaño de cache, threshold de precompute, ventana de agregación, número de shards. Dan control operacional.
- **Saber cuándo NO usar algo** — checkpointing innecesario con ventanas chicas (Ad Click), bloom filter overkill vs índice (Web Crawler), Flink demasiado para mid-level.

---

## Frecuencia de patrones (cuántos de los 20 problemas usan cada uno)

> Conteo sobre los 20 problemas (13 de Hello Interview + 7 de fuentes públicas). De más usado a más específico: los de número alto son el **core que repasás siempre**; los de 1 son **específicos del problema que cae**. El número entre paréntesis es cuántos problemas lo usan.

1. `Back-of-envelope `(17)
2. In-memory cache (14)
3. Ratio read/write primero (12)
4. Idempotencia API (11)
5. Consistencia eventual (10)
6. Sharding vs replicación (10)
7. Cache TTL por volatilidad (9)
8. Elegir DB por acceso (8)
9. Async workers + cola (6)
10. Distribución geográfica (6)
11. Horizontal scaling stateless (6)
12. Distributed lock TTL (5)
13. Read replicas + leader (5)
14. At-least-once (no-ack) (4)
15. CDN (4)
16. Consistent hashing (vs mod) (4)
17. Fingerprinting (4)
18. GSI / índice invertido (4)
19. Locking / double-booking (4)
20. Precomputación (4)
21. Split read/write service (4)
22. Stream Kafka + Flink (4)
23. ACID vs lock distribuido (3)
24. Batching de escrituras (3)
25. Hot key (lectura) (3)
26. No payloads grandes en cola (3)
27. Pipeline multi-etapa (3)
28. Presigned URL (3)
29. Rollups (granularidad) (3)
30. SSE/WS + Pub/Sub broadcast (3)
31. WebSocket persistente (3)
32. Índice geoespacial (3)
33. Algoritmos rate limiting (2)
34. Backoff + visibility + DLQ (2)
35. Bases especializadas TS vs OLAP (2)
36. Batch Spark (MapReduce) (2)
37. CDC sync índice (2)
38. Chunking / multipart (2)
39. Counter atómico + batching (2)
40. Cursor pagination (2)
41. Durable execution (2)
42. Encryption + signed URLs (2)
43. Full-text Postgres vs ES (2)
44. Hash vs counter (IDs) (2)
45. Item vs Inventory (2)
46. Lua script atómico Redis (2)
47. Normalizar relación (2)
48. Nunca confiar en el cliente (2)
49. OCC con version/CAS (2)
50. OLAP columnar (2)
51. Partición geográfica (2)
52. Precompute + cron warming (2)
53. Push + poll híbrido (2)
54. Push notifications nativas (2)
55. Resumable + trust-verify (2)
56. Stream retention + checkpoint (2)
57. Tumbling vs sliding windows (2)
58. 2PC vs Saga (1)
59. ACK + heartbeats (1)
60. Adaptive bitrate + manifest (1)
61. Adaptive intervals (cliente) (1)
62. Aggregate por shard + merge (1)
63. Aislamiento AZ/región (active-active/passive) (1)
64. Anti-entropy con Merkle trees (1)
65. Anti-sniping / soft close (1)
66. At-least-once + idempotencia (jobs) (1)
67. Cache redundante (hot read) (1)
68. Cache-aside vs write-through vs write-back (1)
69. Cell-based architecture (bulkhead) (1)
70. Comprimir antes de cifrar (1)
71. Config poll vs push (1)
72. Connection pooling (1)
73. Content-defined chunking (1)
74. Continuous batching GPU (1)
75. Count-Min Sketch (1)
76. DAG + orquestador (1)
77. Dedup antes del stream (1)
78. Dedup por contenido (1)
79. Delay queue (timeouts) (1)
80. DNS bottleneck (1)
81. Double-entry ledger inmutable (1)
82. Download incremental (1)
83. Evicción LRU vs LFU (1)
84. Fail-open vs fail-closed (1)
85. Fan-out write vs read (1)
86. Gossip protocol (membership) (1)
87. Grafo sobre KV (1)
88. Hot shard suffix (hot write) (1)
89. Híbrido por-entidad (1)
90. Idempotency key con request hash (1)
91. Impression ID + HMAC (1)
92. Inbox / outbox offline (1)
93. Inyección de contexto (stateless) (1)
94. Jerarquía caché búsqueda (1)
95. Kafka buffer de ingesta (1)
96. KV-cache / PagedAttention (1)
97. Lambda architecture (1)
98. Leader election (1)
99. Max depth (anti-trap) (1)
100. Model routing (1)
101. Modelo time-series (name+labels) (1)
102. Operational Transformation (OT) vs CRDT (1)
103. Outbox pattern (1)
104. Partición user vs chat (1)
105. Per-resource lock + jitter (1)
106. Pre-filtrar antes de caro (1)
107. Presence vía conexión (1)
108. Priority starvation: aging + WFQ (1)
109. Proxy bidding (1)
110. Pull vs Push (métricas) (1)
111. Queue + dynamic scaling (1)
112. Quorum N/R/W (1)
113. Reconciliación periódica (1)
114. Request coalescing (1)
115. Sandbox código no confiable (1)
116. Segmentos + múltiples formatos (1)
117. Sequence numbers (gaps) (1)
118. Server-side redirect (1)
119. Shuffle sharding (1)
120. Single-partition transaction (1)
121. Sloppy quorum + hinted handoff (1)
122. Speculative decoding (1)
123. Status implícito (1)
124. Streaming token-by-token SSE (1)
125. Tiering hot/warm/cold + downsampling (1)
126. Timestamp servidor (NTP) (1)
127. Ubicación rate limiter (1)
128. URLs entre workers no archivos (1)
129. Vector clocks (conflictos) (1)
130. Vector search / ANN (RAG) (1)

---


No quedan pendientes de la lista original. Próximos candidatos si seguís ampliando: distributed message queue (Kafka internals), typeahead/autocomplete, Google Maps (routing), notification system, distributed transactions avanzadas (TrueTime/Spanner).

> Nota de procedencia: los 13 primeros vienen de los breakdowns gratuitos de Hello Interview. Los 7 marcados (Google Docs, Payment, Job Scheduler, Distributed Cache, Metrics, Online Auction, ChatGPT) se compilaron de fuentes públicas porque sus breakdowns en Hello Interview son premium — los conceptos centrales son estándar de la industria, pero no reflejan la opinión específica "good/great/bad solution" de Hello Interview.