---
title: 02 - RAG Embeddings in Oracle Vector Stores
libro: RAG-Driven Generative AI - Second Edition
autor: Denis Rothman
capitulo: 2
created: 2026-06-11
tags:
  - libros/rag-driven-generative-ai
  - type/case-study
  - status/permanent
aliases:
  - RAG Embeddings in Oracle Vector Stores
updated: 2026-06-11
---

# 02 - RAG Embeddings in Oracle Vector Stores

> [!info] Capítulo 2 · *RAG-Driven Generative AI - Second Edition* — Denis Rothman
> Del concepto a la ingeniería concreta: construir un vector store enterprise real sobre Oracle Database 23ai en tres notebooks (DBA → Data Engineer → Developer). La **arquitectura de simbiosis** acopla un MAS Python agnóstico (lógica soberana) a la base Oracle (memoria soberana), con vectorización dentro del trust boundary, dual RAG (hechos + instrucciones) y un pipeline Dual-Path que cita evidencia sin alucinar. Navegá: [[_RAG-Driven Generative AI - Second Edition|Mapa del libro]] · anterior [[01 - Why Retrieval-Augmented Generation]] · siguiente [[03 - Building a Live Recruiter Agent]].

## Resumen

El cap. 1 introdujo *traer la inteligencia a los datos*; este capítulo **pasa de la simulación conceptual a la ingeniería concreta**. Es el encuentro de dos soberanías: la lógica fluida y portable de un **multi-agent system (MAS)** y el ecosistema inmutable de una base enterprise. Ya no se discute arquitectura: se construye la **infraestructura real** donde la lógica del código Python anida de forma segura dentro de Oracle Database. Se reemplaza la "magia" de la IA black-box por un **pipeline transparente e ingenierizado** que el arquitecto humano controla completamente.

El recorrido: (1) definir la **arquitectura de simbiosis** —el blueprint que une los agentes Python agnósticos con la escala industrial de Oracle Database 23ai—, manteniendo flexibilidad mientras se apalancan las capacidades vectoriales nativas de la base; (2) provisionar una cuenta **Oracle Cloud Free Tier** con una instancia Autonomous Database 23ai; (3) entrar a la implementación en **tres notebooks**, cada uno una persona/rol distinto: en `1_DBA_Oracle_Management` se adopta la mentalidad de **DBA** (esquema, privilegios, conexión segura); en `2_Oracle_Data_Acquisition` la de **data engineer** (pipeline de ingestión: chunking, embeddings, poblar el vector store); en `3_LLM_Augmented_Query` la de **developer/architect** (cerrar el loop combinando la vector search nativa de Oracle con el razonamiento del LLM para entregar respuestas transparentes y sin alucinaciones que respetan la data sovereignty).

Al final se ha ingenierizado un **vector store enterprise vivo desde cero**, transformando una base pasiva en un motor dinámico de inteligencia. El sistema demuestra **dual RAG** (recuperar tanto *hechos* como *instrucciones*), un pipeline **Dual-Path** que consulta en paralelo `CONTEXT_LIBRARY` (el *cómo*: blueprints de comportamiento) y `KNOWLEDGE_STORE` (el *qué*: evidencia factual), y un caso de uso de **incidente de ciberseguridad** donde la IA correlaciona logs de servidor, tickets de soporte, transcripciones de Slack y políticas internas — citando la evidencia exacta (CVE, timestamps, IPs) y **negándose a alucinar** un data breach que la evidencia no confirma.

![[02-fig-2.1.png]]
*Figure 2.1: Docking our MAS to the enterprise database*

## La arquitectura de simbiosis

No se trata de "conectar un script Python a una base": se ingenieriza una **simbiosis entre dos entidades soberanas**.

- **Sovereign logic (el MAS)** — el multi-agent system que se construye en Python. Liviano, flexible y **fundamentalmente database-agnostic**. Carga las capacidades de razonamiento, las definiciones de agentes y la lógica de orquestación. Es **portable**: podría acoplarse a cualquier motor SQL robusto. Representa la soberanía del **arquitecto**.
- **Sovereign memory (la base enterprise)** — la instancia **Oracle Database 23ai**. Representa la soberanía de la **empresa**. Contiene el ecosistema masivo de registros corporativos, políticas de seguridad y embeddings vectoriales selectivos. Es **inmutable y segura**.

El objetivo: enchufar el MAS ágil sobre la infraestructura enterprise estable. **El MAS provee el razonamiento; la base provee los hechos.** La ingestión y la query **no extraen datos en bulk** hacia un vector store externo (Pinecone u otro): empujan el proceso de vectorización **dentro del trust boundary seguro** de la base. La arquitectura integrada (Figura 2.1):

- **Lado izquierdo**: el entorno Python (MAS) que envía queries e instrucciones de lógica.
- **Lado derecho**: el Oracle Database 23ai (trust boundary) con las tablas relacionales y el vector store en un único motor convergido.
- **La conexión**: un bridge API seguro y de baja latencia donde **solo viajan queries y respuestas, nunca datasets en bulk**.

### La ventaja agnóstica

Aunque se usa Oracle Database 23ai por sus capacidades vectoriales enterprise-grade, **la arquitectura del MAS permanece independiente**: no se hard-codean los agentes para quedar atrapados en el ecosistema de un vendor. Se diseña un MAS que habla el **lenguaje universal de SQL y vectores**. Hoy se acopla a Oracle por su escala y seguridad masivas; la misma lógica podría interfacear con otros entornos. Esta separación garantiza que **la empresa permanece soberana sobre sus datos, pero el arquitecto humano permanece soberano sobre la inteligencia del sistema** — la agilidad del desarrollo Python moderno + la seguridad del data governance enterprise establecido.

### El ecosistema convergido (mapeo D·G·E·T → Oracle)

El cap. 1 presentó el ecosistema RAG como ciclo abstracto Retriever (D) · Generator (G) · Evaluator (E) · Trainer (T). Aquí esos componentes teóricos se **mapean directo a la arquitectura Oracle** (Figura 2.2). El **Retriever (D)** ya no es un set fragmentado de tools sino un proceso unificado corriendo dentro de la base convergida:

| Subcomponente | Qué hace en la arquitectura Oracle |
|---|---|
| **D1 Collect** | En el notebook de ingestión actuamos como collection agent: juntamos artefactos corporativos crudos (logs, documentos de política) del staging area. |
| **D2 Process** | Transformamos los archivos crudos en vectores. El cálculo ocurre vía el embedding model, pero el resultado se empuja inmediatamente al ecosistema de la base. |
| **D3 Storage** | **La innovación core del converged database**: no se manda data a un vector store externo. Se guardan los embeddings en las tablas `KNOWLEDGE_STORE` y `CONTEXT_LIBRARY` — los vectores quedan **justo al lado de la data relacional**. |
| **D4 Retrieve** | La recuperación es una **query SQL usando la función `VECTOR_DISTANCE`**, encontrando registros semánticamente relevantes sin mover data por la red. |

El **Generator (G)** representa el entorno Python donde vive el MAS: **G1 Input** (el usuario provee una query, ej. sobre un incidente de seguridad), **G2 Augmentation** (el script Python intercepta la query, llama a la base para recuperar los hechos de D4 y los combina en una única context window), **G3 Prompt Engineering** (envuelve los hechos recuperados en un instruction set específico), **G4 Output** (el LLM genera la respuesta final basada **solo** en la data soberana de Oracle). El **Evaluator (E)**: **E1 Metrics** (similarity scores de `VECTOR_DISTANCE` para medir relevancia) y **E2 Human Feedback** (el arquitecto revisa el output final).

> [!note] Notebooks self-contained hasta el cap. 3
> Los capítulos 1–3 implementan funcionalidad educativa detallada y los notebooks son **self-contained**. A partir del cap. 3 se empiezan a **externalizar los scripts** y crear librerías de funciones, para tener una aplicación que llame a scripts y funciones.

## Getting started con Oracle 23ai

Oracle no es un ecosistema startup frágil ni un sandbox temporal: es **el cuarto de máquinas de la economía global** (gestiona la data transaccional crítica de los bancos, redes logísticas y sistemas de salud más grandes del mundo). Traer el MAS a esta plataforma es desplegar inteligencia donde más importa: **no se le pide a la empresa mover sus montañas de data a la IA; se trae la IA a la montaña**. Esta es la definición de **enterprise-grade RAG**.

Montar un entorno cloud enterprise exige esfuerzo consciente y atención a la seguridad — **aquí el arquitecto humano es irremplazable** (los agentes automatizados que se construirán luego no pueden navegar estas pantallas de setup ni tomar las decisiones ejecutivas sobre región y protocolos de seguridad). Es un *rite of passage* al cloud engineering: entender la infraestructura antes de automatizarla.

**Crear una Autonomous Database 23ai (Oracle Cloud Free Tier):**
1. Registrarse en el Free Tier en `oracle.com/cloud/sign-in.html`.
2. En la Oracle Cloud Console, ubicar **Autonomous Database** → **Create Autonomous Database**.
3. Configuraciones:
   - **Workload type**: **Autonomous Transaction Processing (ATP)** o **Autonomous JSON** (ambos sirven; ATP es el estándar para workloads mixtos).
   - **Version**: **23ai** — crítico, contiene las capacidades nativas de **AI Vector Search**.
   - **Deployment**: **Always Free** si está disponible en tu región.
   - **Credentials**: una password ADMIN fuerte (la **master key de tu sovereign memory** — anotala).
4. Esperar unos minutos a que provisione; ícono verde = base viva.

**Seguridad vía Wallet** (no es un afterthought, es un prerequisito): Oracle usa un **Wallet** (colección zipeada de certificados SSL y archivos de config) para autenticar la conexión.
1. **Access connection panel**: en Database Details → **DB Connection**.
2. **Download**: **Download Wallet** — se pide crear una password específica del wallet (**distinta** de la admin; protege el archivo de credenciales). Chequear la fecha de expiración del wallet regularmente.
3. **Storage**: descargar el ZIP y tratarlo como **secreto** (en producción jamás se commitea a un repo público). En el libro se usa Google Drive con fines educativos.

## Phase 1: DBA Oracle management (`1_DBA_Oracle_Management.ipynb`)

Se adopta la persona del **Database Administrator (DBA)**. En el mundo real, quien gestiona el esquema y los privilegios suele ser distinto de quien construye la app de IA — **separar concerns** asegura infraestructura segura y robusta. Este notebook establece la conexión, define el esquema y limpia el entorno antes de ingerir data.

![[02-fig-2.3.png|788]]
*Figure 2.3: The three pillars of MAS-Oracle integration*

Los **tres roles/pilares** (Figura 2.3), uno por notebook:
1. **The Gatekeeper (DBA)** — schema definition, privilegios y connection security (`1_DBA_Oracle_Management.ipynb`). *Rol actual.*
2. **The Builder (Data Engineer)** — pipeline de ingestión: chunking de docs, generar embeddings, poblar el vector store (`2_Oracle_Data_Acquisition.ipynb`).
3. **The Architect (Developer)** — lógica de aplicación: prompt engineering y RAG queries (`3_LLM_Augmented_Query.ipynb`).

### Global configuration flags (el panel de control)

La primera celda actúa como **control panel**: flags booleanos que determinan qué tareas administrativas se ejecutan, permitiendo correr el notebook en modo mantenimiento **sin borrar data accidentalmente**:
- `unzip_wallet`: `True` solo en el setup inicial para extraer la config del Oracle wallet; luego `False` para evitar operaciones redundantes.
- `create_tables`: `True` para ejecutar los DDL que inicializan `CONTEXT_LIBRARY` y `KNOWLEDGE_STORE`.
- `empty_tables`: `True` para hacer `TRUNCATE` (clean slate para testing).

```python
unzip_wallet=False  # True to unzip the wallet. False to only unzip the wallet once
create_tables=False # True to create tables
empty_tables=False  # True to empty tables
```

### Setup del entorno y extracción del wallet

Se monta Google Drive para acceder a los archivos de config de forma segura. El Oracle Wallet contiene los certificados SSL/TLS y los archivos de config `tnsnames.ora` y `sqlnet.ora` (necesarios para una conexión segura a la Autonomous Database):

```python
from google.colab import drive
drive.mount('/content/drive')
if unzip_wallet==True:
    !unzip -o "/content/drive/MyDrive/files/oracle/Wallet_*.zip" -d "/content/drive/MyDrive/files/oracle"
wallet_path = "/content/drive/MyDrive/oracle_wallet"
```

### Instalar el driver de Oracle

Se instala el driver Python **`oracledb`** (sucesor modernizado del legacy `cx_Oracle`), pineado a **`3.4.1`** para estabilidad/reproducibilidad. Por defecto opera en **Thin Client Mode**: se comunica directo con la base usando Python puro, **sin requerir los binarios pesados de Oracle Instant Client**.

```python
!pip install oracledb==3.4.1
```

### Conexión segura a la base

Separa estrictamente **código de credenciales** (best practice enterprise): en vez de hard-codear passwords, lee un archivo `credentials.txt` guardado de forma segura en Google Drive. El helper `read_key_value_file` parsea username, password, wallet password y el **DSN (Data Source Name)**. `oracledb.connect()` inicializa la sesión; una query simple (`SELECT 'Connected!' FROM dual`) verifica que la conexión está activa:

```python
import os
# Path to the secure credentials file
creds_path = "/content/drive/MyDrive/files/oracle/credentials.txt"
…
# Initialize connection using secure parameters
import oracledb
connection = oracledb.connect(
    user=user, password=password, dsn=dsn,
    config_dir=wallet_path, wallet_location=wallet_path,
    wallet_password=wallet_password
)
cursor = connection.cursor()
cursor.execute("SELECT 'Connected!' FROM dual")
print(cursor.fetchone())   # -> ('Connected!',)
```

### Crear las tablas (DDL)

Se ejecutan los comandos **DDL (Data Definition Language)** tras chequear el flag `create_tables`. Dos tablas especializadas para RAG:
- **`context_library`** — blueprints estructurados y metadata. `blueprint_json CLOB` (**Character Large Object**, para JSON grandes que definen comportamientos de agentes) + `embedding VECTOR(1536)`.
- **`knowledge_store`** — segmentos de texto no estructurado para retrieval; liga el texto crudo (`text CLOB`) con su representación vectorial.

> [!note] `VECTOR(1536)` — el tipo nativo de Oracle 23ai
> La columna `embedding VECTOR(1536)` es específica de Oracle 23ai, optimizada para vectores de alta dimensión. La dimensión **1536** se elige para matchear el output del modelo de embeddings de OpenAI **`text-embedding-3-small`**.

```python
# Create the vector tables in Oracle 23ai
if create_tables==True:
    cursor.execute("""
    CREATE TABLE context_library (
        id VARCHAR2(200) PRIMARY KEY,
        description CLOB,
        blueprint_json CLOB,
        embedding VECTOR(1536)
    )
    """)
    cursor.execute("""
    CREATE TABLE knowledge_store (
        id VARCHAR2(200) PRIMARY KEY,
        source VARCHAR2(200),
        text CLOB,
        embedding VECTOR(1536)
    )
    """)
    connection.commit()
    print("Vector tables created.")
```

**Auditoría estructural**: tras los DDL se consultan las vistas del Oracle Data Dictionary (`user_tables`, `user_tab_columns`) para confirmar que el esquema matchea el diseño, verificando específicamente que las columnas `EMBEDDING` sean de tipo `VECTOR`. Output confirmatorio: tablas `CONTEXT_LIBRARY` y `KNOWLEDGE_STORE` presentes, ambas con `('EMBEDDING', 'VECTOR', 8200)`.

### Data maintenance

Utilidad para resetear data en ciclos de desarrollo/testing, controlada por `empty_tables`. **Crucialmente se usa `TRUNCATE` y no `DELETE`**: `TRUNCATE` es una operación DDL que resetea instantáneamente el contenido de la tabla, **significativamente más rápida y eficiente en recursos** que borrar filas individualmente — sobre todo con tipos `VECTOR` complejos.

```python
def empty_vector_tables(cursor):
    """Empties the vector tables using TRUNCATE. Faster than DELETE and resets storage."""
    try:
        print("Emptying context_library...")
        cursor.execute("TRUNCATE TABLE context_library")
        print("Emptying knowledge_store...")
        cursor.execute("TRUNCATE TABLE knowledge_store")
        # ...
```

Un quality gate final hace `SELECT COUNT(*)` en ambas tablas, con tags visuales (`[EMPTY]` ✅ / `[NOT EMPTY]` ❌) para que el DBA evalúe el estado de un vistazo.

## Phase 2: Data ingestion (`2_Oracle_Data_Acquisition.ipynb`)

Se adopta la persona del **data engineer** (el builder que construye el pipeline que transforma información cruda en vectores machine-readable). Mientras el DBA se enfocó en seguridad y estructura, el data engineer se enfoca en **contenido y transformación**: tomar archivos de texto crudo, romperlos en chunks manejables, convertirlos en vectores de alta dimensión con los modelos de embedding de OpenAI, e insertarlos en Oracle. Es el **pipeline ETL (Extract, Transform, Load)** del sistema RAG, diseñado robusto y repetible.

**Setup**: además de `oracledb==3.4.1` se instala `openai==2.14.0` (puede conflictuar con dependencias de Colab, de ahí que la celda de install a veces aparezca al inicio o en el cuerpo). Se re-monta Google Drive para el wallet (acceso consistente al wallet es **mandatorio en cada notebook**) y se carga la API key de OpenAI vía `userdata` (Colab Secrets) como env var `OPENAI_API_KEY`. Config de OpenAI:

```python
# Configuration
EMBEDDING_MODEL = "text-embedding-3-small"
EMBEDDING_DIM = 1536 # Dimension for text-embedding-3-small
GENERATION_MODEL = "gpt-5.2"
```

La conexión a la base **reutiliza la lógica segura del notebook 1** (todos los roles —DBA, Engineer, Developer— adhieren a los mismos protocolos de seguridad).

### Dual RAG y los tres estados de la data (Tabla 2.1)

**Innovación clave: dual RAG.** A diferencia de un RAG clásico que solo recupera data factual, este sistema recupera **tanto hechos como instrucciones** para construir un *context library*. Al guardar **blueprints procedurales junto al conocimiento corporativo**, la IA tiene tanto la *evidencia para analizar* como la *lógica específica para formular la respuesta* — y se evita crear instrucciones complejas cada vez. La preparación de datos se divide en tres secciones (Tabla 2.1):

| Sección | Goal | Contenido |
|---|---|---|
| **Initialize staging directory** | Crear un entorno local para almacenamiento temporal de archivos | Creación de la carpeta `incident_documents` para guardar archivos de evidencia. |
| **Knowledge store** | Generar la data factual cruda del sistema | Siete archivos de texto con server logs, transcripciones de chat y reportes. |
| **The context library** | Definir el comportamiento operacional y las instrucciones para la IA | Tres semantic blueprints que actúan como personas para la generación de respuestas. |

### Staging directory e ingestión remota segura

Se crea la carpeta `incident_documents` (staging ground) con `os.makedirs` si no existe — simula un escenario real donde logs, reportes y transcripciones se agregan desde varios departamentos a una ubicación central durante una brecha activa.

```python
import os
if not os.path.exists("incident_documents"):
    os.makedirs("incident_documents")
    print("Directory 'incident_documents' created.")
else:
    print("Directory 'incident_documents' already exists.")
```

La evidencia se descarga (zipeada en `evidence.zip`) desde un repo GitHub en `/enterprise_data/incident_evidence` (simula un corporate data lake). Un flag booleano `retrieve_evidence` permite togglear la ingestión on/off (evita re-descargar artefactos inmutables en reruns, ahorrando bandwidth). Se usa un `time.sleep(10)` para garantizar que el write se commitee a disco antes de extraer.

```python
retrieve_evidence = False #@param {type:"boolean"}
if retrieve_evidence==True:
    !curl -L -H "Authorization: token {pt}" -H "Accept: application/vnd.github.v3.raw" "https://api.github.com/repos/Denis2054/RAG-Driven-Generative-AI-2nd-Edition/contents/enterprise_data/incident_evidence/evidence.zip" -o evidence.zip
```

**Los 7 archivos de evidencia** (un incidente de ciberseguridad de alta severidad: campaña de phishing → SQL injection → ransom):
- `Email_Phishing_Report.txt` — campaña de phishing de alta severidad contra Finance y Admin; usuario comprometido + link de credential harvesting.
- `Patch_Release_Notes.txt` — "Hotfix v2.4.1", update de emergencia para una vulnerabilidad SQL injection crítica en `AdminLoginController.java`.
- `Slack_Channel_Ops.txt` — comunicación real-time entre ops y security al detectar spikes de conexiones y confirmar el ataque SQLi activo.
- `Customer_Support_Tickets.txt` — tickets con reportes de "500 Internal Server Errors" e imposibilidad de acceso.
- `Server_SysLog_001.txt` — logs técnicos del momento exacto del ataque: connection pool exhaustion y un payload SQLi específico "`UNION SELECT`".
- `Hacker_Manifesto.txt` — nota de ransom de "The NullPointer Crew" reclamando robo de data/hashes y exigiendo **50 BTC**.
- `Internal_Policy_DataPrivacy.txt` — políticas de compliance legal, incluyendo la **ventana de reporte de 72 horas** para brechas externas y la clasificación de tiers de data sensible.

**Dos escenarios operacionales con validation gate**: *Caso 1 (VM activa / rerun)* — los archivos persisten; con `retrieve_evidence=False` se skipean curl/unzip preservando bandwidth. *Caso 2 (safety path / cold start con flag en False)* — storage vacío; en vez de crashear, el código chequea `os.path.exists` y solo imprime un warning, manteniendo la app estable.

### El context library — semantic blueprints (las instrucciones)

Innovación crítica: guardar **instrucciones de comportamiento junto a la data factual**. RAG típico guarda solo el "qué" (hechos); aquí también el "cómo" (comportamiento). Los **semantic blueprints** son objetos JSON estructurados que definen personas y formatos de output. Al vectorizarlos, la IA puede recuperar el **rol correcto** (PR Manager, Lead Engineer…) según la *intención* de la query, separando la lógica de aplicación de la data cruda.

**Blueprint 1 — PR Manager**: genera updates tranquilizadores y no-técnicos para clientes; evita jerga como "SQL Injection", usa términos suaves como "Unplanned Outage", sin admitir liability prematuramente.

```python
context_blueprints = [
    {
        "id": "blueprint_public_pr",
        "description": "A blueprint for drafting public-facing statements or press releases regarding service outages or security incidents. The goal is to reassure customers without admitting specific technical liabilities until confirmed.",
        "blueprint": json.dumps({
              "scene_goal": "Manage public perception and reassure customers.",
              "style_guide": "Use a professional, empathetic, but vague tone. Avoid technical jargon like 'SQL Injection' or 'Hacks'. Focus on 'Security Maintenance' or 'Unplanned Outage'.",
              "structure": ["Acknowledgment of Issue", "Current Status (Investigating)", "Estimated Resolution Time", "Apology"],
              "instruction": "Draft a public statement based on the provided facts. Do not reveal sensitive internal details (IPs, specific vulnerabilities)."
            })
    },
```

**Blueprint 2 — Technical Lead Engineer (RCA)**: precisión técnica absoluta; fuerza a citar timestamps, error codes (ORA-XXXX) e IPs para un **Root Cause Analysis (RCA)** accionable.

```python
    {
        "id": "blueprint_technical_rca",
        "description": "A blueprint for creating a technical Root Cause Analysis (RCA) for engineering and security teams. Focuses on specific timestamps, error codes, vulnerabilities (CVEs), and remediation steps.",
        "blueprint": json.dumps({
              "scene_goal": "Provide a precise technical post-mortem of the incident.",
              "style_guide": "Clinical, precise, and fact-based. Use active voice. Explicitly reference log timestamps, error codes (ORA-XXXX), and IP addresses.",
              "structure": ["Incident Timeline", "Root Cause (The Vulnerability)", "Attack Vector", "Remediation (The Fix)"],
              "instruction": "Synthesize the provided logs and chat transcripts into a formal RCA document."
            })
    },
```

**Blueprint 3 — Legal Compliance Officer**: compara los hechos del incidente contra las políticas internas; outputea un risk assessment que determina si la brecha dispara timelines regulatorios (ej. la ventana GDPR de 72h).

```python
    {
        "id": "blueprint_legal_compliance",
        "description": "A blueprint for assessing legal liability and data breach notification obligations. It compares incident facts against internal privacy policies (GDPR/CCPA) to determine if a regulatory filing is required.",
        "blueprint": json.dumps({
              "scene_goal": "Determine legal risk and reporting requirements.",
              "style_guide": "Formal, risk-averse, and citation-heavy. Reference specific clauses from the provided Policy Documents.",
              "structure": ["Incident Summary", "Data Impact Assessment (Tier 1-4)", "Policy Violation Check", "Required Actions (Notification Yes/No)"],
              "instruction": "Analyze the incident against the 'Internal Data Privacy & Compliance Policy'. Determine if the compromised data requires external reporting."
            })
    }
]
print(f"\nPrepared {len(context_blueprints)} context blueprints.")   # -> Prepared 3 context blueprints.
```

### Carga de documentos y utilidades de transformación

Se itera el directorio `incident_documents` leyendo cada `.txt` a un dict `knowledge_base` (mimetiza un ETL de flat files). Output: `📚 Loaded 7 documents into the knowledge base.`

**Chunking robusto** (`chunk_text`): no se puede meter un manual entero en un vector; se rompe en segmentos semánticamente significativos. Se usa la librería **`tiktoken`** (no string-splitting) para contar **tokens, no caracteres**, evitando exceder el context window. Default **400 tokens con overlap de 50** — el overlap es crucial para que **no se pierda contexto en el corte** entre chunks.

```python
import tiktoken
tokenizer = tiktoken.get_encoding("cl100k_base")

def chunk_text(text, chunk_size=400, overlap=50):
    """Chunks text based on token count with overlap.
    Best practice for RAG: Overlap ensures context isn't lost at the cut."""
    tokens = tokenizer.encode(text)
    chunks = []
    for i in range(0, len(tokens), chunk_size - overlap):
        chunk_tokens = tokens[i:i + chunk_size]
        chunk_text = tokenizer.decode(chunk_tokens)
        chunk_text = chunk_text.replace("\n", " ").strip()
        if chunk_text:
            chunks.append(chunk_text)
    return chunks
```

**Batch embedding** (`get_embeddings_batch`): toma una lista de chunks y devuelve vectores de 1536 dims. Envuelto con la librería **`tenacity`** para **exponential backoff** (`@retry` con `wait_random_exponential(min=1, max=60)`, `stop_after_attempt(6)`): ante timeouts o rate limits, el sistema pausa progresivamente y reintenta en vez de crashear el pipeline.

```python
@retry(wait=wait_random_exponential(min=1, max=60), stop=stop_after_attempt(6))
def get_embeddings_batch(texts, model=EMBEDDING_MODEL):
    """Generates embeddings for a batch of texts using OpenAI, with exponential backoff retries."""
    # OpenAI recommends replacing newlines with spaces for best embedding results
    texts = [t.replace("\n", " ") for t in texts]
    response = client.embeddings.create(input=texts, model=model)
    return [item.embedding for item in response.data]
```

### Procesar y subir la data a Oracle

Dos operaciones distintas (manteniendo el principio dual RAG): primero el **context library** (instrucciones), luego el **knowledge store** (evidencia).

**Context library — 3 pasos**: (1) **DB preparation** — `cursor.setinputsizes(embedding=oracledb.DB_TYPE_VECTOR)` le dice al driver que la columna embedding es un **vector type especializado, no un array PL/SQL estándar** (sin esto, error `ORA-01484`). (2) **Vectorizar blueprints** — se genera el embedding **del campo `description`** (estratégico: permite buscar un concepto como "how to write a PR statement" y matchearlo a la descripción, aunque la instrucción real viva separada en JSON). (3) **Insertar** con bind variables.

```python
import oracledb
print("\nProcessing and uploading Context Library to Oracle...")
cursor = connection.cursor()
# CRITICAL: Tell Oracle the "embedding" list is actually a VECTOR type (avoids ORA-01484)
cursor.setinputsizes(embedding=oracledb.DB_TYPE_VECTOR)

vectors_context = []
for item in tqdm(context_blueprints):
    embedding = get_embeddings_batch([item['description']])[0]
    vectors_context.append({
        "id": item['id'], "values": embedding,
        "metadata": {"description": item['description'], "blueprint_json": item['blueprint']}
    })

if vectors_context:
    for item in vectors_context:
        cursor.execute("""
            INSERT INTO context_library (id, description, blueprint_json, embedding)
            VALUES (:id, :description, :blueprint_json, :embedding)
        """, {
            "id": item["id"], "description": item["metadata"]["description"],
            "blueprint_json": item["metadata"]["blueprint_json"], "embedding": item["values"]
        })
    connection.commit()
    print(f"Successfully uploaded {len(vectors_context)} context vectors to Oracle.")
```

**Knowledge store — batches de 100**: a diferencia de los blueprints estructurados, los documentos varían en largo; se pasan por `chunk_text` y se procesan en **`batch_size = 100`** (se acumulan 100 segmentos, se mandan a OpenAI en una sola API call, se insertan — minimiza latencia de red y optimiza los transaction logs). Se genera un **ID único por chunk** (ej. `Server_SysLog_001.txt_chunk_0`) para trazabilidad, y se hace `commit` tras cada batch.

```python
for doc_name, doc_content in knowledge_base.items():
    knowledge_chunks = chunk_text(doc_content)
    for i in tqdm(range(0, len(knowledge_chunks), batch_size), desc=f"  Uploading {doc_name}"):
        batch_texts = knowledge_chunks[i:i+batch_size]
        batch_embeddings = get_embeddings_batch(batch_texts)
        for j, embedding in enumerate(batch_embeddings):
            chunk_id = f"{doc_name}_chunk_{total_vectors_uploaded + j}"
            cursor.execute("""
                INSERT INTO knowledge_store (id, source, text, embedding)
                VALUES (:id, :source, :text, :embedding)
            """, {"id": chunk_id, "source": doc_name, "text": batch_texts[j], "embedding": embedding})
        connection.commit()
    total_vectors_uploaded += len(knowledge_chunks)
```

### Verificación final — semantic search test

Sanity check con una query en lenguaje natural: *"What happened to the admin database?"* — confirma que la base liga una pregunta humana a la evidencia cruda, validando tres componentes: la generación de embeddings, el cálculo `VECTOR_DISTANCE` y la lógica de sorting. Se embebe la query con el mismo `get_embeddings_batch` (mismo espacio de 1536 dims) y se ejecuta:

```python
cursor.setinputsizes(query_vec=oracledb.DB_TYPE_VECTOR)
cursor.execute("""
    SELECT id, source, text,
           VECTOR_DISTANCE(embedding, :query_vec) AS distance
    FROM knowledge_store
    ORDER BY embedding <-> :query_vec
    FETCH FIRST 5 ROWS ONLY
""", {"query_vec": query_embedding})
results = cursor.fetchall()
```

> [!note] El operador `<->` y los CLOB
> `ORDER BY embedding <-> :query_vec` usa el operador **`<->` (Euclidean distance)** para ordenar por proximidad semántica. **Distance más baja = mayor similitud.** Al mostrar resultados hay que leer el campo **CLOB** a string con `.read()` antes de imprimir. Conteos finales esperados: **3 filas en `CONTEXT_LIBRARY`** (un blueprint cada una) y **~7 en `KNOWLEDGE_STORE`** (según cómo el tokenizer chunkó).

## Phase 3: LLM-augmented query (`3_LLM_Augmented_Query.ipynb`)

Se adopta la persona del **architect**: construir el "cerebro" de la app — un pipeline RAG dinámico que conecta el razonamiento del LLM de OpenAI con la sovereign memory en Oracle. El sistema no solo *busca* keywords sino que *piensa* sobre la data: recupera **instrucciones de comportamiento (context)** para entender *cómo* responder, y **evidencia factual (knowledge)** para entender *qué* responder.

![[02-fig-2.5.png|850]]
*Figure 2.5: The Dual-PATH RAG pipeline flowchart*

### Las 8 etapas del pipeline Dual-Path (Figura 2.5)

1. **User query (input)** — input crudo (ej. *"Using the technical RCA blueprint, what happened to the admin database?"*).
2. **Embedding (vectorization)** — la query se pasa a `text-embedding-3-small` → vector `query_embedding` con el significado semántico.
3. **Parallel retrieval (the split path)** — el flujo se divide en dos ramas paralelas, ambas vía `oracle_vector_search()` buscando el closest match al query vector:
   - **Context retrieval** → tabla `CONTEXT_LIBRARY` → hallar instrucciones de comportamiento / personas (ej. PR Manager).
   - **Knowledge retrieval** → tabla `KNOWLEDGE_STORE` → hallar evidencia factual (server logs, tickets).
4. **Data extraction and safety** — como la data vive en columnas distintas según la tabla (index 1 para Context, index 2 para Knowledge), `extract_text_from_row()` estandariza la extracción; internamente llama a `read_clob_or_str()` para manejar **Oracle LOBs** o valores null sin crashear.
5. **Text processing (trimming)** — `trim_snippet()` usa `tiktoken` para truncar a **400 tokens**, asegurando que el contenido entre en el context window.
6. **Prompt assembly** — combina cuatro elementos en un bloque estructurado: **System** (persona), **Context** (blueprints recuperados), **Knowledge** (evidencia recuperada), **User query** (request original).
7. **LLM generation** — el prompt ensamblado va a **gpt-5.2**, que sintetiza el "knowledge" según las instrucciones del "context" para responder la "user query".
8. **Final output** — reporte formateado con la query original, la evidencia citada (para verificación) y la respuesta generada.

### La función `oracle_vector_search` (el motor de retrieval)

Dinámica y fault-tolerant. Tres implementaciones técnicas: **dynamic targeting** (acepta un arg `table`, conmuta entre `KNOWLEDGE_STORE` y `CONTEXT_LIBRARY`), **vector type mapping** (`DB_TYPE_VECTOR`), **Euclidean distance sorting** (operador `<->` en el `ORDER BY` → nearest neighbors).

```python
import oracledb
def oracle_vector_search(table, embedding, top_k=5):
    cursor.setinputsizes(query_vec=oracledb.DB_TYPE_VECTOR)
    if table.lower() == "knowledge_store":
        sql = f"""
            SELECT id, source, text,
                   VECTOR_DISTANCE(embedding, :query_vec) AS distance
            FROM {table}
            ORDER BY embedding <-> :query_vec
            FETCH FIRST {top_k} ROWS ONLY
        """
        cursor.execute(sql, {"query_vec": embedding})
        return cursor.fetchall()
    elif table.lower() == "context_library":
        sql = f"""
            SELECT id, description,
                   VECTOR_DISTANCE(embedding, :query_vec) AS distance
            FROM {table}
            ORDER BY embedding <-> :query_vec
            FETCH FIRST {top_k} ROWS ONLY
        """
        cursor.execute(sql, {"query_vec": embedding})
        return cursor.fetchall()
    else:
        raise ValueError(f"Unsupported table: {table}")
```

### Helpers defensivos: LOB handling y extracción robusta

Al recuperar campos de texto grandes, el driver devuelve **LOB (Large Object) pointers** en vez de strings Python — pasarlos directo al LLM crashea la app. `read_clob_or_str` es la "red de seguridad": si es LOB detecta `.read()` y extrae el stream; si es `None` devuelve string vacío (evita `NoneType` errors); si es string lo pasa sin cambios.

```python
def read_clob_or_str(value):
    """Return string content for CLOB-like objects or plain strings; safe for None."""
    if value is None:
        return ""
    if hasattr(value, "read"):
        try:
            return value.read() or ""
        except Exception:
            return ""
    return str(value)
```

`extract_text_from_row` abstrae que el texto relevante está en **index 1 para Context (`description`)** e **index 2 para Knowledge (`text`)**. Acepta un `preferred_index` con fallback heurístico (si la columna esperada está vacía, prueba index 2, luego 1, o junta todas las columnas string) — el LLM siempre recibe *algún* contenido aunque el esquema evolucione.

```python
def extract_text_from_row(row, preferred_index=None):
    """Extract the most likely text field from a DB row.
    - If preferred_index is provided and valid, use it.
    - Otherwise try common positions (2 then 1) and fall back to joining string-like columns."""
    if row is None:
        return ""
    if preferred_index is not None and preferred_index < len(row):
        return read_clob_or_str(row[preferred_index])
    if len(row) >= 3:
        candidate = read_clob_or_str(row[2])
        if candidate:
            return candidate
    if len(row) >= 2:
        return read_clob_or_str(row[1])
    return " ".join(read_clob_or_str(c) for c in row if read_clob_or_str(c))
```

### Orquestar el pipeline Dual-Path — 3 pasos

**Step 1 — retrieval timer + dual search**: se inicializan timers de performance y se ejecuta la búsqueda dual, con `preferred_index=1` para context y `preferred_index=2` para knowledge.

```python
import time
from datetime import timedelta
import textwrap
from IPython.display import display, Markdown

# system_prompt = "You are a Cyber-Incident Response AI. Use the provided Context Library blueprints and Knowledge Base evidence to fulfill the user's request."
overall_start = time.perf_counter()
retrieval_start = time.perf_counter()

# 1. Retrieve Context (Behavior/Blueprints)
context_results = oracle_vector_search("context_library", query_embedding, top_k=3) \
    if callable(globals().get('oracle_vector_search', None)) else []
# 2. Retrieve Knowledge (Facts/Logs)
knowledge_results = oracle_vector_search("knowledge_store", query_embedding, top_k=5) \
    if callable(globals().get('oracle_vector_search', None)) else []

context_snippets = [extract_text_from_row(r, preferred_index=1) for r in context_results]
knowledge_snippets = [extract_text_from_row(r, preferred_index=2) for r in knowledge_results]
```

**Step 2 — prompt assembly + generation**: se construye `final_prompt` etiquetando **explícitamente** Context Library separado de Knowledge Base (ayuda al LLM a distinguir *instrucciones* de *evidencia*), y se llama al modelo con un approach defensivo.

```python
# user_query = "Using the technical RAG blueprint, what happened to the admin database?"
context_block = "\n\n".join(f"- {s}" for s in context_snippets) if context_snippets else "- (no context found)"
knowledge_block = "\n\n".join(f"- {s}" for s in knowledge_snippets) if knowledge_snippets else "- (no knowledge found)"

final_prompt = f"""
Context Library:
{context_block}

Knowledge Base:
{knowledge_block}

User Query:
{user_query}

Answer:
"""

generation_start = time.perf_counter()
try:
    response = client.chat.completions.create(
        model=GENERATION_MODEL,
        messages=[
            {"role": "system", "content": system_prompt},
            {"role": "user", "content": final_prompt}
        ],
        temperature=0.4,
        max_completion_tokens=4000
    )
    content = None
    if isinstance(response, dict):
        content = response.get("choices", [{}])[0].get("message", {}).get("content")
    else:
        try:
            content = response.choices[0].message.content
        except Exception:
            content = str(response)
except Exception as e:
    generation_error = e
    content = None
```

> [!warning] `max_completion_tokens=4000` — token runway para el reasoning
> Los modelos de razonamiento avanzado como GPT-5.2 pueden generar **muchos internal reasoning tokens** antes de la respuesta final. Si el límite es muy bajo (ej. 800), el modelo agota el budget durante el razonamiento y **falla en producir una respuesta visible**. Un límite mayor garantiza suficiente *token runway* para la respuesta final.

**Step 3 — reporting**: en vez de un `print` simple, se genera un **reporte Markdown** que muestra la evidencia recuperada junto a la respuesta final (transparencia: verificar que la IA cita los logs correctos y sigue los blueprints correctos), vía `display(Markdown(...))`.

### El resultado — enterprise RAG exitoso

La IA **cosió data de formatos completamente distintos**: correlacionó los errores 500 reportados por clientes en los tickets con el minuto exacto en que se agotó el connection pool en los server logs:

> *The attack generated malformed/hostile queries (e.g., UNION SELECT ...) that caused SQL syntax errors, high query latency, and ultimately exhausted the application's DB connection pool (500/500), leading to 500 errors and a watchdog-initiated service restart.*

Identificó la vulnerabilidad exacta en el controller Java tirando directo de las Hotfix Release Notes:

> *Known vulnerable code path: Release notes indicate CVE-2025-9921 (SQLi in AdminLoginController.java) was fixed in hotfix v2.4.1, but it was only deployed to staging and not yet in production at the time of the incident.*

**Lo más importante: evitó alucinaciones.** Como los logs terminaban en el momento del firewall block, la IA **declaró explícitamente que no había modificación ni exfiltración de data confirmada**, ateniéndose estrictamente a los hechos soberanos provistos por Oracle en vez de inventar una narrativa de breach masivo:

> *What is not yet confirmed from provided evidence:*
> *- Data exfiltration from admin_users (security lead explicitly: "Too early to tell… check exfiltration size.")*
> *- Successful SQLi extraction (we see attempts and blocking, but no definitive "rows returned" or outbound transfer logs here.*

Esto confirma el puente entre **sovereign logic y sovereign memory**: una IA que vive dentro del límite seguro de la empresa, capaz de actuar como analista senior de ciberseguridad sobre data privada **sin exponerla jamás** a los riesgos de un training set público.

## Citas

> "We are no longer discussing architecture but building the actual infrastructure where the logic of your Python code nests securely into the Oracle Database."
> "The MAS provides the reasoning, while the database provides the facts."
> "the company remains sovereign over its data, but you, the human architect, remain sovereign over your system's intelligence."
> "we are not asking the enterprise to move its massive data mountains to our AI; we are bringing the AI to the mountain."
> "In a typical RAG application, the database stores only the 'what' (the facts). Here, we also store the 'how' (the behavior)."

## Para aplicar

- **Separar sovereign logic (MAS Python agnóstico) de sovereign memory (base enterprise)** — mantener la lógica portable hablando SQL+vectores; acoplarla a la base sin hard-codear el vendor. El MAS razona, la base aporta hechos.
- **Empujar la vectorización dentro del trust boundary** — guardar los embeddings junto a la data relacional (converged database), no extraer bulk a un vector store externo; solo viajan queries y respuestas.
- **Provisionar Oracle 23ai con AI Vector Search nativo** — elegir version 23ai (no otra), Always Free para desarrollo; tratar el Wallet como secreto (password propia, chequear expiración, nunca commitearlo).
- **Separar roles por notebook (DBA · Data Engineer · Developer)** — separation of concerns: el DBA define esquema/privilegios/conexión, el engineer construye el ETL, el developer la lógica de app.
- **Usar flags booleanos de control** (`unzip_wallet`, `create_tables`, `empty_tables`, `retrieve_evidence`) — correr notebooks en modo mantenimiento sin borrar data; togglear ingestión para ahorrar bandwidth en reruns.
- **Definir columnas `VECTOR(1536)` que matcheen el embedding model** (`text-embedding-3-small`) — y avisar al driver con `cursor.setinputsizes(embedding=oracledb.DB_TYPE_VECTOR)` para evitar `ORA-01484`.
- **Usar `TRUNCATE` (DDL) en vez de `DELETE`** para resetear tablas con vectores — mucho más rápido y eficiente en recursos.
- **Chunkear por tokens con `tiktoken` (400 tokens, overlap 50)** — contar tokens no caracteres; el overlap evita perder contexto en el corte.
- **Batch embeddings (batch_size=100) con exponential backoff (`tenacity`)** — minimiza latencia de red; reintentos automáticos ante timeouts/rate limits en vez de crashear.
- **Aplicar dual RAG: guardar instrucciones (context) junto a hechos (knowledge)** — vectorizar el `description` de cada blueprint para recuperar el rol/persona correcto por intención de la query.
- **Recuperar con `VECTOR_DISTANCE` / operador `<->` y `FETCH FIRST k`** — distancia más baja = mayor similitud; nearest neighbors en el espacio vectorial vía SQL.
- **Escribir helpers defensivos para LOBs** (`read_clob_or_str`, `extract_text_from_row`) — leer CLOB con `.read()`, manejar None y esquemas variables sin crashear.
- **Dar suficiente `max_completion_tokens` (4000)** a modelos de razonamiento — token runway para los internal reasoning tokens antes de la respuesta final.
- **Mostrar la evidencia recuperada junto a la respuesta (reporte Markdown)** — transparencia para verificar citas y blueprints; el grounding en data soberana es lo que evita alucinaciones.

## Conexiones

- [[_RAG-Driven Generative AI - Second Edition|Mapa del libro]] — el MOC del libro.
- [[01 - Why Retrieval-Augmented Generation]] — cap. 1: el paradigma *traer la IA a los datos*, el ecosistema D·G·E·T (aquí mapeado a Oracle), naïve/advanced/modular RAG (este capítulo operacionaliza advanced RAG con vector search real); el `VECTOR_DISTANCE` realiza la cosine/Euclidean distance que el cap. 1 simulaba con TF-IDF.
- [[03 - Building a Live Recruiter Agent]] — cap. 3: pasar de archivos estáticos a **vectorizar data relacional dentro de tablas Oracle** (hybrid RAG); empezar a externalizar scripts en librerías para un MAS-RAG standalone.
- [[_RAG|RAG]] · [[Enterprise RAG Assistant]] · [[Hybrid Search]] · [[Chunking Strategies]] · [[Reranking]] — el patrón RAG y técnicas del vault; chunking (400/50 overlap) y la converged DB como hybrid search SQL+vector.
- [[Change Data Capture]] · [[ACL Filtering en RAG]] — data sovereignty / governance boundary y freshness en el RAG del vault; aquí el dato no cruza el trust boundary.
- [[Grounding]] · [[Hallucinations]] — el caso de uso demuestra grounding en hechos soberanos y rechazo explícito a alucinar lo no confirmado.
- [[Function Calling]] · [[Tool Calling]] · [[Orchestrator]] — `oracle_vector_search` como tool dinámica del MAS; el pipeline Dual-Path orquesta context+knowledge.
- **Oracle Database 23ai · Oracle AI Vector Search · `VECTOR(1536)` · `VECTOR_DISTANCE` · operador `<->` (Euclidean) · Converged Database · Autonomous Database · Oracle Wallet · DSN · `oracledb` (Thin Client) · CLOB/LOB · DDL/`TRUNCATE` · Embeddings · `text-embedding-3-small` · tiktoken · Chunking (overlap) · Exponential backoff (tenacity) · Dual RAG · Context Library · Knowledge Store · Semantic blueprints · Dual-Path RAG pipeline · RCA · Data sovereignty · Multi-Agent System (MAS)** — conceptos/técnicas del capítulo; candidatos a nota propia (varios ya existen en el vault, otros son semillas).
