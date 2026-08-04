---
title: Real-Time Updates
reading:
  total_words: 3060
  read_words: 3060
  pct: 100
  last_read: 2026-07-02
source: HelloInterview (pattern "Real-time Updates") + Ably, GetStream, websocket.org, AlgoMaster, ByteByteGo, System Design Sandbox
author: Compilado para entrevistas de System Design
created: 2026-07-02
tags:
  - system-design/realtime
  - type/pattern
  - status/permanent
aliases:
  - Real-Time Updates
  - Real-time Updates
updated: 2026-07-02
---

> [!note] Definición
> Real-time Updates es el pattern de System Design que resuelve cómo un **servidor empuja datos a un cliente** sin que este los pida (chat, notificaciones, dashboards en vivo, edición colaborativa). Se descompone en dos "hops": el protocolo cliente↔servidor (polling, long polling, SSE, WebSocket, WebRTC) y la distribución interna servidor↔servidor (pull, consistent hashing, pub/sub).

## The Problem

El modelo HTTP request-response clásico funciona para una cantidad sorprendente de casos: el cliente pide, el servidor responde, la conexión se cierra. El problema aparece cuando es el **servidor** quien necesita empujar datos al cliente sin que este los pida — chat, notificaciones, dashboards en vivo, edición colaborativa, precios de bolsa, ubicación de un driver.

El ejemplo motivador clásico es un editor colaborativo tipo Google Docs: cuando un usuario tipea un caracter, los demás deben verlo en milisegundos. No podés tener a cada usuario haciendo polling cada pocos milisegundos sin destruir la infraestructura.

La frase que sintetiza por qué esto cae en entrevistas: si diseñás un chat y decís "el cliente manda mensajes por HTTP POST y hace polling cada segundo", el problema es evidente en cuanto sacás la cuenta — 60 requests por minuto por usuario, para 10 millones de usuarios son 600 millones de requests por minuto solo para preguntar "¿hay algo nuevo?", y la mayoría devuelven vacío.

**Por qué importa a nivel senior/staff:** muchos de estos problemas se resuelven una sola vez por un equipo especializado, así que muchos candidatos con 10+ años de experiencia nunca cruzaron ese puente. Para mid-level, mostrar que lo entendés es un diferenciador; para staff, cierta familiaridad se da por sentada.

---

## The Solution

La solución se piensa en **dos "hops" (saltos)**:

1. **Primer hop — cliente ↔ servidor:** cómo establecés el canal para que el servidor pueda empujar updates al cliente. Es un problema de protocolo de red.
2. **Segundo hop — fuente → servidor:** cómo el update llega desde donde se origina hasta el servidor específico que tiene abierta la conexión con el cliente correcto.

Separar estos dos hops es la clave para no marearse. El primero se resuelve eligiendo protocolo (polling, long polling, SSE, WebSocket, WebRTC). El segundo eligiendo mecanismo de distribución interna (pull, consistent hashing, pub/sub).

> **Regla de oro que atraviesa todo el tema:** empezá simple y escalá solo cuando lo necesites. El error más penalizado es proponer WebSockets porque el problema menciona "real-time", cuando long polling o SSE alcanzaban. Los WebSockets agregan complejidad seria de conexiones stateful.

---

## Client-Server Connection Protocols (primer hop)

### Networking 101

#### Networking Layers

Las redes están construidas en capas (el modelo OSI), lo que abstrae la complejidad para quienes construimos aplicaciones. Cada capa se apoya en la anterior. Lo que importa para la entrevista:

- **Capa 3 (Red) — IP:** direccionamiento y ruteo entre redes.
- **Capa 4 (Transporte) — TCP / UDP:** TCP es orientado a conexión, confiable, ordenado, con control de flujo; UDP no tiene garantías pero es más liviano y rápido. La mayoría de lo real-time confiable va sobre TCP; UDP aparece en WebRTC para media donde perder algún paquete es aceptable.
- **Capa 7 (Aplicación) — HTTP, WebSocket:** donde viven los protocolos que usás. WebSocket vive en capa 7 pero depende de TCP en capa 4; corre sobre los puertos 80 y 443 (este último con TLS), lo que le permite pasar la mayoría de los firewalls y proxies HTTP.

**Dato de infra clave:** que un protocolo sea stateful (WebSocket, SSE) o stateless (HTTP normal) determina cómo lo balanceás. Conexiones persistentes → load balancer L4 (opera a nivel TCP); request-response → L7 (rutea por contenido HTTP).

#### Request Lifecycle

Cuando pedís una página pasa una cadena: resolución DNS (nombre → IP), handshake TCP (3 vías), handshake TLS si es HTTPS, y recién ahí el intercambio HTTP. Cada uno suma round-trips. El dato para la mesa: un round-trip NY↔Londres tiene un piso de ~80ms solo por la velocidad de la luz en fibra, antes de procesar nada. Eso justifica CDNs y despliegues regionales, y explica por qué abrir conexiones nuevas todo el tiempo (como el polling) es caro.

### Simple Polling: The Baseline

**Cómo funciona:** el cliente pregunta al servidor cada X segundos "¿hay algo nuevo?". Como refrescar el inbox manualmente cada pocos minutos.

**Pros:** trivial de implementar, HTTP puro, funciona con toda la infraestructura existente, sin conexiones persistentes que gestionar, stateless (cualquier servidor atiende cualquier request).

**Contras:** desperdicia recursos (la mayoría de las respuestas son "nada nuevo"), la latencia es igual al intervalo de polling, y a escala genera una avalancha de requests inútiles.

**Cuándo usarlo:** cuando latencia de varios segundos o minutos es aceptable y los updates son poco frecuentes. Es el baseline correcto para arrancar — precios que cambian cada 30s, estado de un job de fondo.

**Frase para la mesa:** "Arranco con polling simple; si el intervalo no me da la latencia que necesito o el volumen de requests vacíos se vuelve un problema, escalo a long polling o SSE."

### Long Polling: The Easy Solution

**Cómo funciona:** el cliente hace un request y el servidor **mantiene la conexión abierta** hasta que tiene algo para responder (o hasta un timeout). Al recibir la respuesta, el cliente reconecta inmediatamente. El servidor solo responde cuando tiene algo significativo, eliminando las respuestas "nada nuevo" del polling común.

**Pros:** latencia mucho mejor que polling, sigue siendo HTTP puro (pasa firewalls y proxies sin problema), compatibilidad amplia. Fue la técnica que hizo funcionar los primeros Facebook Messenger y Gmail chat antes de que WebSocket se generalizara.

**Contras:** cada request tiene timeout, así que el cliente debe reconectar periódicamente. El **problema serio a nivel de UX** es en ráfagas: si llegan 5 mensajes seguidos, el primero llega al instante pero los 2-5 se encolan detrás de los ciclos de reconexión — el usuario los ve aparecer de a batches en lugar de uno por uno. El ordering no se garantiza si el mismo cliente abre múltiples conexiones. Es más intensivo en recursos del servidor que WebSocket porque abre/cierra conexiones constantemente.

**Cuándo usarlo:** entornos legacy, firewalls que bloquean conexiones persistentes, o cuando necesitás baja latencia con cambios mínimos de infraestructura. Frameworks como SignalR lo usan como fallback automático cuando WebSocket no está disponible.

**Trampa/frase:** "Long polling es un workaround, no una decisión de diseño." Sirve como puente pero no lo elegirías de entrada para un sistema nuevo con updates de alta frecuencia.

### Server-Sent Events (SSE): The Efficient One-Way Street

**Cómo funciona:** un canal **unidireccional** persistente servidor→cliente sobre HTTP. El cliente abre una request HTTP con la API `EventSource`, y el servidor empuja eventos por ese stream que queda abierto. El cliente **no puede** mandar datos adicionales por la misma conexión SSE.

**Pros:** más simple que WebSocket, funciona sobre infraestructura HTTP estándar, **reconexión automática nativa** (el browser reconecta solo tras errores de red), soporta IDs de evento para resumir donde quedó. Ideal para el patrón "feed": el servidor empuja, el cliente consume.

**Contras:** unidireccional (si el cliente necesita mandar datos seguido, no sirve). En HTTP/1.1 hay un límite de conexiones concurrentes por dominio (~6), aunque con HTTP/2 se multiplexan y deja de ser problema. Ese límite de ~6 era un problema real solo bajo HTTP/1.1: con HTTP/2 (ya estándar de facto en 2026), las conexiones SSE se multiplexan sobre una sola conexión TCP, y es justamente esa mejora la que consolidó a SSE como transporte por defecto para streaming de tokens de LLM en la mayoría de los SDKs de IA. Es una conexión stateful, así que no va detrás de un load balancer normal sin más. — [Ably: WebSockets vs SSE 2026](https://ably.com/blog/websockets-vs-sse), [germano.dev: SSE alternative to WebSockets](https://germano.dev/sse-websockets/)

**Cuándo usarlo:** notificaciones, feeds, dashboards en vivo, precios de bolsa, scores deportivos, streaming de tokens de un LLM (ChatGPT usa SSE), el precio actual de una subasta, el mapa de asientos de Ticketmaster. La regla: **si es server→cliente y no necesitás bidireccionalidad, SSE es el default correcto** — cubre la mayoría de los casos de notificación con complejidad mínima.

**Frase para la mesa:** "Como el flujo es solo del servidor al cliente, uso SSE en vez de WebSocket para no cargar con la complejidad de una conexión full-duplex que no necesito."

### Websockets: The Full-Duplex Champion

**Cómo funciona:** una conexión TCP persistente **bidireccional** (full-duplex). Arranca como un request HTTP normal y hace un "upgrade" a WebSocket. Una vez establecida, tanto cliente como servidor mandan mensajes en cualquier momento con muy baja latencia.

**Pros:** la latencia más baja de todas las opciones, bidireccional real, eficiente para updates de alta frecuencia (ambos eventos llegan con latencia de una sola transmisión porque la conexión ya está abierta), footprint liviano en el servidor una vez establecida, pasa la mayoría de los firewalls (puertos 80/443).

**Contras:** conexiones **stateful** — este es el costo central. No las tirás detrás de un load balancer estándar; necesitás sticky sessions o estado externalizado. El error handling es más complejo (las fallas de WebSocket suelen ser silenciosas y requieren detección explícita con heartbeats). Algunos proxies interfieren. Escalar horizontalmente es no-trivial. WebSocket no maneja reconexión nativamente — la tenés que implementar vos.

**Cuándo usarlo:** cuando la bidireccionalidad de baja latencia es un requisito **central**, no un lujo — chat, edición colaborativa, juegos multiplayer, plataformas de trading, cursores en vivo (Figma actualiza posiciones de cursor muchas veces por segundo, donde long polling se vería a los saltos).

**Trampa nombrada explícitamente:** "un error común es proponer WebSockets cuando HTTP con long polling o SSE andaría bien. Los WebSockets agregan complejidad significativa para mantener conexiones stateful a escala. Solo los usás cuando genuinamente necesitás comunicación bidireccional real-time, no porque el problema menciona 'real-time'."

**Números concretos de capacidad de un servidor:** con tuning de OS y ~16 GB RAM, un solo nodo aguanta 500K+ conexiones idle; un benchmark real reportó 240K conexiones concurrentes con latencia sub-50ms. La restricción real no suele ser la cantidad de conexiones sino el throughput de mensajes: a partir de ~100K conexiones activas o ~50K mensajes/segundo por nodo, el escalado horizontal con load balancer deja de ser opcional. Cada conexión pesa ~20-50 KB en memoria (file descriptors + buffers). Además, a diferencia de SSE, que se beneficia de la multiplexación de HTTP/2, cada WebSocket sigue siendo su propia conexión TCP dedicada — otro argumento concreto (además del de complejidad stateful) para no usar WebSocket "porque sí" cuando el flujo es unidireccional. — [WebSocket.org: WebSockets at Scale](https://websocket.org/guides/websockets-at-scale/), [WebSocket.org: Connection Limits](https://websocket.org/guides/connection-limits/)

### WebRTC: The Peer-to-Peer Solution

**Cómo funciona:** conexión **directa entre clientes (peer-to-peer)**, típicamente sobre UDP para media. El servidor solo interviene en el **signaling** inicial (intercambio de SDP e ICE candidates) y después se sale del camino del media. Cuatro primitivas del browser: `getUserMedia`, `RTCPeerConnection`, `MediaStream`, `RTCDataChannel`.

**El detalle que hay que saber — NAT traversal:** WebRTC no es "magia peer-to-peer". La mayoría de los dispositivos están detrás de NAT/firewalls, así que necesita:
- **STUN:** servidor liviano que le dice al peer cuál es su IP pública. Barato, lo usa casi toda conexión.
- **TURN:** servidor que **relaya** el media cuando la conexión directa falla. Caro en ancho de banda, usado como fallback en ~10-20% de las conexiones (hasta ~18-35% en redes celulares). Regla de producción: **siempre desplegá TURN**.
- **ICE:** el framework que orquesta y prueba los candidatos hasta encontrar un camino que funcione.
- **Signaling server:** un canal chico (normalmente WebSocket) que transporta el handshake. No lo define WebRTC; lo implementás vos.

**La trampa de escala:** P2P puro escala O(N²) — cada peer sube su stream a cada otro peer. Funciona hasta ~4 participantes. Arriba de eso necesitás un **SFU** (Selective Forwarding Unit: cada peer sube una vez, el servidor reenvía) o un **MCU** (mezcla los streams en el servidor).

**Cuándo usarlo:** videollamadas, audio, compartir pantalla, transferencia de datos P2P de baja latencia donde querés evitar el hop por el servidor. Para llamadas grupales, SFU.

### Overview (tabla comparativa)

| Protocolo | Dirección | Latencia | Complejidad | Stateful | Cuándo |
|---|---|---|---|---|---|
| Simple Polling | Cliente pregunta | Alta (= intervalo) | Muy baja | No | Updates infrecuentes, baseline |
| Long Polling | Cliente pregunta, server retiene | Media-baja | Baja | Semi | Legacy, fallback, sin WebSocket |
| SSE | Server → cliente | Baja | Baja-media | Sí | Feeds, notificaciones, precios, un solo sentido |
| WebSocket | Bidireccional | Muy baja | Alta | Sí | Chat, colaboración, juegos, trading |
| WebRTC | Peer ↔ peer | Muy baja | Muy alta | Sí | Video/audio/datos P2P |

**Progresión mental para la mesa:** polling → long polling → SSE → WebSocket, subiendo un escalón solo cuando el anterior no alcanza. WebRTC es un camino aparte, solo para media P2P.

**El sucesor que viene: WebTransport (HTTP/3 + QUIC).** Da streams bidireccionales multiplexados como WebSocket pero sin head-of-line blocking cuando se pierde un paquete, más datagramas no confiables (útiles para juegos/media). A julio 2026 funciona en Chrome y Edge, está parcialmente en Firefox, y **no está implementado en Safari** — por eso todavía no es una opción de producción cross-browser y la elección real en la entrevista sigue siendo SSE vs WebSocket. — [RxDB: WebSockets vs SSE vs WebTransport](https://rxdb.info/articles/websockets-sse-polling-webrtc-webtransport.html), [WebSocket.org: WebSocket vs SSE](https://websocket.org/comparisons/sse/)

---

## Server-Side Push/Pull (segundo hop)

Una vez que el cliente tiene una conexión abierta con **un** servidor específico, ¿cómo llega el update desde donde se origina hasta ese servidor exacto? Porque en producción tenés cientos de servidores y el usuario destino está conectado a uno solo.

### Pulling with Simple Polling

**Cómo funciona:** los servidores que mantienen las conexiones consultan periódicamente la fuente de datos (base, otro servicio) para ver si hay novedades que empujar.

**Pros:** simple, sin infraestructura extra de mensajería.

**Contras:** no escala bien; agrega latencia (igual al intervalo de pull); carga innecesaria sobre la fuente. Es el baseline del segundo hop, análogo al polling del primero.

**Cuándo:** volúmenes bajos, o como primer paso antes de introducir push.

### Pushing via Consistent Hashes

**Cómo funciona:** enrutás cada usuario/recurso a un servidor específico usando **consistent hashing**, de modo que sabés determinísticamente qué servidor tiene la conexión de qué cliente. Cuando ocurre un update, lo mandás directo a ese servidor. Se mantiene un registro (session directory) de qué usuario está en qué nodo.

**Cuándo elegirlo sobre pub/sub:** cuando el procesamiento es **pesado** o necesitás **afinidad/localidad** — mantener usuarios relacionados en la misma máquina. El caso arquetípico es la edición colaborativa (Google Docs): todos los que editan el mismo documento se conectan al mismo servidor (sticky via consistent hashing sobre el document ID), que mantiene el documento en memoria y el operation log. Poner esto en pub/sub sería absurdo porque el estado vive en el servidor.

**Dato:** con modulo hashing, agregar 1 servidor a un cluster de 10 mueve ~90% de las claves; con consistent hashing movés ~10%. En la mesa rara vez necesitás explicar cómo funciona — alcanza con "uso consistent hashing para rutear la conexión al servidor correcto".

**Ejemplo real:** Uber usa Ringpop, su framework de consistent hashing, con las conexiones sharded por UUID de usuario.

### Pushing via Pub/Sub

**Cómo funciona:** un broker (Redis Pub/Sub, Kafka, NATS) donde los servidores se suscriben a topics/canales. Cuando ocurre un evento, se publica, y todos los servidores suscritos lo reciben y lo empujan a sus clientes conectados localmente. Es el **backplane** clásico para escalar WebSocket horizontalmente: Servidor A recibe el mensaje de Alice, publica en un canal Redis, Servidor B (donde está Bob) lo consume y lo empuja.

**Cuándo elegirlo:** cuando querés **desacoplar** al publisher del subscriber, y los updates son **livianos** con fan-out. El publisher no necesita saber qué servidor tiene qué cliente — solo pone el mensaje en un topic. Es el default para chat, notificaciones, live comments.

**Ejemplo real:** WhatsApp usa pub/sub para el fan-out de notificaciones. Es el mecanismo de segundo hop más común y "pasaría la entrevista" en la mayoría de los casos.

**Datos importantes sobre Redis Pub/Sub (para deep dives):**
- Es entrega **"at most once"**: si un subscriber está offline cuando se publica, **pierde** el mensaje. No es durable.
- Mito a desarmar: **no** usa una conexión por canal. La conexión es por nodo — un subscriber mantiene una conexión y recibe todos sus canales por ahí. Millones de canales no significan millones de conexiones.
- Desde Redis 7, el **sharded Pub/Sub** (SPUBLISH/SSUBSCRIBE) rutea cada canal al shard que tiene su slot, y la capacidad escala con el cluster.
- Si necesitás durabilidad, replay o entrega a consumidores que se reconectan: **Redis Streams**, o parear Pub/Sub con una cola (SNS→SQS, Kafka) o el patrón outbox.

**Regla de decisión Pub/Sub vs Consistent Hashing (la decisión central del segundo hop):**

| | Pub/Sub | Consistent Hashing |
|---|---|---|
| Naturaleza del trabajo | Liviano, fan-out | Pesado, con estado en memoria |
| Acoplamiento | Desacopla publisher/subscriber | El publisher sabe a qué servidor ir |
| Caso típico | Chat, notificaciones, live comments | Edición colaborativa, matching |
| Ejemplo | WhatsApp | Google Docs, Uber |

---

## When to Use in Interviews

### Common Interview Scenarios

Real-time updates aparece (a veces como requisito principal, a veces como deep-dive) en:
- **Chat / mensajería** (WhatsApp, Slack): WebSocket + pub/sub.
- **Edición colaborativa** (Google Docs, Figma): WebSocket/SSE + consistent hashing + OT/CRDT.
- **Live comments / reacciones** (FB Live, YouTube Live): SSE + pub/sub particionado.
- **Ubicación en vivo** (Uber): consistent hashing para push a drivers.
- **Precios / market data** (Robinhood): SSE + pub/sub, con coalescing de ticks.
- **Notificaciones / feeds en vivo** (Twitter, Instagram): fan-out + pub/sub.
- **Dashboards y monitoreo en vivo:** SSE.
- **Colas virtuales** (Ticketmaster): SSE/WebSocket para posición en la fila.

### When NOT to Use

- Cuando polling simple alcanza (updates poco frecuentes, latencia de segundos aceptable). No sobre-diseñes.
- Cuando la respuesta es síncrona: si el request se completa y devolvés la respuesta, no necesitás nada real-time.
- Cuando la consistencia fuerte importa más que la inmediatez y podés tolerar refrescar.
- **La trampa:** decir "uso WebSockets" apenas escuchás "real-time". El interviewer quiere que expliques por qué el polling falla a escala, cómo manejás el ruteo cross-server, qué pasa cuando el usuario se desconecta, y cómo trackeás presencia — no solo el nombre del protocolo.

---

## Common Deep Dives

### "¿Cómo manejás fallos de conexión y reconexión?"

Este es el deep-dive más frecuente y el que más separa a los candidatos.

**Detección de conexiones muertas — heartbeats:** WebSocket tiene frames de control nativos Ping/Pong. Mandás un ping cada ~30 segundos; si no llega el pong, la conexión está muerta (útil porque las fallas suelen ser silenciosas a través de NAT/proxies). SSE se apoya en la reconexión automática del browser.

**Reconexión del cliente — exponential backoff con jitter:** al caerse, el cliente no reconecta al instante ni a intervalo fijo (eso genera "reconnect storms" que tumban el servidor). Usás backoff exponencial + jitter aleatorio para dispersar los reintentos.

**No perder mensajes — sequence numbers + buffering:** cada mensaje lleva un ID monotónicamente creciente (por usuario o por conversación) y se persiste **antes** de cualquier push en tiempo real. Al reconectar, el cliente manda su último ID conocido y el servidor le devuelve todos los mensajes faltantes en orden. En Google Docs, el cliente encola ediciones localmente (IndexedDB) y sincroniza contra el último state ID al volver.

**Externalizar el estado de conexión:** en vez de pegar el cliente siempre al mismo servidor (sticky sessions, frágiles ante caídas), guardás el estado de sesión en un store externo (Redis) para que **cualquier** servidor pueda retomar a un cliente que reconecta. Esto habilita failover graceful y rebalanceo. Trade-off: sticky sessions son simples pero si el servidor "sticky" se cae hay riesgo de pérdida de datos; estado externalizado es más resiliente pero agrega latencia y complejidad.

**Presencia (online/offline):** Redis con TTL — el cliente refresca cada X segundos, y si no refresca en el tiempo del TTL se marca offline automáticamente.

**Ejemplo real (Uber RAMEN):** reportaban ACKs solo cada 30s o al reconectar, lo que demora los acknowledgements y hace difícil distinguir pérdidas genuinas de fallos de ACK — justo el trade-off (frecuencia de ACK vs overhead) que conviene discutir.

### "¿Qué pasa cuando un solo usuario tiene millones de followers que necesitan el mismo update?" (el celebrity problem / fan-out)

**El problema:** un usuario con 50-100M followers postea. Con **fan-out on write** puro (empujar a la timeline de cada follower), un solo post dispara decenas de millones de escrituras; el pipeline se satura por minutos y los posts de otros usuarios se atrasan. Es un hot partition.

**Los dos modelos base:**
- **Fan-out on write (push):** al postear, escribís el post_id en la timeline (sorted set de Redis) de cada follower. Lectura rápida (O(1), precomputada), escritura cara. Bueno para usuarios normales — crítico porque las lecturas superan a las escrituras ~100:1.
- **Fan-out on read (pull):** al postear solo guardás el post; la timeline se computa al leer, haciendo merge de los posts de todos los seguidos. Escritura barata, lectura cara (K-way merge de listas ordenadas).

**La solución: híbrido push-pull.** Definís un umbral de "celebrity" (típicamente 10K-100K followers):
- Usuarios normales → push (fan-out on write).
- Celebrities → **no** se hace fan-out; sus posts se guardan una vez. Cuando un follower carga su feed, el sistema chequea si sigue a algún celebrity y hace pull de sus posts recientes, mergeándolos con su timeline precomputada.

El tweet del celebrity se cachea **una vez** (no replicado por follower), así servir a 100M followers cuesta una lectura de cache por request contra una entrada compartida, no 100M escrituras. Así lo resuelven Twitter, Instagram y Facebook.

**Herramientas complementarias (mismo toolbox que el hot-key de consistent hashing):**
- Read replicas para las claves populares.
- Key-space salting: sufijo aleatorio (`taylor-swift-{0..9}`) para que la clave caliente se disperse en varios nodos.
- Adaptive rebalancing: mover rangos de claves de nodos sobrecargados.
- Snowflake IDs para los tweets: timestamp + datacenter + secuencia, de modo que ordenar por ID equivale a ordenar por tiempo sin contador centralizado.

**La distinción fina que suma:** virtual nodes previenen el desbalance **estructural** (distribución despareja de claves); replication y salting previenen el desbalance de **tráfico** (carga despareja sobre claves iguales).

**La trampa:** comprometerte con un solo modelo sin caveat. Apenas lo hacés, el interviewer te pregunta por el usuario de 50M followers y quedás pintado en la esquina. Mejor: presentá los dos modelos, sus trade-offs, y proponé el híbrido, mostrando que entendés cuándo cada uno se rompe.

### "¿Cómo mantenés el ordering de mensajes entre servidores distribuidos?"

**El problema:** con múltiples servidores y conexiones, los mensajes pueden llegar en distinto orden; algunos se retrasan. Garantizar el orden a nivel de transporte a través de reconexiones requeriría coupling fuerte y amplifica las fallas.

**La solución — sequence numbers a nivel de aplicación:** cada mensaje lleva un ID monotónicamente creciente **por conversación/recurso** (no global, que sería un cuello de botella), persistido antes del push. El cliente reordena por ID y detecta huecos (si le falta el ID N, pide reenvío).

**Particionar por key:** hacés que todo lo de un mismo recurso (una conversación, un documento) pase por la **misma** partición/servidor, de modo que el orden se preserva dentro de esa partición. En Kafka, particionás por gateway_id o por conversation_id: cada partición la consume exactamente un nodo, y el orden dentro de la partición está garantizado.

**Trade-off a nombrar:** ordering estricto global es caro y rara vez necesario. Ordering **por conversación** casi siempre alcanza y es mucho más barato. Agregar timestamps/sequence numbers tiene overhead, que a escala hay que considerar.

---

## Conclusion

El tema se domina con tres decisiones encadenadas:

1. **Protocolo (primer hop):** empezá en polling y subí a long polling → SSE → WebSocket solo cuando el anterior no da. WebRTC solo para media P2P. La pregunta clave: ¿necesito bidireccionalidad? Si no, SSE.
2. **Distribución (segundo hop):** pub/sub para fan-out liviano y desacoplado (WhatsApp); consistent hashing para procesamiento pesado con estado en memoria (Google Docs).
3. **Robustez (deep dives):** heartbeats + backoff con jitter para reconexión; sequence numbers + buffering para no perder ni desordenar mensajes; híbrido push/pull para el celebrity problem.

La actitud que buscan a nivel senior/staff: no aplicar la técnica más sofisticada, sino justificar el default y el umbral exacto en que conviene desviarse. La frase que resume el tema: "real-time es un espectro" — cada feature tiene requisitos distintos y el arte está en elegir el punto justo, no el más impresionante.

## References

- HelloInterview — pattern "Real-time Updates" (fuente base del documento)
- [WebSocket.org: WebSockets at Scale](https://websocket.org/guides/websockets-at-scale/)
- [WebSocket.org: Connection Limits](https://websocket.org/guides/connection-limits/)
- [WebSocket.org: WebSocket vs SSE](https://websocket.org/comparisons/sse/)
- [Ably: WebSockets vs Server-Sent Events 2026](https://ably.com/blog/websockets-vs-sse)
- [germano.dev: Server-Sent Events, the alternative to WebSockets](https://germano.dev/sse-websockets/)
- [RxDB: WebSockets vs SSE vs Long-Polling vs WebRTC vs WebTransport](https://rxdb.info/articles/websockets-sse-polling-webrtc-webtransport.html)

## Related

- [[Server-Sent Events]]
- [[Bidirectional Streaming]]
- [[Pub-Sub]]
- [[Webhooks]]
- [[Request-Response]]
