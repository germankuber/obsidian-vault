---
title: 09 - Building a Conversational RAG Agent
libro: RAG-Driven Generative AI - Second Edition
autor: Denis Rothman
capitulo: 9
created: 2026-06-11
tags:
  - libros/rag-driven-generative-ai
  - type/case-study
  - status/permanent
aliases:
  - Building a Conversational RAG Agent
  - Multimodal RAG Video Store Agent
updated: 2026-06-11
---

# 09 - Building a Conversational RAG Agent

> [!info] Capítulo 9 · *RAG-Driven Generative AI - Second Edition* — Denis Rothman
> Entrada al **multimodal RAG**: del texto/relacional al **video**. Arquitectura tri-planar (control/semantic/visual). Se profesionaliza la infra con un **Oracle DBA console** (schema-as-code vía `create_script.py` + registry), se verifica el ground truth de videos AI-generados (Sora) con OpenCV, y se construye un **conversational Video Store agent** que vectoriza metadata (descripciones+timestamps), busca con `VECTOR_DISTANCE` sobre `MEDIA_SEGMENTS`, y reproduce el clip exacto en lenguaje natural. Navegá: [[_RAG-Driven Generative AI - Second Edition|Mapa del libro]] · anterior [[08 - Boosting RAG Performance with Human Feedback]] · siguiente [[10 - Building an Agent with Spatial-RAG and GraphRAG]].

## Resumen

El MAS se expande a la forma de datos más dinámica: **video**. Se pasa de documentos de texto estáticos a procesar **experiencias visuales** — la entrada al **multimodal RAG**, donde la IA debe **salvar el gap semántico** entre el request en lenguaje natural del usuario y una librería de artefactos visuales. Tres etapas: (1) **profesionalizar la infraestructura** con un **Oracle DBA console** (un control plane unificado que gestiona el lifecycle de los schemas vía un schema registry central — *database-as-code*); (2) **establecer el ground truth** con un *video dataset visualization notebook* (actuar como data analysts: autenticar, descargar `.mp4` AI-generados, inspeccionar propiedades con computer vision, generar thumbnails, verificar frame rates); (3) construir el **conversational RAG Video Store agent** (ingerir metadata semántica, generar embeddings, chat interface para recuperar momentos visuales específicos por lenguaje natural). El RAG actúa como un **bibliotecario** de la video store.

La arquitectura es **tri-planar** (Figura 9.1): **control plane** (infra, NB1 DBA Console), **semantic plane** (metadata+embeddings, NB3 Video Store Agent), **visual plane** (assets `.mp4`, NB2 Video Visualization). El flujo: el usuario hace una query → el agente usa **embeddings human-verified** para semantic search y recupera el ID del clip más relevante → el ID se pasa al visual plane que lo mapea al `.mp4` físico → se renderiza un video player con el segmento exacto. Crucial: los CSV de metadata son **ground truth de alta calidad** — parcialmente auto-generados pero **refinados vía el loop de human feedback (RLHF) del cap. 8**, garantizando embeddings basados en descripciones verificadas y semánticamente ricas, no captions automáticos ruidosos.

El núcleo técnico: schema-as-code (un `create_script.py` versionado en el repo remoto como *source of truth*, evitando *configuration drift*), un esquema dual `MEDIA_ASSETS` (catálogo relacional: filename, category, video_url) + `MEDIA_SEGMENTS` (vector store: timestamp, description CLOB, `description_vector VECTOR(1536, FLOAT32)`, FK al asset), **ingesta idempotente** (chequear `user_tables` / filename antes de insertar para evitar duplicados y `ORA-00955`/`ORA-00933`), y un `VideoStoreAgent` que combina **JOIN relacional + `VECTOR_DISTANCE`** en una query nativa sobre la converged database, sintetiza con GPT-4o citando filename+timestamp, y reproduce el clip auto-seeking al momento exacto (`#t=<seconds>`).

![[09-fig-9.1.png]]
*Figure 9.1: The architecture of Multimodal RAG*

## Arquitectura del multimodal RAG (tri-planar)

Framework tri-planar que separa **infraestructura, assets visuales y metadata semántica** en zonas operacionales distintas, con un repo remoto + 3 notebooks (NB1/NB2/NB3) del raw data a la interacción del usuario.

- **Control plane (infraestructura)** — en la cima de la jerarquía; gobierna la infra de base. **NB1: Oracle DBA Console** (`Oracle_DBA_Console.ipynb`) inicializa/mantiene el **Schema Registry**, estrictamente definido por el artefacto `create_script.py` **pulled del repo remoto** (version-controlled, consistente entre deployments). *Database schemas as code*: en vez de crear tablas a mano, se hace pull del `create_script.py` versionado y CSV → deployment **determinístico**, sin *configuration drift*, garantizando que los agentes operan contra una estructura validada y actualizada (local o cloud).
- **Semantic plane (metadata)** — el "cerebro" del retrieval (bottom-right). **NB3: Video Store Agent** (`Conversational_RAG_Video_Store_Agent.ipynb`) ingesta metadata y genera embeddings; consume `.csv` del repo para construir el contexto. **Ground truth de alta calidad**: la metadata fue *parcialmente auto-generada* pero **significativamente refinada vía el loop de human feedback (RLHF)** del cap. 8 → embeddings basados en descripciones verificadas, no captions ruidosos.
- **Visual plane (assets)** — el heavy lifting de media (paralelo al semantic). **NB2: Video Visualization** (`Video_dataset_visualization.ipynb`) descarga `.mp4` del repo, verifica y genera thumbnails, asegurando que los assets matcheen su metadata antes de presentarlos.

> [!note] El system flow (path numerado)
> **(1) Query** — el usuario manda su query en lenguaje natural a NB3. **(2) Retrieval** — el agente usa los embeddings human-verified para semantic search y recupera el **ID del clip** más relevante. **(3) Display** — el ID se pasa al visual plane (NB2), que lo mapea al `.mp4` físico. **(4) Output** — un **video player renderizado** con el segmento exacto que responde a la intención.

## Universal Oracle DBA console

En caps. previos se scripteaban los schemas a mano (document store cap. 2, hybrid recruitment cap. 3). Al expandirse el sistema, gestionar schemas con scripts desconectados es **ineficiente y error-prone** → un **control plane unificado**. El DBA console **desacopla la definición de las tablas de la ejecución** de las operaciones de base; en vez de hard-codear SQL en cada notebook, usa un registry central `create_script.py`. Gestiona el lifecycle completo (creación, verificación, truncado, borrado) desde una interfaz.

![[09-fig-9.2.png]]
*Figure 9.2: The workflow of the Oracle DBA console*

### Schema registry y scope

Innovación core: **externalizar las definiciones de schema** (no atrapadas en un `.ipynb`, difícil de versionar). Se descarga `create_script.py` (dict de `CREATE` statements por capítulo/use case). Se define el **scope** con `CURRENT_SCOPE` (qué set de tablas targetear — un DBA rara vez opera sobre toda la base) y un `TABLE_REGISTRY` que mapea capítulos → tablas (context-aware):

```python
CURRENT_SCOPE = 'CHAPTER_3'   # 'CHAPTER_2' / 'CHAPTER_3' / 'CHAPTER_9'

TABLE_REGISTRY = {
    'CHAPTER_2': ['CONTEXT_LIBRARY', 'KNOWLEDGE_STORE'],
    'CHAPTER_3': [
        'CONTEXT_LIBRARY', 'KNOWLEDGE_STORE',   # Shared infrastructure
        'CANDIDATE_POOL', 'RECRUITMENT_RULES'   # CH3 Specific
    ]
}
```

Las tablas del cap. 9 (en el `create_script.py`): **`MEDIA_ASSETS`** (catálogo de videos) y **`MEDIA_SEGMENTS`** (segmentos vectorizados, con `description_vector VECTOR(1536, FLOAT32)` y FK a MEDIA_ASSETS).

### Schema operations engine — `manage_schema`

Workhorse del console. Acepta un `mode`. Antes de actuar hace un **idempotency check** consultando `user_tables` (idempotencia = correr la misma operación N veces no produce efecto adicional, porque chequea si el cambio ya se aplicó):

```python
def manage_schema(cursor, mode='CLEAN'):
    """mode: 'CLEAN' (truncate) or 'DROP' (delete table structure)."""
    print(f"\n--- ⚙️ Starting Schema Operation: {mode} ---")
    for table in active_tables:
        try:
            cursor.execute(f"SELECT count(*) FROM user_tables WHERE table_name = '{table}'")
            exists = cursor.fetchone()[0]
            if exists:
                if mode == 'DROP':
                    cursor.execute(f"DROP TABLE {table} PURGE")
                elif mode == 'CLEAN':
                    cursor.execute(f"TRUNCATE TABLE {table}")
            else:
                print(f"   ⚠️ Skipping {table}: Not found in database.")
        except Exception as e:
            print(f"   ❌ Error processing {table}: {e}")
```

**`verify_schema`** (auditor — *trust but verify*): itera `active_tables`, chequea existencia y cuenta filas con tags visuales `✅ [EMPTY]` / `📦 [DATA PRESENT]` / `❌ [DROPPED / NOT FOUND]`.

### Interactive DBA console

Se envuelve la lógica en `ipywidgets`. El **`CREATE` es especial**: requiere importar y **recargar** el módulo `create_script` (`importlib.reload`) para captar cambios del DDL externo sin reiniciar el kernel:

```python
import importlib
def manage_schema(cursor, scope, mode):
    if mode == 'CREATE':
        import create_script
        importlib.reload(create_script)
        catalog = create_script.DDL_CATALOG.get(scope)
        for entry in catalog:
            table_name = entry['table_name']; sql = entry['sql']
            cursor.execute(f"SELECT count(*) FROM user_tables WHERE table_name = '{table_name}'")
            if not cursor.fetchone()[0]:
                cursor.execute(sql)   # create only if not exists
            else:
                print(f"   ⚠️ Skipping {table_name}: Already exists.")
        connection.commit()
        return
    # ... DROP/CLEAN ...
```

UI con dropdowns **Scope** (`CHAPTER_2/3/9`) y **Action** (`VERIFY`, `DISPLAY`, `CREATE`, `CLEAN`, `DROP`) + un handler que rutea y **auto-verifica** tras cada mantenimiento:

```python
def on_button_clicked(b):
    scope = scope_dropdown.value; action = action_dropdown.value
    if action == 'VERIFY':
        verify_schema(cursor, scope)
    elif action == 'DISPLAY':
        display_table_data(cursor, scope)
    else:
        manage_schema(cursor, scope, action)
        verify_schema(cursor, scope)   # Always auto-verify after maintenance
```

> [!warning] Crear las tablas del cap. 9 antes de seguir
> Hay que crear las tablas de este capítulo vía el DBA console **antes** del Video Store agent. Cuidado: si se hace `CLEAN`/`EMPTY` hay que repoblar; si se hace `DROP` hay que recrear.

## Video dataset visualization console

Para un multimodal RAG real hay que extender la arquitectura al **video** (explotando en contextos educativos/training). El notebook `Video_dataset_visualization.ipynb` es la tool de exploración inicial: descargar, inspeccionar y mostrar videos **AI-generados con OpenAI Sora**, estableciendo el ground truth.

**Setup**: `cv2` (OpenCV, procesamiento de video), `base64` (encoding para display), PIL, pandas, numpy, BytesIO. Acceso seguro a un repo GitHub privado vía token.

**`download`** — construye un comando `curl` con el token de auth (headers + redirects); `download_video` lo abstrae al directorio `Chapter09/videos`:

```python
def download(directory, filename):
    base_url = 'https://raw.githubusercontent.com/Denis2054/RAG-Driven-Generative-AI-2nd-Edition/main/'
    file_url = f"{base_url}{directory}/{filename}"
    try:
        curl_command = f'curl -L -o "{filename}" "{file_url}"'
        subprocess.run(curl_command, check=True, shell=True)   # check=True raises on failure
        print(f"Downloaded '{filename}' successfully.")
    except subprocess.CalledProcessError:
        print(f"Failed to download '{filename}'.")
```

**Display utilities**: `display_video` (lee el `.mp4`, lo encodea a Base64 y lo embebe en un `<video>` HTML — reproduce nativo en el notebook); `display_video_frame` (OpenCV captura un **frame específico**, lo convierte a imagen y lo muestra — thumbnail rápido sin reproducir todo):

```python
def display_video_frame(file_name, frame_number, size):
    cap = cv2.VideoCapture(file_name)
    cap.set(cv2.CAP_PROP_POS_FRAMES, frame_number)
    success, frame = cap.read()
    if not success:
        return "Failed to grab frame"
    frame = cv2.cvtColor(frame, cv2.COLOR_BGR2RGB)   # BGR -> RGB
    img = Image.fromarray(frame).resize(size, Image.LANCZOS)
    # ... save to BytesIO as JPEG, base64-encode, embed in <img> HTML ...
```

**Análisis de propiedades** (metadata técnica esencial para el chunking del RAG): `CAP_PROP_FRAME_COUNT`, `CAP_PROP_FPS`, y duración = frames/fps. Ej. del intro `AI_Professor_Introduces_New_Course.mp4`: **340 frames, 30.0 fps, 11.33 segundos**.

**Batch processing**: una lista `lfiles` de ~21 videos de deportes/actividades AI-generados (jogging, skiing, soccer, surfer, swimming, basketball, football, hockey, alpinist…) — el catálogo de la video store. Se descargan en loop y se inspeccionan (metadata + thumbnail) como un **contact sheet** para verificar la diversidad del dataset.

## Implementar el Video Store agent

*Una librería es inútil sin un bibliotecario*: el usuario no puede browsear cientos de videos para hallar *"video of a surfer riding a big wave"*. El **conversational RAG Video Store agent** conecta la pregunta del usuario a la metadata específica.

![[09-fig-9.11.png]]
*Figure 9.11: The workflow of the conversational agent*

**Las 5 etapas del workflow**: (1) **User query** (*"I want to see a surfer"*); (2) **Vectorization** (OpenAI Embedding node, `text-embedding-3-small`, texto→vector); (3) **Semantic search** (Cosine Similarity engine calcula la distancia entre la query y la Video Metadata del knowledge base); (4) **Context retrieval** (los "Top Results" al GPT-4 Agent); (5) **Generative output** (sintetiza la metadata recuperada en una respuesta conversacional Y dispara el Video Player con el `.mp4` exacto, ej. `surfer1.mp4`).

### El esquema dual — `MEDIA_ASSETS` + `MEDIA_SEGMENTS`

Se popula un esquema relacional en Oracle 23ai (ya no file-based). **`MEDIA_ASSETS`** = *relational source of truth* del catálogo (resuelve clips a su ubicación física):

```sql
CREATE TABLE MEDIA_ASSETS (
    asset_id NUMBER GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
    filename VARCHAR2(255),
    category VARCHAR2(100),
    video_url VARCHAR2(1000)
)
```

**`MEDIA_SEGMENTS`** = *sovereign memory* donde ocurre el multimodal-RAG (representaciones vectorizadas de momentos específicos):

```sql
CREATE TABLE MEDIA_SEGMENTS (
    segment_id NUMBER GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
    asset_id NUMBER,
    timestamp_start VARCHAR2(50),
    description CLOB,
    description_vector VECTOR(1536, FLOAT32),   -- Pinned for text-embedding-3-small
    CONSTRAINT fk_asset FOREIGN KEY (asset_id) REFERENCES MEDIA_ASSETS(asset_id)
)
```

- **`timestamp_start`** — habilita **in-video retrieval** marcando la ubicación temporal exacta del evento visual.
- **`description_vector VECTOR(1536, FLOAT32)`** — optimizado para `text-embedding-3-small`; **`FLOAT32`** asegura alineación matemática high-performance con las librerías numéricas de Python.
- **`fk_asset`** — integridad referencial estricta entre segmentos y su asset padre. (Separar metadata high-level de los vector segments high-frequency permite resolver instantáneamente la source URL de cualquier clip recuperado.)

### Crawler — ingesta idempotente

Un "crawler interno" que visita el repo GitHub (file server de la empresa), fetch de la metadata (CSVs) + video URLs, genera embeddings y los inserta. **Idempotente**: chequea si un cambio es necesario antes de hacerlo (re-correr no duplica).

```python
def get_embedding(text, model="text-embedding-3-small"):
    text = text.replace("\n", " ")
    return client.embeddings.create(input=[text], model=model).data[0].embedding
```

**Provisioning loop** (gatekeeper inteligente — evita `ORA-00955` "name already used"):

```python
schemas = {"MEDIA_ASSETS": assets_sql, "MEDIA_SEGMENTS": segments_sql}
for table_name, create_query in schemas.items():
    cursor.execute(f"SELECT count(*) FROM user_tables WHERE table_name = '{table_name}'")
    if cursor.fetchone()[0] == 0:
        cursor.execute(create_query)   # create if missing
        print(f"✅ {table_name} provisioned successfully.")
    else:
        print(f"✔️ {table_name} already exists.")
```

**`ingest_video_metadata`** — por cada video: (1) **idempotency check** (`SELECT asset_id FROM MEDIA_ASSETS WHERE filename = :1` → skip si existe); (2) fetch del CSV; (3) insertar en `MEDIA_ASSETS` capturando el PK auto-generado con **`RETURNING asset_id INTO :4`** (para linkear los segmentos); (4) parsear el CSV con pandas, generar embedding por fila (comment→vector), y **bulk insert** en `MEDIA_SEGMENTS` con `executemany`:

```python
def ingest_video_metadata(cursor, connection):
    for video_file in video_files:
        # 1. IDEMPOTENCY CHECK
        cursor.execute("SELECT asset_id FROM MEDIA_ASSETS WHERE filename = :1", [video_file])
        if cursor.fetchone():
            print(f"   ⚠️ Asset already exists. Skipping ingestion.")
            continue
        # ... fetch CSV from GitHub ...
        # B. Insert catalog entry, capture auto-generated PK
        out_id_var = cursor.var(int)
        cursor.execute(
            "INSERT INTO MEDIA_ASSETS (filename, category, video_url) VALUES (:1, :2, :3) RETURNING asset_id INTO :4",
            [video_file, category, video_url, out_id_var])
        asset_id = out_id_var.getvalue()
        if isinstance(asset_id, list): asset_id = asset_id[0]
        # C. Process text descriptions -> embeddings -> bulk insert
        df = pd.read_csv(io.StringIO(response.text))
        rows_to_insert = []
        for index, row in df.iterrows():
            vector_data = get_embedding(str(row[comment_col]))
            vector_array = array.array('f', vector_data)   # 'f' = FLOAT32
            rows_to_insert.append((asset_id, str(row[timestamp_col]), str(row[comment_col]), vector_array))
        if rows_to_insert:
            cursor.executemany(
                "INSERT INTO MEDIA_SEGMENTS (asset_id, timestamp_start, description, description_vector) VALUES (:1, :2, :3, :4)",
                rows_to_insert)
        connection.commit()
```

Output: `⚠️ Asset already exists (ID: 159). Skipping ingestion.` cuando la idempotencia lo detecta.

### La clase `VideoStoreAgent`

Encapsula la lógica (puente entre usuario, vector DB y modelo generativo). `__init__` con `client`, `cursor` y un **system prompt** que define la persona ("expert Video Library Assistant" que cita siempre filename+timestamp):

```python
class VideoStoreAgent:
    def __init__(self, client, cursor):
        self.client = client
        self.cursor = cursor
        self.system_prompt = """
        You are an expert Video Library Assistant.
        1. Analyze the user's request.
        2. Use the database search results to answer.
        3. Always cite the filename and timestamp.
        4. Be concise and helpful.
        """
    def get_embedding(self, text):
        text = text.replace("\n", " ")
        vec = self.client.embeddings.create(input=[text], model="text-embedding-3-small").data[0].embedding
        return array.array('f', vec)   # same semantic space as ingestion
```

**`search_videos`** — el poder de la **converged database**: una query SQL nativa que combina **JOIN relacional + matemática de vectores**. `VECTOR_DISTANCE` halla el segmento semánticamente más cercano, **`ORDER BY ... ASC`** (menor distancia = mejor match primero), `FETCH FIRST :2 ROWS ONLY`:

```python
def search_videos(self, query, top_k=1):
    query_vector = self.get_embedding(query)
    sql = """
    SELECT a.filename, a.video_url, s.timestamp_start, s.description
    FROM MEDIA_SEGMENTS s
    JOIN MEDIA_ASSETS a ON s.asset_id = a.asset_id
    ORDER BY VECTOR_DISTANCE(s.description_vector, :1) ASC
    FETCH FIRST :2 ROWS ONLY
    """
    self.cursor.execute(sql, [query_vector, top_k])
    return self.cursor.fetchall()
```

**`display_video_clip`** — descarga el video a memoria, lo encodea a Base64 y lo embebe en un **HTML5 player con autoplay**; parsea robustamente el timestamp ("MM:SS" o segundos crudos) y usa **`#t=<seconds>`** para **auto-seeking** al momento exacto.

**`ask`** — orquesta el pipeline RAG: (1) **Retrieval** (`search_videos`), (2) **Generation** (construye un prompt con la query + el contexto de la base, lo manda a **GPT-4o**), (3) **Output** (imprime la explicación), (4) **Display** (reproduce el clip):

```python
def ask(self, user_query):
    results = self.search_videos(user_query, top_k=1)
    if not results:
        return "I couldn't find any matching videos."
    top_match = results[0]
    context = f"File: {top_match[0]}, Time: {top_match[2]}, Desc: {top_match[3]}"
    prompt = f"User: {user_query}\nContext: {context}"
    response = self.client.chat.completions.create(
        model="gpt-4o",
        messages=[
            {"role": "system", "content": self.system_prompt},
            {"role": "user", "content": prompt}
        ])
    print(response.choices[0].message.content)
    self.display_video_clip(top_match[1], top_match[0], top_match[2])
```

### El agente en acción

```python
agent = VideoStoreAgent(client, cursor)
agent.ask("Find me a clip of a basketball player scoring a basket")
```

Pipeline completo: log de la búsqueda en la base → respuesta de GPT-4o (cita `basketball3.mp4` at the 91-second mark) → descarga del video → player. Otras queries: *"I want to see a surfer"*, *"I want to see a soccer player"*.

![[09-fig-9.12.png]]
*Figure 9.12: The video output of a basketball player scoring*

> [!tip] Cierra el loop multimodal
> Se conecta **la pregunta del usuario → el entendimiento semántico de la metadata (refinada por RLHF) → la recuperación del archivo de video físico**, transformando un file server pasivo en una experiencia conversacional inteligente.

## Citas

> "This chapter marks our entry into multimodal RAG, where AI must bridge the semantic gap between a user's natural language request and a library of visual artifacts."
> "we treat database schemas as code."
> "However, a library is useless without a librarian."
> "We successfully closed the loop between the user's question, the semantic understanding of the metadata, and the retrieval of the physical video file."

## Para aplicar

- **Separar la arquitectura multimodal en planos** (control/semantic/visual) — infra, metadata+embeddings, assets; cada uno en su notebook/zona, comunicados por IDs y data contracts.
- **Tratar los schemas como código (schema-as-code)** — externalizar las definiciones en un `create_script.py` versionado en el repo remoto (source of truth); deployment determinístico, sin configuration drift.
- **Construir un DBA console unificado** — desacoplar definición de ejecución; gestionar el lifecycle (CREATE/VERIFY/DISPLAY/CLEAN/DROP) por dropdowns; `importlib.reload` para captar cambios del DDL sin reiniciar kernel; auto-verificar tras cada mantenimiento.
- **Hacer toda operación idempotente** — chequear `user_tables` / filename antes de crear/insertar; evita `ORA-00955`/duplicados y permite re-correr notebooks sin corromper el vector store.
- **Verificar el ground truth antes de automatizar** — actuar como data analyst: descargar, inspeccionar propiedades con OpenCV (frame count, fps, duración), generar thumbnails (contact sheet) para verificar diversidad e integridad.
- **Refinar la metadata con RLHF** — los embeddings se basan en descripciones verificadas (cap. 8), no en captions automáticos ruidosos; ground truth de alta calidad = retrieval de calidad.
- **Diseñar un esquema dual: catálogo relacional + vector store** — `MEDIA_ASSETS` (filename/category/url) separado de `MEDIA_SEGMENTS` (timestamp, description CLOB, `VECTOR(1536, FLOAT32)`, FK); permite **in-video retrieval** por timestamp y resolver la source URL al instante.
- **Pinear el vector a FLOAT32 y usar `array.array('f', ...)`** — alineación matemática high-performance con `text-embedding-3-small` y el driver Oracle.
- **Capturar el PK auto-generado con `RETURNING ... INTO`** — para linkear los segmentos hijos al asset padre; bulk insert con `executemany`.
- **Hacer hybrid query con JOIN + `VECTOR_DISTANCE` (`ORDER BY ... ASC`)** — sobre la converged database; menor distancia = mejor match; `FETCH FIRST k`.
- **Embeber el mismo modelo de embedding en ingesta y query** — para que la query viva en el mismo espacio semántico que las descripciones almacenadas.
- **Reproducir el clip auto-seeking con `#t=<seconds>`** — parsear robustamente el timestamp (MM:SS o segundos), Base64 + HTML5 player con autoplay; el agente cita filename+timestamp y muestra la evidencia visual ground truth.

## Conexiones

- [[_RAG-Driven Generative AI - Second Edition|Mapa del libro]] — el MOC del libro.
- [[08 - Boosting RAG Performance with Human Feedback]] — cap. 8: la metadata de los videos fue **refinada vía el loop RLHF/human feedback** de allí → ground truth de alta calidad para los embeddings.
- [[02 - RAG Embeddings in Oracle Vector Stores]] — cap. 2: el patrón `VECTOR(1536)`, `VECTOR_DISTANCE`, `setinputsizes`/idempotencia con `user_tables` reusado; el DBA console profesionaliza el schema-scripting de aquí.
- [[03 - Building a Live Recruiter Agent]] — cap. 3: el `CANDIDATE_POOL`/`RECRUITMENT_RULES` que el `TABLE_REGISTRY` cataloga; la **converged database** (relacional + vector en una query) es la misma idea, ahora multimodal.
- [[01 - Why Retrieval-Augmented Generation]] — cap. 1: el ecosistema RAG; modular RAG extendido a multimodal; cosine/vector distance.
- [[10 - Building an Agent with Spatial-RAG and GraphRAG]] — cap. 10: extiende el **DBA console / schema registry** de aquí (5 updates); Spatial-RAG + GraphRAG sobre la converged DB.
- [[_RAG|RAG]] · [[Enterprise RAG Assistant]] · [[Hybrid Search]] · [[Reranking]] · [[Chunking Strategies]] — el patrón RAG del vault; multimodal RAG como extensión.
- [[Embeddings]] · [[Cosine Similarity]] · [[Vector Database]] — los embeddings de descripciones, la semantic search, el vector store (candidatos a nota propia).
- [[Function Calling]] · [[Tool Calling]] · [[Orchestrator]] — el `VideoStoreAgent` como agente con tools (search + display); el DBA console como control plane.
- [[Grounding]] · [[Hallucinations]] — el agente cita filename+timestamp y reproduce la **evidencia visual ground truth** (grounding multimodal).
- **Multimodal RAG · In-video retrieval · Tri-planar architecture (control/semantic/visual plane) · Schema-as-code / configuration drift · `create_script.py` registry · DBA console · Idempotency (`user_tables`, `ORA-00955`) · `MEDIA_ASSETS` / `MEDIA_SEGMENTS` · `VECTOR(1536, FLOAT32)` · `RETURNING ... INTO` · `executemany` · OpenCV (`cv2`) · OpenAI Sora (video gen) · Base64 video player / `#t=` auto-seek · GPT-4o · `text-embedding-3-small`** — conceptos/técnicas del capítulo; candidatos a nota propia (varios ya existen en el vault, otros son semillas).
