---
title: 10 - Building an Agent with Spatial-RAG and GraphRAG
libro: RAG-Driven Generative AI - Second Edition
autor: Denis Rothman
capitulo: 10
created: 2026-06-11
tags:
  - libros/rag-driven-generative-ai
  - type/case-study
  - status/permanent
aliases:
  - Building an Agent with Spatial-RAG and GraphRAG
  - Spatial-RAG and GraphRAG
updated: 2026-06-11
---

# 10 - Building an Agent with Spatial-RAG and GraphRAG

> [!info] Capítulo 10 · *RAG-Driven Generative AI - Second Edition* — Denis Rothman
> RAG más allá de la semántica: **espacio y relaciones como señales de retrieval first-class**. **Spatial-RAG** (`SDO_GEOMETRY`) + **GraphRAG** (SQL Property Graphs / SQL/PGQ), nativos en la **converged database** de Oracle 23ai → razonamiento spatial+graph+vector+relacional en **un solo SQL**. Se extiende el DBA console del cap. 9 y se construye un **hyper-contextual Recruiter** que filtra por significado, dinero, ubicación y relaciones a la vez. Navegá: [[_RAG-Driven Generative AI - Second Edition|Mapa del libro]] · anterior [[09 - Building a Conversational RAG Agent]] · siguiente [[11 - Scaling AI Workloads with Oracle Exadata]].

## Resumen

Hasta ahora los agentes operaban en un **vacío de puro significado**: podían responder *qué* contiene un datapoint (vía vectores), pero no *dónde* existe en representaciones de alto nivel del mundo real. Este capítulo extiende RAG **más allá de la semántica**, introduciendo **espacio y relaciones como señales de retrieval first-class**: **Spatial-RAG** usando el motor espacial nativo de Oracle (`SDO_GEOMETRY`) y **GraphRAG** usando **SQL Property Graphs (SQL/PGQ)**. Ambos corren **nativamente dentro de la Converged Database de Oracle 23ai**, permitiendo que el razonamiento spatial, graph, vector y relacional opere en **un único contexto de ejecución SQL**.

Se abandona la **polyglot persistence** (motores especializados separados por tipo de dato — que obliga a mover data por la red, con latencia y problemas de sincronización) por una **arquitectura convergida**: relacional, vectores y property graphs en **un único engine que comparte memoria y storage**. Esto habilita **hybrid searches**: filtrar por inventory status, buscar similitud visual con vectores, y traversar relaciones sociales con grafos — **todo en una sola query SQL sin overhead de red**. El cap. construye sobre la inversión del cap. 9 **extendiendo el universal Oracle DBA console** para soportar los nuevos requisitos convergidos (spatial + graph schemas), sin escribir SQL ad-hoc.

El resultado es un **hyper-contextual Recruiter**: en caps. previos el agente solo respondía por similitud semántica (encontraba un *Python Developer* pero sin concepto de realidad física ni confianza social). Ahora responde preguntas complejas del mundo real: *"¿Está este candidato a 30 minutos de commute?"*, *"¿Lo refirió un empleado de confianza?"* — ejecutando **un solo SQL query que filtra por meaning, money, location y relationships a la vez**. El núcleo es la **hyper-query** que combina tres engines: **vector** (`VECTOR_DISTANCE` skills↔resume), **spatial** (`SDO_GEOM.SDO_DISTANCE` radio desde la oficina), **graph** (`GRAPH_TABLE` traversal de paths de referral válidos). Evita el *data movement tax* (no ETL a un GIS externo como ArcGIS/PostGIS ni a un graph DB como Neo4j). El agente envuelve esto con GPT-5.2 para explicar *por qué* un candidato es la mejor opción, valorando **trust (referrals) y presence (location) tanto como skill**.

![[10-fig-10.1.png]]
*Figure 10.1: The converged architecture*

## La arquitectura convergida

Se abandona la **polyglot persistence** (motores especializados separados → mover data por red = latencia + sync). En su lugar, **converged architecture** con Oracle 23ai: relacional + vector embeddings + property graphs en un engine que **comparte memoria y storage** → hybrid searches en una sola query SQL sin overhead de red.

### Vector search en Oracle 23ai

Tipo `VECTOR` nativo (no un blob ni array genérico, sino **especializado y optimizado para CPU**). **Formatos de precisión** para optimizar storage al escalar a billones de filas: **FLOAT32** (32-bit, máxima accuracy), **INT8** (8-bit signed, reducción significativa), **BINARY** (bits, compresión extrema). **Índices ANN** (escanear cada vector sería muy lento): **HNSW** (índice grafo in-memory, mejor performance/accuracy pero más memoria — vive en un vector memory pool dedicado) e **IVF (Inverted File Flat)** (neighbor-partition por clustering, en disco + cacheado, apto para datasets masivos que exceden RAM). Métricas: dot product o cosine similarity (alineadas con el embedding model).

### SQL Property Graphs

Para entender **relaciones estructurales** (nodes + edges). Históricamente requería un graph DB separado + lenguaje especializado; aquí se usa el estándar **SQL/PGQ** que define property graphs **sobre las tablas relacionales existentes** — **sin exportar ni duplicar data**. Se define una graph view sobre el schema relacional y se usa el operador **`GRAPH_TABLE`** para pattern matching declarativo dentro de SQL estándar; los resultados de un traversal se **joinean directo con tablas relacionales** en la misma query. Habilita **vector-enhanced graph traversals**: un vector search identifica nodos de inicio, luego los edges se traversan para descubrir resultados estructuralmente relacionados → esto es **GraphRAG**.

### Capa de aplicación Python

`python-oracledb` en **thin mode** (Python puro, conecta directo sin client libraries locales; deploy simple + soporte completo de vector/graph). Tres módulos: **DBA module** (schema + índices), **Data engineering module** (embeddings + ingesta), **Analyst notebook** (hybrid queries + visualizaciones).

## Extender el DBA console para spatial y graph

Se construye sobre la infra del cap. 9 (control plane unificado, schema registry centralizado) en vez de scripts aislados. Solo **5 updates** al control plane existente: actualizar el registry externo, registrar el nuevo scope, ejecutar el create workflow.

![[10-fig-10.4.png]]
*Figure 10.4: Implementing the vertex table, the edge table, and the Property Graph*

> [!note] El flujo convergido (Figura 10.1)
> Una user query se parsea y ejecuta across **tres engines**: el **Spatial Engine** filtra candidatos por radio, el **Graph Engine** valida paths de social trust, el **Vector Engine** los rankea por skill. Los tres operan sobre la tabla `CANDIDATE_POOL_GEO` antes de pasar los resultados al **AI Recruiter**.

### Actualizar el schema registry (`create_script.py`)

Se agrega la key `CHAPTER_10` al `DDL_CATALOG` con tres tablas. **`CANDIDATE_POOL_GEO`** incluye la columna **`home_location SDO_GEOMETRY`** (formato nativo de Oracle para puntos/líneas/polígonos) + `resume_vector VECTOR(1536)`; **`EMPLOYEES`** (graph vertex: referrers); **`REFERRALS`** (graph edge: linkea empleados↔candidatos, con FKs a ambos):

```python
"CHAPTER_10": [
    {
        "table_name": "CANDIDATE_POOL_GEO",
        "description": "Spatial table: Copy of Candidate Pool with added Geolocation data.",
        "sql": """
        CREATE TABLE candidate_pool_geo (
            candidate_id VARCHAR2(50) PRIMARY KEY,
            full_name VARCHAR2(100),
            summary CLOB,
            skills VARCHAR2(1000),
            years_experience NUMBER,
            salary_expectation NUMBER,
            resume_vector VECTOR(1536),
            home_location SDO_GEOMETRY
        )
        """
    },
    {
        "table_name": "EMPLOYEES",
        "description": "Graph Vertex: Internal employees who can refer candidates.",
        "sql": "CREATE TABLE employees (emp_id VARCHAR2(50) PRIMARY KEY, name VARCHAR2(100), role VARCHAR2(100))"
    },
    {
        "table_name": "REFERRALS",
        "description": "Graph Edge: Links Employees to Candidates.",
        "sql": """
        CREATE TABLE referrals (
            referrer_id VARCHAR2(50),
            candidate_id VARCHAR2(50),
            relationship VARCHAR2(50),
            PRIMARY KEY (referrer_id, candidate_id),
            CONSTRAINT fk_referrer FOREIGN KEY (referrer_id) REFERENCES employees(emp_id),
            CONSTRAINT fk_candidate FOREIGN KEY (candidate_id) REFERENCES candidate_pool_geo(candidate_id)
        )
        """
    }
]
```

### Actualizar el `TABLE_REGISTRY` y la UI

En el notebook se agrega `'CHAPTER_10': ['CANDIDATE_POOL_GEO', 'EMPLOYEES', 'REFERRALS']` al `TABLE_REGISTRY` (context-aware: VERIFY/DROP targetean solo estas 3, ignorando las de caps. previos), y se agrega `'CHAPTER_10'` a las options del `scope_dropdown` (default). Esto **expone las capacidades sin alterar la lógica de `manage_schema`**.

**Crear + verificar**: Scope=CHAPTER_10, Action=CREATE → el console **recarga `create_script.py`** y ejecuta los DDL (idempotente, chequea existencia primero). Luego Action=VERIFY confirma las 3 tablas `✅ [EMPTY]`. *Sin una línea de SQL ad-hoc en los notebooks.*

## Construir el hyper-contextual Recruiter

En caps. previos el agente solo respondía por similitud semántica (encontraba un *Python Developer* pero sin **realidad física ni social trust**). Ahora responde *"¿está a 30 min de commute?"* y *"¿lo refirió un empleado de confianza?"* con **un solo SQL que filtra por meaning, money, location y relationships**.

**Setup**: `oracledb==3.4.1` (pineado por estabilidad/vector features), thin mode. (Posible conflicto menor de deps OpenAI/Colab: si pide restart, reiniciar; o instalar OpenAI primero en sesión nueva.)

### La dimensión espacial: geolocation

Para no corromper el dataset educativo original (`CANDIDATE_POOL` del cap. 3), se trabaja con una **shadow copy** `CANDIDATE_POOL_GEO`. Se copia la data y se simulan ubicaciones físicas actualizando `HOME_LOCATION` (`SDO_GEOMETRY`). Coordenadas: algunos en downtown SF, suburbs, NY:

```python
cursor.execute("TRUNCATE TABLE candidate_pool_geo")   # idempotency

# Populate from Chapter 3 source (explicit columns to match new schema)
cursor.execute("""
    INSERT INTO candidate_pool_geo (candidate_id, full_name, summary, skills, years_experience, salary_expectation, resume_vector)
    SELECT candidate_id, full_name, summary, skills, years_experience, salary_expectation, resume_vector
    FROM candidate_pool
""")
print(f"✅ Data copied. Rows inserted: {cursor.rowcount}")   # -> 5

# Group A: San Francisco Downtown
cursor.execute("""
    UPDATE candidate_pool_geo
    SET home_location = SDO_GEOMETRY(2001, 4326, SDO_POINT_TYPE(-122.4194, 37.7749, NULL), NULL, NULL)
    WHERE full_name IN ('Alex V.', 'Casey M.')
""")
# Group B: SF Suburbs (-122.25, 37.80) WHERE full_name LIKE 'Quinn%'
# Group C: New York (-74.0060, 40.7128) WHERE full_name LIKE 'Jordan%'
connection.commit()
```

> [!note] `SDO_GEOMETRY(2001, 4326, SDO_POINT_TYPE(lon, lat, NULL), ...)`
> **2001** = tipo geometría punto 2D; **4326** = SRID **WGS 84** (el sistema de coordenadas GPS estándar lat/lon). `SDO_POINT_TYPE(lon, lat, NULL)` — nótese el orden **longitud, latitud**.

### La dimensión social: SQL Property Graphs

Para juzgar la confianza se usan **referrals**. Tres componentes (Figura 10.4): **vertex table `EMPLOYEES`** (referrers), **edge table `REFERRALS`** (linkea empleado↔candidato en `CANDIDATE_POOL_GEO`), y el **Property Graph Object** (vista virtual que habilita la sintaxis de traversal).

```python
cursor.execute("TRUNCATE TABLE referrals")   # Child first (DDL auto-commits)
cursor.execute("DELETE FROM employees")      # Parent second (DML)
connection.commit()   # ← commit el DELETE para liberar row locks (previene ORA-12860)

cursor.execute("INSERT INTO employees VALUES ('EMP_001', 'Alice Stark', 'Principal Architect')")
cursor.execute("INSERT INTO referrals VALUES ('EMP_001', 'CAND_005', 'Former Colleague')")
cursor.execute("INSERT INTO referrals VALUES ('EMP_001', 'CAND_002', 'University Friend')")
connection.commit()
```

**Crear el property graph** (idempotente: drop/create):

```python
try:
    cursor.execute("DROP PROPERTY GRAPH recruitment_graph")
except: pass

cursor.execute("""
    CREATE PROPERTY GRAPH recruitment_graph
    VERTEX TABLES (
        employees KEY (emp_id) LABEL employee PROPERTIES (name),
        candidate_pool_geo KEY (candidate_id) LABEL candidate PROPERTIES (full_name, candidate_id)
    )
    EDGE TABLES (
        referrals
            KEY (referrer_id, candidate_id)
            SOURCE KEY (referrer_id) REFERENCES employees (emp_id)
            DESTINATION KEY (candidate_id) REFERENCES candidate_pool_geo (candidate_id)
            LABEL referred_by PROPERTIES (relationship)
    )
""")
```

> [!warning] `ORA-12860` — commit el DELETE antes del INSERT
> Al resetear la tabla padre con `DELETE` (DML, no auto-commit como `TRUNCATE`), hay que **`commit()` antes de insertar** para liberar los row locks; si no, `ORA-12860`.

### La hyper-query (vector + spatial + graph en un SQL)

El **core de la Converged Database**: no se extrae data a un GIS (ArcGIS/PostGIS) ni a un graph DB (Neo4j) — Oracle 23ai tiene engines nativos enterprise-grade de **spatial y graph dentro del kernel**, evitando el **data movement tax** (no ETL para saber si un candidato vive a 25 millas). Un **único SQL** con tres operaciones simultáneas: **vector engine** (`VECTOR_DISTANCE` job description↔resume), **spatial engine** (`SDO_GEOM.SDO_DISTANCE` radio desde HQ), **graph engine** (`GRAPH_TABLE` traversal del `recruitment_graph` para paths de referral válidos).

**Step 1 — constraints**: job title (meaning), referrer (trust), coordenadas+radio (space).

```python
query_text = "Python Backend Developer"
referrer_name = "Alice Stark"
max_distance_miles = 25
sf_office_lat = 37.7749; sf_office_lon = -122.4194
```

**Step 2 — embedding**: `query_vec = get_embedding(query_text)` (OpenAI → vector).

**Step 3 — la converged query**. El `SELECT` calcula dos scores on-the-fly: `dist_miles` (distancia física) y `semantic_score` (similitud resume↔query). El **graph JOIN** es el *trust filter* (`INNER JOIN GRAPH_TABLE` con un `MATCH` pattern). El `WHERE` es el *spatial filter* (radio):

```python
sql_hyper = """
    SELECT
        c.full_name, c.salary_expectation, c.summary,
        -- Spatial Engine: distance in miles
        ROUND(SDO_GEOM.SDO_DISTANCE(
            c.home_location,
            SDO_GEOMETRY(2001, 4326, SDO_POINT_TYPE(:lon, :lat, NULL), NULL, NULL),
            0.005, 'unit=MILE'
        ), 1) as dist_miles,
        -- Vector Engine: semantic similarity
        ROUND(VECTOR_DISTANCE(c.resume_vector, :vec, DOT), 4) as semantic_score
    FROM candidate_pool_geo c
    -- Graph Engine: trust filter via GRAPH_TABLE
    INNER JOIN GRAPH_TABLE(recruitment_graph
        MATCH (e IS employee WHERE e.name = :ref_name)
        -[r IS referred_by]->
        (c_target IS candidate)
        COLUMNS (c_target.candidate_id AS g_id)
    ) gt ON c.candidate_id = gt.g_id
    WHERE
        -- Spatial Engine: filter by radius
        SDO_GEOM.SDO_DISTANCE(
            c.home_location,
            SDO_GEOMETRY(2001, 4326, SDO_POINT_TYPE(:lon, :lat, NULL), NULL, NULL),
            0.005, 'unit=MILE'
        ) < :max_dist
    ORDER BY semantic_score DESC
"""
```

El `MATCH` define el patrón `(Employee) -[Referred By]-> (Candidate)`; el `INNER JOIN` filtra el result set a **solo** los candidatos hallados en ese traversal.

**Step 4 — ejecutar** (bind vars, `setinputsizes` para el vector):

```python
cursor.setinputsizes(vec=oracledb.DB_TYPE_VECTOR)
cursor.execute(sql_hyper, {
    "lon": sf_office_lon, "lat": sf_office_lat, "vec": query_vec,
    "max_dist": max_distance_miles, "ref_name": referrer_name
})
results = cursor.fetchall()
```

**Step 5 — display**: si un candidato aparece, **pasó los tres tests** (skills/score, vive cerca/spatial, es de confianza/graph):

```text
🔎 Searching for: 'Python Backend Developer'
📍 Spatial Filter: Within 25 miles of SF Office
🤝 Graph Filter:   Must be referred by 'Alice Stark'
--- 🏆 Hyper-Query Results ---
✅ Quinn R.
   - Distance: 9.4 miles from HQ
   - Semantic Score: -0.4134
   - Status: Verified Referral
```

La query **filtró candidatos semánticamente relevantes pero demasiado lejos o sin referral de confianza**.

### Generar la recomendación del Recruiter

`generate_hyper_recommendation` orquesta retrieval + alimenta el contexto multi-dimensional al LLM, instruido como **Hyper-Contextual Recruiter** que valora **TRUST (referrals) y PRESENCE (location) tanto como SKILL**. Pasos: (1) rol+params; (2) `get_embedding(user_query)`; (3) ejecutar la hyper-query (vector+spatial+graph sobre `candidate_pool_geo`); (4) formatear el contexto (name, salary, miles, referral verified, score, summary); (5) prompt que **fuerza al modelo a articular *por qué* location y referral hacen al candidato una contratación más segura que un desconocido**; (6) llamar GPT-5.2:

```python
def generate_hyper_recommendation(user_query, referrer, max_dist_miles):
    query_vec = get_embedding(user_query)
    cursor.setinputsizes(vec=oracledb.DB_TYPE_VECTOR)
    cursor.execute(sql_hyper, {"lon": sf_office_lon, "lat": sf_office_lat,
        "vec": query_vec, "max_dist": max_distance_miles, "ref_name": referrer})
    candidates = cursor.fetchall()
    if not candidates:
        return "⚠️ No candidates found. Try relaxing the distance or referral constraints."
    # format context_text per candidate (read CLOB summary) ...
    system_prompt = "You are a Hyper-Contextual Recruiter. You value TRUST (Referrals) and PRESENCE (Location) as much as SKILL."
    user_prompt = f"""
    USER REQUEST: "{user_query}"
    CANDIDATES FOUND (Filtered by Graph & Spatial DB):
    {context_text}
    TASK:
    1. Recommend the best candidate.
    2. Explicitly mention why their location ({max_dist_miles} mile radius) and referral ({referrer}) make them a safer hire than an unknown candidate.
    """
    response = client.chat.completions.create(
        model="gpt-5.2",
        messages=[{"role": "system", "content": system_prompt},
                  {"role": "user", "content": user_prompt}])
    return response.choices[0].message.content

rec_output = generate_hyper_recommendation(
    user_query="Senior Python Engineer", referrer="Alice Stark", max_dist_miles=25)
```

La IA recomienda **Quinn R.** y articula que la **location (9.4 mi, commutable)** reduce attendance/attrition risk, y el **referral verificado de Alice Stark** agrega trust signals que un applicant desconocido no tiene (Alice apuesta su reputación) — *"even with a weaker vector match score, the combination of close proximity and a verified referral makes Quinn a materially safer, lower-variance hire"*. Mimetiza el razonamiento de un hiring manager que mitiga riesgo.

### Verificar la integridad (shadow table)

Se confirma que `CANDIDATE_POOL_GEO` tiene la columna `HOME_LOCATION: SDO_GEOMETRY` (consultando `user_tab_columns`) **dejando intacta la tabla original `CANDIDATE_POOL`** del cap. 3 — la estrategia shadow-copy respetó la integridad de datos.

## Citas

> "They could answer what a datapoint contains, but not where it exists within higher-level representations of the real world."
> "we extend RAG beyond semantics by introducing space and relationships as first-class retrieval signals."
> "we avoid the data movement tax. We don't have to ETL our candidate records to an external system to find out if they live within 25 miles of the office."
> "a true converged database does not merely store different data types but actively harmonizes them, allowing a single SQL query to weigh the competing priorities of meaning, money, location, and trust simultaneously."

## Para aplicar

- **Extender RAG con espacio y relaciones como señales first-class** — Spatial-RAG (`SDO_GEOMETRY`) + GraphRAG (SQL/PGQ); responder *dónde* y *con quién*, no solo *qué*.
- **Usar una converged database para evitar el data movement tax** — vector + spatial + graph + relacional en un engine; no ETL a GIS (ArcGIS/PostGIS) ni graph DB (Neo4j); menos latencia, datos consistentes y seguros en un lugar.
- **Definir property graphs sobre tablas relacionales (SQL/PGQ)** — vertex tables + edge tables + `CREATE PROPERTY GRAPH`; sin exportar/duplicar data; traversar con `GRAPH_TABLE`/`MATCH` y joinear con relacional en la misma query.
- **Combinar los tres engines en una sola hyper-query** — `VECTOR_DISTANCE` (skill) + `SDO_GEOM.SDO_DISTANCE` (radio) + `GRAPH_TABLE MATCH` (trust path); un candidato que aparece pasó los tres tests.
- **Extender el DBA console en vez de scripts ad-hoc** — agregar el scope al `DDL_CATALOG`, `TABLE_REGISTRY` y dropdown; idempotente, version-controlled, sin SQL ad-hoc en notebooks.
- **Usar shadow tables para no corromper datasets originales** — `CANDIDATE_POOL_GEO` copia y extiende con `HOME_LOCATION` dejando `CANDIDATE_POOL` intacta; verificar con `user_tab_columns`.
- **Almacenar geolocation con `SDO_GEOMETRY(2001, 4326, SDO_POINT_TYPE(lon, lat, NULL)...)`** — tipo punto 2D, SRID 4326 (WGS 84); orden longitud, latitud.
- **Commit el DELETE de la tabla padre antes del INSERT** — liberar row locks para evitar `ORA-12860`; `TRUNCATE` auto-commitea pero `DELETE` no.
- **Elegir formato de vector e índice según escala** — FLOAT32/INT8/BINARY (storage), HNSW (in-memory, accuracy) vs IVF (disk, datasets masivos); métrica DOT/cosine alineada al embedding.
- **Instruir al LLM a valorar TRUST y PRESENCE como SKILL** — forzar que articule por qué location+referral reducen riesgo vs un desconocido; el LLM sintetiza meaning+money+location+trust en una recomendación de negocio.
- **Usar `python-oracledb` en thin mode** — Python puro, sin client libraries; soporte completo de vector/graph; deploy simple.

## Conexiones

- [[_RAG-Driven Generative AI - Second Edition|Mapa del libro]] — el MOC del libro.
- [[09 - Building a Conversational RAG Agent]] — cap. 9: este capítulo **extiende el DBA console / schema registry** de allí (5 updates); la converged DB (relacional+vector) se amplía a spatial+graph.
- [[03 - Building a Live Recruiter Agent]] — cap. 3: el `CANDIDATE_POOL` original (la shadow `CANDIDATE_POOL_GEO` lo copia); el hybrid search (vector+SQL) ahora hyper (vector+spatial+graph); `VECTOR_DISTANCE(..., DOT)` reusado.
- [[02 - RAG Embeddings in Oracle Vector Stores]] — cap. 2: `VECTOR`, `VECTOR_DISTANCE`, `setinputsizes`, HNSW/IVF, idempotencia con `user_tables`.
- [[01 - Why Retrieval-Augmented Generation]] — cap. 1: modular RAG → hyper-converged; el ecosistema RAG ampliado a múltiples engines.
- [[11 - Scaling AI Workloads with Oracle Exadata]] — cap. 11: escalar a nivel enterprise con Oracle Cloud y Exadata (RAGOps); extiende el mismo DBA console (CHAPTER_11) y mide la performance del kernel sobre estos tipos vector.
- [[_RAG|RAG]] · [[Enterprise RAG Assistant]] · [[Hybrid Search]] · [[Reranking]] · [[Chunking Strategies]] — el patrón RAG del vault; Spatial-RAG y GraphRAG como extensiones.
- [[Function Calling]] · [[Tool Calling]] · [[Orchestrator]] — el hyper-Recruiter como agente que orquesta los tres engines vía una query.
- [[Grounding]] · [[Hallucinations]] — el agente fundamenta la recomendación en hechos verificables (distancia exacta, referral verificado) — grounding multi-dimensional.
- **Spatial-RAG · GraphRAG · Converged database / polyglot persistence · `SDO_GEOMETRY` / `SDO_GEOM.SDO_DISTANCE` · SRID 4326 (WGS 84) · GIS · SQL Property Graphs / SQL/PGQ · `GRAPH_TABLE` / `MATCH` / `CREATE PROPERTY GRAPH` · vertex/edge tables · `VECTOR_DISTANCE(..., DOT)` · HNSW / IVF · FLOAT32/INT8/BINARY · Data movement tax · `python-oracledb` thin mode · `ORA-12860` · Hyper-query · Shadow table · GPT-5.2** — conceptos/técnicas del capítulo; candidatos a nota propia (varios ya existen en el vault, otros son semillas).
