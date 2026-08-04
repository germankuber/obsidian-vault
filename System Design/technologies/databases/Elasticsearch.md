---
title: Elasticsearch
created: 2026-07-01
tags:
  - system-design/databases
  - type/technology
  - status/permanent
aliases:
  - Elasticsearch
  - ES
updated: 2026-07-05
reading:
  total_words: 3343
  read_words: 0
  pct: 0
  last_read: 2026-07-01
---

# Elasticsearch

> [!note] Tesis operativa
> Elasticsearch NO es una base de datos: es un **framework de orquestación distribuida** sobre **Apache Lucene** (el motor de búsqueda de bajo nivel que hace el trabajo pesado). ES aporta coordinación de cluster, APIs, agregaciones y near-real-time; Lucene aporta las estructuras (segmentos inmutables, inverted index, doc values) que hacen la búsqueda rápida. Todo lo demás —[[Sharding]], réplicas, coordinating nodes— es escalabilidad y disponibilidad sobre "una gran bolsa de índices de Lucene".

## Marco mental (leé esto primero)

1. **Dos capas, dos responsabilidades.** Lucene es la librería Java que sabe indexar y buscar texto en un solo proceso; Elasticsearch la envuelve y la distribuye — cluster, réplicas, routing, APIs REST/JSON. Cuando algo es rápido (búsqueda) es mérito de Lucene; cuando algo es escalable (miles de millones de docs) es mérito de ES.
2. **Todo gira alrededor del mapping.** El mapping decide qué es buscable y cómo — la diferencia entre `keyword` y `text` es la bisagra de todo: exact-match tipo hash table vs full-text tokenizado con **Inverted Index**. Mapear de más desperdicia memoria; mapear de menos deja campos inconsultables.
3. **Inmutabilidad de los segmentos de Lucene explica casi todo el comportamiento interno.** Escribir, actualizar y borrar en ES son en el fondo variaciones de "insertar en un segmento nuevo" — de ahí sale el near-real-time, el costo de los updates, el rol del merge y hasta por qué la búsqueda es tan rápida (todo cacheable porque nada cambia).
4. **ES no es tu source of truth.** Case típico: Postgres/DynamoDB como store autoritativo, replicado a ES vía **Change Data Capture** (CDC) para full-text search, analytics y (cada vez más) búsqueda vectorial/semántica.

## Conceptos básicos

- **Documento**: unidad buscable = objeto JSON (un libro, un post, una reseña).
- **Índice**: colección de documentos ≈ tabla de una DB relacional. ⚠️ término sobrecargado — también nombra las estructuras internas (**Inverted Index**) que aceleran la búsqueda.
- **Mapping**: el schema del índice — define fields, tipos y cómo se procesan/indexan. Decide qué es buscable.
- **Field**: cada par clave-valor, tipado por el mapping.

![[elasticsearch-01-key-concepts.svg]]
*Conceptos básicos: Documents / Indices / Mappings / Fields*

Documento ejemplo:

```json
{
  "id": "XYZ123",
  "title": "The Great Gatsby",
  "author": "F. Scott Fitzgerald",
  "price": 10.99,
  "createdAt": "2024-01-01T00:00:00.000Z"
}
```

**`keyword` vs `text`** (la distinción clave del mapping): `keyword` = valor completo, NO tokenizado → lookup exacto tipo hash table (ej. `id`). `text` = tokenizado → **Inverted Index** para full-text search (ej. `title`). ⚠️ Costo de mapping: mapear 10 campos cuando solo 2 son buscables desperdicia memoria del índice.

```json
{
  "properties": {
    "id": { "type": "keyword" },
    "title": { "type": "text" },
    "author": { "type": "text" },
    "price": { "type": "float" },
    "createdAt": { "type": "date" }
  }
}
```

## Uso básico — API REST

Crear índice (mapping dinámico, 1 shard, 1 réplica):

```json
// PUT /books
{
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 1
  }
}
```

Setear mapping (notar `reviews` como **Nested fields**):

```json
// PUT /books/_mapping
{
  "properties": {
    "title": { "type": "text" },
    "author": { "type": "keyword" },
    "description": { "type": "text" },
    "price": { "type": "float" },
    "publish_date": { "type": "date" },
    "categories": { "type": "keyword" },
    "reviews": {
      "type": "nested",
      "properties": {
        "user": { "type": "keyword" },
        "rating": { "type": "integer" },
        "comment": { "type": "text" }
      }
    }
  }
}
```

Trade-off nested vs índice separado: si las reviews se actualizan poco y se consultan mucho → anidarlas. Si no → índice separado.

Agregar documento (`POST /books/_doc`) devuelve `_version` (base del **Optimistic Concurrency Control**):

```json
{
  "_index": "books",
  "_id": "kLEHMYkBq7V9x4qGJOnh",
  "_version": 1, // NOTE!
  "result": "created",
  "_shards": { "total": 2, "successful": 1, "failed": 0 },
  "_seq_no": 0,
  "_primary_term": 1
}
```

Actualizar — 3 estrategias: PUT documento entero (riesgoso) · `PUT ?version=1` (**Optimistic Concurrency Control** — falla si la versión cambió) · `POST /_update` con doc parcial.

```json
// POST /books/_update/kLEHMYkBq7V9x4qGJOnh
{ "doc": { "price": 14.99 } }
```

Search (JSON, tipo SQL). `match` + `bool`/`must`/`range` + `nested`:

```json
// GET /books/_search
{
  "query": {
    "nested": {
      "path": "reviews",
      "query": {
        "bool": {
          "must": [
            { "match": { "reviews.comment": "excellent" } },
            { "range": { "reviews.rating": { "gte": 4 } } }
          ]
        }
      }
    }
  }
}
```

Respuesta trae `took`, `hits.total`, `max_score`, `_score` por resultado, `_source`.

## Búsqueda geoespacial

Dos tipos de campo: **geo_point/geo_shape** — `geo_point` (lat/lon puntual) y `geo_shape` (polígonos/líneas/círculos). Query más común `geo_distance`:

```json
// GET /restaurants/_search
{
  "query": {
    "geo_distance": {
      "distance": "5km",
      "location": { "lat": 40.7128, "lon": -74.0060 }
    }
  }
}
```

Por debajo: **Geohash** + **BKD Tree** (variante de k-d tree para block storage) + estructuras tipo R-tree.

## Sort

5 variantes: básico (`price` asc) · multi-campo · por script (Painless, `doc['price'].value * 0.9`) · sobre nested (`reviews.rating`, `mode: max`) · por **Relevance scoring** (`_score`, default **BM25** — ver más abajo).

## Relevancia y scoring

**Relevance scoring** es el corazón de por qué un resultado aparece antes que otro. Desde ES 5.0 (Lucene 6, ~2016) el default de ranking es [[BM25]], que reemplazó al **TF-IDF** puro. Tres parámetros gobiernan el score: `k1` (≈1.2, satura el peso de la frecuencia de término — "la décima ocurrencia de una palabra no debería pesar como la primera"), `b` (≈0.75, normalización por longitud de documento; `0` = apagado, `1` = normalización completa) y el componente IDF (términos raros pesan más). `_score` es, por default, el resultado de este cálculo.

Para inclinar el ranking manualmente se usa **function_score**/boosting a nivel query (el boosting a nivel índice está deprecado — preferí siempre boost en query-time). Funciones de decay disponibles: `gauss` (campana, buen default), `exp` (cae rápido y después se aplana, bueno para long-tail), `linear` (corte duro), todas parametrizadas por `origin`/`scale`/`decay`; más `field_value_factor` (multiplica el score por un campo numérico como popularidad, típicamente amortiguado con log). Esta es la respuesta esperada a "¿cómo boostearías resultados recientes/populares/cercanos?".

## Analyzers y tokenización

Un **Analyzer** (**Analyzers** es la categoría general) = **Tokenization** + filtros (`lowercase`, **Stemming**, **Synonyms**). Corre en index-time (solo campos `text`) y en query-time. Esto es exactamente POR QUÉ importa la distinción `keyword` vs `text` de más arriba: `text` pasa por el analyzer y termina en el **Inverted Index**; `keyword` no se toca.

Para autocomplete hay dos caminos: **Edge n-grams** (indexás prefijos del término, tipo "search-as-you-type") o el **Completion Suggester** (estructura dedicada respaldada por un **FST** — finite state transducer — mucho más eficiente en memoria que n-grams para sugerencias).

## Aggregations — ES como motor de analytics

Más allá de la búsqueda, **Aggregations** convierten a Elasticsearch en un motor de analytics sobre el mismo índice: agregaciones *bucket* (agrupan, tipo `GROUP BY`) y *metric* (`avg`/`sum`/`percentiles`), anidables entre sí. Es una capacidad distinta de la búsqueda — el mismo dato que indexaste para full-text sirve también para dashboards y métricas, sin duplicar a otro sistema.

## Paginación — cadena causal de 3 estrategias

1. **From/Size** — simple (`from`/`size`), pero ineficiente pasado ~10.000 resultados → por eso...
2. **Search After** — usa los sort values del último resultado como cursor. Eficiente, pero *stateful* + forward-only + inconsistente → por eso...
3. **Point in Time** (cursors/PIT) — `POST /_pit?keep_alive=1m` + `search_after` da una vista congelada consistente, a costa de overhead. Es la versión ES de [[Cursor Pagination]] — ver también [[Pagination]].

```json
// GET /my_index/_search
{
  "size": 10,
  "query": { "match": { "title": "elasticsearch" } },
  "sort": [ {"date": "desc"}, {"_id": "desc"} ],
  "search_after": [1463538857, "654323"]
}
```

## Búsqueda vectorial y semántica

El tema caliente de toda entrevista de ES en 2025/26. Un campo `dense_vector` guarda embeddings, y se consulta con `kNN` (k-nearest-neighbors). Por debajo, ES usa **HNSW** (Hierarchical Navigable Small World) — un grafo por segmento de Lucene, parametrizado por `m` (≈16 aristas por nodo) y `ef_construction` (≈100, calidad de construcción del grafo), y `num_candidates` en tiempo de query controla el trade-off recall/latencia. HNSW es **ANN** (approximate nearest neighbor) — la alternativa exacta es `script_score` (fuerza bruta, O(n)), solo viable en sets chicos o pre-filtrados.

La métrica de similaridad importa y es una decisión real de pipeline de embeddings: `cosine` (auto-normaliza, score = `(1+cos)/2`) vs `dot_product` (el más rápido, pero exige vectores pre-normalizados a norma unitaria) vs `max_inner_product`. Elegir mal (`dot_product` con vectores no normalizados) rompe el ranking silenciosamente.

Los vectores tienen que vivir en memoria (off-heap) para dar buena latencia — por eso **Quantization** (int8/int4/BBQ, hasta 32x de reducción) es central para el costo en producción, no un detalle de implementación.

[[Hybrid Search]] combina [[BM25]] (léxico) + vector (semántico), fusionados con [[Reciprocal Rank Fusion]] (RRF, `rank_constant` default 60) en vez de sumar los scores directamente. La razón: BM25 es *unbounded* (~0–30+) y `cosine` está acotado a [0,1] → sumarlos ingenuamente deja que una señal domine a la otra según la escala. RRF es *rank-based*, por lo tanto invariante a la escala — es una pregunta clásica de entrevista ("¿por qué no sumás los scores?").

Awareness-level: **ELSER** es el modelo de semántica dispersa (*sparse*) propio de Elastic (`sparse_vector`/`text_expansion`) — un punto intermedio entre BM25 (sin semántica) y **Embeddings** densos (necesitan tuning de dominio), sin requerir fine-tuning. Relacionado: **Vector Database** como categoría — ES compite ahí, pero nació como motor de texto.

## Escalado operativo — ILM y shard sizing

**ILM** (Index Lifecycle Management) automatiza `rollover` (crear un índice nuevo cuando el actual supera un umbral), índices por ventana de tiempo, y mueve datos entre tiers **hot-warm-cold-frozen** según costo/latencia. Es el patrón clásico para logs/observabilidad a escala ("10TB/día de logs → índices time-based + ILM + frozen en S3").

Regla de dedo para **Shard sizing**: ~50GB por shard; el oversharding (demasiados shards chicos) agrega overhead de coordinación y memoria sin beneficio. El número de shards de un índice se fija en su creación — por eso existe `rollover`: en vez de resharding, creás un índice nuevo con la config correcta y rotás.

## Cómo funciona por dentro

### Arquitectura de cluster — 5 tipos de nodo

![[elasticsearch-02-sequence.svg]]
*Arquitectura de cluster: flujo ingest → data → coordinating*

| Nodo | Rol |
|---|---|
| **Master nodes** | operaciones de cluster (topología, metadata) |
| **Data nodes** | almacenan documentos, en tiers hot/warm/cold/frozen |
| **Coordinating nodes** | reciben/enrutan search, hacen **Query Planning** |
| **Ingest nodes** | transforman documentos antes de indexar |
| **ML nodes** | tareas de machine learning |

Un nodo puede combinar roles. Seed nodes (master-eligible) participan de una elección de líder → siempre hay un único master activo.

### Data Nodes — muñecas rusas

Separan el `_source` crudo de los índices Lucene. Un request de búsqueda corre en 2 fases: *query* (identifica qué documentos matchean) + *fetch* (trae el contenido de esos IDs).

![[elasticsearch-03-russian-dolls.svg]]
*Data nodes en capas: índice → shards → Lucene → segmentos*

- **Shards** — parten los datos entre hosts (búsquedas en paralelo, merge de resultados en el coordinator). Cada shard es 1:1 con un índice de Lucene.
- **Réplicas** — copia exacta de un shard → alta disponibilidad + throughput de lectura (X TPS × Y réplicas).

### Lucene Segment CRUD — inmutabilidad

Los índices de Lucene están hechos de **Lucene Segment**s inmutables.

![[elasticsearch-04-lucene-segments.svg]]
*Lucene Segment CRUD: cómo insert/update/delete interactúan con la inmutabilidad*

- **Insert**: se agrega a un segmento en memoria → se batchea → **Flush** a disco.
- **Segment Merge**: fusiona varios segmentos en uno nuevo, más eficiente.
- **Delete**: no borra in-place — marca un **Soft Delete** (se limpia recién en el próximo merge).
- **Update**: soft delete del documento viejo + insert del nuevo → ⚠️ un update es estructuralmente más caro que un insert.

6 beneficios de la **Immutability**: mejor performance de escritura · cacheo seguro (nada cambia bajo el caché) · concurrencia simple (sin locks de escritura sobre datos existentes) · recuperación fácil ante fallos · mejor compresión · búsquedas más rápidas.

### Las 2 estructuras clave del segmento

**Inverted Index** — "el corazón de Lucene": mapea cada token → la lista de documentos que lo contienen. Convierte un scan O(n) en un lookup O(1).

![[elasticsearch-05-inverted-index.svg]]
*Inverted Index: token → documentos*

**Doc Values** — representación columnar contigua de un solo field para todos los docs del segmento → hace eficientes el sort y las agregaciones (que necesitan barrer un campo entero, no un documento entero).

### Coordinating Nodes — query planning

**Query Planning**: ejemplo clásico, buscar "bill nye" — el token "bill" tiene millones de entradas en el **Inverted Index**, "nye" unos cientos → el ORDEN en que se intersectan las listas de postings cambia la performance en órdenes de magnitud (intersectar la lista corta primero). ES usa estadísticas del índice para elegir ese orden.

## Write path y near-real-time

Elasticsearch se llama a sí mismo *near-real-time*, no *real-time*, y hay una razón concreta: el **Refresh interval** (default 1 segundo) — un documento escrito NO es buscable hasta que corre el próximo refresh, que abre un nuevo segmento en memoria visible para búsqueda. Esto conecta directo con la sección de inmutabilidad de arriba: cada refresh es, en el fondo, "crear un segmento nuevo".

Durabilidad: el **Translog** cumple el mismo rol que un [[Write-Ahead Log]] — cada escritura se loguea ahí antes de confirmarse, para no perderla si el nodo cae antes del próximo **Flush** (que persiste los segmentos a disco y limpia el translog). El **Segment Merge** visto arriba compacta segmentos en el tiempo, a costa de I/O periódico.

Para ingesta de alto volumen se usa la **Bulk API** (batchea muchos writes en un solo request) — y una práctica común es subir el `refresh_interval` (o desactivarlo) durante una carga masiva, porque cada refresh de más es overhead que no aporta nada si nadie está leyendo todavía esos datos.

### Consistencia y escalado de lecturas

**Consistency model**: réplicas primary-replica con replicación síncrona limitada por `wait_for_active_shards` (cuántas copias deben confirmar antes de reconocer la escritura). Combinado con el near-real-time de arriba, una lectura inmediatamente después de un write puede no verlo todavía. En términos de **CAP Theorem**, ES prioriza disponibilidad/throughput sobre linealizabilidad estricta.

Escalar lecturas ≠ escalar escrituras: las réplicas suman **Read scaling**, hay **Query cache** para queries repetidas, y **Adaptive replica selection** enruta cada request a la réplica menos cargada. Ninguno de estos mecanismos ayuda si el cuello de botella es la tasa de escritura del shard primario.

## En tu entrevista

ES casi siempre se conecta vía [[Change Data Capture|CDC]] (ej. Debezium sobre Kafka) a un store autoritativo (Postgres / [[DynamoDB]]). 6 precauciones:

1. **No es una base de datos** — no la trates como source of truth.
2. **Read-heavy** por diseño — optimizada para consultas, no para ser el sistema transaccional.
3. **Consistencia eventual** — ver near-real-time y consistency model arriba.
4. **No relacional** — desnormalizá, igual que en Cassandra: no hay JOINs baratos.
5. **No todo lo necesita** — con menos de ~100k documentos, una query simple sobre la DB relacional alcanza; sumar ES es complejidad sin retorno.
6. **Sincronización = drift** — todo pipeline CDC puede desincronizarse; necesitás reconciliación o reindex periódico.

5 lecciones que se llevan a cualquier entrevista de diseño: inmutabilidad como estrategia de performance · separar la capa de query de la de storage (coordinating vs data nodes) · estrategias de indexación (mapping, analyzers) según el access pattern · trade-offs distribuidos (**CAP Theorem**) · estructuras de datos eficientes (skip lists, finite state transducers) como la diferencia real entre un motor de búsqueda y un `LIKE '%...%'`.

## ¿Cuándo usar Elasticsearch? (decisión)

| Usalo si… | Evitalo si… |
|---|---|
| Full-text search, relevancia (BM25/vector/híbrido) | Necesitás ACID / transacciones fuertes |
| Analytics/agregaciones sobre el mismo dato indexado | El dataset es chico (<100k docs) y una query SQL alcanza |
| Autocomplete, geo search, logs a escala (ILM) | Necesitás que sea el source of truth |
| Búsqueda vectorial/semántica (kNN, hybrid search) | No podés tolerar staleness (near-real-time, no real-time) |
| Ya tenés un store autoritativo y sumás ES vía CDC | No tenés capacidad operativa para mantener un cluster + pipeline de sync |

## 🎯 Preguntas de retrieval

> [!question] ¿Por qué un update en Elasticsearch es más caro que un insert?
> Porque los segmentos de Lucene son inmutables: un update no modifica el documento in-place, hace un soft delete del viejo (lo marca como borrado, sin liberar espacio todavía) e inserta uno nuevo. El espacio del viejo recién se recupera en el próximo segment merge.

> [!question] ¿Por qué `text` habilita full-text search y `keyword` no?
> Porque `text` pasa por un analyzer (tokenización + lowercase + stemming + synonyms) y termina en el inverted index, mientras que `keyword` guarda el valor completo sin tokenizar para lookup exacto tipo hash table. Mapear un campo como el tipo equivocado lo hace inconsultable de la forma que necesitás.

> [!question] ¿Por qué Search After es mejor que From/Size pasados los primeros miles de resultados, pero no es la solución final?
> From/Size se degrada porque cada página tiene que traer y descartar todos los resultados anteriores. Search After usa los sort values del último resultado como cursor, evitando ese costo — pero es stateful, solo avanza hacia adelante, y no da una vista consistente si el índice cambia mientras paginás. Por eso para consistencia real se usa Point in Time + search_after.

> [!question] ¿Por qué en hybrid search no podés simplemente sumar el score de BM25 y el de similitud vectorial?
> Porque tienen escalas incompatibles: BM25 es unbounded (puede dar 0 a 30+), mientras que la similitud coseno está acotada a [0,1]. Sumarlos directamente deja que la señal de mayor magnitud domine el ranking sin que eso refleje relevancia real. Reciprocal Rank Fusion resuelve esto fusionando por **rank** en vez de por score, lo que lo hace invariante a la escala de cada señal.

> [!question] ¿Por qué Elasticsearch es "near-real-time" y no consistente al instante?
> Porque un documento recién escrito vive en un segmento en memoria que no es buscable hasta el próximo refresh (default cada 1 segundo). Antes de ese refresh, el dato está durable en el translog pero no aparece en resultados de búsqueda.

> [!question] ¿Por qué importa la quantization en búsqueda vectorial si "solo" reduce tamaño en disco?
> Porque los vectores densos necesitan vivir en memoria (off-heap) para dar latencias de kNN aceptables, y los embeddings de alta dimensión son caros en RAM. Quantization (int8/int4/BBQ) reduce ese footprint hasta 32x, lo cual es la diferencia entre poder o no poder pagar el cluster a la escala de producción.

> [!question] ¿Cuándo NO usarías Elasticsearch?
> Cuando el dataset es chico (menos de ~100k documentos, una query SQL simple alcanza), cuando necesitás que sea el source of truth (no tiene ACID real ni es transaccional), o cuando el caso de uso es write-heavy con necesidad de consistencia inmediata — ES está optimizado para lectura y búsqueda, no para eso.

## 🔗 Conexión con el vault

- [[_Databases|Databases]] — MOC de bases de datos.
- [[Cassandra]] y [[Redis]] — los otros dos deep-dives de bases NoSQL del vault; contrastá: Cassandra prioriza write throughput con LSM trees, Redis prioriza latencia en memoria, Elasticsearch prioriza relevancia de búsqueda sobre Lucene.
- [[Sharding]], [[Primary-Replica]], [[Consistent Hashing]] — patrones de distribución que ES comparte con Cassandra y Redis, aunque implementados distinto (shards fijos por índice, no consistent hashing).
- [[Write-Ahead Log]] — el translog de ES es una instancia concreta de este patrón.
- [[Change Data Capture]] — el mecanismo típico para mantener ES sincronizado con el store autoritativo.
- [[Pagination]] — Search After y Point in Time son la implementación ES de este problema general.
- [[BM25]], [[Hybrid Search]], [[Reciprocal Rank Fusion]] — el trío de relevancia/ranking que conecta este note con el subárbol de AI/RAG del vault.
