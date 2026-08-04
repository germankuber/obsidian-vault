---
title: Networking Essentials
source: https://www.hellointerview.com/learn/system-design/core-concepts/networking-essentials
author: Hello Interview
created: 2026-06-16
tags:
  - system-design/patterns
  - type/concept
  - status/permanent
aliases:
  - Networking Essentials
  - Networking
  - Network Protocols
reading:
  total_words: 3163
  read_words: 0
  pct: 0
  last_read: ""
updated: 2026-06-16
---

# Networking Essentials

> [!note] Tesis operativa
> Toda la red es una **pila de abstracciones que se compran y se pagan**: cada capa y cada protocolo existe porque resuelve un problema que el de abajo no podía, y cobra por hacerlo (overhead, latencia, estado, complejidad). Entender networking no es memorizar definiciones: es saber decir, en cada bifurcación, *"elijo X sobre Y porque tolero (o no) este costo a cambio de esta garantía"*. Default de entrevista: **TCP, REST, HTTP**.

## Marco mental (leé esto primero)

Tres ideas sostienen todo lo demás; sin ellas el resto no se entiende:

1. **La red NO es confiable.** Pierde, reordena y duplica paquetes, y los enlaces se caen. *Todo* lo que sigue —los protocolos de transporte y los patrones de resiliencia ([[Timeout]], [[Retry with Backoff]], [[Circuit Breaker]])— es una respuesta a este hecho.
2. **Subimos por capas (L3 → L4 → L7) porque cada capa le da a la de arriba una abstracción más simple.** **IP** mueve paquetes; encima **TCP** da un flujo confiable; encima **HTTP** da request-response con verbos. Cada nivel esconde la suciedad del de abajo.
3. **Una conexión es estado caro que ambos extremos mantienen.** Abrirla cuesta round-trips de handshake. *Por eso* casi toda optimización de red —keep-alive, multiplexing, conexiones persistentes— consiste en **no volver a pagar ese estado**.

![[osi-layers.svg]]
*OSI model layers*

## La pila como historia: IP → TCP/UDP → QUIC

La forma honesta de entender el transporte es como una cadena de "esto faltaba, así que apareció lo siguiente".

**L3 — IP** entrega paquetes *best-effort* entre direcciones IP: hace lo que puede, pero **puede perder, reordenar o duplicar y no te avisa**. En la práctica, **DHCP** te asigna una IP local cuando te conectás a una red, mientras que un **RIR** (Regional Internet Registry) reparte los bloques globales a las organizaciones —por eso `17.x.x.x` es de Apple—. El problema es que la aplicación normalmente quiere fiabilidad y orden, cosa que IP no da.

**L4 — TCP** se construye *encima* de IP y agrega justo lo que IP no daba: entrega **confiable** (retransmite lo perdido), **ordenada**, con **control de flujo** y **control de congestión**. El precio de esas garantías:
- Un **handshake de 3 vías** (SYN → SYN-ACK → ACK) antes de mandar nada útil: round-trips puros de setup.
- **Estado** mantenido en *ambos* extremos.
- Una cabecera de **20-60 bytes** (contra 8 de UDP).
- En reverso: todas esas garantías **cuestan latencia**.

**UDP** es el otro extremo: conserva la velocidad cruda de IP y **tira las garantías**. Sin handshake, sin estado, sin orden, cabecera de **8 bytes**.

Por eso elegir TCP vs UDP se reduce a **una sola pregunta**: *¿tolero perder datos a cambio de menos latencia?*
- **No** (web, APIs, file transfer, email) → **TCP**, el default.
- **Sí, porque un dato viejo ya no sirve** (gaming en tiempo real, voz/video en vivo) → **UDP**, pero justificándolo.

### Comparación factor por factor — TCP vs UDP

| Factor | TCP | UDP |
|---|---|---|
| Conexión | Orientado a conexión | Sin conexión |
| Fiabilidad | Garantizada (retransmite) | No garantizada |
| Orden | Ordena los paquetes | Sin orden |
| Control de flujo | Sí | No |
| Control de congestión | Sí | No |
| Cabecera | 20-60 bytes | 8 bytes |
| Velocidad | Más lento | Más rápido |
| Casos de uso | Web, email, file transfer | Streaming, gaming, VoIP, DNS |

**QUIC** es, en una frase, *"un mejor TCP"*: quiere la **velocidad de UDP con las garantías de TCP**. Se construye sobre **UDP** pero **reimplementa fiabilidad y orden** en espacio de aplicación, con un handshake más rápido que **fusiona transporte y cifrado** en menos round-trips, y elimina el *head-of-line blocking* de TCP. Es la base de **HTTP/3**.

> [!question] 🎯 ¿Por qué QUIC corre sobre UDP si UDP no da garantías y QUIC sí las quiere?
> Porque UDP es el **sobre crudo, una pizarra en blanco**: no impone ningún comportamiento que haya que desarmar. QUIC reimplementa en *espacio de aplicación* exactamente lo que TCP hacía en el kernel (fiabilidad, orden), pero más rápido y sin las décadas de rigidez de TCP. Construir sobre TCP lo habría atado a sus limitaciones; construir sobre UDP le dio libertad total.

## El web request como espina dorsal

Un simple GET es la mejor forma de ver el costo del estado en acción, paso a paso:

1. **DNS** resuelve el dominio a una IP (su propia ida y vuelta).
2. **Handshake TCP**: SYN → SYN-ACK → ACK.
3. **GET** sobre la conexión ya abierta.
4. **Response** del servidor.
5. **Cierre** de 4 pasos: FIN → ACK → FIN → ACK.

![[web-request-flow.svg]]
*Simple HTTP request — full flow*

Lo importante: **antes de mandar un solo byte útil ya pagaste DNS + handshake**, y al final pagás un cierre de 4 pasos. Abrir una conexión nueva por *cada* imagen de una página repagaría todo eso una y otra vez. *Por eso* existen:
- **keep-alive**: reusar la misma conexión TCP entre múltiples requests.
- **HTTP/2 multiplexing**: mandar requests concurrentes sobre *una sola* conexión.

Todo el ahorro es el mismo principio: **no re-pagar el setup**.

## Paradigmas de API: una progresión

Cada paradigma de API resuelve un dolor del anterior.

### HTTP — el piso request-response

HTTP da un modelo **request-response stateless** con **verbos** y **status codes**. Los verbos son GET / POST / PUT / PATCH / DELETE; de ellos, **GET y DELETE son idempotentes** y **GET no lleva body** (ver [[HTTP Methods]]). Los status codes ([[HTTP Status Codes]]) comunican el resultado:
- **2xx** éxito: `200 OK`, `201 Created`.
- **3xx** redirecciones: `301`, `302`.
- **4xx** errores del cliente: `401` (no autenticado), `403` (sin permiso), `404` (no existe), `429` (rate-limited → ver [[Rate Limiting]]).
- **5xx** errores del servidor: `500`, `502`.

Los **headers** dan flexibilidad: por ejemplo el cliente manda `Accept-Encoding` y el servidor responde con `Content-Encoding` (gzip/br) para comprimir. **HTTPS** es simplemente HTTP sobre **TLS/SSL**.

> [!warning] Encrypted ≠ trusted
> TLS cifra el canal, pero no valida la intención de quien lo usa. Siempre validá el **body** del request: si confiás en un user ID que viene del cliente sin chequear, abrís la puerta a que un usuario actúe en nombre de otro.

### REST — organizar HTTP alrededor de recursos

REST es el **default** de las APIs públicas: estructura HTTP en torno a **recursos + verbos** (ver [[REST API]] y [[REST Constraints]]).

```http
GET    /users/123          # leer un recurso
PUT    /users/123          # reemplazar un recurso
POST   /users              # crear un recurso
GET    /users/123/orders   # recursos anidados
```

La clave es **modelar operaciones como recursos**: en vez de un RPC `updateUser`, hacés `PUT /users/{id}`; en vez de `startGame`, hacés `PATCH /games {status: "started"}`. ElasticSearch, por ejemplo, expone toda su API como REST. Alrededor del diseño de una API REST viven decisiones vecinas como [[Pagination]] (cómo devolver listas grandes), [[API Versioning]] (cómo evolucionar sin romper clientes) y poner un [[API Gateway]] al frente.

Pero REST tiene dos dolores cuando los recursos no calzan con lo que la pantalla necesita:
- **Under-fetching**: un recurso no trae todo, así que el cliente hace **muchos round-trips** para componer una vista.
- **Over-fetching**: el recurso trae **campos de más** que el cliente no usa, desperdiciando ancho de banda.

![[under-fetching.svg]]
*Under-fetching — multiple round-trips to assemble one view*

![[over-fetching.svg]]
*Over-fetching — fields the client never uses*

### GraphQL — el cliente pide la forma exacta

**GraphQL** (Facebook, 2015) cura ambos dolores: un **único endpoint** donde el cliente describe en una sola **query** la forma exacta de los datos que quiere.

```graphql
query {
  user(id: "123") {
    name
    orders {
      total
      items { title }
    }
  }
}
```

El costo es **complejidad**: el caching se vuelve difícil (ya no hay URLs cacheables por recurso), hay que escribir **resolvers**, y un cliente puede mandar **queries caras** que peguen duro al backend. Para entrevistas suele ser terreno *murky* —úsalo solo cuando el under/over-fetching es un dolor real—.

### gRPC — eficiencia en el cable

El tercer escalón ataca otra ineficiencia: **JSON sobre HTTP es texto verboso**. **gRPC** usa **HTTP/2 + Protocol Buffers** (un formato binario). El ahorro es concreto: un objeto que pesa **~40 bytes en JSON** pesa **~15 bytes en protobuf**, dando hasta **~10x más throughput**.

Mirá el mismo objeto en ambos formatos:

```json
{"id": 123, "name": "john doe"}
```

se codifica en protobuf como:

```
0A 03 31 32 33 12 08 6A 6F 68 6E 20 64 6F 65
```

Son **tags + longitudes + bytes crudos**: no hay comillas, ni llaves, ni nombres de campo repetidos. El reverso es que **no es human-readable** y **necesita el schema `.proto`** para decodificarse. *Por eso* gRPC vive **service-to-service, interno, no público**: el patrón típico es **REST hacia afuera + gRPC hacia adentro**. Aporta además streaming, deadlines y client-side load balancing.

![[grpc-internal-rest-external.svg]]
*gRPC internal + REST external*

> [!warning] Premature optimization
> No metas gRPC por defecto. Si no tenés un cuello de botella real de throughput entre servicios, la complejidad del schema y la pérdida de legibilidad no se pagan sola.

```text
¿Qué API elijo?
├── ¿Es pública?  ───────────────────────────────► REST
├── ¿Under/over-fetching grave + formas a medida? ► GraphQL
└── ¿Interno, service-to-service, throughput?     ► gRPC
```

## Realtime: la escalera (server-push → bidireccional → P2P)

HTTP es request-response: **el servidor no puede iniciar** la comunicación. Para que el servidor empuje datos hay una escalera de soluciones, cada peldaño más potente y más caro. **Regla: no agarres un peldaño más alto del que tu caso necesita.**

### SSE — Server-Sent Events

El servidor empuja **muchos mensajes sobre UNA response HTTP que queda abierta**, mandándolos en **chunks** (`data: {...}` por línea) en vez de un blob único al final (ver [[Server-Sent Events]]). El cliente usa `EventSource`, que **reconecta solo** y manda el *last message ID*, así el servidor le reenvía lo que se perdió. Es **unidireccional** (solo server → cliente).
- **Usos**: notificaciones, feeds, precio de una subasta en vivo, tokens de un LLM a medida que se generan.

> [!warning] Conexiones largas frágiles
> Los [[Load Balancing|balanceadores]] y proxies suelen **cerrar conexiones largas**, y las redes malas **batchean** los chunks, rompiendo el efecto "tiempo real".

### WebSockets — canal bidireccional persistente

Un canal **bidireccional y persistente** (ver [[Bidirectional Streaming|WebSockets]]). Arranca como una conexión HTTP normal y se *promueve* con un **HTTP Upgrade**. Los 4 pasos:

1. El cliente manda **1 request con `Upgrade: websocket`**.
2. El servidor responde **`101 Switching Protocols`**.
3. Queda abierto un **canal WS full-duplex**.
4. Ambos extremos mandan mensajes hasta que **alguno cierra**.

![[websocket-api.svg]]
*WebSocket API example*

WebSocket **no dicta el protocolo de aplicación** (lo común es mandar JSON por encima).
- **Usos**: chat, edición colaborativa, juegos. Google Docs usa WebSockets.

> [!warning] La infra tiene que soportarlo
> Firewalls, proxies y [[Load Balancing|LBs]] deben entender WebSockets; si no, la conexión persistente se cae.

### WebRTC — peer-to-peer

Los clientes hablan **directo entre sí**, sin pasar todo el tráfico por tu servidor. Es el **único protocolo de aplicación que corre sobre UDP**. Los 4 pasos:

1. **Signaling**: los peers intercambian metadata a través de un **servidor de señalización** —solo para presentarse, no para el tráfico real—.
2. **STUN** (Session Traversal Utilities for NAT): cada peer **descubre su IP pública** para poder ser alcanzado a través del **NAT**.
3. **TURN** (Traversal Using Relays around NAT): si el NAT es muy restrictivo y la conexión directa falla, TURN **retransmite el tráfico** por un servidor intermedio.
4. Los peers **intercambian datos directamente**.

![[webrtc-setup.svg]]
*WebRTC setup*

**STUN y TURN existen ambos por culpa del NAT**: STUN ayuda a *atravesarlo*, y TURN se *rinde* y hace de relay cuando atravesarlo es imposible. Para apps colaborativas, los **CRDTs** son una alternativa para fusionar ediciones concurrentes sin conflicto.
- **Usos**: video/audio calling, conferencing. Es **de nicho y un dolor de cabeza** de operar.

```text
¿El servidor necesita empujar datos?
├── No ─────────────────────────────────► HTTP / REST
├── Sí, solo server → cliente ──────────► SSE
├── Sí, bidireccional persistente ──────► WebSockets
└── Cliente ↔ cliente directo ─────────► WebRTC
```

## Load balancing

Hacer **horizontal scaling** ([[Horizontal Scaling]]) —agregar más servidores— crea inmediatamente la pregunta *"¿a cuál de N le mando este request?"*. El **load balancing** ([[Load Balancing]]) la responde. (La alternativa, **vertical scaling** ([[Vertical Scaling]]) —una máquina más grande— el autor la prefiere por simplicidad, pero las entrevistas favorecen horizontal.)

![[vertical-vs-horizontal-scaling.svg]]
*Vertical vs horizontal scaling*

### Primera bifurcación: ¿quién decide?

**Client-side LB** — el **cliente elige** a qué backend pegar, **sin un salto extra**. El cliente conoce la lista de servidores y reparte él mismo. Ejemplos: **Redis Cluster** (los nodos se descubren por *gossip*; si pegaste al equivocado, te devuelve un redirect `MOVED`) y la **rotación DNS** (un dominio resuelve a varias IPs).
- **Debilidad**: los updates de topología son **lentos** (atados al TTL del DNS), y el DNS puede ser un **SPOF** (lo mitigás con dos LBs + rotación).
- Viene **integrado en gRPC**. Detrás suele haber un registry de [[Service Discovery]].
- **Usar** para un set chico y controlado, o un set grande que **tolera updates lentos**.

**Dedicated LB** — una **pieza intermedia** dedicada. Cuesta un **salto extra**, pero da **updates rápidos** y control fino. Es vecino conceptual de un [[Reverse Proxy]] o un [[API Gateway]].

### Segunda bifurcación (dentro del dedicado): ¿necesita ver el contenido?

**L4 LB** (capa de transporte): **no lee el contenido**, balancea a nivel **TCP/conexión** y mantiene una **conexión persistente** a un backend elegido. Es **rápido**. La postura del artículo es que es bueno para **WebSockets** (conexiones de larga vida).

![[l4-load-balancer.svg]]
*L4 load balancer*

> [!warning] Punto disputado
> Los comentarios del artículo cuestionan que L4 sea obligatorio para WebSockets: **Envoy** y **AWS ALB** manejan WS perfectamente en **L7**. No es una regla absoluta.

**L7 LB** (capa de aplicación): **lee URL, headers y cookies**, así que rutea **por contenido** (mandar `/api/*` a un pool y `/static/*` a otro, o pegar a un usuario a un backend por su cookie). Es bueno para **HTTP**.

![[l7-load-balancer.svg]]
*L7 load balancer*

### Health checks y algoritmos

Los **health checks** ([[Health Check]]) sacan de rotación al backend caído: un *TCP check* verifica que **acepte conexiones**; un *L7 check* verifica que responda **HTTP 200** (y no 500 o nada). Para repartir:
- **Round-robin / random** para requests **stateless** cortas y parejas.
- **Least-connections** para conexiones **persistentes de duración dispar** (SSE, WebSockets), donde importa cuántas conexiones vivas tiene cada backend.

Implementaciones: **F5** (hardware), **HAProxy / NGINX / Envoy** (software), **AWS ELB** (ALB = L7, NLB = L4; con equivalentes en GCP y Azure). El hardware llega a **cientos de millones de rps**.

## Regionalización: la luz no negocia

Hay un piso de latencia que **ningún protocolo puede optimizar**: la **velocidad de la luz**. En fibra la señal viaja a ~2/3 de *c*, es decir **~200.000 km/s**. La distancia NY ↔ London es **~5.600 km**, así que el mínimo físico es **~56 ms solo de ida**, sin contar procesamiento. En la práctica, dentro del **mismo datacenter** la latencia es **<1 ms**, mientras que **cruzar el mundo es >80 ms**.

La **única salida** es acercar los datos al usuario:
- **CDN** ([[Content Delivery Network|CDN]]): cachear **contenido estático** en **edge servers** cerca del usuario (cientos o miles de ciudades). Por ejemplo, la búsqueda de Facebook se sirve desde el edge.
- **Particionamiento regional**: poner los **datos dinámicos** en su región. Ejemplo Uber: los conductores de Miami no necesitan vivir junto a los de NY, porque un rider de Miami **nunca** va a pedir un conductor de NY. Co-localás DB + servidores por región (vecino de [[Sharding]] y [[Consistent Hashing]]) y usás **availability zones** para aislar fallos dentro de la región.

## Manejo de fallos: una cadena donde cada eslabón arregla al anterior

Volvemos a la primera ley del marco mental: **la red no es confiable**. La robustez no es un patrón único sino una **composición** donde cada arreglo abre un agujero nuevo que el siguiente tapa.

1. **El servidor puede no responder** → ponés [[Timeout|timeouts]] y reintentos.
2. **Reintentar duplica el efecto** si la primera request *sí* llegó pero se perdió la *respuesta* (¡doble cobro!) → reintentar solo es seguro si la operación es **idempotente** (ver [[Idempotency]]).
3. **¿Cómo hago idempotente un pago?** → con una **idempotency key** ([[Idempotency Key]]), por ejemplo `userID + fecha`: el servidor la registra y, si la ve dos veces, **deduplica**.
4. **Si miles de clientes reintentan a la vez**, los retries se **sincronizan** y pegan en oleadas → **backoff con jitter** ([[Retry with Backoff]]): esperar cada vez más (**Exponential Backoff**) más un componente **aleatorio** que *desincroniza* a los clientes. La frase mágica de entrevista: *"retry with exponential backoff"*.
5. **Si el servicio ya está caído**, seguir reintentando lo **hunde más** (*cascading failures*, *Thundering Herd* —ej. bootear una DB recibiendo todo el tráfico de golpe—) → **circuit breaker** ([[Circuit Breaker]]): envuelve la dependencia y, tras N fallos, **abre el circuito** para fallar rápido.

Los **5 estados** del circuit breaker:

```text
monitor ──(N fallos)──► open ──(timeout)──► half-open ──(prueba OK)──► close
  ▲                      │                       │                       │
  └──────────────────────┴───────────────────────┴───(prueba falla)──► reopen
```

- **monitor**: operación normal, contando fallos.
- **open**: falla rápido, **no deja pasar** requests a la dependencia.
- **half-open**: tras un timeout, **deja pasar una request de prueba**.
- **close**: vuelve a la normalidad si la prueba salió bien.
- **reopen**: si la prueba falla, vuelve a abrir.

> [!question] 🎯 ¿Por qué no alcanza con timeout + retry para tener fiabilidad?
> Porque **cada arreglo abre un agujero nuevo**: el *retry* puede **duplicar** efectos (lo tapa la *idempotency key*); muchos retries se **sincronizan** y golpean en oleadas (lo tapa *backoff + jitter*); y reintentar contra un servicio **caído lo hunde más** (lo tapa el *circuit breaker*). La fiabilidad no es un patrón, es la **composición de todos**.

## Cierre: defaults razonados

La entrevista se gana eligiendo el default y sabiendo cuándo abandonarlo:
- **Transporte**: **TCP**, salvo que toleres pérdida → **UDP** (justificado), o **QUIC/HTTP3** si querés lo mejor de ambos.
- **API**: **REST afuera, gRPC adentro**; **GraphQL** solo cuando el under/over-fetching duela de verdad.
- **Realtime**: subí la escalera **SSE → WebSockets → WebRTC** solo hasta donde el caso lo exija.
- **Load balancing**: **L7 para HTTP**, **L4 para conexiones persistentes** (con la salvedad de que L7 también puede).
- **Latencia**: acercá los datos al usuario (**CDN + partición regional**), porque la luz no negocia.
- **Fallos**: la cadena completa que culmina en *"retry with exponential backoff"* + circuit breakers.

> [!tip] Follow-ups para profundizar
> **Wireshark** para inspeccionar packets reales, y **Network Link Conditioner** para simular redes degradadas y ver cómo se comportan tus protocolos.

## References

- Source: [Networking Essentials](https://www.hellointerview.com/learn/system-design/core-concepts/networking-essentials) — Hello Interview
- Las figuras embebidas son SVGs de `files.hellointerview.com`.
- El punto disputado sobre L4/L7 y WebSockets proviene de los comentarios del artículo fuente.

## Related

- [[_System Design|System Design]]
- [[Consistent Hashing]]
- [[Load Balancing]]
- [[REST API]]
- [[Server-Sent Events]]
- [[Bidirectional Streaming|WebSockets]]
- [[Circuit Breaker]]
- [[Retry with Backoff]]
- [[Content Delivery Network|CDN]]
- [[API Gateway]]
- [[Idempotency]]
