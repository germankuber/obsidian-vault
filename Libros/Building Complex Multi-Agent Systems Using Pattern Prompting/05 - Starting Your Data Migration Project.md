---
title: Starting Your Data Migration Project
libro: Building Complex Multi-Agent Systems Using Pattern Prompting
autor: Packt
capitulo: 5
created: 2026-06-20
tags:
  - libros/building-complex-multi-agent-systems-using-pattern-prompting
  - type/concept
  - status/permanent
aliases:
  - Starting Your Data Migration Project
  - Cap 5 - Starting Your Data Migration Project
updated: 2026-07-05
---
# Starting Your Data Migration Project

> [!info] Capítulo 5 · *Building Complex Multi-Agent Systems Using Pattern Prompting* — Packt (ISBN 9781806114290)
> El **primer paso para construir aplicaciones agénticas** es **encontrar y categorizar la data valiosa** para la app y **migrarla a una [[Vector Database|vector database]]**, vía un workflow que primero **chunkea** la data, luego crea **[[Embeddings|embeddings]]** de esos chunks y finalmente los **almacena**. El valor enterprise no viene de mandarle solo texto al LLM, sino de **incluir textos e imágenes adicionales con el prompt** (información, fórmulas, algoritmos, definiciones). El capítulo operacionaliza esto **a escala**: identificar **documentos de datos Y de proceso**, construir ingestion pipelines con un **[[ETL]] tool maduro** ([[Airbyte]]) para data distribuida multi-formato, y atender los engineering considerations (throughput vía **[[EPS (Embeddings per Second)|EPS]]**, scalability, cost). También cubre cómo mantener la data actual ([[Incremental Sync|incremental updates]] / [[Change Data Capture|CDC]]), asegurar **security/privacy/compliance** (PII, GDPR, SOX) y mejorar el retrieval ([[Data Cleaning|data cleaning]], [[Hybrid Search|hybrid search]], [[Knowledge Graph|knowledge structuring]]). Navegá: [[_Building Complex Multi-Agent Systems Using Pattern Prompting|Building Complex Multi-Agent Systems]] · anterior [[04 - Building Your First RAG App]] · siguiente [[06 - Ingesting Data Using Airbyte and Pinecone]].

## Resumen

El cap. 4 cerró prometiendo los **data pipelines que mueven datos a la vector DB**; este capítulo arranca ese **proyecto de data migration**. La idea central: para que una app GenAI dé valor enterprise hay que **incluir data adicional con el prompt** (no solo texto), y eso requiere **encontrar los documentos relevantes y migrarlos a una [[Vector Database|vector database]]** con el workflow **chunk → embed → store**, para luego recuperar los chunks relevantes vía **semantic query** en tiempo de consulta. El capítulo operacionaliza este proceso **a escala**, recorriendo cinco grandes temas en el orden real del cuerpo: **identificar los documentos**, **elegir el ETL tool**, **planear el pipeline**, **optimizar costos/throttling** y **evaluar la calidad del retrieval**.

Primero, **Identifying documents**: hay que buscar no solo docs de **raw data** sino docs de **procesos y fórmulas** que el LLM debe usar. El **ejemplo ZZZ Auto Insurance** lo ilustra: para analizar estados financieros con el estándar **non-GAAP IFRS 17**, los contadores suben el **manual IFRS 17** (información "how to" + ejemplos) junto a los **datos financieros** (lo que se opera), y un prompt experto instruye al LLM a usar el manual. Es un **paradigma poderoso** —la **separación data/funciones de una clase, pero a nivel documento** (Fig 5.1, analogía OO).

Segundo, **Selecting the right ETL tool**: los desafíos enterprise (docs dispersos en muchos sistemas/medios y formatos, sin categorizar por topic, con estructuras distintas) más un **checklist de 6 preguntas** (errores, security regulatoria, updates, versioning, recolección por topic, paralelismo) llevan a elegir entre **tres opciones** (code your own, GenAI tools como LangChain, mature ETL platforms). El libro elige la **opción 3** y usa **[[Airbyte]]** (open-source, gran comunidad, multi-lenguaje, sin obligar a aprender Python), retomando del cap. 1 la advertencia sobre la **"frothy exuberance"** alrededor de la AI.

Tercero, **Planning your data pipeline** (los 5 ítems previos): **capacity planning** con la métrica correcta **[[EPS (Embeddings per Second)|EPS]]** (no gigabytes), análogo a **RPS**; **handling change** con **full reloads vs [[Incremental Sync|incremental updates]]** (cursor field `updated_at`, hash, **[[Change Data Capture|CDC]]**, la metáfora del **conveyor belt**); **security/privacy/compliance** (classification levels, masking, encryption, audit trails, versioning); **parallel pipelines y CI/CD** (3 stages concurrentes Extract/Transform/Load, DevOps discipline); y **testing/validation** (unit por stage, integration con top-5, continuous validation muestreando 1%). Cuarto, **Cost optimization and throttling**: rate limits, **dynamic throttling** con **back-off exponencial**, batch embeddings. Quinto, **Evaluating retrieval quality**: **precision@k / recall@k**, baseline **precision@5 > 0.8**, y técnicas avanzadas opcionales (**taxonomy discovery**, **hybrid search/BM25**, **data cleaning**, **graph databases/GraphRAG**) que transforman la vector DB en una **knowledge platform**. El **Summary** enmarca todo como un ejercicio de **systems-engineering** más que de plumbing; el **cap. 6** construye el pipeline completo y funcional con **[[Airbyte]] + [[Pinecone]]**, con observability y error recovery.

## Identifying documents to ingest into the vector database

Cuando usamos un LLM en el día a día (ChatGPT, Grok, Gemini) solemos conformarnos con escribir solo texto. Pero **el verdadero beneficio enterprise aparece cuando incluís textos e imágenes adicionales con los prompts**, permitiéndole al LLM considerar esa data en su respuesta. La data puede ser **información, fórmulas, algoritmos, definiciones, o cualquier cosa que el LLM encuentre útil**.

Al buscar documentos relevantes hay que pensar no solo en **documentos que aportan raw data**, sino también en **documentos que contienen información sobre procesos y fórmulas** que querés que el LLM use. Estos suelen combinarse con los documentos que contienen la data sobre la que el LLM debe operar.

> [!note] El paradigma poderoso de este capítulo: combinar **documentos de información/proceso** (el "cómo") con **documentos de datos** (el "qué"). Es análogo al concepto de CS de **separar data y funciones en una clase**, solo que esta vez la separación es **a nivel documento** (Fig 5.1).

El ejemplo canónico del capítulo (reproducido verbatim como aparece destacado en el original):

> **ZZZ Auto Insurance**
> ZZZ Auto Insurance wants to analyze its financial statements using the non-standard (non-GAAP) accounting standard IFRS 17 and see how the results differ from GAAP. The accountants upload the IFRS 17 manual along with the company's financial documents. An LLM can then generate a summary of the IFRS 17-compatible report using an expertly written prompt that instructs the LLM to use the IFRS manual to determine how to generate the report.

Este ejemplo sube **dos tipos de data**:

- **El manual IFRS 17** — contiene información **"how to"** y ejemplos.
- **Los datos financieros** — la data sobre la que se opera.

Para usar ambos (información + proceso), escribirías un prompt del estilo `"use the IFRS 17 manuals to analyze the financial document provided…"` junto con **chunks del documento generados por una semantic query** a la vector database.

![[B34134_5_1.png]]
*Figure 5.1 – Separating data and logic in OO programming languages and RAG architecture*

> [!tip] Antes de aprovechar esta idea, hay que **encontrar los documentos y migrarlos a la vector database** para que los chunks relevantes puedan recuperarse en query time. La tarea de migración puede ser **chica** (unas pocas docenas de docs) o un **proyecto enorme** (miles de docs). De cualquier modo, **elegir un buen ETL tool te facilita mucho la vida**.

## Selecting the right ETL tool

Las empresas que usan documentos internos para potenciar GenAI enfrentan muchos desafíos. El primero: pueden tener **miles de documentos en varios sistemas internos**, y **no se pueden adjuntar más que unos pocos** al llamar al LLM —por las limitaciones de recursos del LLM y por el riesgo de que exponerlo a mucha data irrelevante le haga devolver resultados inesperados—. Hay que encontrar la forma de proveerle al prompt **solo las páginas, párrafos o secciones más relevantes** de la colección. El **primer paso de todo proyecto GenAI** es juntar todos los documentos que el LLM usará y **chunkearlos hacia una vector database**.

> [!note] **Desafíos al mover documentos a la vector DB en un entorno enterprise:**
> - Los documentos **no están en una sola ubicación ni en el mismo medio**: pueden estar dispersos en file servers, drives, relational databases, **SharePoint, Workday, AWS S3, GCS, HTTPS y SFTP**, entre otros.
> - Varios **tipos de documento** a ingestar: **PDF, spreadsheet, plain text, DOCX**.
> - El contenido a recuperar puede estar **disperso en cientos o miles** de documentos.
> - Los documentos pueden **no estar categorizados por topic**, requiriendo un **preprocessing step** para identificar topics.
> - Cada documento tiene **estructura distinta**: algunos PDFs con chapters/subchapters, otros unstructured.

A esto se suma definir estrategias para errores, políticas y versionado. El **checklist a considerar al planear la migración** (6 preguntas):

1. **¿Cómo manejás errores?** Cuando ocurre un error en el pipeline, ¿hacés **rollback del batch entero**, **logueás el file**, o tomás otra acción?
2. **¿Cómo aplicás las security rules** alrededor de requisitos regulatorios como **PII** (Personally Identifiable Information), **SOX** y **GDPR**?
3. Cuando los documentos cambian, **¿cómo actualizás la información, y cada cuánto?**
4. **¿Cómo manejás el versioning** cuando se libera una nueva versión de un documento?
5. **¿Cómo recolectás data de una colección por topic?**
6. **¿Cuántos pipelines de ingestión paralelos** deberías correr?

Lo mejor es arrancar buscando un **mature ETL tool** que se ajuste a las necesidades y expertise de la organización. Si la empresa ya tiene licenciado un ETL tool, ese sería el candidato top. Sino, hay tres opciones:

### Tabla 5.1 — Las 3 opciones de ETL

| Opción | Cuándo conviene | Riesgo / contra |
|---|---|---|
| **Code your own** | Solo si hay **relativamente pocos data repositories**. Librerías como **Spring Boot's ETL pipeline** dan la plumbing para un pipeline DIY | Esfuerzo de construcción; no escala a muchos repos |
| **GenAI tools como LangChain** | Opción que vale explorar | Dado lo **recientes** que son y su **limitada exposición enterprise**, no sorprendería que **underperformen en throughput, security y madurez** |
| **Mature ETL platforms** | Plataforma **testeada en muchos sistemas enterprise**, con soporte y documentación robustos | — (la elegida por el libro) |

> [!tip] El libro elige la **opción 3** (mature ETL platform). Muchos ETL tools **tardaron años** en alcanzar su madurez actual y ganarse la confianza de líderes técnicos y developers; mirando sus adoption curves, **varios años después de su desarrollo todavía estaban madurando**.

Como se mencionó en el **[[01 - Introduction Patterns, Abstractions, and the GenAI Landscape|Capítulo 1]]**, en tiempos de **frothy exuberance** —como hoy con la AI— emergen muchas tools que intentan reemplazar productos ya existentes, solo para **perder fuerza o flame out** por completo. Hay que **evaluar las tools nuevas con cuidado**, preguntando qué ofrecen **genuinamente nuevo** y si **alcanzaron la madurez** de las existentes.

> [!note] **La elección del libro: [[Airbyte]].** Tiene una **gran comunidad**, es **open-source** y soporta **múltiples lenguajes**. Consistente con el tema del libro, eligieron un producto que **no obliga a aprender un lenguaje nuevo como Python** para construir apps GenAI — aunque si ya sabés Python, Airbyte tiene un **excelente Python SDK**.

Más allá de la selección de tecnología, hay que **planear el proyecto cuidadosamente**: decisiones sobre inputs/outputs, **acceptable latency**, security y otras constraints. Los equipos de **Business e IT deben trabajar juntos** para identificar targets y constraints a corto y largo plazo.

## Planning your data pipeline

Antes de empezar a construir el data pipeline, hay que hacer **cinco cosas**:

1. **Capacity planning y throughput analysis**.
2. **Planear el incremental synchronization** de nuevas versiones de documentos.
3. **Construir para management robusto, testing y CI/CD pipelines**.
4. **Identificar requisitos de security, PII y compliance**.
5. **Diseñar un testing and validation plan** para ingesta continua.

### Capacity planning and throughput

Si construiste ETL pipelines antes, probablemente dimensionabas jobs calculando **gigabytes** (cuántos GB de PDFs procesar). Para vector ingestion, **la métrica correcta NO es gigabytes** sino **cuántos [[EPS (Embeddings per Second)|embeddings per second (EPS)]]** se pueden producir y escribir en la vector DB **sin exceder rate limits ni budget**.

> [!note] Así como los ingenieros tradicionales piensan en **requests per second (RPS)** en web services, los data engineers que trabajan con vector databases deben pensar en **embeddings per second (EPS)**. Cada **embedding call** cuenta contra una **API quota**, consume **network bandwidth** y **genera costo**.

Un **design goal realista** para un ingestion pipeline enterprise es **entre 50 y 200 EPS**, según el embedding provider. Más allá de eso, el bottleneck típicamente **se corre de compute a I/O** o a los **throttling limits** de la embedding API; si se necesita más EPS, hay que considerar **parallelization strategies**.

Hugging Face mantiene el **[[MTEB|MTEB Leaderboard]]** de embedding models con **speed metrics** (filtrar por **Speed**): `https://huggingface.co/spaces/mteb/leaderboard`. Para benchmarkear EPS uno mismo, el utility **`mteb`** es práctico: `https://github.com/embeddings-benchmark/mteb`.

> [!tip] **Estimar throughput** = multiplicar **tokens por chunk × cantidad promedio de chunks por documento**. Ejemplo concreto: un **PDF de 50 páginas ≈ 200 chunks**; ingestar **10,000 documentos** → **2 millones de embeddings**; a **100 EPS ≈ 5½ horas**. Estas cuentas back-of-the-envelope son **cruciales** porque, una vez iniciada la ingesta, **parar y reiniciar desde cero es costoso**.

La misma lógica aplica a updates: un **batch nocturno** que agrega o modifica el **1%** de los documentos puede representar **decenas de miles de embeddings**. El pipeline debe estar preparado para manejar estos **incremental loads sin frenar** otros procesos que dependen del vector store.

### Handling change and continuous operations

Las empresas modernas **rara vez tienen data estática**: los contratos se renuevan, los manuales se revisan, las regulaciones se actualizan. Un pipeline robusto debe **asumir que cualquier documento puede cambiar en cualquier momento**. Dos estrategias:

### Tabla 5.2 — Full reloads vs Incremental updates

| Estrategia | Qué hace | Trade-off |
|---|---|---|
| **Full reloads** | **Borrar y re-embeber** la colección entera | **Simple pero caro** |
| **Incremental updates** | **Detectar y re-embeber solo lo cambiado** | Reduce costos y mantiene la vector DB actual |

**[[Airbyte]] y similares** soportan ambos vía su feature de **[[Incremental Sync|incremental sync]]**: definís un **cursor field** como `updated_at` o tomás un **hash del contenido**; cuando el field cambia, **solo ese record se reprocesa**.

Para requisitos **real-time**, se usan connectors de **[[Change Data Capture|Change Data Capture (CDC)]]**: CDC **vigila el transaction log** del source system y **streamea inserts, updates y deletes** hacia el pipeline. Aunque el true event-by-event streaming no siempre es necesario, **CDC con un polling interval corto** (por ejemplo, **un minuto**) logra **near-real-time synchronization**.

> [!note] **Modelo mental: el conveyor belt.** Imaginá una cinta transportadora — los documentos nuevos se agregan por un extremo, los procesados salen por el otro, y **la cinta nunca para**. El pipeline debe ser capaz de **operación continua aun cuando connectors individuales o APIs externas fallen**. Cada fallo debe **loguearse, reintentarse, y si hace falta ponerse en cuarentena** para inspección manual.

> [!warning] En una organización moderna, **el pipeline no será aprobado** salvo que demuestres que cumple los requisitos de **security and privacy compliance**. Es crítico que la solución ETL tenga una **security architecture apropiada**, incluyendo **encryption at rest y encryption in transit**.

### Ensuring security, privacy, and compliance

La **security debe abordarse ANTES de armar el pipeline**. Los documentos suelen contener **PII, trade secrets o correspondencia confidencial**; moverlos por **third-party services sin controles** es riesgoso y puede **violar leyes como GDPR o SOX**.

> [!note] **Los embeddings cargan la misma sensibilidad que los documentos fuente.** Antes de la ingesta, definí **classification levels** —por ejemplo **public, internal, restricted y confidential**— que determinan **qué connector, storage system o embedding service** puede procesar la data. Los documentos muy sensibles podrían embeberse con un **modelo on-premises** en vez de una cloud API.

### Tabla 5.3 — Técnicas de privacy y security

| Técnica | Qué implica |
|---|---|
| **Field-level masking** | **Redactar** phone numbers, IDs o account numbers **antes de embeber** |
| **Encryption at rest and in transit** | **TLS** para transfers y **AES-256** para chunks y embeddings almacenados |
| **Audit trails** | Guardar metadata de **quién subió qué y cuándo** |
| **Versioning** | Al aparecer una versión nueva del documento, **retener la anterior** para trazabilidad |

> [!tip] En organizaciones grandes, los **compliance teams** pueden exigir **controles demostrables**. **Airbyte Enterprise**, por ejemplo, soporta **role-based access, secret vault integration y encrypted logs**.

### Managing parallel pipelines and CI/CD

El **paralelismo acorta drásticamente el tiempo de ingestión**, pero puede causar **inestabilidad** si no se planea bien. Cada connector debe declarar su **concurrency level**. Para tasks **CPU-bound** (como **text extraction de PDFs**) se puede **scale horizontally**; para tasks **API-bound** (como el embedding), subir la concurrency **demasiado agresivamente lleva a throttling y requests fallidos**.

Un diseño balanceado típicamente separa el pipeline en **tres stages concurrentes**:

1. **Extraction** — **leer y parsear** documentos, produciendo **clean text**.
2. **Embedding** — **generar los vector embeddings**, posiblemente **en batches**.
3. **Loading** — **escribir embeddings y metadata** a la vector database.

![[B34134_5_2.png]]
*Figure 5.2 – The three stages of an ETL pipeline: Extract, Transform, Load*

> [!note] **Cada stage escala independientemente**, comunicándose a través de **message queues o temp storage** (los queues del [[04 - Building Your First RAG App|cap. 4]]). Cada stage puede **acknowledge la completitud de los mensajes**, asegurando que **ningún documento se pierda** aunque un subproceso crashee a mitad de camino.

Un **GenAI pipeline es software** y merece la **misma DevOps discipline que cualquier microservice**:

- **Connector configurations y transformation code bajo version control.**
- **Testear en staging** antes de pushear a producción.
- Tratar los **schema changes** (como alterar metadata fields) como **migrations** que requieren **review y rollback plans**.
- **Continuous Integration (CI)** — cada cambio (por ejemplo, pasar de `text-embedding-ada-002` a `text-embedding-3-large`) dispara **tests automáticos** que verifican **compatibility de vector dimensions y retrieval accuracy**.
- **Continuous Deployment (CD)** — promueve el cambio a producción **cuando todos los tests pasan**.

> [!tip] Combinado con **CDC o incremental syncs**, CI/CD permite **evolucionar el pipeline de forma segura sin downtime**: los ingenieros pueden rollout de mejoras en **chunking logic, embedding parameters o security filters** con confianza de que la data existente permanecerá consistente.

### Testing and validation

Testear un ingestion pipeline va **más allá de chequear si los files llegaron** al destino: hay que confirmar que **el significado semántico sobrevivió el viaje**.

- **Unit tests por stage:**
  - **Extraction tests** — verificar que la conversión **PDF-to-text preserva paragraphs y headings**.
  - **Embedding tests** — **comparar cosine similarity** entre **pares conocidos de oraciones**.
  - **Load tests** — asegurar que las **vector dimensions matchean** el output del modelo.
- **Integration tests con sample prompts** — consultar la vector DB con **algunas preguntas conocidas** e **inspeccionar manualmente los top 5 chunks** devueltos. Si los matches son **semánticamente relevantes**, los embeddings y el metadata schema están sanos; si no, **revisar chunk size o preprocessing logic**.
- **Continuous validation** — a medida que llegan documentos nuevos, **samplear el 1% de los embeddings y recomputar similarities**. Los **drifts en average similarity o retrieval accuracy** suelen indicar **silent corruption o preprocessing inconsistente** entre runs.

## Cost optimization and throttling

Como **cada token enviado a embeber representa un costo**, el **throttling no es solo un control técnico sino financiero**. La mayoría de los embedding providers **publican rate limits** (por ejemplo, **3,500 requests per minute**). Muchos ETL tools permiten **per-connector rate limits**, para asegurar que el tráfico se mantenga dentro de las quotas.

> [!warning] **Error común: throttlear demasiado agresivo**, dejando **GPUs idle mientras los documentos se encolan**. El enfoque correcto es el **dynamic throttling**: ajustar la concurrency según **feedback en tiempo real de los API response codes**.

Los **back-off algorithms** como el **exponential delay (1 s, 2 s, 4 s, 8 s)** funcionan bien. Si la latencia de creación de embeddings o de inserción en la vector DB se vuelve un problema, **reducir incrementalmente** el número de requests enviados.

> [!tip] Para **backlogs muy grandes**, schedule la ingestión en **off-peak hours** o usá **batch embeddings** (varios chunks combinados en un solo request). Estas optimizaciones pueden **cortar costos a la mitad** en sistemas de alto volumen.

## Evaluating retrieval quality

Una vez que la data está en el vector store, hay que **medir cuán bien performa**. La retrieval quality se cuantifica con métricas estándar:

- **[[precision@k]]** — la **proporción de los top-k chunks devueltos que son relevantes**.
- **[[recall@k]]** — la **proporción de todos los chunks relevantes que se recuperan**.

> [!note] **Baseline:** **`precision@5 > 0.8`** es generalmente aceptable para enterprise documentation.

Se automatiza manteniendo un **test suite de pares (question → expected answer)**, y **rerun periódico** para detectar **regresiones tras code o model updates**. Esta práctica **transforma la experimentación GenAI en una disciplina de ingeniería** anclada en outcomes medibles.

### Discovering taxonomies (advanced, optional)

Cuando la data es compleja o voluminosa, se emplean técnicas avanzadas. Tras ingestar, el sistema **recupera chunks semánticamente relevantes pero no entiende cómo se relacionan con estructuras conceptuales más amplias**. En sistemas tradicionales esa estructura es una **taxonomy** (clasificación jerárquica de conceptos y topics). En las empresas las taxonomies **existen fragmentadas** (product catalogs, compliance classifications, customer-support topic hierarchies) pero **rara vez unificadas**; al ingestar de distintos sistemas, **esa estructura semántica se pierde**.

> [!note] **[[Taxonomy Discovery|Taxonomy discovery]]** identifica **estructuras conceptuales latentes** en la data ingestada: en vez de jerarquías predefinidas, **descubre clusters de conceptos relacionados directamente de los embeddings** (que representan significado en un espacio de alta dimensión) usando **clustering como k-means o hierarchical clustering**. Los documentos sobre conceptos similares **forman clusters naturalmente**.

**Ejemplo de aseguradora:** miles de **policy documents, claims procedures y regulatory guidelines**; sin taxonomy el retrieval devuelve los docs correctos pero **no revela cómo se relacionan** con conceptos más amplios como **underwriting rules, claims handling o regulatory reporting**. El clustering **infiere agrupaciones**; los ingenieros las **revisan y etiquetan** → **categorías de conocimiento explícitas**.

> [!tip] Ventaja clave: la taxonomy **evoluciona con la data** (re-clustering periódico detecta **emerging topics**) — útil en industrias de terminología cambiante como **cybersecurity** (nuevas threat categories). Otra técnica: **keyword extraction (TF-IDF o transformer-based) + embedding similarity** → **candidate taxonomy trees** que los domain experts refinan. La taxonomy se vuelve una **capa sobre la vector DB**: sus labels sirven de **metadata filters** para restringir el retrieval a una rama conceptual → mejora **performance e interpretability**, transformando la vector DB de simple similarity-search en un **structured knowledge system**.

### Hybrid search and the limits of pure vector retrieval

Las vector databases ganaron popularidad porque habilitan la **semantic search**: en vez de matchear keywords exactos, recuperan documentos cuyos **embeddings están cerca en el vector space** — genial para preguntas en lenguaje natural. Pero el retrieval **puramente semántico tiene límites**: ciertas queries requieren **exact matches** (un **product code, un legal clause number, un technical identifier**) donde el **keyword search tradicional gana**.

Por eso muchas arquitecturas GenAI modernas usan **[[Hybrid Search|hybrid search]]**, que combina vector similarity con métodos de **lexical search como [[BM25]]**. Cada enfoque **compensa la debilidad del otro**.

### Tabla 5.4 — Lexical vs Vector vs Hybrid search

| Aspecto | **Lexical search ([[BM25]])** | **Vector search** | **Hybrid search** |
|---|---|---|---|
| Fortaleza | Recupera **palabras/frases exactas**; rápido, determinístico, bien entendido | Captura **significado** ("vehicle insurance" ≈ "automobile coverage" cerca en el espacio) | **Combina los scores de ambos** con una weighted formula; los docs que rinden bien en ambos rankean más alto |
| Debilidad | Falla con **variación semántica** (busca "vehicle coverage rules", el doc dice "automobile insurance policy") | A veces recupera docs **relacionados pero no lo bastante precisos** | — (suele dar la **mejor retrieval quality** enterprise) |
| Extra | — | — | Mejora **transparency**: inspeccionar el componente lexical explica **por qué matcheó** (clave en entornos regulados con decisiones explicables) |

> [!tip] **Implementación:** calcular un **lexical score** y un **vector similarity score**, y **mergearlos con una weighted formula**. Muchas vector DBs soportan hybrid **directamente**; otras integran con **Elasticsearch u OpenSearch** en una **arquitectura de dos etapas**: el lexical search **recupera candidatos** y el vector similarity los **rerankea**, balanceando performance y relevance.

### The importance of data cleaning in GenAI pipelines

La efectividad del retrieval **depende fuertemente de la calidad de la data subyacente**: **ni los embedding models más sofisticados compensan texto mal preparado**. Los documentos ingestados suelen traer **formatting artifacts, duplicated sections o irrelevant metadata** que degradan el retrieval.

> [!note] **[[Data Cleaning|Data cleaning]]** es un **stage crítico**: antes de embeber, los documentos deben **normalizarse a una representación textual consistente**, **removiendo headers, footers y page numbers repetidos** — estos fragmentos repetidos **distorsionan los embeddings** introduciendo **similitud artificial entre documentos no relacionados**.

Aspectos clave de la data preparation:

- **Scanned PDFs / OCR** — los sistemas de OCR a veces introducen errores como **merged words o caracteres incorrectos**; si quedan, los embeddings **no reflejan fielmente el significado**. Mitigar con **spell correction y sentence segmentation**.
- **Deduplication** — las empresas suelen guardar **múltiples copias del mismo documento** en distintos sistemas; si se embeben por separado, el retrieval **devuelve chunks idénticos** y baja la **diversidad de resultados**. El **hashing** identifica duplicados **antes de ingestar**.
- **Metadata enrichment** — adjuntar **metadata contextual** (department, project, regulatory category) a cada chunk **mejora la precision** porque las queries pueden **filtrar por document type, author o date**.
- **Standardizing terminology** — las organizaciones usan **múltiples nombres para el mismo concepto** ("customer", "client", "policyholder"); aunque los embedding models capturan similitud semántica, la **terminología consistente mejora la interpretability y reduce ambigüedad**.

> [!tip] Un buen data cleaning **mejora la reliability general** del sistema GenAI: asegura que los embeddings **representen contenido con significado** en vez de **formatting artifacts**. A medida que crece la colección, los beneficios del preprocessing cuidadoso **se vuelven cada vez más evidentes**.

### Graph databases and the emergence of knowledge graphs

Las vector databases son excelentes en **similarity search** pero **menos efectivas representando relaciones explícitas entre entidades** (ownership, dependencies, causal connections). Las **[[Graph Database|graph databases]]** representan info como **nodes y edges**: cada **node** es una entidad (product, document, organization, concept) y cada **edge** una relación.

> [!note] **Combinadas con vector retrieval:** el **vector search identifica las piezas relevantes** de información, mientras el **graph traversal revela cómo se relacionan**. Ejemplo aseguradora: un vector search recupera passages de **accounting rules**; la graph DB las **conecta con regulatory bodies, financial instruments y compliance procedures** → razonamiento más estructurado.

Los **[[Knowledge Graph|knowledge graphs]]** soportan **explainability**: muestran la **cadena de relaciones** que llevó a una conclusión — clave en **finance, healthcare y law**, donde las decisiones deben ser **auditables**.

**Construcción** (empieza en el ingestion pipeline):

1. **Named entity recognition (NER)** — identifica entidades (people, organizations, products) en el texto.
2. **Relationship extraction** — infiere **cómo se conectan** esas entidades.
3. Las entidades y relaciones se **guardan en la graph database** junto a los documentos originales.

> [!tip] Una vez que el grafo existe, el retrieval **combina vector search con graph traversal**: una query primero **recupera docs por embeddings**, luego **expande por relaciones del grafo** para juntar más contexto — técnica llamada **[[GraphRAG]]**, que **enriquece las respuestas con structured knowledge**. Las graph databases extienden GenAI **más allá del similarity search**, introduciendo **explicit reasoning structures**.

### Toward integrated knowledge retrieval

Estas técnicas son una **evolución natural de los pipelines GenAI**. Los sistemas tempranos se enfocaban en ingestar docs y generar embeddings; con la experiencia, las organizaciones reconocen que **la semantic search sola no alcanza para tareas de conocimiento complejas**.

> [!note] **Cómo encajan las cuatro piezas:** **Taxonomy discovery** aporta estructura conceptual; **hybrid search** mejora accuracy semántica + lexical; **data cleaning** asegura que los embeddings reflejen fielmente el significado; **graph databases** dan un framework relacional. Juntas **transforman una vector DB simple en una knowledge platform comprehensiva**.

A medida que las apps GenAI maduran, las arquitecturas que **combinan vector databases + lexical search engines + graph databases** son cada vez más comunes — **espejan cómo los humanos organizan el conocimiento: por similitud Y por relaciones**. El resultado es un retrieval system capaz de **soportar tareas analíticas complejas**, desde responder preguntas en lenguaje natural hasta explorar conceptos interconectados en grandes colecciones de documentos.

## Citas

> "The first step when building agentic applications is finding and categorizing the data that is valuable to the application, and then migrating that data to a vector database."

> "This is an extremely powerful paradigm. It is analogous to the computer science concept of separating data and functions in a class, only this time the separation is at the document level."

> "For vector ingestion, the correct metric is not gigabytes but rather how many embeddings per second (EPS) can be produced and written to the vector database without exceeding rate limits or budget constraints."

> "A useful mental model is to imagine a conveyor belt: new documents are added at one end, processed documents come off the other end, and the belt never stops."

> "The embeddings themselves carry the same sensitivity as the source documents."

> "Because every token sent for embedding represents a cost, throttling is not merely a technical control but a financial one."

> "These hybrid systems mirror the way humans organize knowledge: through both similarity and relationships."

> "The only novelty lies in what you move (embeddings instead of records) and how you measure success (semantic relevance instead of row counts)."

## Para aplicar

- **Buscar documentos de proceso, no solo de datos** — al armar la colección, sumá **manuales, fórmulas y definiciones "how to"** (como el manual IFRS 17) junto a la raw data, y orquestá el prompt para que el LLM **use los docs de proceso para interpretar los docs de datos** (separación data/logic a nivel documento).
- **Elegir un ETL tool maduro** — para enterprise preferí una **mature ETL platform** ([[Airbyte]]) sobre code-your-own (solo si hay pocos repos) o GenAI tools recientes (LangChain); recorré el **checklist de 6 preguntas** (errores, PII/SOX/GDPR, updates, versioning, recolección por topic, paralelismo) antes de empezar.
- **Dimensionar en EPS, no en GB** — calculá throughput como **tokens/chunk × chunks/doc**; apuntá a **50–200 EPS**, estimá el tiempo total (10k docs × 200 chunks = 2M embeddings ≈ 5½ h a 100 EPS) y benchmarkeá con el utility **`mteb`** filtrando el MTEB Leaderboard por Speed.
- **Diseñar para el cambio continuo** — soportá **full reloads E incremental updates**; definí un **cursor field** (`updated_at`) o **hash de contenido** para reprocesar solo lo cambiado, y usá **CDC con polling corto (~1 min)** para near-real-time; pensá el pipeline como un **conveyor belt** que loguea, reintenta y pone en cuarentena los fallos.
- **Resolver security ANTES del pipeline** — definí **classification levels** (public/internal/restricted/confidential), aplicá **field-level masking, encryption (TLS + AES-256), audit trails y versioning**; para data muy sensible embebé **on-premises**; en grandes orgs usá controles demostrables (Airbyte Enterprise: RBAC, secret vault, encrypted logs).
- **Paralelizar en 3 stages con DevOps discipline** — separá **Extraction (CPU-bound, scale horizontal) / Embedding (API-bound, cuidado con throttling) / Loading**, comunicados por message queues con acknowledge; poné connector configs y transformation code **bajo version control**, testeá en staging y tratá los schema changes como migrations; con **CI/CD** validá vector dimensions y retrieval accuracy en cada cambio de modelo.
- **Testear el significado, no solo la llegada** — unit tests por stage (extraction preserva paragraphs/headings; embedding compara cosine similarity de pares conocidos; load matchea dimensions), integration con **top-5 chunks** sobre preguntas conocidas, y **continuous validation** muestreando **1%** para detectar drift.
- **Throttlear dinámicamente** — ajustá concurrency según los **API response codes** con **back-off exponencial (1/2/4/8 s)**; para backlogs grandes usá **off-peak scheduling o batch embeddings** (puede cortar costos a la mitad).
- **Medir retrieval quality** — automatizá un suite de pares **(question → expected answer)** con baseline **precision@5 > 0.8** y rerun periódico; cuando la semantic search no alcance, sumá **taxonomy discovery** (clustering k-means/hierarchical), **hybrid search** (vector + BM25), **data cleaning** (dedup por hashing, metadata enrichment, terminología consistente) y **graph databases/GraphRAG** (NER + relationship extraction).

## Conexiones

- [[_Building Complex Multi-Agent Systems Using Pattern Prompting|Building Complex Multi-Agent Systems]] — el MOC del libro.
- [[04 - Building Your First RAG App]] — capítulo anterior: cerró prometiendo construir los **data pipelines / [[ETL]]** que mueven datos a la vector DB; este capítulo **arranca ese proyecto de data migration**. Los **message queues** de la app RAG reaparecen acá como el medio de comunicación entre los 3 stages del pipeline.
- [[06 - Ingesting Data Using Airbyte and Pinecone]] — capítulo siguiente (placeholder): construye un **pipeline completo y funcional** con **[[Airbyte]]** conectando documentos no estructurados a la vector database **[[Pinecone]]**, instrumentado para **observability y error recovery** → un ingestion framework reproducible reusable.
- [[03 - Building with GenAI Parameters, Tuning, and Project Phases]] — había prometido el **[[ETL]] avanzado para los caps. 5 y 6**; introdujo las 3 fases del proyecto, el [[Chunking|chunking]] (chunk size) y el pipeline ingestion/retrieval/generation que acá se hace operacional.
- [[02 - Embeddings The Language of AI]] — los **[[Embeddings|embeddings]]**, la **[[Cosine Similarity|cosine similarity]]**, las **[[Vector Database|vector databases]]** (data migration vs live query) y el **[[Chunking|chunking]]** que este pipeline mueve y mide.
- [[01 - Introduction Patterns, Abstractions, and the GenAI Landscape]] — la advertencia sobre la **"frothy exuberance"** y evaluar tools nuevas con cuidado, que justifica elegir un ETL tool maduro.
- **[[Airbyte]]** — la mature ETL platform open-source elegida (incremental sync, CDC, Airbyte Enterprise con RBAC/secret vault/encrypted logs).
- **[[EPS (Embeddings per Second)]]** — la métrica de capacity planning para vector ingestion (análoga a RPS).
- **[[Change Data Capture]]** · **[[Incremental Sync]]** — estrategias para mantener la vector DB actual sin re-embeber todo.
- **[[Hybrid Search]]** · **[[BM25]]** — combinar vector + lexical search para cubrir exact matches y variación semántica.
- **[[Taxonomy Discovery]]** — descubrir estructura conceptual latente por clustering de embeddings.
- **[[Knowledge Graph]]** · **[[Graph Database]]** · **[[GraphRAG]]** — representar relaciones explícitas entre entidades y combinarlas con vector search.
- **[[Data Cleaning]]** — normalización, dedup, metadata enrichment y standardización de terminología antes de embeber.
- **[[precision@k]]** · **[[recall@k]]** — métricas de retrieval quality (baseline precision@5 > 0.8).
- **[[Pinecone]]** — la vector database que el cap. 6 conecta vía Airbyte (candidato a nota propia).
- [[MTEB]] — leaderboard de embedding models con speed metrics, usado para benchmarkear EPS.
- [[ETL]] · [[Vector Database]] · [[RAG]] — el pipeline ETL que alimenta la vector DB del sistema RAG.
