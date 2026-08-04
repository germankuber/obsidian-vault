---
title: 06 - Ingesting Data Using Airbyte and Pinecone
libro: Building Complex Multi-Agent Systems Using Pattern Prompting
autor: Packt
capitulo: 6
created: 2026-06-20
tags:
  - libros/building-complex-multi-agent-systems-using-pattern-prompting
  - type/case-study
  - status/permanent
aliases:
  - Ingesting Data Using Airbyte and Pinecone
updated: 2026-07-05
---

# 06 - Ingesting Data Using Airbyte and Pinecone

> [!info] Capítulo 6 · *Building Complex Multi-Agent Systems Using Pattern Prompting* — Packt (ISBN 9781806114290)
> El capítulo baja a tierra el proyecto de data migration del [[05 - Starting Your Data Migration Project|cap. 5]] construyendo un **data pipeline práctico y funcional**: lee PDFs desde **[[Amazon S3]]** y los ingesta en la vector database **[[Pinecone]]**, usando **[[Airbyte]]** para todo el workflow **[[ETL]] (Extract, Transform, Load)**. Tesis: el tooling ETL moderno (no-code) simplifica preparar data para [[RAG]] porque por debajo aplica [[Enterprise Integration Patterns|EIP]] conocidos ([[Message Channel]], [[Channel Adapter]]) y el [[Strategy]] pattern para intercambiar source/destination sin tocar la arquitectura. Navegá: [[_Building Complex Multi-Agent Systems Using Pattern Prompting|Libro]] · anterior [[05 - Starting Your Data Migration Project]] · siguiente [[07 - Tips and Best Practices]].

## Resumen

Este capítulo cierra el bloque ETL prometido (los "caps. 5 y 6"): si el [[05 - Starting Your Data Migration Project|cap. 5]] **planeó** la data migration como ejercicio de systems-engineering (tooling, capacity planning, security, CI/CD), el cap. 6 lo **construye funcional**. El objetivo es un **ingestion pipeline que transforma documentos no estructurados en vector embeddings searchables en [[Pinecone]]**. La arquitectura (Fig 6.1) es lineal: **[[Amazon S3]] → [[Airbyte]] (Load Data → Create Embeddings → Ingest Data) → [[Pinecone]] Vector Database**. El capítulo demuestra cómo el **tooling ETL moderno** simplifica preparar data para sistemas [[RAG]]; al terminar, el lector tiene un pipeline corriendo que lee PDFs de un S3 bucket, los chunkea, genera embeddings e ingesta en Pinecone.

El recorrido tiene dos mitades. La primera, *Analysis of applicable patterns*, mira **bajo el capó** del ETL: [[Airbyte]] (como casi todo tool ETL) se apoya en patrones del catálogo **[[Enterprise Integration Patterns|EIP]]** de **Gregor Hohpe y Bobby Woolf**. Dos son centrales — **[[Message Channel]]** (desacopla producer de consumer enviando mensajes por un channel, permitiendo escalar independientemente) y **[[Channel Adapter]]** (conecta un communication channel externo, como [[RabbitMQ]] o una HTTP API, traduciendo data entre el formato del channel y la representación interna). Para crear un pipeline hay que especificar solo **dos cosas** — el **data source** y el **destination system** —; Airbyte usa **internal strategy y adapter components** para conectarlos, lo que permite reconfigurar fácilmente (S3 hoy, SharePoint o un RDBMS mañana) **sin cambiar la arquitectura subyacente**. Para minimizar configuración, Airbyte expone esto vía un **[[Strategy]] pattern** sobre una **no-code interface**: el usuario solo elige el adapter y provee connection details.

La segunda mitad, *Building our ETL pipeline*, ejecuta los **5 pasos** del proyecto: (1) instalar vector DB y Airbyte (open-source), (2) analizar las colecciones de documentos, (3) configurar Airbyte, (4) migrar los documentos y (5) testear resultados. Se instala **[[Airbyte]]** open-source local (o free trial sin tarjeta) y **[[Pinecone Local]]** vía Docker. El análisis de documentos lleva a la **[[Taxonomy Discovery|taxonomy creation]]** previa al ETL (text classification / topic modeling para enriquecer metadata, ref. al taxonomy discovery del cap. 5). Airbyte se puede configurar de **3 formas** (no-code UI, RESTful API, Python SDK); el cap. usa la no-code: se configura el **Source S3** (bucket + API key) y el **Destination Pinecone** (chunk size, text splitting, embedding model, indexing), y en la tab **Builder** se importa un **[[YAML]] manifest**. El YAML usa `syncMode: incremental` + `destinationSyncMode: append` para procesar solo los files nuevos/modificados (modo `Deduped` si no se quieren versiones viejas). Tras correr el pipeline hay que vigilar el **drift** y validarlo con las sample queries del [[02 - Embeddings The Language of AI|cap. 2]]. El **[[07 - Best Practices and Advanced Data Management|cap. 7]]** cerrará con best practices y advanced options de data management.

![[B34134_6_1.png]]
*Figure 6.1 – Pipeline architecture: Amazon S3 → Airbyte (Load Data, Create Embeddings, Ingest Data) → Pinecone Vector Database.*

## Analysis of applicable patterns

[[Airbyte]], like many ETL tools, relies on well-known software design patterns to implement scalable data pipelines. Estos patrones provienen del catálogo **[[Enterprise Integration Patterns|EIP]]** descrito por **Gregor Hohpe y Bobby Woolf** (los mismos referidos desde el [[01 - Introduction Patterns, Abstractions, and the GenAI Landscape|cap. 1]]). Dos patrones son particularmente relevantes para el pipeline de este capítulo: **[[Message Channel]]** y **[[Channel Adapter]]**.

Un **[[Message Channel]]** habilita la comunicación entre componentes que producen y consumen data: en vez de conectar sistemas directamente, los mensajes se envían a través de un **channel**. Esto **desacopla el producer del consumer** y permite que el pipeline **escale independientemente** entre distintos sistemas. Un **[[Channel Adapter]]** es un componente que conecta un **external communication channel** (como [[RabbitMQ]] o una HTTP API) a una aplicación, **traduciendo data** entre el formato del channel y la representación interna del sistema. En nuestro caso, el **producer** lee documentos de **[[Amazon S3]]** y manda la data extraída al pipeline; los componentes downstream la procesan y finalmente la guardan en **[[Pinecone]]**.

> [!note] Estos dos patrones forman el fundamento de cómo los tools ETL manejan el data movement en la práctica: el Message Channel desacopla, y el Channel Adapter traduce e integra los extremos.

### Tabla — Los 2 EIP centrales del pipeline

| Patrón EIP | Qué hace | Rol en el pipeline |
|---|---|---|
| **[[Message Channel]]** | Habilita comunicación entre componentes que producen y consumen data; los mensajes van por un channel en vez de conexión directa | Desacopla producer (S3 reader) de consumer (Pinecone writer) y permite escalar cada lado independientemente |
| **[[Channel Adapter]]** | Conecta un communication channel externo (RabbitMQ, HTTP API) a una app, traduciendo data entre el formato del channel y la representación interna | Traduce los documentos extraídos al/del mensaje que viaja por el channel en cada extremo (source y destination) |

![[B34134_6_2.png]]
*Figure 6.2 – Message Channel and Channel Adapter patterns: Message Producer → Messaging System → Receiver Application, with a Source Strategy / Channel Adapter on each side.*

### Integration patterns implemented by ETL

Los sistemas ETL típicamente **combinan varios integration patterns** para mover data entre sistemas, y [[Airbyte]] los implementa mediante **configurable connectors y adapters**. Para crear un pipeline ETL con Airbyte hay que especificar **dos cosas**:

- **El data source** del que se recuperará información.
- **El destination system** donde se guardará la data procesada.

Airbyte usa entonces **internal strategy y adapter components** para conectar esos sistemas y mover data entre ellos. Este diseño permite **reconfigurar pipelines fácilmente**: un pipeline que hoy lee de **[[Amazon S3]]** podría más tarde modificarse para leer de **SharePoint** o de una **relational database** *sin cambiar la arquitectura subyacente*.

> [!tip] La separación source/destination + internal strategy/adapter es lo que hace al pipeline portable: cambiás el adapter, no la arquitectura.

### Channel adapter

Como se mencionó, un **[[Channel Adapter]]** es un integration component que conecta una aplicación a un **messaging system**, permitiéndole enviar y recibir mensajes. Es una **implementación del Channel Adapter EIP**: el adapter interactúa con la API o data de una app y la **traduce a un mensaje** para un message channel, o viceversa.

[[Airbyte]] tiene **dos tipos** de channel adapters:

- **Inbound channel adapter** — integración **one-way** para **traer data adentro** de una messaging application. Puede ser **message-driven** (reacciona a mensajes nuevos a medida que llegan) o **polling-based** (chequea periódicamente si hay mensajes nuevos).
- **Outbound channel adapter** — integración **one-way** para **sacar data afuera** de una messaging application. Escucha mensajes en un channel y los manda a un sistema externo, como un **file**, una **database** o una **API**.

Como la mayoría de los tools ETL, Airbyte ofrece varios channel adapters configurables por el usuario. Por ejemplo, se puede configurar un **outbound channel adapter** para cualquiera de varias vector database implementations como **Qdrant, [[Pinecone]] o Milvus**. Para minimizar el esfuerzo de configuración, Airbyte usa un **[[Strategy]] pattern**: el usuario especifica qué adapter usar y provee los connection details a través de una **no-code interface**.

### Tabla — Inbound vs Outbound channel adapter

| Tipo | Dirección | Mecanismo | Ejemplo |
|---|---|---|---|
| **Inbound channel adapter** | One-way, **trae data adentro** de la messaging app | **Message-driven** (reacciona al llegar el mensaje) o **polling-based** (chequea periódicamente) | Leer documentos desde [[Amazon S3]] hacia el pipeline |
| **Outbound channel adapter** | One-way, **saca data afuera** de la messaging app | Escucha mensajes en un channel y los manda a un sistema externo (file, database, API) | Escribir embeddings a una vector DB: Qdrant, [[Pinecone]], Milvus |

### Strategy pattern

El **[[Strategy]] pattern** es un **behavioral design pattern** que permite a un programa **seleccionar un algoritmo de una familia de algoritmos en runtime**. Lo logra **definiendo algoritmos intercambiables**, **encapsulando cada uno en una clase separada** y haciéndolos **seleccionables en runtime** (a menudo pasando uno de los strategy objects a un **context object**). Este enfoque hace el código más **flexible, intercambiable y extensible** sin modificar su core structure. (Es el mismo [[Strategy]] que el [[04 - Building Your First RAG App|cap. 4]] usó para los behaviors pluggables del LLM client.)

> [!note] En Airbyte, el Strategy pattern es el mecanismo **no-code** que intercambia source/destination adapters: el usuario elige el adapter y aporta los connection details; el resto lo resuelve el context.

#### Strategy pattern (producer)

Para configurar Airbyte para **leer de un source** como [[Amazon S3]], solo hace falta proveer el **source type en el context** junto con la **connection y security information** que el adapter necesita para autenticar y conectar.

![[B34134_6_3.png]]
*Figure 6.3 – Strategy Pattern – Airbyte Producer: Client → Context → Data Source Adapters (Amazon S3, SharePoint, RDBMS, …).*

#### Strategy pattern (receiver)

Para configurar Airbyte para **escribir a un destination**, hay que proveer el **destination type en el context** junto con la **connection y security information** del adapter que se quiere usar.

![[B34134_6_4.png]]
*Figure 6.4 – Strategy Pattern – Airbyte Receiver: Client → Context → Vector Database Adapters (ChromaDB, Pinecone, Weaviate, …).*

Con esta intuición de lo que pasa *under the hood* de un ETL, el capítulo pasa al ejemplo concreto: usando Airbyte, crear un pipeline que lee documentos de un **Amazon S3 bucket**, los **chunkea** y escribe **vector embeddings** a la vector database **[[Pinecone]]**.

## Building our ETL pipeline

Hecha la tool selection (cap. 5) y revisados los patrones relevantes, el capítulo construye el pipeline ETL en **5 pasos**, en orden:

1. **Install a vector database and Airbyte** (open-source version).
2. **Analyze the document collections**.
3. **Configure Airbyte**.
4. **Migrate the documents**.
5. **Test the results**.

### Installation of Pinecone and Airbyte

La **versión open-source de [[Airbyte]]** es directa de instalar en la máquina local; alternativamente, se puede registrar un **free trial** en el sitio de Airbyte **sin tarjeta de crédito**. **[[Pinecone]]** ofrece una local development tool llamada **[[Pinecone Local]]** que corre vía **Docker**, dando una experiencia *open-source-like* para desarrollo y testing local. El repositorio de GitHub del libro contiene instrucciones para configurar Pinecone, Airbyte y [[Amazon S3]] para este demo. Los tres links de referencia:

```text
https://github.com/airbytehq/airbyte?tab=readme-ov-file
https://docs.airbyte.com/platform/using-airbyte/getting-started/oss-quickstart
https://docs.pinecone.io/guides/get-started/overview
```

### Analysis of documents

El **primer paso** es **survey y documentar las colecciones** de documentos que se quieren migrar a la vector database, investigando cómo se categorizan. Preguntas a hacerse:

- ¿Los documentos contienen **PII**?
- ¿Qué tipo de **autenticación** se requiere para acceder?
- ¿Qué **tipos de documentos** hay: PDF, Excel, plain text?
- ¿Qué **structural patterns** aparecen dentro de los documentos?
- ¿Los PDFs tienen **múltiples chapters y subchapters**?

Si se encuentran varios repositorios de documentos con **mala categorización**, conviene considerar algoritmos de **text classification** o **topic-modeling** *antes* de correr el pipeline ETL: estos algoritmos pueden construir una **[[Taxonomy Discovery|taxonomy]]** que enriquece la **metadata** de cada documento (retomando el taxonomy discovery del [[05 - Starting Your Data Migration Project|cap. 5]]). Hay varias formas de extraer taxonomies, desde construir un **taxonomy extractor simple con un prompt + third-party LLM libraries**, hasta **cloud tools sofisticadas de Google, Amazon y Microsoft**.

> [!tip] Está fuera del scope del libro cubrir todas las opciones de taxonomy creation, pero para **colecciones grandes mal categorizadas** conviene una **commercial o open-source taxonomy extraction tool**.

![[B34134_6_5.png]]
*Figure 6.5 – Taxonomy creation: a source document is classified into a hierarchy of categorized sub-documents.*

### Options for configuring Airbyte

Para tener el pipeline ETL corriendo con Airbyte hay que proveer config data de modo que Airbyte **(a)** sepa **a qué endpoints** se va a conectar, y **(b)** tenga la **connection information** (URLs y security credentials) que necesita para **autenticar** a esos endpoints. Airbyte se puede configurar de **tres formas**:

| Forma de configurar | Detalle |
|---|---|
| **No-code interface** | La UI gráfica de Airbyte; es la que usa este capítulo |
| **RESTful API** | Endpoints invocables desde cualquier lenguaje maduro como **TypeScript** o **Java** (el repo tiene un ejemplo en TypeScript) |
| **Python SDK** | Configuración programática vía SDK de Python |

> [!note] El cap. muestra la **no-code interface**; ejemplos de configurar Airbyte vía la API con **TypeScript** y **Java** están en el repositorio de GitHub del libro.

#### Configuring Airbyte using its no-code interface

Al loguearte en Airbyte, ves la página **Connections** (las Sources se enlazan con Destinations a través del Airbyte connector hub).

![[B34134_6_6.png]]
*Figure 6.6 – Airbyte Connections page: Sources link to Destinations through the Airbyte connector hub.*

Para configurar un connector a **[[Amazon S3]]**: clic en **Sources** (panel izquierdo) → se ven muchos source connectors disponibles → buscar `S3` en el campo **Source name** → clic en el ícono del S3 bucket para abrir la pantalla de configuración del S3 bucket. Ahí se ingresa el **bucket name** y la **Amazon API key** (campos opcionales). Esto le dice a Airbyte que use el S3 bucket y que **mande la API key con cada fetch request**, de modo de estar autorizado a acceder al bucket.

![[B34134_6_7.png]]
*Figure 6.7 – Airbyte S3 Source configuration: enter Source name, Bucket, streams to sync, and Delivery Method.*

Luego, clic en **Destinations** (sidebar izquierdo) y seleccionar **[[Pinecone]]** de la lista (puede hacer falta filtrar por vector databases o por nombre). Esto abre una pantalla para configurar la conexión a Pinecone, incluyendo settings para **chunk size, text splitting, embedding model e indexing**.

![[B34134_6_8.png]]
*Figure 6.8 – Airbyte Pinecone Destination configuration: Processing (chunk size, text splitter), Embedding (OpenAI API key), and Indexing (Pinecone index, environment, API key).*

Después, clic en la tab **Builder** (sidebar izquierdo) para crear el pipeline. Se ven **tres opciones para construir connectors**:

| Opción del Builder | Descripción |
|---|---|
| **Importing a YAML manifest** | Proveer un archivo YAML para customizar y subir (la que usa el capítulo) |
| **Fork an existing connector** | Partir de un connector existente y modificarlo |
| **Start from scratch** | Construir el connector desde cero |

El capítulo usa la **primera** opción: proveer un YAML file para customizar y subir.

#### Importing a YAML manifest

**[[YAML]]** es un **human-readable data serialization language** usado principalmente para **configuration files**. El YAML file está en el repositorio de GitHub del libro y se puede abrir con cualquier editor de texto estándar (**Notepad++, VS Code, Sublime Text, Vim, Atom**). El manifest del pipeline S3 → Pinecone:

```yaml
# Airbyte connection: S3 (documents) → Pinecone
resourceType: connection
name: s3-pdfs → pinecone
workspaceId: ${AIRBYTE_WORKSPACE_ID}

# Octavia fills these from the referenced files during `generate`, but you can
# leave them as relative references or apply after generating via CLI.
source:      ../sources/s3_pdfs/configuration.yaml
destination: ../destinations/pinecone/configuration.yaml

# Minimal catalog config. Airbyte will infer the schema;
# we include the stream name.
syncCatalog:
  streams:
    - stream:
        name: pdf_docs
        jsonSchema: {}          # let Airbyte manage schema details
        supportedSyncModes: ["full_refresh", "incremental"]
        namespace: null
      config:
        syncMode: incremental
        destinationSyncMode: append
        primaryKey: [["document_key"]]   # de-dupe / stable upserts
        selected: true

# Schedule (change as desired)
schedule:
  timeUnit: hours
  units: 24

status: active

# Optional: connector resources (CPU/mem) or normalization can be added here.
```

Notar que se setea `syncMode` a `incremental` y `destinationSyncMode` a `append`. Esto asegura que cuando se **agrega un documento nuevo** al S3 bucket, o hay una **versión actualizada** de un documento existente, **solo ese file se procesa**; las versiones viejas quedan disponibles. Si las versiones viejas **no** se necesitan, el modo **`Deduped`** se puede usar para **ahorrar espacio** no reteniéndolas. La documentación de Airbyte cubre estas y otras features en detalle.

> [!note] El `primaryKey: [["document_key"]]` habilita el **de-dupe / stable upserts**: identifica de forma estable cada documento para que las re-ejecuciones actualicen en vez de duplicar.

#### Drift y validación

Cada ejecución del data pipeline tiene el potencial de crear **"drift"** — un **cambio en el comportamiento del sistema** causado por usar un dataset **nuevo o modificado**. Tras correr el pipeline, hay que **verificar que las semantic queries sigan devolviendo resultados aceptables**.

> [!warning] El drift es un riesgo real cada vez que re-corrés el pipeline con un dataset cambiado: el sistema puede comportarse distinto sin avisar. Validá siempre con tus sample queries después de cada run.

Acá se demuestra el valor de la best practice del [[02 - Embeddings The Language of AI|cap. 2]]: escribir **unas pocas docenas de sample queries** al inicio del proyecto GenAI. Ese trabajo previo es justo lo que permite detectar el drift y confirmar que el retrieval sigue sano tras la ingesta.

## Citas

> "This chapter demonstrates how modern ETL tooling can simplify the process of preparing data for RAG systems."

> "A message channel enables communication between components that produce and consume data. Instead of connecting systems directly, messages are sent through a channel."

> "For example, a pipeline that reads from Amazon S3 today could later be modified to read from SharePoint or a relational database without changing the underlying architecture."

> "The strategy pattern is a behavioral design pattern that allows a program to select an algorithm from a family of algorithms at runtime."

> "Each execution of the data pipeline has the potential to create \"drift\" — a change in the behavior of the system caused by using a new or modified dataset."

> "Having worked through the preceding chapters, you should now have an excellent grounding in the fundamentals required to build LLM-based systems. You can be confident starting any project that uses an LLM, whether in a RAG architecture or a microservice architecture."

## Para aplicar

- **Instalá el stack local** — Airbyte open-source (o free trial sin tarjeta) + [[Pinecone Local]] vía Docker; seguí las instrucciones del repo de GitHub del libro para Pinecone, Airbyte y [[Amazon S3]].
- **Auditá tus colecciones antes de migrar** — preguntá por PII, autenticación, tipos de doc (PDF/Excel/plain text), structural patterns y si los PDFs tienen chapters/subchapters; si están mal categorizados, corré **text classification / topic-modeling** para construir una **[[Taxonomy Discovery|taxonomy]]** que enriquezca la metadata.
- **Elegí cómo configurar Airbyte** — no-code UI (este cap.), RESTful API (TypeScript/Java) o Python SDK; para el demo usá la **no-code interface**.
- **Configurá el Source S3** — Sources → buscar `S3` → ingresá bucket name + Amazon API key (para autorizar cada fetch request).
- **Configurá el Destination Pinecone** — Destinations → seleccioná Pinecone → seteá **Processing** (chunk size, text splitter), **Embedding** (OpenAI API key) e **Indexing** (Pinecone index, environment, API key).
- **Importá el YAML manifest** en la tab **Builder** — usá `syncMode: incremental` + `destinationSyncMode: append` para procesar solo lo nuevo/modificado; pasá a `Deduped` si querés ahorrar espacio descartando versiones viejas; usá `primaryKey: [["document_key"]]` para upserts estables.
- **Vigilá el drift tras cada run** — corré tus **sample queries** (del [[02 - Embeddings The Language of AI|cap. 2]]) y verificá que el retrieval sigue devolviendo resultados aceptables.

## Conexiones

- [[_Building Complex Multi-Agent Systems Using Pattern Prompting|Libro]] — el MOC del libro.
- [[05 - Starting Your Data Migration Project]] — capítulo anterior: planea el ETL (Airbyte, Pinecone, capacity planning, taxonomy discovery); el cap. 6 lo **construye funcional**.
- [[07 - Tips and Best Practices]] — capítulo siguiente (placeholder): best practices + advanced options para manejar data en apps GenAI.
- [[02 - Embeddings The Language of AI]] — la best practice de escribir sample queries que acá valida el drift.
- [[01 - Introduction Patterns, Abstractions, and the GenAI Landscape]] — los EIP de Hohpe & Woolf y [[RabbitMQ]] como ejemplo de channel.
- [[04 - Building Your First RAG App]] — el [[Strategy]] pattern y los EIP ya aplicados sobre RabbitMQ.
- [[Enterprise Integration Patterns]] · **[[Message Channel]]** · **[[Channel Adapter]]** — los EIP que implementan los ETL (inbound/outbound channel adapters).
- [[Strategy]] — el behavioral pattern que Airbyte usa como mecanismo no-code para intercambiar source/destination adapters.
- [[Airbyte]] · [[Pinecone]] · **[[Pinecone Local]]** · **[[Amazon S3]]** · **[[YAML]]** — el tooling concreto del pipeline (Pinecone Local vía Docker; YAML manifest para el Builder; S3 como source).
- [[Taxonomy Discovery]] · [[ETL]] · [[Incremental Sync]] · [[RAG]] — taxonomy creation previa al ETL; el pipeline ETL funcional; incremental sync vía YAML; el sistema RAG al que alimenta.
