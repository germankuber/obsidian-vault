---
title: 04 - Building Sovereign Enterprise Agents
libro: RAG-Driven Generative AI - Second Edition
autor: Denis Rothman
capitulo: 4
created: 2026-06-11
tags:
  - libros/rag-driven-generative-ai
  - type/case-study
  - status/permanent
aliases:
  - Building Sovereign Enterprise Agents
  - Sovereign Enterprise Agents
updated: 2026-06-11
---

# 04 - Building Sovereign Enterprise Agents

> [!info] Capítulo 4 · *RAG-Driven Generative AI - Second Edition* — Denis Rothman
> De scripts lineales en notebooks a **software de ingeniería modular**: desacoplar la capa de datos de la capa de razonamiento. Se refactoriza la lógica de los caps. 2-3 en módulos `.py` stateless que exponen una interfaz MCP estandarizada: `oracle_lib.py` (`OracleManager` singleton), `agent_archivist.py` (retriever no estructurado), `agent_recruiter.py` (hybrid retriever estructurado), validados en un unit-test notebook. El resultado: Oracle 23ai como componente **plug-and-play** para cualquier framework agéntico. Navegá: [[_RAG-Driven Generative AI - Second Edition|Mapa del libro]] · anterior [[03 - Building a Live Recruiter Agent]] · siguiente [[05 - Building a Universal Context Engine]].

## Resumen

El cap. 3 probó el hybrid RAG dentro de un notebook (lógica lineal). Este capítulo es una **decisión arquitectónica pivotal**: pasar de ejecución manual y lineal a **modularidad automatizada y event-driven**. La meta es **desacoplar la capa de datos de la capa de razonamiento**: si se hard-codea la lógica de base directo en el reasoning engine, se crea un sistema frágil que no se adapta a entornos nuevos. La solución es el **interface pattern** — envolver la lógica de los caps. 2-3 en módulos `.py` **stateless** que exponen una signatura input/output estandarizada (el **MCP, Model Context Protocol**) que cualquier *Universal Context Engine* puede consumir, sin necesitar saber si habla con una vector DB cloud o una converged database on-premise.

El rol del lector cambia de **data scientist** (caps. 2-3) a **enterprise software engineer**: no se reinventa la lógica del Recruiter ni del Archivist, se la **repurposea** en software desplegable. La arquitectura es en capas, con el **Jupyter Notebook como engine/orchestrator** (mínimo, solo flujo de ejecución y visualización) y el heavy lifting abstraído en `.py` puros. Cuatro componentes en cuatro fases: (1) **Infrastructure layer** (`oracle_lib.py`) — la clase `OracleManager` (singleton) que centraliza connection pooling, credenciales y error handling; (2) **Unstructured data agent** (`agent_archivist.py`) — el **Oracle Archivist**, retriever puro de similitud semántica sobre `KNOWLEDGE_STORE` (contraparte del agente "Researcher" que en el cap. 5 consulta Pinecone); (3) **Structured data agent** (`agent_recruiter.py`) — el **Oracle Recruiter**, hybrid retriever que parsea constraints escalares (salary/experience) + vector sobre `CANDIDATE_POOL`; (4) **Verification suite** (`Unified_Agents_Unit_Test.ipynb`) — unit tests que validan que cada agente parsea inputs, ejecuta queries y devuelve el formato MCP estricto. Beneficios: **reproducibilidad** (orden de ejecución definido en el engine), **modularidad** (lógica unit-testeable independiente del notebook), **claridad** (el notebook se enfoca en la *narrativa*, no en la implementación). Las dependencias `.py` se zipean en `unit_library.zip` y se suben al repo para que el notebook las descargue.

![[04-fig-4.1.png]]
*Figure 4.1: Building a central MAS engine*

## Arquitectura del sistema y diseño del framework

Se abandona el enfoque **monolítico** (conexiones, lógica de negocio y flujo de ejecución entrelazados en un script lineal) por una **arquitectura en capas** con separación estricta entre el código persistente y el runtime transitorio. El **Jupyter Notebook no es un scratchpad**: es el **central engine y orchestrator** — maneja el flujo de alto nivel, la inyección de config y la visualización; el heavy lifting (lógica, clases, procesamiento) se abstrae en módulos `.py` puros.

La capa de orquestación:
- **Engine** (`Unified_Agents_Unit_Test.ipynb`) — el command center. Inicializa el entorno, dispara la carga de datos, ejecuta la lógica core y renderiza outputs. **Intencionalmente mínimo**: no contiene lógica de negocio pesada, solo llama funciones de los módulos (`loader.load_data()`, `processor.run()`). Es la interfaz visual del usuario.
- **Peripheral modules** (`helpers.py`) — el toolbox multipropósito. Consolida funciones de interacción con el LLM, generación de embeddings y seguridad, **imponiendo los estándares MCP (Model Context Protocol)** que usan todos los agentes. Importado tanto por el Engine como por los Agent modules → error handling, retries y message formatting consistentes.

> [!note] MCP (Model Context Protocol)
> Los agentes se comunican vía un **diccionario MCP estandarizado** (creado con el helper `create_mcp_message`). Cada agente acepta un `mcp_message` y devuelve un dict MCP — diseñado para **consumo machine-to-machine**. Esto es lo que convierte funciones locales en **agentes soberanos** con una interfaz uniforme, intercambiables dentro de cualquier framework agéntico.

### Lo nuevo y lo que no — la evolución (Tabla 4.1)

Vital distinguir la *lógica* validada en el cap. 3 de la *arquitectura* que se construye ahora. En el cap. 3 se operaba como **data scientist** (scripts lineales para probar que el hybrid RAG funciona); aquí se cambia a **enterprise software engineer** (repurposear esa lógica en software desplegable):

| Componente | Cap. 3 — origen funcional | Cap. 4 — evolución de implementación | Status |
|---|---|---|---|
| **Data y schema** | *Active creation*: se corrieron DDL para crear `CANDIDATE_POOL` y se ingirió `hr_data.json`. | *Passive consumption*: se asume que el schema y la data **ya existen**; los agentes solo consultan el estado persistente establecido antes. | Repurposed data |
| **Ubicación de la lógica core** | *Notebook-bound*: la lógica vivía en celdas de `3_LLM_Augmented_Hybrid_Query.ipynb`, ejecutada linealmente. | *Modularized*: la lógica se extrae a archivos `.py` externos y stateless (`agent_recruiter.py`) importables en cualquier app. | New structure |
| **Conexión a la base** | *Scripted*: `oracledb.connect(...)` llamado manual y repetidamente en celdas. | *Infrastructure layer*: `oracle_lib.py` usa una clase singleton (`OracleManager`) que maneja connection pools y seguridad centralmente. | New pattern |
| **Los agentes** | *Local functions*: `find_candidates_hybrid()` imprimía resultados directo al output del usuario. | *Sovereign agents*: `agent_oracle_recruiter` devuelve un diccionario MCP (JSON) estandarizado para consumo machine-to-machine. | Evolved interface |
| **Flujo de ejecución** | *Linear narrative*: celdas top-to-bottom para contar la historia de ingestión y retrieval. | *Unit test suite*: `Unified_Agents_Unit_Test.ipynb` como staging ground que valida que los módulos pasan constraints estrictos antes del deploy. | New methodology |

La meta ya **no es solo obtener una respuesta**, sino construir una **interfaz stateless robusta** que permita a *cualquier* sistema externo (de los capítulos siguientes) acceder a la lógica.

## Phase 1: Infrastructure layer (`oracle_lib.py`)

En el cap. 3 se usaban connection strings ad-hoc dispersas en las celdas (aceptable para scripting, no para software enterprise). Aquí se construye `OracleManager`, una **clase singleton** que centraliza connection pooling, credenciales y error handling → cada agente accede a la base por un **único gateway seguro y performante**, desacoplando la conectividad de los agentes.

![[04-fig-4.2.png]]
*Figure 4.2: The singleton Oracle functionality*

> [!note] Prerequisito
> Se reusa la misma funcionalidad de conexión del cap. 3, repurposeada en un archivo `.py`. La conexión funciona corriendo **primero los notebooks del cap. 3** (la data y el schema deben existir).

### Imports y logging

`oracledb` (conectividad), `os` (paths), `contextmanager` (resource handling seguro), y `logging` con feedback timestamped (esencial para debuggear interacciones asíncronas de agentes):

```python
import oracledb
import os
import logging
from contextlib import contextmanager

# Configure logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - [OracleLib] - %(levelname)s - %(message)s')
```

### Estructura singleton

`OracleManager` con variables a nivel de clase `_connection` y `_creds` para mantener el estado compartido entre todos los agentes. A diferencia de una clase estándar (un objeto nuevo por interacción), el singleton asegura que **solo exista un connection pool en todo el lifecycle de la app**:

```python
class OracleManager:
    """A Singleton-style wrapper for Oracle Database connections.
    It manages the connection pool and handles secure wallet authentication."""
    _connection = None
    _creds = {}
```

### Lógica de inicialización

El método `initialize` (classmethod) es el entry point seguro: acepta paths al wallet y credenciales, conecta, y hace un **heartbeat query inmediato** (`SELECT 'Heartbeat OK' FROM dual`) para **fail fast** si la base es inalcanzable:

```python
    @classmethod
    def initialize(cls, creds_path=None, wallet_path=None):
        """Reads credentials and establishes the primary connection.
        This must be called once before any agent attempts to work."""
        logging.info("Initializing Oracle Connection Manager...")
        if not creds_path:
            creds_path = "/content/drive/MyDrive/files/oracle/credentials.txt"
        if not wallet_path:
            wallet_path = "/content/drive/MyDrive/files/oracle"

        # 1. Load Credentials
        if not os.path.exists(creds_path):
            logging.error(f"Credentials file not found at {creds_path}")
            raise FileNotFoundError(f"Credentials file missing: {creds_path}")
        cls._creds = cls._read_key_value_file(creds_path)

        # 2. Connect
        try:
            cls._connection = oracledb.connect(
                user=cls._creds.get("user"), password=cls._creds.get("password"),
                dsn=cls._creds.get("dsn"), config_dir=wallet_path,
                wallet_location=wallet_path, wallet_password=cls._creds.get("wallet_password")
            )
            logging.info("✅ Secure connection to Oracle 23ai established.")
            with cls._connection.cursor() as cursor:   # Simple heartbeat check
                cursor.execute("SELECT 'Heartbeat OK' FROM dual")
                result = cursor.fetchone()
                logging.info(f"   Database says: {result[0]}")
        except Exception as e:
            logging.error(f"Failed to connect to Oracle: {e}")
            raise e
```

### Context manager — `get_cursor`

Manejar cursores a mano es propenso a errores (olvidar cerrar, no hacer rollback tras excepción). El context manager `get_cursor` yield-ea un cursor y maneja automáticamente el **commit/rollback** según el operación tenga éxito o falle:

```python
    @classmethod
    @contextmanager
    def get_cursor(cls):
        """A context manager that yields a cursor.
        Ensures the cursor is closed after use, even if errors occur."""
        if cls._connection is None:
            raise ConnectionError("OracleManager is not initialized. Call initialize() first.")
        cursor = cls._connection.cursor()
        try:
            yield cursor
            cls._connection.commit()   # auto-commit (RAG agents are mostly read-only / self-contained)
        except Exception as e:
            cls._connection.rollback()
            logging.error(f"Database operation failed: {e}")
            raise e
        finally:
            cursor.close()
```

### Utilidades y cleanup

`_read_key_value_file` parsea el `credentials.txt` (formato `key=value`, ignora comentarios `#`); `close` termina la conexión limpiamente al apagar la app:

```python
    @staticmethod
    def _read_key_value_file(path):
        """Helper to parse the standard credentials.txt format."""
        creds = {}
        with open(path, "r", encoding="utf-8") as f:
            for line in f:
                line = line.strip()
                if not line or line.startswith("#"):
                    continue
                if "=" not in line:
                    continue
                k, v = line.split("=", 1)
                creds[k.strip()] = v.strip()
        return creds

    @classmethod
    def close(cls):
        """Closes the main connection cleanly."""
        if cls._connection:
            cls._connection.close()
            cls._connection = None
            logging.info("Oracle connection closed.")
```

## Phase 2: Unstructured data agent (`agent_archivist.py`)

El **Oracle Archivist**: encapsula la lógica de vector search puro del cap. 2 en una interfaz MCP estandarizada. Trata la query a `KNOWLEDGE_STORE` como un **servicio self-contained** → el MAS recupera texto no estructurado sin entender el SQL/vector math subyacente. Es la contraparte directa del agente "Researcher" (cap. 5) que consulta Pinecone — pero el Archivist consulta Oracle 23ai.

![[04-fig-4.3.png]]
*Figure 4.3: Implementing the Agent Archivist*

### Signatura y dependency injection

Importa `oracledb`, `create_mcp_message` (del `helpers`) y el `OracleManager`. La firma acepta un `mcp_message` (dict), un `client` de OpenAI (para embeddings) y el modelo — patrón de **dependency injection** que mantiene el agente **stateless**:

```python
import logging
import oracledb
from helpers import create_mcp_message
from oracle_lib import OracleManager

logging.basicConfig(level=logging.INFO,
    format='%(asctime)s - [Archivist] - %(levelname)s - %(message)s')

def agent_oracle_archivist(mcp_message, client, embedding_model="text-embedding-3-small"):
    """Retrieves unstructured text from Oracle 23ai's KNOWLEDGE_STORE table."""
    agent_name = "OracleArchivist"
    logging.info(f"[{agent_name}] Activated.")
```

### Parsing del input y vectorización

Extrae la intención (acepta `intent_query` **o** `topic_query` para flexibilidad de cómo otros agentes lo direccionan) y la vectoriza reusando el `client` inyectado:

```python
    try:
        content = mcp_message.get('content', {})
        user_query = content.get('intent_query') or content.get('topic_query')
        if not user_query:
            raise ValueError(f"[{agent_name}] Input requires 'intent_query' or 'topic_query'.")
        response = client.embeddings.create(input=user_query, model=embedding_model)
        query_vector = response.data[0].embedding
```

### Ejecución del vector search

Usa el `OracleManager.get_cursor()` (context manager) y `cursor.setinputsizes(v=oracledb.DB_TYPE_VECTOR)` (crítico para performance: le dice al driver que `:v` es vector nativo, evitando overhead de type inference). Métrica **DOT** consistente con el cap. 2; lee CLOBs con `.read()`:

```python
        retrieved_text_blocks = []
        with OracleManager.get_cursor() as cursor:
            sql = """
                SELECT text, VECTOR_DISTANCE(embedding, :v, DOT) as score
                FROM knowledge_store
                ORDER BY score DESC
                FETCH FIRST 3 ROWS ONLY
            """
            cursor.setinputsizes(v=oracledb.DB_TYPE_VECTOR)
            cursor.execute(sql, {"v": query_vector})
            rows = cursor.fetchall()
            for row in rows:
                text_chunk = row[0].read() if hasattr(row[0], 'read') else str(row[0])
                score = row[1]
                logging.info(f"[{agent_name}] Found match (Score: {score:.3f})")
                retrieved_text_blocks.append(text_chunk)
```

### Output con formato MCP

Fallback graceful si no hay docs; si hay, junta los chunks en un context block y lo envuelve en el dict MCP estándar vía `create_mcp_message`. El `except` devuelve un **error message seguro** en vez de crashear todo el engine:

```python
        if not retrieved_text_blocks:
            logging.warning(f"[{agent_name}] No relevant documents found.")
            combined_context = "No relevant documents found in the Knowledge Store."
        else:
            combined_context = "\n\n--- DOCUMENT EVIDENCE ---\n".join(retrieved_text_blocks)
        output_content = {
            "retrieved_context": combined_context,
            "source": "Oracle 23ai (KNOWLEDGE_STORE)"
        }
        return create_mcp_message(agent_name, output_content)
    except Exception as e:
        logging.error(f"[{agent_name}] Failed: {e}")
        return create_mcp_message(agent_name, {"error": str(e)})
```

## Phase 3: Structured data agent (`agent_recruiter.py`)

El **Oracle Recruiter**: el componente más complejo. Encapsula el hybrid RAG del cap. 3, transformando la SQL cruda en una función flexible que **parsea constraints JSON**. A diferencia del Archivist (solo similitud semántica), el Recruiter interpreta **reglas de negocio rígidas** (salary caps, experience thresholds) y las fusiona dinámicamente con los vector similarity scores. Esconde la complejidad del hybrid filtering tras una interfaz limpia.

![[04-fig-4.4.png]]
*Figure 4.4: Building the Recruiter agent*

### Signatura

Igual al Archivist pero importa también `json` (para los payloads estructurados más complejos):

```python
import logging
import oracledb
import json
from helpers import create_mcp_message
from oracle_lib import OracleManager

def agent_oracle_recruiter(mcp_message, client, embedding_model="text-embedding-3-small"):
    """Retrieves candidates from Oracle 23ai's CANDIDATE_POOL using Hybrid Search."""
    agent_name = "OracleRecruiter"
    logging.info(f"[{agent_name}] Activated.")
```

### Parsing de los hybrid constraints

Lo que diferencia al Recruiter de un retriever estándar: extrae **no solo la query semántica** sino también los **constraints escalares** del dict `constraints` del MCP. Defaults razonables (cap de $1M, exp 0) para que la query ejecute aun con constraints parciales:

```python
    try:
        content = mcp_message.get('content', {})
        user_query = content.get('intent_query')
        constraints = content.get('constraints', {})
        max_salary = constraints.get('max_salary', 1000000)
        min_experience = constraints.get('min_experience', 0)
        if not user_query:
            raise ValueError(f"[{agent_name}] Input requires 'intent_query'.")
        logging.info(f"[{agent_name}] Searching: '{user_query}' (Sal <= {max_salary}, Exp >= {min_experience})")
        response = client.embeddings.create(input=user_query, model=embedding_model)
        query_vector = response.data[0].embedding
```

### Ejecución de la hybrid query

La query que **define la converged database**: `WHERE` para los escalares (`salary_expectation`, `years_experience`) + `ORDER BY VECTOR_DISTANCE(resume_vector, :v, DOT)` para lo semántico. Cada candidato devuelto es **a la vez match semántico y opción de negocio viable**. Se castea el score a `float` para que sea JSON serializable:

```python
        candidates_found = []
        with OracleManager.get_cursor() as cursor:
            sql = """
                SELECT candidate_id, full_name, years_experience, salary_expectation, summary,
                       VECTOR_DISTANCE(resume_vector, :v, DOT) as similarity
                FROM candidate_pool
                WHERE salary_expectation <= :max_sal
                  AND years_experience >= :min_exp
                ORDER BY similarity DESC
                FETCH FIRST 3 ROWS ONLY
            """
            # ... (setinputsizes, execute, fetch) ...
            logging.info(f"[{agent_name}] Match: {name} (Score: {score:.3f}, Sal: {sal})")
            candidates_found.append({
                "id": c_id, "name": name, "experience": exp, "salary": sal,
                "match_score": float(score),   # Ensure JSON serializable
                "summary": summary_text
            })
```

### Output estructurado dual

A diferencia del Archivist (texto crudo), el Recruiter devuelve una **lista estructurada de candidatos** (`candidate_list`) **y** un `retrieved_context` (text block) para compatibilidad con agentes downstream (ej. un Writer que prefiera consumir un string simple):

```python
        formatted_text_block = ""
        for c in candidates_found:
            formatted_text_block += (
                f"--- CANDIDATE: {c['name']} (ID: {c['id']}) ---\n"
                f"Score: {c['match_score']:.3f}\n"
                f"Experience: {c['experience']} years\n"
                f"Salary: ${c['salary']:,}\n"
                f"Summary: {c['summary']}\n\n"
            )
        output_content = {
            "candidate_list": candidates_found,
            "retrieved_context": formatted_text_block,   # Standard key for Writer compatibility
            "source": "Oracle 23ai (Hybrid Search)"
        }
        return create_mcp_message(agent_name, output_content)
    except Exception as e:
        logging.error(f"[{agent_name}] Failed: {e}")
        return create_mcp_message(agent_name, {"error": str(e)})
```

## Phase 4: Verification suite (`Unified_Agents_Unit_Test.ipynb`)

*Never assume code works until you test it.* Un notebook dedicado como **staging ground** que replica el entorno del engine completo **sin el overhead del planner y el tracer**, validando que los agentes parsean inputs, ejecutan queries y devuelven el formato estricto requerido.

### Setup e imports

Instala dependencias pineadas (`oracledb==3.4.1 openai==2.14.0 tiktoken==0.7.0 tqdm==4.67.1`), carga la API key vía Colab `userdata`, e **importa los módulos custom como librerías Python estándar** (primera vez que `agent_archivist`/`agent_recruiter` se tratan como libs), inicializando el singleton:

```python
!pip install oracledb==3.4.1 openai==2.14.0 tiktoken==0.7.0 tqdm==4.67.1 --quiet

import os
from google.colab import userdata
from openai import OpenAI
api_key = userdata.get("API_KEY")
os.environ["OPENAI_API_KEY"] = api_key
client = OpenAI()

from oracle_lib import OracleManager
from agent_archivist import agent_oracle_archivist
from agent_recruiter import agent_oracle_recruiter
from helpers import create_mcp_message
OracleManager.initialize()   # Initialize the Singleton Connection
```

### Test Case A — Archivist

Mensaje MCP de test (con `create_mcp_message`, formato idéntico al que envía el engine real), se llama al agente y se **valida la estructura del output** (que exista `retrieved_context`):

```python
archivist_input = create_mcp_message("Tester", {
    "intent_query": "What happened during the cyber incident?"
})
response_a = agent_oracle_archivist(archivist_input, client)
content_a = response_a.get("content", {})
if "retrieved_context" in content_a:
    print(f"[Result] Source: {content_a.get('source')}")
    print(f"[Result] Context Snippet: {content_a['retrieved_context'][:100]}...")
    print("✅ Archivist Test Passed.")
```

Output: `Source: Oracle 23ai (KNOWLEDGE_STORE)`, snippet *"Server logs indicate unauthorized access at 03:00..."*, ✅ pasó.

### Test Case B — Recruiter (hybrid)

Valida el hybrid RAG con constraints estrictos (max salary 200k, min exp 4 años) buscando "Python Team Lead": el agente debe **respetar los límites escalares** mientras halla el mejor match semántico:

```python
recruiter_input = create_mcp_message("Tester", {
    "intent_query": "Find a Python Team Lead",
    "constraints": {"max_salary": 200000, "min_experience": 4}
})
response_b = agent_oracle_recruiter(recruiter_input, client)
content_b = response_b.get("content", {})
candidates = content_b.get("candidate_list", [])
if candidates:
    print(f"[Result] Candidates Found: {len(candidates)}")
    top_candidate = candidates[0]
    print(f"[Result] Top Candidate: {top_candidate['name']} (Score: {top_candidate['match_score']:.3f})")
    print("✅ Recruiter Test Passed.")
```

Output: `Source: Oracle 23ai (Hybrid Search)`, `Candidates Found: 3`, `Top Match: Quinn (Score: 0.890)`, ✅ pasó — confirma que la lógica de converged database funciona correctamente detrás de la API.

### Cleanup

*Good software citizenship*: cerrar la conexión cuando ya no se necesita.

```python
OracleManager.close()   # ✅ Connection closed.
```

Los agentes están **modularizados, implementados y verificados** — listos para deployarse en el multi-agent engine del cap. 5.

## Citas

> "If we hard-code our database logic directly into the reasoning engine, we create a brittle system that cannot adapt to new environments. The goal of this chapter is to decouple the data layer from the reasoning layer."
> "the Jupyter Notebook is not merely a scratchpad; it serves as the central engine and orchestrator."
> "We are not reinventing the logic of the Recruiter or the Archivist agents. We are repurposing it"
> "In software engineering, we never assume code works until we test it."
> "transforms Oracle Database 23ai from a passive storage engine into an active participant in our intelligent system."

## Para aplicar

- **Desacoplar la capa de datos de la de razonamiento** — no hard-codear la lógica de base en el reasoning engine; envolverla en módulos `.py` stateless detrás de una interfaz estandarizada (interface pattern).
- **Tratar el notebook como engine/orchestrator mínimo** — flujo de ejecución, config y visualización en el notebook; clases y lógica pesada en `.py` puros → reproducibilidad, modularidad (unit-testeable), claridad narrativa.
- **Estandarizar la comunicación con MCP** (`create_mcp_message`) — cada agente acepta un `mcp_message` y devuelve un dict MCP para consumo machine-to-machine; agentes intercambiables en cualquier framework.
- **Centralizar la conexión con un singleton (`OracleManager`)** — un único connection pool por lifecycle; `initialize` con heartbeat (`SELECT ... FROM dual`) para fail-fast; `close` para cleanup; evitar que cada agente maneje su propia conexión.
- **Usar un context manager para los cursores** (`get_cursor`) — commit/rollback automático según éxito/fallo, `finally: cursor.close()`; sesión estable pase lo que pase dentro del agente.
- **Hacer los agentes stateless con dependency injection** — pasar `client`/modelo como parámetros; no estado interno; reusar el `client` inyectado para embeddings.
- **Distinguir agente no estructurado (Archivist) vs estructurado (Recruiter)** — el Archivist devuelve texto crudo (`retrieved_context`); el Recruiter parsea `constraints` y devuelve lista estructurada (`candidate_list`) **+** text block para compatibilidad con downstream (Writer).
- **Castear scores a `float` para JSON-serializabilidad** y leer CLOBs con `.read()` — la salida MCP debe ser serializable; manejar LOBs defensivamente.
- **Devolver errores seguros en vez de crashear** — el `except` de cada agente retorna `create_mcp_message(agent, {"error": ...})`, manteniendo el engine vivo.
- **Construir una verification suite antes de deployar** — unit-test notebook como staging ground (sin el overhead de planner/tracer); validar parsing de inputs, ejecución de queries y formato de output con constraints estrictos.
- **Empaquetar las dependencias (`unit_library.zip`)** — zipear los `.py` y subirlos para que el notebook los descargue/descomprima; portabilidad del set de módulos.

## Conexiones

- [[_RAG-Driven Generative AI - Second Edition|Mapa del libro]] — el MOC del libro.
- [[03 - Building a Live Recruiter Agent]] — cap. 3: este capítulo **repurposea** su lógica (`find_candidates_hybrid` → `agent_oracle_recruiter`), su schema y data (consumidos pasivamente), su hybrid query (`WHERE` + `VECTOR_DISTANCE DOT`). Tabla 4.1 mapea explícitamente cap. 3 → cap. 4.
- [[02 - RAG Embeddings in Oracle Vector Stores]] — cap. 2: el vector search puro sobre `KNOWLEDGE_STORE` que el Archivist encapsula; `setinputsizes`/CLOB/`VECTOR_DISTANCE` reusados.
- [[01 - Why Retrieval-Augmented Generation]] — cap. 1: el `RetrievalComponent` modular (agente primitivo self-contained) anticipaba exactamente esta modularización en agentes soberanos.
- [[05 - Building a Universal Context Engine]] — cap. 5: integra estos agentes soberanos en un **Universal Context Engine** multi-agente; se registran en `registry.py` y el Planner los selecciona dinámicamente; el "Researcher" (Pinecone) como contraparte del Archivist.
- [[_RAG|RAG]] · [[Enterprise RAG Assistant]] · [[Hybrid Search]] · [[Reranking]] · [[Chunking Strategies]] — el patrón RAG del vault; Archivist = retriever semántico, Recruiter = hybrid search.
- [[Function Calling]] · [[Tool Calling]] · [[Orchestrator]] — los agentes soberanos como tools MCP del orchestrator/engine; interface pattern para tool-calling.
- [[Grounding]] · [[Hallucinations]] — agentes que devuelven evidencia con `source` (trazabilidad/grounding) y errores seguros.
- **Singleton pattern · `OracleManager` · Context manager (`contextmanager`/`get_cursor`) · MCP (Model Context Protocol) · `create_mcp_message` · Interface pattern · Stateless agents · Dependency injection · Sovereign agents · Oracle Archivist · Oracle Recruiter · Universal Context Engine · Connection pooling · Heartbeat check · `unit_library.zip` · Unit test suite** — conceptos/técnicas del capítulo; candidatos a nota propia (varios ya existen en el vault, otros son semillas).
