---
title: ZooKeeper
created: 2026-07-01
tags:
  - system-design/coordination
  - type/technology
  - status/permanent
aliases:
  - ZooKeeper
  - Apache ZooKeeper
  - Zookeeper
updated: 2026-07-01
reading:
  total_words: 4317
  read_words: 0
  pct: 0
  last_read: 2026-07-01
---

# ZooKeeper

> [!note] Tesis operativa
> ZooKeeper es un **servicio de coordinación distribuido**: expone un kernel chico de primitivas (un árbol de znodes + watches + sesiones) sobre el que construís servicios de alto nivel — locks distribuidos, **Leader Election**, service discovery, config management, group membership. La idea central es que coordinar procesos distribuidos *correctamente* (sin race conditions ni deadlocks) es notoriamente difícil, así que ZooKeeper te da un núcleo replicado y fuertemente consistente (**CP**) en vez de que cada quien reimplemente el consenso y meta la pata. NO es un data store: los znodes pesan <1MB, es un store de **metadata de coordinación**. En entrevistas es la respuesta a "¿cómo hago un lock distribuido / elijo un líder / detecto qué nodos están vivos?".

## Marco mental (leé esto primero)

Cuatro ideas sostienen todo lo demás. Si no las tenés claras, el resto no cierra:

- **No es una DB, es un coordinador**: znodes chicos (<1MB), carga read-heavy (~10:1 reads sobre writes), todo en memoria. Si querés meter datos de negocio adentro, estás usando la herramienta equivocada — ese es el error conceptual número uno.
- **Todo se construye con 3 piezas**: el **árbol de znodes** (datos + metadata jerárquicos), los **watches** (notificaciones push *one-shot* ante cambios) y las **sesiones** (que le dan vida a los znodes efímeros y detectan fallos vía heartbeat). Las recipes — lock, leader election, discovery — no son features de ZooKeeper: son *combinaciones* de estas 3 piezas.
- **Es CP, no AP**: bajo partición de red ZooKeeper prioriza consistencia sobre disponibilidad — rechaza el request antes que servir estado sin **[[Quorum|quórum]]**. Las escrituras son linearizables; las lecturas NO (pueden ser stale). Esa asimetría es el matiz que separa a un junior de un senior hablando de ZooKeeper.
- **El "por qué existe" (nivel senior)**: coordinar en sistemas asíncronos choca con la **FLP impossibility** — no podés distinguir un nodo lento de uno caído. ZooKeeper esquiva FLP asumiendo *sincronía parcial* y usando consenso (**ZAB**) para imponer un orden total sobre las escrituras.

## La cadena causal — de las 3 piezas a una recipe

El valor de ZooKeeper no está en ninguna primitiva aislada sino en cómo se combinan. Seguí el hilo:

1. Un **znode efímero** vive exactamente mientras vive la sesión que lo creó. Cuando la sesión muere (crash, GC pause, partición), el cluster borra ese znode solo. Esto es lo que convierte "estoy vivo" en un hecho observable por todos.
2. Un **znode secuencial** le agrega al nombre un contador monótono padded a 10 dígitos, único por padre. Combinado con efímero (`EPHEMERAL_SEQUENTIAL`) te da una cola ordenada de participantes que se autolimpia cuando alguno cae.
3. Un **watch** te avisa cuando un znode cambia o desaparece — sin tener que hacer polling. Es la pieza que transforma el estado del árbol en eventos.
4. Junta las tres y tenés un **[[Distributed Lock|lock distribuido]]**: cada cliente crea un efímero-secuencial bajo `/locks/`, el de número más bajo tiene el lock, y los demás ponen un watch sobre su predecesor inmediato. Cuando el que tiene el lock cae, su efímero desaparece, y despierta *exactamente uno* de los que esperaban. Esa es la recipe completa — y la misma primitiva, sin cambiar una línea, es también leader election.

## 1. Data model — el árbol de znodes

El namespace es jerárquico tipo filesystem (`/app/config`). Reglas: no hay paths relativos, el path `zookeeper` está reservado, y cada znode pesa **<1MB**. Cada znode guarda **data + metadata**, y esa metadata (la **Stat structure**) es la que habilita optimistic concurrency y detección de cambios.

**Stat structure** (metadata por znode):

| Campo | Definición |
|---|---|
| **czxid** | zxid del cambio que lo creó |
| **mzxid** | zxid del último cambio a la data |
| **pzxid** | zxid del último cambio a los hijos |
| **version** / **cversion** / **aversion** | nº de cambios a data / hijos / ACL |
| **ephemeralOwner** | session id del dueño si es efímero; 0 si no lo es |
| **dataLength** / **numChildren** | longitud de la data / cantidad de hijos |

El campo **version** es load-bearing: es lo que hace posible el CAS (compare-and-set) y el **fencing token**, como vas a ver en la sección de API.

**Tipos de znode** (`CreateMode`, 7 valores exactos):

| Tipo | Comportamiento |
|---|---|
| **PERSISTENT** | Persiste hasta un borrado explícito |
| **EPHEMERAL** | Se borra al terminar la sesión que lo creó. **No puede tener hijos** |
| **PERSISTENT_SEQUENTIAL** | Persistente + contador monótono (10 dígitos padded), único por padre |
| **EPHEMERAL_SEQUENTIAL** | Efímero + secuencial (el patrón de locks / leader election) |
| **CONTAINER** (3.5.3+) | Candidato a borrado al perder su último hijo (pensado para recipes) |
| **PERSISTENT_WITH_TTL** / **PERSISTENT_SEQUENTIAL_WITH_TTL** (3.5.3+) | Persistente + TTL en ms |

> [!warning] No existe ephemeral+TTL
> Un efímero con TTL sería redundante — el efímero ya muere solo con la sesión, así que un TTL encima no tiene sentido. Solo los persistentes tienen variante TTL. Es una pregunta capciosa clásica.

## 2. Sessions

El cliente se conecta pasando un **connection string** multi-host (`host1:2181,host2:2181,...`) — así, si un server se cae, el cliente prueba el siguiente. Opcionalmente agrega un **chroot** (`.../app/a`) que ancla todas sus operaciones en un subpath, útil para aislar aplicaciones en el mismo ensemble.

El timeout de sesión se negocia dentro de un rango de **2× a 20× el `tickTime`**. Si la conexión queda idle, el cliente manda un **PING** para mantenerla viva. El punto clave: **la expiración la maneja el cluster, no el cliente**. Cuando el cluster deja de recibir comunicación de una sesión dentro de su timeout, borra **todos los znodes efímeros** de esa sesión (los encuentra vía `ephemeralOwner`) y notifica a los watchers. Por eso "detectar que un proceso murió" es gratis: no lo detectás vos, lo detecta el cluster por ausencia de heartbeat.

La sesión está atada al **cluster, no a un server puntual** → el cliente puede reconectar a otro server reteniendo la misma sesión. Estados del ciclo de vida: `CONNECTING → CONNECTED → CLOSED`; los eventos `KeeperState` incluyen `Disconnected`, `SyncConnected`, `Expired`, entre otros. `Expired` es terminal — la sesión ya no existe y hay que crear una nueva.

## 3. Watches

La definición canónica: *"one-time trigger, sent to the client that set the watch, when the data changes."* Tres propiedades que hay que internalizar:

- **One-shot**: el watch se dispara UNA vez y se remueve. Hay que re-registrarlo si querés seguir escuchando.
- **Entrega asíncrona**: el evento llega push, sin polling.
- **Específico por tipo**: un *data watch* (de `getData`/`exists`) es distinto de un *child watch* (de `getChildren`).

Qué operación te avisa de qué:

| Operación | Eventos que dispara el watch |
|---|---|
| `exists()` | Created, Deleted, Changed |
| `getData()` | Deleted, Changed |
| `getChildren()` | Child |

**Orden**: el cliente ve el watch event ANTES de ver la nueva data — nunca al revés.

> [!warning] Se pierden cambios intermedios
> Entre que el watch se dispara y lo re-registrás hay una ventana de latencia. Si el dato cambia varias veces en ese hueco, ZooKeeper **NO te entrega cada cambio** — solo te garantiza que vas a ver el *estado final*. El patrón correcto es siempre **watch → recibís el evento → releés el valor + re-registrás el watch**, nunca asumir que el evento trae el dato nuevo adentro. Los persistent/recursive watches (3.6.0+) vía `addWatch()` no se remueven al dispararse y evitan parte de este baile.

## 4. ZAB — el protocolo de consenso

**ZAB** (ZooKeeper Atomic Broadcast) es un protocolo de atomic broadcast primario-backup (paper DSN 2011). Importante para entrevistas: **NO es Paxos ni Raft** — es un protocolo propio. Garantiza *reliable delivery* + **total order** + *causal order*, todo sobre canales FIFO (TCP). En el fondo ZAB es **total order broadcast**, y ese problema es **equivalente al consensus** — resolver uno es resolver el otro.

**zxid** (64 bits) es el corazón del orden total: los **32 bits altos son el epoch** (cambia con cada nueva jefatura) y los **32 bits bajos son el contador** (uno por transacción). Cada líder nuevo arranca su epoch fijando el zxid en `(epoch_max + 1, 0)`. Como el epoch es el bit más significativo, cualquier transacción de un líder nuevo siempre ordena *después* de todas las del líder viejo — sin necesidad de coordinar relojes.

**Fases** (el paper describe 4; la implementación real colapsa Discovery+Sync en "recovery", quedando 3):

| Fase | Qué ocurre |
|---|---|
| **Leader Election** | **Fast Leader Election (FLE)**: elige al peer con el **mayor zxid** (el historial más actualizado) → así minimiza cuánto estado hay que transferir después |
| **Discovery** *(recovery)* | Los followers se conectan; el líder fija el nuevo epoch |
| **Synchronization** *(recovery)* | El líder pone al día a los followers atrasados; recién ahí el ensemble procesa transacciones nuevas |
| **Broadcast** | Operación normal: propuestas → ACK → COMMIT |

**Protocolo de escritura (2PC-ish)** — la cadena exacta de una escritura:

1. El cliente manda el write al server al que está conectado.
2. Si ese server no es el líder, **reenvía** el write al líder.
3. El líder asigna un zxid + crea una **propuesta**, y la manda a los followers en orden.
4. Cada follower persiste la propuesta en su **[[Write-Ahead Log|WAL]]** y responde **ACK**.
5. Cuando el líder junta ACKs de un **quórum** (mayoría), manda **COMMIT**.
6. Cada server aplica la transacción a su DB en memoria en el mismo orden total.
7. El líder responde al cliente.

Es un [[Two-Phase Commit|2PC]] simplificado: propuesta (fase 1) → commit tras quórum (fase 2). La diferencia con 2PC clásico es que ZAB solo necesita *mayoría*, no *unanimidad* — por eso tolera fallos sin bloquearse.

**Quórum**: con `2f+1` servers tolerás `f` fallos. La garantía surge de que dos quórums cualesquiera *siempre* se intersectan en ≥1 server → ese solapamiento es lo que **previene el split-brain** (no puede haber dos mayorías disjuntas eligiendo cosas distintas). Ver [[Quorum]].

## 5. Arquitectura del ensemble

El ensemble tiene tres roles, y entender quién hace qué explica el perfil read-heavy:

- **Leader**: coordina TODAS las escrituras y es el único que asigna zxid (o sea, el único que define el orden total).
- **Followers**: atienden clientes, votan (ACK) las propuestas del líder, y sirven lecturas desde su copia local.
- **Observers**: replican el estado pero **NO votan** → escalan lecturas y dan fan-out geográfico *sin* engordar el quórum, por lo que no penalizan la latencia de escritura. Son la respuesta cuando necesitás muchos reads sin frenar los writes.

La regla operativa clave: **todas las escrituras pasan por el líder (con quórum); las lecturas las sirve cualquier server desde su memoria local, sin quórum**. De ahí sale a la vez la ventaja read-heavy *y* el motivo de que las lecturas puedan ser stale (un follower atrasado sirve un valor viejo). Si necesitás una lectura fresca, **`sync()`** fuerza al server a alinearse con el líder antes de leer.

**Por qué número impar (3/5/7)**: un ensemble de 4 tolera exactamente lo mismo que uno de 3 (1 fallo), porque `2f+1` con f=1 son 3 y con f=1 el cuarto server no compra tolerancia extra. El par simplemente desperdicia un server y encima aumenta el riesgo de split-vote en la elección. Regla: un ensemble impar tolera `N/2` caídas (parte entera).

## 6. Garantías de consistencia (el matiz senior/staff)

ZooKeeper promete cinco garantías, pero hay que leerlas con cuidado porque no dicen "linearizable para todo":

| Garantía | Definición |
|---|---|
| **Sequential Consistency** | Los updates de un cliente se aplican en el orden en que ese cliente los envió (FIFO por cliente) |
| **Atomicity** | Todo o nada — nunca resultados parciales |
| **Single System Image** | Misma vista lógica sin importar a qué server te conectes |
| **Reliability** | Un update aplicado persiste hasta que otro lo sobreescriba |
| **Timeliness** | La vista de un cliente está up-to-date dentro de un bound (decenas de segundos) |

**La asimetría clave**: las **escrituras son linearizables** (pasan por ZAB, quórum y orden total); las **lecturas NO son linearizables** por default entre clientes distintos — solo son sequential/FIFO respecto del *propio* cliente, así que **pueden ser stale**. El paper le pone nombre a este punto medio: **Ordered Sequential Consistency (OSC)**, que vive entre la sequential consistency y la linearizability.

> [!warning] Dato de Jepsen (staff-level)
> Ni siquiera `sync()` + read es *estrictamente* linearizable. El `sync` solo llega al líder, y existe una ventana en la que el líder *cree* seguir siéndolo pero ya perdió el quórum (se eligió un líder nuevo) → puede devolver un valor stale. El análisis de Jepsen nunca encontró que ZooKeeper dropeara un write ya confirmado (los writes son sólidos), pero **las lecturas son el punto débil**. Moraleja: no bases correctitud crítica en la frescura de un read; basala en el fencing token del write.

## 7. Persistencia

Cómo ZooKeeper sobrevive a un reinicio sin perder datos confirmados:

- **Transaction log (WAL)**: se escribe ANTES de aplicar el cambio en memoria y ANTES de mandar el ACK, con **fsync antes de reconocer** — esa es la garantía de durabilidad (un write confirmado ya está en disco). El flag `forceSync` lo controla, y desactivarlo es *unsafe*.
- **Fuzzy snapshots**: snapshots periódicos *no bloqueantes* del árbol en memoria, tomados "en caliente". No son un corte exacto en un instante — por eso el recovery no se apoya solo en el snapshot.
- **Recovery**: último snapshot + **replay del transaction log** posterior. El snapshot te acerca, el WAL te completa hasta el último write confirmado.

Config verbatim de un ensemble de 3:

```
tickTime=2000
dataDir=/var/lib/zookeeper/
clientPort=2181
initLimit=5
syncLimit=2
server.1=zoo1:2888:3888
server.2=zoo2:2888:3888
server.3=zoo3:2888:3888
```

> [!tip] Separá los discos en producción
> Poner **`dataLogDir`** (el WAL, escritura secuencial constante y en el camino crítico del ACK) en un disco distinto de **`dataDir`** (los snapshots) evita contención de I/O. El WAL está en el path de latencia de cada escritura, así que no querés que un snapshot le pise el disco.

## 8. API / operaciones

Firmas Java (subset síncrono):

```java
String create(String path, byte[] data, List<ACL> acl, CreateMode createMode)
void delete(String path, int version)
Stat exists(String path, boolean watch)
byte[] getData(String path, boolean watch, Stat stat)
Stat setData(String path, byte[] data, int version)
List<String> getChildren(String path, boolean watch)
void sync(String path)
List<OpResult> multi(Iterable<Op> ops)
```

- **Todas las escrituras toman un `version`** → optimistic concurrency (CAS): si la version que pasás no coincide con la actual del znode, tirás `BadVersionException`; pasar `-1` desactiva el chequeo. **Este `version` ES el fencing token** — la pieza que hace que un lock sea *correcto* aun con GC pauses.
- `multi()` → una transacción atómica multi-operación (todo o nada), para agrupar varios cambios sin dejar estados intermedios visibles.
- `sync()` → asíncrono, fuerza al server a alinearse con el líder antes de la próxima lectura.

**ACL con formato `scheme:id:permissions`** — y ojo, **NO se hereda a los hijos** (cada znode tiene su propia ACL). Los permisos son **CRWDA**: Create / Read / Write / Delete / Admin — notá que **C** y **D** están separados de **W**, para poder dar control fino (dejar crear hijos pero no borrar, por ejemplo). Schemes disponibles: `world`, `auth`, `digest`, `ip`, `x509`.

## 9. Recipes de coordinación (los "ZooKeeper recipes")

Todas las recipes son la misma idea reciclada: **efímeros/secuenciales + watches**.

- **Distributed lock**: cada cliente crea un `EPHEMERAL_SEQUENTIAL` bajo `/locks/lock-`; el de **número más bajo** tiene el lock; los demás ponen un **watch sobre el znode inmediato anterior** (NO sobre el mínimo). Cuando el predecesor desaparece, despierta **exactamente UN** cliente. Esto es lo que **evita el herd effect**: si todos watchearan el mismo znode mínimo, al liberarse el lock se despertarían *todos* a la vez y se pisarían — watchear al predecesor los pone en fila india. Ver [[Distributed Lock]].
- **Leader election**: idéntico al lock — el secuencial más chico es el líder, los demás watchean a su predecesor. Cuando el líder muere, su efímero desaparece y el siguiente en la fila se entera. Lock y leader election son literalmente **la misma primitiva**.
- **Service discovery / group membership**: cada instancia registra un `EPHEMERAL` bajo `/services/<svc>/`; si cae, su znode desaparece solo. Un `getChildren` + watch te da la membership viva sin polling. Ver [[Service Discovery]].
- **Config management**: guardás la config en la **data** de un znode persistente; un watch sobre ese znode + un `setData` propaga el cambio a todos los que escuchan.
- **Barriers / double barriers**: sincronizar el inicio y/o el fin de N procesos (arrancan todos juntos, o terminan todos juntos).
- **Distributed queue**: `PERSISTENT_SEQUENTIAL` bajo `/queue/`; el consumidor toma el de menor número → FIFO natural.

> [!warning] No implementes las recipes a mano
> En producción usá **Apache Curator** (nació en Netflix, hoy es Apache): trae las recipes ya probadas (`InterProcessMutex`, `LeaderSelector`, etc.) más el manejo automático de reconexión y retries. Las implementaciones caseras se rompen justo en los edge cases de reconexión — que son los que importan.

## 10. Quién lo usa (casos reales)

| Sistema | Uso |
|---|---|
| **[[Kafka]]** (legacy, pre-KRaft) | Metadata del cluster + controller election. Reemplazado por **KRaft** en Kafka 4.0 |
| **HDFS** | El `ZKFailoverController` coordina el failover Active/Standby del NameNode |
| **HBase** | HMaster election + asignación de regiones |
| **Solr / Mesos / NiFi / Druid** | Cluster coordination, leader election, service discovery |

## 11. ZooKeeper vs alternativas (comparación de entrevista)

| Dimensión | ZooKeeper | etcd | Consul |
|---|---|---|---|
| Consenso | **ZAB** (propio) | **Raft** | Raft + Gossip |
| Data model | Jerárquico (znodes) | KV plano + MVCC/revisions | KV + catálogo de servicios |
| API | Binario propio (Java/C) | gRPC/HTTP | HTTP/DNS |
| Caso principal | Coordinación genérica | Backing store de **Kubernetes** | Service discovery + health + multi-DC |
| Nivel | Primitivo (armás las recipes vos) | Primitivo similar | Alto nivel, "batteries included" |

**Por qué Kafka migró a KRaft**: sacar la dependencia externa significa un sistema distribuido menos que operar; además el mecanismo de watches de ZooKeeper era un cuello de botella con cientos de miles de particiones, y el failover de controller pasó de varios segundos a **<1s**, bajando el footprint de infra un 30/40%.

> [!tip] ZK/etcd vs Redis (Redlock) para locks — EL punto senior
> **Redlock** (el lock distribuido sobre **Redis**) es AP-ish: depende de timing frágil — la expiración del lock contra el delay de red contra un GC pause — y **no genera fencing tokens**, así que dos clientes pueden creer que tienen el lock a la vez. ZooKeeper (y **etcd**) son **CP con consenso real** y te dan un **fencing token** (la version del znode) → la correctitud NO depende de que los relojes anden bien. La conclusión de Kleppmann: *si la correctitud de tu sistema depende del lock, usá ZK/etcd, no [[Redis]] Redlock*. Ver [[Redis]].

## 12. Cuándo usar / cuándo NO

**SÍ**:
- Coordinación fuerte (leader election, locks *correctos*, barriers) donde necesitás linearizabilidad en las escrituras.
- Ya tenés un stack Hadoop / Kafka-legacy / HBase que lo trae de arrastre.
- Volumen de coordinación bajo-medio.

**NO**:
- Como data store — los znodes son <1MB, no es para eso.
- High-throughput de escritura — todos los writes van al líder + quórum, es un cuello de botella deliberado.
- Service discovery greenfield — **etcd**/**Consul** suelen encajar mejor y con menos ceremonia.
- Si ya estás en Kubernetes — ya tenés **etcd** corriendo, no dupliques la pieza de coordinación.

## 13. Gotchas / operación

> [!warning] GC pauses → session expiry (el gotcha número uno)
> La causa más común de una expiración *inesperada* no es un crash, es un **GC pause largo** que impide mandar el heartbeat a tiempo → el cluster da la sesión por muerta → borra los efímeros → se "suelta" un lock o un leadership **sin que el proceso se haya muerto**. Por eso es peligrosa toda lógica que asuma "si tengo el znode, tengo el lock para siempre": verificá el **fencing token** al *ejecutar* la operación protegida, no solo al *adquirir* el lock.

- **jute.maxbuffer**: el límite de payload (~1MB, `0xfffff`). Si necesitás subirlo, hay que hacerlo en **TODO** el ensemble (server *y* cliente) o quedan desalineados.
- **Subir el `session_timeout` como parche es un anti-patrón**: no arregla la causa (el GC) y encima alarga la ventana en la que un efímero "zombie" sigue vivo → split-brain lógico. Atacá el GC, no el síntoma.

## 14. Escala / staff-level

- **Throughput**: del orden de decenas de miles de ops/s, fuertemente read-heavy — los reads son baratos porque no requieren atomic broadcast (se sirven de memoria local), solo los writes pagan el quórum.
- **Ensemble size vs latencia**: más nodos = writes más lentos (quórum más grande que satisfacer) pero más durabilidad y más capacidad de reads. Los **Observers** son la vía para escalar reads *sin* penalizar los writes, porque no votan.
- **FLP impossibility** (Fischer-Lynch-Paterson, 1985): en un sistema asíncrono con al menos 1 fallo posible, ningún algoritmo determinístico garantiza a la vez *agreement*, *validity* y *termination*. Los sistemas reales lo esquivan asumiendo **sincronía parcial**: mantienen *safety* siempre y consiguen *liveness* durante los períodos de sincronía. ZooKeeper es exactamente eso.
- **Evolución 2025+**: la industria se mueve a stacks basados en **Raft** (etcd, KRaft, Consul). ZooKeeper sigue relevante donde ya está desplegado; para greenfield hay que justificar *por qué NO* Raft directo.

## 🎯 Preguntas de retrieval

> [!question] ¿Por qué se watchea al predecesor y no al znode mínimo actual en un lock?
> Para evitar el **herd effect**: si todos watchearan el mismo znode mínimo, al liberarse el lock se despertarían *todos* los que esperan a la vez y se pisarían. Watcheando cada uno a su predecesor inmediato, al liberarse el lock despierta **exactamente uno** — los pone en fila india.

> [!question] ¿Qué pasa si el cliente que tiene el lock se cae?
> Su sesión expira → el cluster borra el znode efímero que sostenía el lock → el lock se libera solo, sin intervención. Esa autolimpieza es precisamente para qué existe el tipo efímero.

> [!question] ¿ZooKeeper es CP o AP?
> **CP**. Bajo partición prefiere rechazar el request antes que servir estado sin quórum. La consistencia gana sobre la disponibilidad.

> [!question] ¿Las lecturas de ZooKeeper son consistentes?
> **NO** por default: son sequential/FIFO respecto del propio cliente y pueden ser **stale** entre clientes distintos. `sync()` las acerca al líder, pero ni así son *estrictamente* linearizables (dato de Jepsen: hay una ventana donde el líder ya perdió el quórum y no lo sabe).

> [!question] ¿Por qué un número impar de nodos?
> Porque un ensemble de 4 tolera lo mismo que uno de 3 (1 fallo): el server par no compra tolerancia extra, solo desperdicia recursos y agrega riesgo de split-vote en la elección.

> [!question] ¿Qué es exactamente un zxid?
> Un identificador de 64 bits: 32 altos de **epoch** (cambia con cada líder) + 32 bajos de **contador** (por transacción). Da un **orden total** sobre todas las transacciones, y como el epoch es el bit más significativo, todo lo del líder nuevo ordena después de lo del viejo.

> [!question] ¿ZAB es Paxos o Raft?
> Ninguno de los dos — es un protocolo propio de atomic broadcast. Ese atomic/total-order broadcast es **equivalente al problema de consensus**, pero la implementación es de ZooKeeper, no Paxos ni Raft.

> [!question] ¿Por qué Kafka migró a KRaft?
> Para sacarse de encima la dependencia externa (un sistema distribuido menos que operar); además los watches de ZooKeeper eran un cuello de botella con cientos de miles de particiones, y el failover de controller bajó de varios segundos a <1s.

> [!question] ¿Por qué ZK/etcd para un lock y no Redis Redlock?
> Porque ZK/etcd son **CP con consenso real** y dan **fencing token** (la version del znode) → la correctitud no depende de relojes. Redlock es AP-ish, depende de timing frágil y no da fencing token, así que dos clientes pueden creerse dueños del lock a la vez.

> [!question] ¿Qué distingue a Leader, Follower y Observer?
> El **leader** ordena todas las escrituras (asigna zxid). El **follower** vota (ACK) propuestas y sirve reads locales. El **observer** replica el estado pero **no vota** → escala reads y fan-out geográfico sin engordar el quórum ni frenar los writes.

> [!question] ¿Cómo se recupera un server al reiniciar?
> Carga el **último snapshot** y le hace **replay del transaction log (WAL)** posterior. El snapshot lo acerca al estado reciente y el WAL lo completa hasta el último write confirmado.

## References

- Este resumen viene de research multi-fuente (4 agentes en paralelo), no de un único artículo: docs oficiales de Apache ZooKeeper (Programmer's Guide, Internals/ZAB, Overview, Admin, Java API) + los papers USENIX ATC 2010 ("Wait-free coordination") y DSN 2011 (Zab) + material de preparación de entrevistas (HelloInterview — deep-dive de ZooKeeper, *parcialmente paywalled* —, AlgoMaster, DesignGurus/Grokking Advanced, System Design Handbook) + análisis de consistencia de Jepsen/aphyr + DDIA cap. 9 + MIT 6.824 lecture 8 + material de Confluent/Redpanda sobre KRaft + Kleppmann/antirez sobre Redlock + docs de Curator.
- Los enriquecimientos de comparación (vs etcd/Consul, Redlock/fencing, FLP, matices de linearizability) provienen de ese cuerpo de preparación de system design, no de una fuente única.

## 🔗 Conexión con el vault

- Volvé al MOC del subtema: [[_Coordination|Coordination]].
- Otras tecnologías deep-dive (tecnologías hermanas de la serie): [[Kafka]] (el caso de uso histórico más famoso de ZooKeeper, hoy migrado a KRaft), [[Flink]], [[Cassandra]], [[Redis]] (el contraste Redlock vs ZK para locks), [[Elasticsearch]], [[DynamoDB]].
- Patrones y conceptos relacionados: [[Distributed Lock]], [[Quorum]], [[Two-Phase Commit]], [[Write-Ahead Log]], [[Service Discovery]], [[Primary-Replica]], [[Consistent Hashing]], [[Sharding]], [[Idempotency]], [[Event-Driven Architecture]], [[Vector Clocks]].
