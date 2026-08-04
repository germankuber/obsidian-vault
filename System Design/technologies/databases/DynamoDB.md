---
title: DynamoDB
created: 2026-07-01
tags:
  - system-design/databases
  - type/technology
  - status/permanent
aliases:
  - DynamoDB
  - Amazon DynamoDB
updated: 2026-07-01
reading:
  total_words: 5604
  read_words: 0
  pct: 0
  last_read: 2026-07-01
---

# DynamoDB

## Tesis operativa
DynamoDB es la base NoSQL key-value **fully-managed** de AWS: AWS se ocupa de hardware, configuración, patching y scaling; escala automáticamente y sin downtime; y, a diferencia del prejuicio clásico sobre NoSQL, soporta **ACID Transactions** — la crítica "NoSQL = sin transactions" ya no aplica. No es open-source, así que sus internals no se leen en un repo: se arman combinando la documentación de AWS con el **Dynamo Paper** (USENIX ATC 2022, la versión moderna del paper de 2007). Antes de proponerlo en una entrevista conviene preguntarle al interviewer si está permitido usarlo: algunos equipos lo descartan de entrada por **Vendor Lock-in**.

## Marco mental (leé esto primero)
- **Fully-managed**: no administrás nodos ni particiones a mano — eso es justo lo que la separás de [[Cassandra]], donde vos operás el cluster.
- **Key-value con soporte de transactions**: DynamoDB no es un key-value "tonto"; tiene índices secundarios, queries por rango y transacciones multi-tabla. Sin este matiz, secciones enteras de abajo (GSI/LSI, TransactWriteItems) no tienen sentido.
- **Partición física con techo duro de throughput**: cada partición tiene un límite fijo de RCU/WCU. Esto es la raíz de casi todos los problemas de diseño avanzados (hot partitions, single-table design, write sharding) — conviene tenerlo en la cabeza antes de leer esas secciones.
- **Todo es per-request, no per-tabla**: consistency (`ConsistentRead`), y hasta el modelo de costos (RCU/WCU), se deciden en cada llamada, no como propiedad fija de la tabla.

## The Data Model
La jerarquía es Tables → Items → Attributes. Una **Table** es el contenedor top-level: exige una primary key obligatoria y opcionalmente soporta índices secundarios. Un **Item** es el equivalente a una fila — una colección de **Attributes**, con un límite duro de 400KB por item (contando todos sus attributes). Un **Attribute** es un par clave-valor: puede ser un escalar (strings, numbers, booleans), un set, o un documento anidado.

La tabla es **Schema-less**: no hay un schema previo que validar contra la base — dos items en la misma tabla pueden tener attributes completamente distintos, y la validación de forma queda en manos de la aplicación. JSON en este contexto es *solo el formato de transporte* (lo que ves en la consola AWS o mandás por la SDK); el storage interno es propietario.

![[dynamodb-05-create-table.png|859]]

```json
{
  "PersonID": 101,
  "LastName": "Smith",
  "FirstName": "Fred",
  "Phone": "555-4321"
},
{
  "PersonID": 102,
  "LastName": "Jones",
  "FirstName": "Mary",
  "Address": {
    "Street": "123 Main",
    "City": "Anytown",
    "State": "OH",
    "ZIPCode": 12345
  }
},
{
  "PersonID": 103,
  "LastName": "Stephens",
  "FirstName": "Howard",
  "Address": {
    "Street": "123 Main",
    "City": "London",
    "PostalCode": "ER3 5K8"
  },
  "FavoriteColor": "Blue"
}
```

Notá el item 103: tiene `FavoriteColor` (que ningún otro item tiene) y una `Address` con forma distinta a la del item 102 (`PostalCode` en vez de `State`+`ZIPCode`). Eso es Schema-less en la práctica.

## Partition Key y Sort Key
La primary key se arma con uno o dos attributes. La **Partition Key** es obligatoria: se hashea para decidir en qué partición física (storage node) vive el item. La **Sort Key** es opcional y, cuando existe, forma junto a la Partition Key una **Composite Primary Key** — permite ordenar los items que comparten la misma Partition Key y hacer range queries sobre ellos.

![[dynamodb-01-primary-key.svg]]

En entrevista, la tarea es especificar PK y, opcionalmente, SK: buscás una Partition Key que optimice los patrones de query más comunes *y* que distribuya la carga parejo entre particiones (evitar hot keys). Ejemplo clásico — un chat grupal: `chat_id` como Partition Key + `message_id` como Sort Key, así "traeme los últimos N mensajes de este chat" es una sola Query eficiente.

Un detalle que suele trabarse en la entrevista: `message_id` **no debería ser un timestamp**. Los timestamps no son únicos — varios mensajes pueden llegar en el mismo milisegundo — así que como Sort Key rompen el orden o colisionan. Mejor usar un ID monotónico: contadores auto-incrementales, **UUID v7** (preferido sobre v1: es timestamp-first y ordenable como string, sin exponer la MAC address), **Snowflake ID**, o **ULID**.

Por debajo (under the hood), DynamoDB usa **Hash Partitioning**: hashea la Partition Key, y un request router consulta un **Partition Metadata Service** para saber a qué storage node mandar la request. Esto se parece a [[Consistent Hashing]] pero con una diferencia de arquitectura importante: acá el mapa partición→nodo es **centralizado** (un servicio de metadata + placement service que maneja split/merge automáticamente), no un ring peer-to-peer como el del Dynamo paper de 2007. Dentro de cada partición, si hay Sort Key, los items se guardan en un **B-tree** indexado por esa Sort Key — así las range queries dentro de la partición son eficientes. Para una operación con composite key, primero se hashea la Partition Key para llegar al nodo correcto, y después se hace un traverse del B-tree por la Sort Key. Este two-tier (hash a nivel partición + B-tree a nivel item) es lo que da a la vez horizontal scalability y querying eficiente.

## Secondary Indexes
Cuando necesitás consultar por un attribute que no es la primary key, DynamoDB ofrece dos tipos de índice secundario.

**GSI (Global Secondary Index)**: define una Partition Key y Sort Key distintas a las de la tabla base; vive en particiones físicas separadas, replicadas de forma independiente.

![[dynamodb-02-gsi.svg]]

**LSI (Local Secondary Index)**: mantiene la misma Partition Key que la tabla, pero con una Sort Key distinta — sirve para range queries/sorting alternativos dentro de la misma partición (co-located con los datos base).

![[dynamodb-03-lsi.svg]]

Volviendo al ejemplo del chat (tabla con PK `chat_id` + SK `message_id`): si necesitás "todos los mensajes de un usuario en todos sus chats", eso requiere un **GSI** con PK `user_id` + SK `message_id`. Si en cambio necesitás "los mensajes con más adjuntos dentro de un chat", eso es una **LSI** sobre `num_attachments` (misma PK `chat_id`, distinta SK). Ojo con esta última: una LSI **solo se puede definir al crear la tabla**, no se puede agregar después.

| Feature | GSI | LSI |
|---|---|---|
| Definition | Index con partition key distinto al de la tabla | Index con la misma partition key, distinto sort key |
| When to Use | Query por attributes que no son la primary key | Sort keys adicionales para query dentro de la misma partition key |
| Size Restrictions | Sin restricción de tamaño | Limitado a 10 GB por partition key |
| Throughput | RCU/WCU propios, separados de la tabla base | Comparte los RCU/WCU de la tabla base |
| Consistency | Solo eventually consistent | Eventual (default) y strongly consistent |
| Creation | Se agrega/quita en cualquier momento | Solo al crear la tabla, no se quita |
| Deletion | Borrar un GSI no afecta la tabla base | No se borra sin borrar la tabla |
| Maximum Count | Hasta 20 GSIs por tabla | Hasta 5 LSIs por tabla |
| Use Case | Búsqueda global entre particiones (ej. email en users) | Búsqueda local dentro de particiones (ej. órdenes recientes de un cliente) |

Por debajo: las GSIs son, en esencia, tablas internas separadas — sus updates son **asíncronos** respecto a la tabla base, por eso solo ofrecen eventual consistency. Las LSIs, en cambio, están co-located con la partición base, mantienen un B-tree separado indexado por la Sort Key de la LSI, y sus updates son **síncronos** — por eso pueden ofrecer tanto eventual como strong consistency.

### Sort-key y patrones de índice
Una Query sobre la Sort Key soporta operadores de rango: `=`, `<`, `<=`, `>`, `>=`, `BETWEEN` y `BEGINS_WITH` — esto es lo que habilita las range queries. Un patrón avanzado es la **Composite Sort Key**: concatenar varios campos en una sola Sort Key (por ejemplo `STATUS#DATE`) para poder filtrar por una combinación de condiciones dentro de una sola Query. Otro patrón es el **Sparse Index**: una GSI que solo indexa los items que efectivamente *tienen* el attribute usado como su Partition Key — así el índice queda chico y barato, útil para queries "filtradas" (ej. una GSI sobre `isOpen` que solo contiene las órdenes abiertas, porque las cerradas nunca tuvieron ese attribute). Y el **Inverted Index**: una GSI que invierte Partition Key y Sort Key de la tabla base, para poder consultar una relación many-to-many en la dirección contraria a la que la tabla base favorece.

## Accessing Data
Hay dos formas de leer datos: **Scan** recorre *todos* los items de la tabla (paginado) — es ineficiente y en general hay que evitarlo. **Query** busca por primary key o por un índice, y es la forma eficiente de leer, incluyendo range queries sobre la Sort Key. Se accede vía AWS SDK o la consola. **PartiQL** es una capa de conveniencia SQL-compatible que soporta `SELECT`/`INSERT`/`UPDATE`/`DELETE` sobre estas mismas operaciones.

```sql
SELECT * FROM users WHERE user_id = 101
```

```javascript
const params = {
  TableName: 'users',
  KeyConditionExpression: 'user_id = :id',
  ExpressionAttributeValues: {
    ':id': 101
  }
};

dynamodb.query(params, (err, data) => {
  if (err) console.error(err);
  else console.log(data);
});
```

```sql
SELECT * FROM users
```

```javascript
const params = {
  TableName: 'users'
};

dynamodb.scan(params, (err, data) => {
  if (err) console.error(err);
  else console.log(data);
});
```

> [!warning] Trampa de RCU con ProjectionExpression
> Yelp documentó este problema: por default, una lectura trae el item **entero**. `ProjectionExpression` solo recorta el bandwidth de red de la respuesta — el item completo igual se lee del storage, y pagás el RCU completo según el tamaño total del item. Esto es distinto de `SELECT columna` en SQL, donde limitar columnas sí reduce el costo de lectura. Si tenés items grandes, la solución es normalizar (por ejemplo, mover reviews largas a una tabla separada) en vez de confiar en projection para ahorrar.

Para escrituras/lecturas en lote existen **BatchWriteItem** (hasta 25 writes por call) y **BatchGetItem** (hasta 100 reads por call), ambas con un tope de 16MB por call y **sin atomicidad** — si algo falla, la respuesta trae `UnprocessedItems` y hay que reintentarlos con backoff. Esto contrasta con **TransactWriteItems** (hasta 100 acciones, atómico all-or-nothing, tope de 4MB): el trade-off es throughput/costo (batch, más barato, más rápido, pero parcial) contra atomicidad (transaction, todo o nada).

## CAP Theorem
DynamoDB ofrece dos consistency models para lecturas, y la elección es **por request**, no una propiedad fija de la tabla — se controla con `ConsistentRead=true` en `GetItem`, `Query` o `Scan`.

- **Eventual** (default): máxima disponibilidad y menor latencia; podés no ver el último write. Es el comportamiento AP con semántica **BASE**.
- **Strong** (`ConsistentRead=true`): refleja todos los writes previos confirmados; cuesta el doble de read capacity (1 RCU por 4KB, contra 0.5 RCU del modo eventual) y tiene mayor latencia.

En entrevista, la heurística es: strong consistency para flujos tipo booking (donde ver un dato viejo genera un bug de negocio), eventual para read-heavy donde la frescura no es crítica.

DynamoDB también soporta **ACID Transactions** vía `TransactWriteItems`/`TransactGetItems`, con serializable isolation, hasta 100 items across múltiples tablas. Advertencia importante: las strong reads solo están disponibles en la tabla base y en LSIs — las **GSIs solo ofrecen eventual consistency**, sin excepción.

Por debajo: una eventual read puede resolverse contra cualquiera de las 3 réplicas (el leader propaga los writes de forma asíncrona a los followers después de confirmar el quorum), y cuesta 0.5 RCU por 4KB. Una strong read se resuelve siempre contra el **leader** (que tiene el dato más actual), cuesta 1 RCU por 4KB, y por eso no está disponible en GSIs (que no tienen ese leader único con el dato fresco).

## Architecture & Scalability
**Scalability**: auto-sharding + load balancing. Cuando una partición llega a su límite (de tamaño o de throughput) se divide automáticamente y sus datos se redistribuyen, sin downtime. El Hash Partitioning es lo que garantiza que esa distribución sea pareja. **Global Tables** replican en tiempo real cross-region, dando lecturas y escrituras locales en cualquier parte del mundo, con múltiples Availability Zones por región. En una entrevista, mencionar Global Tables suele alcanzar como respuesta a "¿cómo lo hacés global?" — no hace falta profundizar más salvo que pregunten.

**Fault Tolerance**: cada partición se replica automáticamente en 3 Availability Zones por región (esto **no es configurable**). Cada partición mantiene 3 réplicas — 1 leader + 2 followers — gestionadas por AWS. Por debajo, cada partición corre **Multi-Paxos** sobre un replication group de 3 nodos: el leader maneja los writes, genera una entrada de **Write-Ahead Log**, la manda a los peers, y el write se confirma (acknowledge) recién cuando el **Quorum** (2 de 3) persistió la entrada. Las strong reads van siempre al leader; las eventual reads pueden ir a cualquiera de las 3 réplicas.

### Global Tables — active-active y resolución de conflictos
En un setup multi-región, cada réplica de una Global Table acepta *tanto* reads como writes (multi-active, no hay un primary fijo entre regiones). Cuando dos escrituras concurrentes tocan el mismo item en regiones distintas, la resolución de conflictos es **Last-Writer-Wins** por timestamp interno del item — es un mecanismo best-effort. Hay dos versiones del feature: v2019.11.21 (un solo stream record por write, más barata, permite agregar/quitar réplicas con datos ya cargados) y la v2017.11.29 anterior.

> [!warning] Trampa de CDC cross-región
> La replicación entre regiones usa internamente el stream de cada réplica. Un write que llegó desde la Region A aparece en el stream de la Region B **indistinguible** de un write local hecho en B. Si tenés un consumer de Change Data Capture escuchando ese stream sin cuidado, esto genera loops infinitos o doble procesamiento — la mitigación es taggear cada item con su `origin-region` y que el consumer la chequee antes de reaccionar.

Y el caveat silencioso de Last-Writer-Wins: si dos regiones escriben el mismo item casi al mismo tiempo, uno de los dos writes se **descarta sin aviso** — no hay merge, no hay error visible para el cliente que "perdió".

## Security
Encrypted at rest por default. TLS forzado en todas las llamadas a la API (in transit, siempre, sin necesidad de configurarlo). Soporta IAM con permisos fine-grained, y VPC endpoints para no exponer la tabla a internet. En entrevista, mencionar encryption at rest + TLS suele ser suficiente — profundizar en IAM/VPC es overkill salvo que pregunten específicamente.

## Pricing Model
Hay dos modos de capacity: **On-Demand** (cobra por request, impredecible pero simple) y **Provisioned Capacity** (RCU/WCU fijos, facturados por hora, más barato si el uso es predecible pero desperdicia capacidad si está subutilizada). **RCU** (Read Capacity Unit) = leer 4KB por segundo. **WCU** (Write Capacity Unit) = escribir 1KB por segundo.

| Feature | Cost | Details |
|---|---|---|
| Read Capacity Unit (RCU) | $1.12 por millón de reads (4KB c/u) | Un strongly consistent read/s hasta 4KB, o dos eventually consistent reads/s |
| Write Capacity Unit (WCU) | $5.62 por millón de writes (1KB c/u) | Un write/s hasta 1KB |

Los números concretos: cada partición soporta hasta 3,000 RCU y 1,000 WCU, lo que equivale a 12MB/s de lecturas (3000 × 4KB) y 1MB/s de escrituras (1000 × 1KB) **por partición**. Ejercicio tipo entrevista con YouTube views: cada write cuenta como mínimo 1 WCU (redondea hacia arriba a 1KB) — con 10M views/segundo necesitás del orden de 10,000 particiones solo para absorber ese throughput. En modo provisioned (~$0.00065 por WCU-hora en us-east-1), eso da 10M × $0.00065 × 24h ≈ $156,000/día (on-demand saldría todavía más caro). El objetivo de este tipo de cuenta no es el número exacto, sino demostrar que sabés hacer el gut check de costos con los factores correctos.

### Capacity modes y throttling
En modo provisioned existe **Auto-scaling**: target-tracking alrededor de ~70% de utilización, con un mínimo y máximo configurables. El modo on-demand absorbe hasta 2× el peak de tráfico previo dentro de una ventana de 30 minutos — superar ese margen sin pre-warm produce throttling. El error de throttling es `ProvisionedThroughputExceededException`; las SDKs oficiales ya vienen con exponential backoff + jitter por default para reintentar.

## Advanced Features — DAX
**DAX** es un cache in-memory purpose-built para DynamoDB que baja las lecturas read-heavy a microsegundos — en muchos casos evita tener que sumar Redis o Memcached aparte. Se integra reemplazando el client normal por el DAX client SDK (disponible para Java, .NET, Node, Python, Go): la API es compatible y los cambios de código son mínimos, pero **no es totalmente transparente** (no es un drop-in sin tocar nada). Funciona con [[Read-Through]] y [[Write-Through]].

![[dynamodb-04-dax.svg]]

> [!warning] DAX solo invalida lo que pasa por él
> DAX auto-invalida su caché únicamente para los writes que pasan **a través de DAX**. Si alguien hace un update directo a DynamoDB (bypaseando DAX), esa entrada queda **stale** en el caché hasta que expire por TTL o sea evicted.

DAX mantiene dos caches activos simultáneamente: un *item cache* (para `GetItem`/`BatchGetItem`) y un *query cache* (para `Query`/`Scan`). Un detalle importante: DAX **no cachea strongly consistent reads** — solo eventual.

## Advanced Features — Streams (CDC)
**DynamoDB Streams** es el mecanismo de **Change Data Capture** built-in de la base: captura cada insert/update/delete como un stream record en tiempo casi real. Casos de uso típicos: mantener un índice de [[Elasticsearch]] en sync con la tabla, alimentar analytics en tiempo real (Streams → **Kinesis Data Streams** → **Kinesis Data Firehose** → S3/Redshift/OpenSearch — ojo, Firehose *no* lee Streams directamente, hace falta pasar por Kinesis Data Streams o por una Lambda intermedia), o disparar Change Notifications vía Lambda ante cada modificación.

Detalles operativos que suelen aparecer en preguntas de seguimiento: la retención del stream es de 24 horas · hay 1 stream por tabla (una Global Table tiene 1 stream por réplica regional) · los shards del stream mapean a particiones pero son efímeros (se dividen/mezclan igual que las particiones, con un linaje parent→child) · el orden solo está garantizado *dentro* de un shard (por partition key), nunca a nivel global · cada shard soporta como máximo 2 consumers simultáneos · la forma recomendada de consumir es con el adaptador KCL (Kinesis Client Library) · las **GSI no generan stream records propios** — el stream solo refleja modificaciones de la tabla base · y la entrega es at-least-once, así que los consumers tienen que ser idempotentes (ver [[Idempotency]]).

### TTL
El **TTL** (Time To Live) se configura como un attribute de tipo Number con un timestamp Unix en segundos; el borrado es asíncrono y best-effort (puede tardar hasta un par de días), y es gratis — no consume WCU en el momento del borrado. Un detalle no obvio: los deletes por TTL **sí aparecen en el Stream**, marcados con `principalId=dynamodb.amazonaws.com` y `type=Service`, lo que permite armar pipelines de archival que reaccionen específicamente a expiraciones. Usos típicos: expiración de sesiones/cache, retención por GDPR, limpieza de eventos viejos.

## Escrituras condicionales y concurrencia
Un **Conditional Write** usa `ConditionExpression` para condicionar el write a un estado previo. El caso más común es `attribute_not_exists(PK)`, que convierte un `PutItem` en un insert idempotente (ver [[Idempotency]] e [[Idempotency Key]]): si el item ya existe, el write falla en vez de sobrescribir silenciosamente.

Sobre esa misma primitiva se construye **Optimistic Locking**: agregás un attribute `version` al item, y cada update lleva `ConditionExpression: version = :expected` — es un compare-and-swap. Si otro proceso ya modificó el item (y por lo tanto ya incrementó `version`), la condición no matchea y la call falla con `ConditionalCheckFailedException`, indicando que hay que reintentar leyendo el estado fresco.

Para contadores existe una primitiva distinta: **Atomic Counter**, usando `UpdateExpression` con `ADD`, que incrementa el valor del lado del servidor sin la race condition de un read-modify-write manual. Ojo: un Atomic Counter **no es idempotente** — si un `ADD` se reintenta después de un timeout (sin saber si el primero llegó a aplicarse), el contador puede quedar contado dos veces.

Esta es, en general, la respuesta esperada a "¿cómo prevenís lost updates?" en DynamoDB sin necesidad de levantar un [[Distributed Lock]] externo: la propia base te da compare-and-swap vía conditional writes.

## Single-table design
El patrón avanzado más característico de DynamoDB es modelar **todas** las entidades de la aplicación en una sola tabla, con Partition Key y Sort Key genéricos y sobrecargados (`PK`/`SK` sin nombre semántico fijo, reutilizados con distinto significado según el tipo de item). La pieza clave es el concepto de **Item collection**: el conjunto de items que comparten la misma Partition Key. Una sola Query sobre esa Partition Key trae de un viaje el "padre" y todos sus "hijos" — es, en la práctica, un *pre-join hecho en el momento de escribir*, no en el momento de leer.

Ejemplo (de Alex DeBrie): con `PK=USER#123`, el item con `SK=USER#123` guarda el perfil del usuario, y cada `SK=ORDER#<id>` guarda una de sus órdenes — una sola Query por `PK=USER#123` trae el perfil *y* todas las órdenes en un solo round trip, en vez de un query al usuario y N queries a sus órdenes. El principio detrás (Rick Houlihan): "lo que se accede junto, se guarda junto".

DynamoDB no tiene `JOIN` por diseño — no escala horizontalmente si tuviera que resolver joins arbitrarios entre particiones — y single-table design es la forma de compensar eso al momento de modelar.

El trade-off es real y hay que nombrarlo en entrevista:
- Exige conocer **todos** los access patterns de antemano, antes de diseñar el esquema. Un patrón nuevo que no se previó puede requerir un ETL (scan + backfill) sobre toda la tabla.
- Es rígido frente a cambios de requerimientos.
- El análisis ad-hoc (analytics) es difícil directamente sobre la tabla — la solución típica es Streams → Firehose → S3 → Athena, para no pelear con el modelo operacional.

Cuándo **no** conviene: proyectos greenfield donde los access patterns todavía no están claros, GraphQL (donde cada resolver suele necesitar su propio patrón de acceso por campo), o un caso de uso key-value simple donde una tabla por entidad ya es suficientemente eficiente.

## Hot partitions y throughput
Cada partición física tiene un techo **duro**: 3,000 RCU y 1,000 WCU por segundo, sin importar cuánta capacity tenga aprovisionada la tabla en total. Si una key concentra tráfico desproporcionado — un usuario "celebridad", una Partition Key tipo "today" que todo el tráfico del día golpea, o una Sort Key monotónica mal elegida — esa partición se convierte en una **Hot Partition** y empieza a tirar throttling aunque el resto de la tabla tenga capacidad de sobra.

DynamoDB mitiga esto con **Adaptive Capacity**: aísla el item/partición caliente para que pueda usar el techo completo de 3,000/1,000 disponible, redistribuyendo throughput no usado de otras particiones. Es importante entender qué hace y qué no hace: adaptive capacity **redistribuye** capacidad, no **eleva** el techo por partición — ese límite de 3,000 RCU / 1,000 WCU sigue siendo duro.

La única forma real de superar ese techo por partición es **Write Sharding** (también llamado salting): agregarle un sufijo a la key (`key#0`, `key#1`, ..., `key#N`) para esparcir artificialmente ese hot key en N particiones distintas, multiplicando por N el throughput disponible. El costo es que los reads ya no pueden ir a una sola partición: hay que hacer scatter-gather (leer las N variantes) y mergear los resultados en la aplicación.

Como colchón adicional existe **Burst Capacity**: DynamoDB retiene hasta 300 segundos (5 minutos) de capacity no usada, disponible para absorber picos puntuales sin throttling.

## DynamoDB vs otras bases
Esta es, en la práctica, la comparación #1 que un entrevistador va a pedir: ¿por qué DynamoDB y no [[Cassandra]] o Postgres?

| Feature | DynamoDB | Cassandra | PostgreSQL |
|---|---|---|---|
| Storage engine | B-tree por partición (read-optimized) | **LSM tree** (write-optimized: commit log → memtable → SSTables + compaction) | B-tree/heap relacional |
| Write path | Al leader de la partición, WAL + quorum de réplicas | A cualquier nodo (coordinator), append-only al commit log + memtable | Al primary, WAL + heap |
| Read path | Leader (strong) o cualquier réplica (eventual) | Read repair entre réplicas, tunable | Primary o réplicas read-only |
| Partitioning | Hash Partitioning + Partition Metadata Service **centralizado** | [[Consistent Hashing]] en ring peer-to-peer + gossip + vnodes | Sin partitioning nativo (requiere [[Sharding]] externo) |
| Consistency | `ConsistentRead` per-request (eventual/strong) | [[Quorum]] tunable por request (ONE/QUORUM/ALL) | Fuerte por default (ACID) |
| Secondary indexes | GSI/LSI | Índices secundarios limitados, poco recomendados | Ricos (B-tree, GIN, GiST, parciales) |
| CDC | DynamoDB Streams | CDC vía herramientas externas (ej. Debezium) | Logical replication / WAL decoding |
| Managed? | Sí, fully-managed AWS | No — vos operás el cluster (o DataStax Astra) | Depende (self-hosted o RDS/Cloud SQL managed) |
| Best-for | Alta escala predecible, key-value/access patterns fijos, serverless | Write-heavy masivo, multi-datacenter propio | Modelo relacional, queries flexibles, transacciones complejas |

El contraste de fondo es de storage engine: DynamoDB usa un **B-tree** por partición (óptimo para lectura), mientras Cassandra usa una **LSM tree** (óptimo para escritura: todo entra secuencial al commit log y al memtable, y la compactación reorganiza después). A nivel de partición, DynamoDB centraliza el mapa partición→nodo en un Partition Metadata Service, mientras Cassandra distribuye ese conocimiento peer-to-peer con [[Consistent Hashing]] + gossip. Y en consistency, Cassandra expone un [[Quorum]] tunable explícito (ONE, QUORUM, ALL) mientras DynamoDB simplifica a un booleano per-request (`ConsistentRead`).

La heurística de entrevista que vale oro acá: **"empezá con Postgres salvo que tengas una razón concreta DynamoDB-shaped (escala predecible, key-value, serverless, sin necesidad de joins/queries ad-hoc)"** — porque migrar de Postgres a DynamoDB, una vez que conocés tus access patterns reales, es mucho más fácil que migrar al revés, cuando ya modelaste todo alrededor de una sola tabla sin joins.

## En tu entrevista
**Cuándo usarlo**: DynamoDB sirve como persistence layer para casi cualquier sistema que necesite ser scalable, durable, con soporte de transactions, y con latencias de single-digit milliseconds (o hasta microsegundos si sumás DAX).

**Cuándo no** — los 4 límites a nombrar:
1. **Cost Efficiency**: puede volverse caro en escenarios de volumen muy alto (ver el ejercicio de YouTube views arriba).
2. **Complex Query Patterns**: no hace joins ni aggregations; sí tiene transactions (hasta 100 items multi-tabla), pero no el querying flexible de SQL.
3. **Data Modeling Constraints**: exige modelado cuidadoso desde el día uno; si tu dominio necesita muchos GSI/LSI para cubrir access patterns variables, probablemente estás peleando contra la herramienta y PostgreSQL encaja mejor.
4. **Vendor Lock-in**: te ata a AWS — no hay un "DynamoDB self-hosted" real.

El resumen que hay que dejar claro: DynamoDB soporta ACID Transactions (incluso multi-tabla), así que la vieja crítica de "NoSQL = sin transactions" ya no es un argumento válido en contra.

## Extras de nivel staff
El **Dynamo Paper** original de 2007 (peer-to-peer, gossip, [[Vector Clocks]], Merkle trees) **no es el mismo sistema** que el servicio DynamoDB que existe desde 2012 en adelante — el servicio moderno es centralizado, basado en Multi-Paxos con un leader por partición, y no expone vector clocks al cliente. Nombrar esta distinción en una entrevista es una señal de nivel staff (para lo que sí usaba el paper de 2007, ver [[Vector Clocks]]). Del paper de 2022 sobre el servicio actual vale la pena conocer: el request router es stateless (cachea la metadata de particiones), las log replicas son solo-WAL y se recuperan (heal) en cuestión de segundos, existe un **GAC** (Global Admission Control) basado en tokens para limitar throughput agregado, y la tesis de diseño explícita es "predictable performance over efficiency" — preferir latencia consistente antes que exprimir el hardware al máximo.

Otros dos detalles de producción: **PITR** (point-in-time recovery) permite restaurar la tabla a cualquier segundo dentro de los últimos 35 días, complementado con backups on-demand y export a **S3**. Y para items que superan el límite de 400KB, el patrón estándar es guardar el payload grande en **S3** y dejar en DynamoDB solo un puntero (claim-check pattern).

## 🎯 Preguntas de repaso
> [!question] ¿Por qué `message_id` no debería ser un timestamp como Sort Key?
> Porque los timestamps no son únicos — varios mensajes pueden compartir el mismo milisegundo — y eso rompe el orden o genera colisiones. Conviene un ID monotónico (UUID v7, Snowflake, ULID) que sea único y ordenable.

> [!question] ¿Cuál es la diferencia real entre GSI y LSI más allá de "global vs local"?
> GSI vive en particiones separadas con replicación asíncrona (solo eventual consistency, RCU/WCU propios, se agrega/quita cuando quieras). LSI está co-located con la partición base, replica sincrónicamente (por eso soporta strong consistency), comparte capacity con la tabla, y solo se define al crear la tabla.

> [!question] ¿Por qué una strong read cuesta el doble de RCU que una eventual read?
> Porque una strong read siempre va al leader de la partición (el único nodo garantizado con el dato más reciente), mientras que una eventual read puede resolverse contra cualquiera de las 3 réplicas — DynamoDB cobra ese costo de garantía como 1 RCU/4KB contra 0.5 RCU/4KB.

> [!question] ¿`ProjectionExpression` reduce el costo de una lectura?
> No. Solo reduce el bandwidth de la respuesta de red — el item completo se lee igual del storage y se cobra el RCU completo según el tamaño total del item, a diferencia de un `SELECT columna` en SQL.

> [!question] ¿Cómo particiona DynamoDB comparado con consistent hashing clásico?
> Usa Hash Partitioning con un Partition Metadata Service **centralizado** que gestiona el mapa partición→nodo y hace split/merge automático — no es un ring peer-to-peer descentralizado como el del Dynamo paper de 2007.

> [!question] ¿Cómo llega DynamoDB a durabilidad con solo 3 réplicas?
> Con Multi-Paxos: el leader escribe a un Write-Ahead Log y propaga a los 2 followers; el write se confirma recién cuando el Quorum (2 de 3) persistió la entrada.

> [!question] ¿Por qué DAX no sirve como caché para strongly consistent reads?
> Porque DAX solo cachea el resultado de eventual reads — cachear una strong read rompería la garantía misma que la strong read promete (dato siempre fresco).

> [!question] ¿Cuáles son los 4 motivos para NO usar DynamoDB?
> Cost efficiency en volumen alto, falta de complex query patterns (joins/aggregations), data modeling constraints (exige conocer access patterns de antemano), y vendor lock-in a AWS.

> [!question] ¿Adaptive Capacity resuelve una hot partition o igual necesitás write sharding?
> Adaptive Capacity solo redistribuye/aísla throughput no usado hacia la partición caliente — el techo de 3,000 RCU/1,000 WCU por partición sigue siendo duro. Para superar ese techo hace falta Write Sharding (salting de la key en N particiones).

> [!question] ¿Por qué preferir single-table design en vez de una tabla por entidad?
> Porque permite pre-armar el join en el momento de escribir: una Item collection (items que comparten Partition Key) se trae completa en una sola Query, evitando múltiples round trips — DynamoDB no tiene JOIN por diseño.

> [!question] ¿Cómo se resuelven conflictos en Global Tables multi-región?
> Con Last-Writer-Wins por timestamp interno del item, de forma best-effort. El caveat es que, ante escrituras concurrentes al mismo item en dos regiones, una de las dos se descarta silenciosamente.

> [!question] ¿Cómo hacés un insert idempotente o prevenís lost updates sin un lock externo?
> Con Conditional Writes: `attribute_not_exists(PK)` para un insert idempotente, o un compare-and-swap con un attribute `version` (Optimistic Locking) para updates seguros bajo concurrencia.

> [!question] ¿DynamoDB o Cassandra para un workload write-heavy masivo?
> Cassandra tiene ventaja estructural ahí por su LSM tree (write-optimized por diseño), pero a costa de operar vos el cluster. DynamoDB, siendo B-tree y managed, también soporta cargas de escritura altas, pero el trade-off real suele ser operational overhead (Cassandra) vs vendor lock-in y costo a escala (DynamoDB).

> [!question] ¿Qué NO aparece en un DynamoDB Stream?
> Los writes que solo tocan una GSI — el stream únicamente refleja modificaciones de la tabla base.

## ¿Qué aplica a mi caso? (árbol de decisión)
- ¿Necesitás joins, aggregations o queries ad-hoc flexibles? → Postgres (o similar relacional), no DynamoDB.
- ¿Tus access patterns ya están claros y son estables? → DynamoDB, y considerá single-table design si hay relaciones parent-child que siempre se leen juntas.
- ¿Access patterns todavía inciertos / greenfield / GraphQL con resolvers por campo? → esperá con single-table design; empezá con tablas por entidad o directamente Postgres.
- ¿Escritura masiva sostenida y controlás vos la infraestructura? → evaluá [[Cassandra]] (LSM tree, write-optimized) contra el costo/ops de mantener el cluster vos mismo.
- ¿Necesitás lecturas en microsegundos sobre hot keys read-heavy? → sumá DAX antes de asumir que hace falta Redis aparte.
- ¿Un patrón de acceso concentra tráfico en pocas keys (hot partition)? → Adaptive Capacity ayuda, pero si el techo de 3,000 RCU/1,000 WCU por partición no alcanza, la salida es Write Sharding.
- ¿Necesitás la tabla disponible con lectura/escritura local en varias regiones? → Global Tables, aceptando Last-Writer-Wins como mecanismo de resolución de conflictos.
- ¿Preocupa el vendor lock-in a largo plazo? → es un costo real de DynamoDB; sopesalo contra el ahorro operacional de no correr vos el cluster.

## 🔗 Conexión con el vault
- MOC: [[_Databases|Databases]]
- Contraste directo de storage engine y particionamiento: [[Cassandra]] (LSM tree, write-optimized, [[Consistent Hashing]] en ring)
- Otras bases del vault: [[Redis]], [[Elasticsearch]] (target típico de DynamoDB Streams para sync de índices)
- Streaming/eventos: [[Kafka]]
- Patrones de particionamiento y distribución: [[Consistent Hashing]], [[Sharding]], [[Primary-Replica]], [[Quorum]]
- Durabilidad y replicación: [[Write-Ahead Log]], [[Vector Clocks]]
- Change Data Capture: [[Change Data Capture]]
- Idempotencia y concurrencia: [[Idempotency]], [[Idempotency Key]], [[Distributed Lock]]
- Transacciones distribuidas: [[Two-Phase Commit]], [[Saga]]
- Patrones de arquitectura relacionados: [[CQRS]]
- Patrones de caching (DAX): [[Cache-Aside]], [[Read-Through]], [[Write-Through]]
