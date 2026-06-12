---
title: 03 - Building a Live Recruiter Agent
libro: RAG-Driven Generative AI - Second Edition
autor: Denis Rothman
capitulo: 3
created: 2026-06-11
tags:
  - libros/rag-driven-generative-ai
  - type/case-study
  - status/permanent
aliases:
  - Building a Live Recruiter Agent
  - Recruiter Agent
updated: 2026-06-11
---

# 03 - Building a Live Recruiter Agent

> [!info] Capítulo 3 · *RAG-Driven Generative AI - Second Edition* — Denis Rothman
> De vectorizar archivos a **vectorizar tablas relacionales Oracle directamente**. Se construye un Recruiter agent con **hybrid RAG**: combina vector search (skills semánticos) con filtrado SQL preciso (salario, experiencia) en una sola query sobre la **converged database**, donde datos estructurados (`NUMBER`) y vectores (`VECTOR(1536)`) viven en la misma fila. Misma metodología de 3 fases (DBA → Engineer → Developer) que el cap. 2. Navegá: [[_RAG-Driven Generative AI - Second Edition|Mapa del libro]] · anterior [[02 - RAG Embeddings in Oracle Vector Stores]] · siguiente [[04 - Building Sovereign Enterprise Agents]].

## Resumen

El cap. 2 acopló un MAS a Oracle ingiriendo data **no estructurada** desde un file server externo. Este capítulo evoluciona la arquitectura **moviéndose estrictamente dentro del motor de la base**: se construye un **Recruiter agent vivo** que vectoriza **tablas relacionales Oracle directamente** (no archivos). El blueprint arquitectónico permanece **simétrico** al cap. 2 (las mismas tres fases: DBA management, data ingestion, augmented query), pero la tecnología subyacente se upgradea para soportar **hybrid RAG**: combinar el entendimiento semántico de los vectores (matchear los skills de un candidato) con la precisión del filtrado SQL estructurado (matchear un rango salarial o de experiencia).

El concepto central es la **converged database** (Oracle Database 23ai): almacenar tablas relacionales, documentos JSON y vector embeddings **en un único motor**, eliminando la necesidad de bases especializadas separadas. La **data convergence** lleva esto al extremo: *datos de negocio estructurados y vectores de IA no estructurados viven en la misma fila*. Esto permite **SQL filtering preciso + semantic search en una sola query**. El caso de uso resuelve requests complejos como *"Find me a Python developer with a salary under 80k"*: vector search para "Python developer" + cláusula SQL `WHERE salary < 80000`.

Las tres fases: (1) **Phase 1 — DBA**: definir un **hybrid schema** con tablas especializadas `CANDIDATE_POOL` (atributos estructurados + vector del resume) y `RECRUITMENT_RULES` (personas y criterios de evaluación de los agentes); columnas `NUMBER` (salary/experience) junto a `VECTOR(1536)` en la misma fila. (2) **Phase 2 — Data Engineer**: pipeline ETL de **dual-stream ingestion** que parsea JSON, mapea hechos duros (salary, experience) a columnas SQL escalares vía `MERGE` (idempotente) y, en una pasada de **vector enrichment** separada, genera embeddings del summary y hace `UPDATE` de las columnas vector. (3) **Phase 3 — AI Architect**: el Recruiter agent con **Hybrid Search** (`find_candidates_hybrid`) que aplica el `WHERE` para constraints duros y `VECTOR_DISTANCE(..., DOT)` para ranking semántico; recupera dinámicamente **hiring personas** de la base (Technical Screener, Culture Fit Officer, University Recruiter) para que el LLM cambie su rúbrica de evaluación on demand, y genera una recomendación de contratación grounded y compliant.

![[03-fig-3.1.png]]
*Figure 3.1: The hybrid RAG architecture*

## Arquitectura del Recruiter agent

El blueprint es **simétrico al cap. 2** (mismas 3 fases) pero upgradeado a **hybrid RAG**. El foco es la **data convergence**: data de negocio estructurada y vectores de IA no estructurados en la **misma fila**. La progresión respecto del cap. 2 (Figura 3.1) es un **pipeline dual-stream** donde la data JSON se divide en **SQL scalars** (hard filtering) y **vector embeddings** (semantic search), convergiendo finalmente en una única tabla Oracle:

- **Phase 2 (izquierda/medio)**: el proceso arranca con un "Candidate Profile" (**JSON**) y lo **divide activamente**: *salary* y *experience* suben al path **Structured (SQL)**; el *summary de la persona* baja al path **Semantic (Vector)**.
- **Phase 1 (derecha)**: ambos paths reconvergen en **Oracle 23ai** — una única tabla `CANDIDATE_POOL` que sostiene data SQL (azul) y Vector (naranja) en la misma fila.
- **Phase 3 (abajo)**: el Recruiter agent traduce una query en lenguaje natural a una **Hybrid Query** que apunta simultáneamente a las columnas SQL y a los vector embeddings.

> [!note] Las tres fases del capítulo
> **Phase 1 (DBA / infrastructure)**: schema relacional HR — tablas `CANDIDATE_POOL` y `RECRUITMENT_RULES` (en vez del genérico text storage); vectores `VECTOR` junto a scalars (`SALARY`, `NAME`) en la misma fila. **Phase 2 (Data Engineer / ETL)**: data flow enterprise con JSON estructurado (candidate profiles + job requisitions); parseo que mapea "5 years" a columnas SQL mientras genera embeddings de texto no estructurado (semantic mapping, no simple text chunking). **Phase 3 (LLM-augmented hybrid query)**: el Recruiter agent con **Hybrid Search** que combina vector search + `WHERE salary < 80000` — una IA que entiende **lenguaje humano y constraints de base** a la vez.

## Phase 1: DBA Oracle management (la infraestructura)

Se asume el rol de **DBA**: traducir los requisitos arquitectónicos a SQL **DDL** concreto. A diferencia del cap. 2 (buckets genéricos para texto crudo), aquí se construye un **hybrid schema**: una estructura que **impone reglas de negocio estrictas** (salary caps, niveles de experiencia) mientras simultáneamente **habilita flexibilidad semántica** para el skill matching. Es el bedrock de la converged database.

![[03-fig-3.2.png]]
*Figure 3.2: The role of the DBA as the database owner*

### Global configuration flags

El panel de control con un flag nuevo respecto del cap. 2: `drop_tables` (destructivo). Defaultear a `False` previene overwrites accidentales; el script de creación **skipea tablas existentes** (sobreviven múltiples runs); para reiniciar un experimento, `empty_tables = True` limpia las filas sin destruir las tablas.

```python
unzip_wallet = False  # Set to True only if extracting the wallet for the first time
create_tables = False # Set to True to build the schema (safely skips existing tables)
empty_tables = False  # Set to True to TRUNCATE rows (your safe "reset button")
drop_tables = False   # Set to True only for a destructive clean slate (DROP)
```

### Conexión segura

Mismo patrón que el cap. 2: `oracledb==3.4.1`, lee `credentials.txt` del drive (separa código de credenciales), conecta vía wallet y verifica con `SELECT 'Connected!' FROM dual`. Aquí se usa el helper `read_key_value_file` que devuelve un dict (`creds.get("user")`, etc.).

```python
import oracledb
connection = oracledb.connect(
    user=creds.get("user"), password=creds.get("password"), dsn=creds.get("dsn"),
    config_dir=creds.get("wallet_path"), wallet_location=creds.get("wallet_path"),
    wallet_password=creds.get("wallet_password")
)
cursor = connection.cursor()
cursor.execute("SELECT 'Connected!' FROM dual")
print(cursor.fetchone())   # -> ('Connected!',)
```

### El hybrid schema — `CANDIDATE_POOL` y `RECRUITMENT_RULES`

Se implementa el patrón **converged database**: data relacional estructurada en la **misma fila** que el vector embedding no estructurado → SQL filtering preciso + semantic search en una query.

**`CANDIDATE_POOL`** (la tabla híbrida): mezcla tipos SQL estándar y vector. `years_experience` y `salary_expectation` como `NUMBER` permiten respetar **constraints de negocio duros** que un vector search puro ignoraría.

```python
# 1. CANDIDATE_POOL: The Hybrid Table (Structured + Vector)
cursor.execute("""
CREATE TABLE candidate_pool (
    candidate_id VARCHAR2(50) PRIMARY KEY,
    full_name VARCHAR2(100),
    summary CLOB,                   -- The text we will vectorise
    skills VARCHAR2(1000),          -- Comma-separated list for keywords
    years_experience NUMBER,        -- For SQL Filtering (e.g. > 5 years)
    salary_expectation NUMBER,      -- For SQL Filtering (e.g. < 120k)
    resume_vector VECTOR(1536)      -- The Semantic Brain
)
""")
print("Table CANDIDATE_POOL created.")
```

**`RECRUITMENT_RULES`** (instrucciones domain-specific): separa la lógica de los agentes del `CONTEXT_LIBRARY` genérico (domain isolation). Las personas de hiring (ej. "Culture Fit Officer", "Technical Screener") se aíslan y gestionan independientemente.

```python
# 2. RECRUITMENT_RULES: Domain-Specific Instructions
cursor.execute("""
CREATE TABLE recruitment_rules (
    rule_id VARCHAR2(50) PRIMARY KEY,
    agent_persona CLOB,             -- e.g., "Culture Fit Officer" vs "Technical Screener"
    evaluation_criteria CLOB,       -- The specific rubric for this agent
    rule_vector VECTOR(1536)
)
""")
print("Table RECRUITMENT_RULES created.")
connection.commit()
print("HR Schema initialized successfully.")
```

### Verificar el converged schema (4/4 tablas)

*Trust but verify.* Se confirma que la base aloja **tanto el document store del cap. 2 como el schema híbrido del cap. 3** (4 tablas). Se consulta `user_tables` y se inspeccionan las columnas de `CANDIDATE_POOL` para confirmar visualmente que un `NUMBER` (salary) está **adyacente a un `VECTOR`** (semantic) en la misma definición.

```python
cursor.execute("""
SELECT table_name FROM user_tables
WHERE table_name IN ('CONTEXT_LIBRARY', 'KNOWLEDGE_STORE', 'CANDIDATE_POOL', 'RECRUITMENT_RULES')
ORDER BY table_name
""")
tables = cursor.fetchall()
print(f"Tables found ({len(tables)}/4):", tables)
```

Output clave (la convergencia visible — `NUMBER` y `VECTOR` en la misma tabla):
```text
Tables found (4/4): [('CANDIDATE_POOL',), ('CONTEXT_LIBRARY',), ('KNOWLEDGE_STORE',), ('RECRUITMENT_RULES',)]
--- Column definitions for CANDIDATE_POOL (Hybrid Table) ---
('CANDIDATE_ID', 'VARCHAR2', 50)
('FULL_NAME', 'VARCHAR2', 100)
('SUMMARY', 'CLOB', 4000)
('SKILLS', 'VARCHAR2', 1000)
('YEARS_EXPERIENCE', 'NUMBER', 22)
('SALARY_EXPECTATION', 'NUMBER', 22)
('RESUME_VECTOR', 'VECTOR', 8200)
```

### Data maintenance — TRUNCATE y DROP

**`empty_vector_tables`** (no destructivo): `TRUNCATE` (DDL, resetea contenido y storage high-water mark instantáneamente; mucho más rápido que `DELETE`) sobre las 4 tablas — RAG del cap. 2 (`CONTEXT_LIBRARY`, `KNOWLEDGE_STORE`) + HR del cap. 3 (`CANDIDATE_POOL`, `RECRUITMENT_RULES`). **`verify_tables_empty`** itera las 4 tablas con `SELECT COUNT(*)`, chequea existencia en `user_tables` primero (evita crashes), aplica tags `[EMPTY]` ✅ / `[NOT EMPTY]` ❌ y hace **forensic sampling** (primeras 2 filas) si algo quedó.

**`drop_tables`** (destructivo): `DROP TABLE ... PURGE` (libera storage inmediatamente, sin pasar por el Recycle Bin) en orden de dependencia inverso; envuelto en error handling que **ignora `ORA-00942`** (table does not exist) en runs frescos.

```python
if drop_tables == True:
    tables_to_drop = ["RECRUITMENT_RULES", "CANDIDATE_POOL", "KNOWLEDGE_STORE", "CONTEXT_LIBRARY"]
    for table in tables_to_drop:
        try:
            cursor.execute(f"DROP TABLE {table} PURGE")
            print("✅ DROPPED.")
        except oracledb.Error as e:
            error_obj = e.args[0]
            if error_obj.code == 942:        # ORA-00942: table does not exist (safe to ignore)
                print("⏭️ SKIPPED (Does not exist).")
            else:
                print(f"❌ FAILED: {e}")
```

## Phase 2: Data ingestion (el pipeline ETL)

Rol de **data engineer**. En el cap. 2 era leer archivos de texto crudo; aquí el desafío evoluciona a **hybrid ingestion**: parsear payloads **JSON estructurados**, separar los hechos de negocio duros (salary, experience) de las narrativas semánticas blandas (summaries/cover letters), y rutearlos a las columnas convergidas. Es el bridge que convierte una base estática en un recruitment engine dinámico.

![[03-fig-3.3.png]]
*Figure 3.3: The role of the data engineer (hybrid ingestion)*

### Setup y adquisición de data

Se reconecta a Oracle (la ingestión suele ser un proceso operacional separado, a veces con otro service account). Se descarga el dataset HR (JSON remoto) con `curl` y se carga a memoria:

```python
!curl -L "https://api.github.com/repos/Denis2054/RAG-Driven-Generative-AI-2nd-Edition/contents/enterprise_data/talent_acquisition/hr_data.json" -o hr_data.json

if os.path.exists("hr_data.json"):
    with open("hr_data.json", 'r', encoding='utf-8') as f:
        hr_data = json.load(f)
    print(f"✅ HR Data Loaded Successfully")
    print(f"   - Candidates found: {len(hr_data.get('candidates', []))}")   # -> 5
    print(f"   - Recruitment Rules found: {len(hr_data.get('rules', []))}")  # -> 3
```

**Utilidad de vectorización** reusada (`get_embeddings_batch` con `text-embedding-3-small`):

```python
def get_embeddings_batch(texts, model="text-embedding-3-small"):
    texts = [t.replace("\n", " ") for t in texts]
    response = client.embeddings.create(input=texts, model=model)
    return [item.embedding for item in response.data]
```

### Dual-stream data ingestion (el momento definitorio)

No se trata la data como un blob monolítico: se divide el procesamiento en **dos streams**. *Stream 1* maneja la data estructurada (`years_experience`, `salary_expectation` → columnas relacionales). *Stream 2* maneja la data semántica (el summary se flaggea para vectorización posterior).

**Stream 1** usa un `MERGE` SQL que asegura **idempotencia** (correr el pipeline múltiples veces sin duplicar registros). Se repite para `RECRUITMENT_RULES`:

```python
# Stream 1: Populate Structured SQL Columns
candidates = hr_data.get("candidates", [])
for cand in candidates:
    cursor.execute("""
        MERGE INTO candidate_pool target
        USING (SELECT :candidate_id AS id FROM dual) source
        ON (target.candidate_id = source.id)
        WHEN NOT MATCHED THEN
            INSERT (candidate_id, full_name, years_experience, salary_expectation, skills, summary)
            VALUES (:candidate_id, :full_name, :years_experience, :salary_expectation, :skills, :summary)
    """, {
        "candidate_id": cand['candidate_id'], "full_name": cand['full_name'],
        "years_experience": cand['years_experience'], "salary_expectation": cand['salary_expectation'],
        "skills": cand['skills'], "summary": cand['summary']
    })
print(f"-> Merged {len(candidates)} rows into CANDIDATE_POOL.")
```

Output: `-> Merged 5 rows into CANDIDATE_POOL.` y `-> Merged 3 rows into RECRUITMENT_RULES.` (las columnas vector quedan `NULL` por ahora).

### Vector enrichment (Stream 2)

Pasada de enriquecimiento que **desacopla** la carga transaccional de los datos de negocio del proceso computacionalmente intensivo de vectorización: query por filas donde `resume_vector IS NULL`, lee el summary (CLOB con `.read()`), genera embeddings y hace `UPDATE` poniendo el vector en la celda correcta.

```python
# Stream 2: Vector Enrichment
cursor.execute("SELECT candidate_id, summary FROM candidate_pool WHERE resume_vector IS NULL")
rows_to_process = cursor.fetchall()
if rows_to_process:
    summaries = [row[1].read() for row in rows_to_process]   # read CLOBs
    vectors = get_embeddings_batch(summaries)
    for i, (cand_id, _) in enumerate(rows_to_process):
        cursor.setinputsizes(vec=oracledb.DB_TYPE_VECTOR)
        cursor.execute("""
            UPDATE candidate_pool SET resume_vector = :vec WHERE candidate_id = :id
        """, {"vec": vectors[i], "id": cand_id})
    connection.commit()
    print("-> Candidates updated with vectors.")   # Vectorizing 5 candidates...
```

(Se repite para `RECRUITMENT_RULES` para indexar semánticamente las personas del agente.)

### Verificación de hybrid query

Para probar la ingestión se ejecuta una **Hybrid Query**: hallar un candidato que matchea una descripción semántica (*"Leadership and team building"*) **pero también** satisface un constraint SQL duro (`salary_expectation <= 170000`). Usa el vector del summary para el sort y el escalar salary para el filtro:

```python
user_query = "Leadership and team building experience"
max_budget = 170000
query_vector = get_embeddings_batch([user_query])[0]

cursor.setinputsizes(v=oracledb.DB_TYPE_VECTOR)
cursor.execute("""
    SELECT candidate_id, full_name, salary_expectation,
           VECTOR_DISTANCE(resume_vector, :v, DOT) as similarity
    FROM candidate_pool
    WHERE salary_expectation <= :budget
    ORDER BY similarity DESC
    FETCH FIRST 3 ROWS ONLY
""", {"v": query_vector, "budget": max_budget})
results = cursor.fetchall()
```

Output (candidatos dentro de budget, ordenados por similitud):
```text
Candidate: Riley S. (ID: CAND_004)   Salary: $140,000   Match Score: -0.2118
Candidate: Jordan L. (ID: CAND_002)  Salary: $85,000    Match Score: -0.2528
Candidate: Alex V. (ID: CAND_001)    Salary: $165,000   Match Score: -0.3072
```

## Phase 3: LLM-augmented hybrid query (el Recruiter agent)

Rol de **AI architect**. En el cap. 2 el agente era un retriever simple (similitud semántica sola); aquí evoluciona a **hybrid agent**: piensa semánticamente mientras respeta lógica de negocio rígida (ej. `Salary < 120k` es un constraint duro, no una sugerencia). Sintetiza **NLU (Natural Language Understanding)** con lógica de base estructurada para producir una recomendación grounded y compliant.

![[03-fig-3.4.png]]
*Figure 3.4: The Role of the AI Architect (hybrid agent)*

### Dynamic context retrieval — hiring personas

El contexto no se limita a documentos: incluye la **identidad de la IA misma**. En vez de hard-codear "You are a helpful assistant", se recuperan **hiring personas** de `RECRUITMENT_RULES` → la organización gestiona centralmente cómo se comportan los distintos recruiters. Se cargan a un dict `active_personas` (manejando la conversión LOB con `.read()`):

```python
cursor.execute("SELECT rule_id, agent_persona, evaluation_criteria FROM recruitment_rules")
rules = cursor.fetchall()
active_personas = {}
for r in rules:
    r_id, persona_lob, criteria_lob = r
    persona = persona_lob.read() if hasattr(persona_lob, 'read') else str(persona_lob)
    criteria = criteria_lob.read() if hasattr(criteria_lob, 'read') else str(criteria_lob)
    active_personas[r_id] = {"persona": persona, "criteria": criteria}
```

Las **3 personas** disponibles (el ID es descriptivo, el texto define el título profesional):
- `rule_tech_screener` → *Senior Technical Architect* ("find rock-star coders").
- `rule_culture_screener` → *Culture Fit Officer* ("build a collaborative...").
- `rule_junior_scout` → *University Recruiter* ("find diamonds in the rough...").

### El hybrid search engine — `find_candidates_hybrid`

El motor core. Tres pasos: **(1) vectorizar** la query del usuario; **(2) construir el SQL híbrido** que combina un `WHERE` (constraints duros) con `VECTOR_DISTANCE` (ranking semántico) — usa la métrica **dot product (`DOT`)**, donde más alto = mejor match; **(3) ejecutar** avisando al driver que `:v` es vector type.

```python
def find_candidates_hybrid(user_query, max_salary=1000000, min_experience=0):
    """Performs a Hybrid Search:
    1. VECTOR: Semantically matches the user_query against candidate resumes.
    2. SQL: Filters out candidates who don't meet the salary/experience constraints."""
    query_vector = get_embeddings_batch([user_query])[0]   # 1. Vectorize

    sql = """
        SELECT candidate_id, full_name, years_experience, salary_expectation, summary,
               VECTOR_DISTANCE(resume_vector, :v, DOT) as similarity
        FROM candidate_pool
        WHERE salary_expectation <= :max_sal
          AND years_experience >= :min_exp
        ORDER BY similarity DESC
        FETCH FIRST 3 ROWS ONLY
    """
    cursor.setinputsizes(v=oracledb.DB_TYPE_VECTOR)   # 3. Crucial: :v is a vector
    cursor.execute(sql, {"v": query_vector, "max_sal": max_salary, "min_exp": min_experience})
    results = cursor.fetchall()
    return results
```

> [!note] Hybrid Search = `WHERE` (SQL duro) + `VECTOR_DISTANCE` (semántico)
> Lo único que hace única a esta query es **combinar en una sola sentencia** un `WHERE salary_expectation <= :max_sal AND years_experience >= :min_exp` (constraints de negocio inviolables) con el `ORDER BY VECTOR_DISTANCE(..., DOT) DESC` (ranking por significado). El SQL filtra *quién es elegible*; el vector ordena *quién es más relevante*. Es la culminación de la converged database.

Unit test ("Experienced Python Developer", salary ≤ $150k, exp ≥ 2): encuentra 2 candidatos estrictamente dentro del rango (`Jordan L. $85,000`, `Riley S. $140,000`).

### El Recruiter agent — `generate_hiring_recommendation`

El ensamble final del pipeline RAG (conecta usuario, base y LLM), en pasos: **(1)** validar la persona pedida y cargar su rúbrica del dict en memoria (setea el contexto cognitivo); **(2)** llamar a `find_candidates_hybrid` (evidencia); **(3)** **context injection** — formatear las filas de la base en un bloque de texto legible; **(4)** construir el prompt combinando user query + perfiles inyectados + rúbrica de la persona; **(5)** generar con `gpt-5.2` a `temperature=0.3` (adherencia estricta a los hechos).

```python
def generate_hiring_recommendation(user_query, max_salary, min_exp, persona_id="rule_tech_screener"):
    # 1. Validation
    if persona_id not in active_personas:
        return f"❌ Error: Persona '{persona_id}' not found."
    # 2. Retrieve Context (Who is the AI?)
    persona_data = active_personas[persona_id]
    system_persona = persona_data['persona']
    grading_rubric = persona_data['criteria']
    # 3. Retrieve Evidence (The Hybrid Search)
    candidates = find_candidates_hybrid(user_query, max_salary, min_exp)
    if not candidates:
        return "⚠️ No candidates found matching these strict criteria."
    # 4. Augment the Prompt (Context Injection) — build a readable text block per candidate
    context_block = ""
    for c in candidates:
        c_id, name, exp, sal, summary_lob, score = c
        summary = summary_lob.read() if hasattr(summary_lob, 'read') else str(summary_lob)
        context_block += f"""
        --- CANDIDATE PROFILE ---
        ID: {c_id}
        Name: {name}
        Cost: ${sal:,} (Budget: ${max_salary:,})
        Experience: {exp} years
        Resume Summary: {summary}
        -------------------------
        """
    # 5. Construct the Final Prompt + 6. Generate
    user_message = f"""
    USER REQUEST: "{user_query}"
    CANDIDATES FOUND (Database Output):
    {context_block}
    INSTRUCTIONS:
    Based on your persona rules below, evaluate these candidates.
    1. Select the BEST fit.
    2. Explain WHY, referencing their specific skills.
    3. If they are over budget or underqualified, mention it as a risk.
    YOUR GRADING RUBRIC:
    {grading_rubric}
    """
    response = client.chat.completions.create(
        model="gpt-5.2",
        messages=[
            {"role": "system", "content": system_persona},
            {"role": "user", "content": user_message}
        ],
        temperature=0.3
    )
    return response.choices[0].message.content
```

### Simulación en vivo

Request: *"We need a Python Backend developer who can lead a team."*, budget $160,000, min 4 años, persona `rule_tech_screener` (foco técnico, no soft skills).

```python
search_query = "We need a Python Backend developer who can lead a team."
max_budget = 160000
min_experience = 4
agent_role = "rule_tech_screener"
recommendation = generate_hiring_recommendation(
    user_query=search_query, max_salary=max_budget, min_exp=min_experience, persona_id=agent_role)
print(recommendation)
```

La IA identifica la persona, filtra por salary+experience, rankea por vector similarity y genera una justificación detallada — recomienda **CAND_005 (Quinn R.)**: strong Python background (core requirement), Engineering Manager con mentorship/team growth (cubre "lead a team"), $155k dentro del budget de $160k. Recomienda proceder pero **screenear el delivery hands-on reciente de Python** (risk awareness). El sistema recomendó un candidato que satisface **tanto los requisitos semánticos del job description como los constraints financieros duros** del departamento.

## Citas

> "While the physical location of the data changes from a file system to relational tables, the architectural blueprint remains symmetrical to the one we saw in Chapter 2."
> "the separation between structured business data and unstructured semantic data is artificial."
> "structured relational data lives in the exact same row as the unstructured vector embedding."
> "an AI that understands both human language and database constraints."
> "the converged database allows an LLM to act not just as a creative writer but as a precise decision-maker"

## Para aplicar

- **Vectorizar tablas relacionales directamente, no solo archivos** — activar la data dormida en las tablas Oracle existentes; el blueprint de 3 fases (DBA→Engineer→Developer) se mantiene simétrico al de archivos.
- **Diseñar un hybrid schema (converged database)** — columnas `NUMBER`/`VARCHAR2` para constraints de negocio junto a `VECTOR(1536)` en la **misma fila**; SQL filtering preciso + semantic search en una query.
- **Aislar las reglas de dominio en su propia tabla** (`RECRUITMENT_RULES` separada de `CONTEXT_LIBRARY`) — domain isolation; gestionar las personas/criterios independientemente.
- **Usar flags de control incluyendo `drop_tables` con `PURGE`** — `TRUNCATE` para reset no destructivo, `DROP ... PURGE` para clean slate destructivo; ignorar `ORA-00942` en runs frescos.
- **Ingerir en dual-stream: hechos duros a SQL, narrativa a vectores** — parsear el JSON y rutear `salary`/`experience` a columnas escalares y el `summary` a embeddings.
- **Cargar con `MERGE` para idempotencia** — `MERGE ... WHEN NOT MATCHED THEN INSERT` permite re-correr el pipeline sin duplicar registros.
- **Desacoplar el vector enrichment de la carga transaccional** — query `WHERE resume_vector IS NULL`, embeber y `UPDATE`; separa la carga rápida de business data del proceso costoso de vectorización.
- **Hacer Hybrid Search: `WHERE` (constraints duros) + `VECTOR_DISTANCE(..., DOT)` (ranking)** — el SQL decide elegibilidad, el vector decide relevancia; dot product, más alto = mejor.
- **Recuperar la persona del agente dinámicamente desde la base** — la identidad/rúbrica del LLM como data gobernable (Technical Screener / Culture Fit / University Recruiter), cambiable on demand.
- **Inyectar la evidencia recuperada como context block + rúbrica + temperature baja (0.3)** — context injection para grounding; el agente cita skills específicos, marca over-budget/underqualified como riesgo y es a la vez creativo y compliant.
- **Avisar al driver del vector type** (`cursor.setinputsizes(v=oracledb.DB_TYPE_VECTOR)`) y leer CLOBs con `.read()` — en cada query/insert con vectores y al formatear summaries.

## Conexiones

- [[_RAG-Driven Generative AI - Second Edition|Mapa del libro]] — el MOC del libro.
- [[02 - RAG Embeddings in Oracle Vector Stores]] — cap. 2: este capítulo es **simétrico** (mismas 3 fases DBA→Engineer→Developer), reusa `oracledb`/wallet/`VECTOR_DISTANCE`/`setinputsizes`/`get_embeddings_batch`; evoluciona del file server a tablas relacionales y de semantic search puro a hybrid (SQL+vector).
- [[01 - Why Retrieval-Augmented Generation]] — cap. 1: el **modular RAG** que conmuta naïve (exact/SQL) ↔ advanced (vector) se materializa aquí como hybrid search en una sola query; Oracle AI Vector Search soporta hybrid (mencionado en el cap. 1).
- [[04 - Building Sovereign Enterprise Agents]] — cap. 4: refactorizar esta lógica (`find_candidates_hybrid` → `agent_oracle_recruiter`) en **módulos `.py` stateless** con interfaz MCP; el schema/data de aquí se consumen pasivamente.
- [[_RAG|RAG]] · [[Hybrid Search]] · [[Enterprise RAG Assistant]] · [[Reranking]] · [[Chunking Strategies]] — el **Hybrid Search** del vault es exactamente este patrón (SQL filtering + vector ranking sobre la converged DB).
- [[ACL Filtering en RAG]] · [[Change Data Capture]] — el `WHERE` como filtrado pre-vector (paralelo a ACL filtering); freshness de la data relacional.
- [[Grounding]] · [[Hallucinations]] — context injection + temperature baja para grounding; el agente marca riesgos (over-budget) en vez de inventar.
- [[Function Calling]] · [[Tool Calling]] · [[Orchestrator]] — `find_candidates_hybrid` como tool del Recruiter agent; las personas como configuración del comportamiento.
- **Converged Database · Data convergence · Hybrid RAG / Hybrid Search · `CANDIDATE_POOL` / `RECRUITMENT_RULES` · `VECTOR_DISTANCE(..., DOT)` (dot product) · SQL `MERGE` (idempotencia) · Dual-stream ingestion · Vector enrichment · `DROP TABLE ... PURGE` / `ORA-00942` · Hiring personas · Context injection · NLU · Filtered vector search** — conceptos/técnicas del capítulo; candidatos a nota propia (varios ya existen en el vault, otros son semillas).
