---
title: 05 - Building a Universal Context Engine
libro: RAG-Driven Generative AI - Second Edition
autor: Denis Rothman
capitulo: 5
created: 2026-06-11
tags:
  - libros/rag-driven-generative-ai
  - type/case-study
  - status/permanent
aliases:
  - Building a Universal Context Engine
  - Universal Context Engine
updated: 2026-06-11
---

# 05 - Building a Universal Context Engine

> [!info] Capítulo 5 · *RAG-Driven Generative AI - Second Edition* — Denis Rothman
> De agentes soberanos aislados a un **sistema unificado**: el **Universal Context Engine**, un MAS-RAG cross-domain donde un reasoning engine **indiferente al medio de almacenamiento** consulta el `registry.py` (routing table), y un **Planner** (LLM) elige dinámicamente los agentes según el `get_capabilities_description()` — *architectural prompt engineering*. Se integran el Archivist y el Recruiter del cap. 4, se configura el runtime híbrido (OpenAI + OracleManager) y se prueban escenarios de convergencia (búsqueda semántica + constraint SQL + generación de email). Navegá: [[_RAG-Driven Generative AI - Second Edition|Mapa del libro]] · anterior [[04 - Building Sovereign Enterprise Agents]] · siguiente [[06 - Operationalizing the Universal Context Engine]].

## Resumen

El cap. 4 construyó agentes soberanos aislados (component construction); este capítulo da el salto a la **unificación del sistema** (system unification). Es un capítulo hands-on que **fusiona múltiples librerías en un único engine** que se usará para el resto del libro. La **tesis arquitectónica fundamental**: *un reasoning engine debe ser indiferente al medio de almacenamiento*. Sea que la data viva en un vector store o en una tabla relacional, **la lógica del engine permanece estática**. Esto se logra con el **Universal Context Engine** — un context engine liviano y domain-agnostic para construir MAS transparentes con dual RAG, memoria y orquestación protocol-driven; ofrece una arquitectura **glass-box** con execution traces, token analytics y reasoning flows observables (comportamiento de agentes auditable), reusable en múltiples dominios sin reescribir la lógica del engine.

El framework se apoya en dos assets críticos: (1) el **Registry** (`registry.py`) — la *routing table* del sistema, que centraliza los agent handlers y sus dependencias de infraestructura, **desacoplando el "How" (ejecución) del "Nature" (estructura de datos)**; evoluciona de una lista simple de funciones a un **dynamic capability provider** que hace al engine plug-and-play (agregar agentes Oracle = agregar dimensiones cognitivas nuevas **sin tocar el reasoning loop core**); (2) el **Planner** (driven por el LLM) — **no tiene conocimiento hard-coded de las tools**; depende enteramente del string `get_capabilities_description()` del registry. Refinar esas descripciones es **escribir el manual de instrucciones del Planner**: si el manual dice que el Recruiter maneja salary constraints, el Planner lo incluirá automáticamente en un plan de HR. El registry es **donde se define la intención del sistema** — esto es *architectural prompt engineering*.

Se fusiona la lógica madura del **core engine library (CEL)** —derivada del repo open-source `Context-Engineering-for-Multi-Agent-Systems`, que provee planning, ejecución y context management— con los **agentes soberanos del cap. 4** (`unit_library.zip`). Roadmap en 3 pasos: (1) **actualizar el registry** importando `agent_archivist`/`agent_recruiter`, agregándolos al `AGENT_TOOLKIT` y "programando" el Planner vía `get_capabilities_description()`; (2) **configurar el runtime híbrido** mergeando los helpers cloud (OpenAI) con `oracle_lib` (el singleton `OracleManager`); (3) **deployar escenarios de convergencia** — un plan cross-domain/cross-modal que fuerza al Planner a elegir dinámicamente los agentes Oracle para combinar constraints SQL estructurados con vector search semántico sobre datasets distintos. El resultado: un **MAS-RAG cross-domain universal** donde el engine es estático pero el registry lo adapta dinámicamente.

![[05-fig-5.1.png]]
*Figure 5.1: The Universal Context Engine architecture*

## Arquitectura del Universal Context Engine

Transición de *component construction* (cap. 4) a *system unification*. Los agentes soberanos son componentes stateless y modulares que adhieren a una interfaz estricta, **desacoplando la lógica de base del reasoning engine**. Es "universal" porque usa **un único framework para múltiples dominios y tareas**. El Engine se sienta al centro, **completamente aislado de los drivers específicos de base**; se comunica **solo con el Registry**, que conmuta el path de ejecución según el goal.

> [!note] Los dos assets críticos
> **Registry (`registry.py`)** — la routing table; centraliza handlers + dependencias de infra, desacoplando el *How* (ejecución) del *Nature* (estructura de datos). Es un **dynamic capability provider**: agregar agentes Oracle suma dimensiones cognitivas **sin modificar el reasoning loop core**. **Planner (LLM)** — sin conocimiento hard-coded de las tools; depende del `get_capabilities_description()` del registry. Refinar esas descripciones = escribir el manual del Planner = **architectural prompt engineering**. El Registry es donde se define la intención del sistema.

### Qué hay de nuevo — CEL + agentes soberanos

Primer paso de un **MAS-RAG cross-domain universal**. Se fusiona la lógica madura del **core engine library (CEL)** (del repo `Context-Engineering-for-Multi-Agent-Systems/commons/engine/`, librería algorítmica primaria con planning/ejecución/context management) con los agentes enterprise del cap. 4. La evolución de cada componente:

| Componente | Fuente | Evolución respecto del cap. 4 |
|---|---|---|
| **Orchestration engine** (`engine.py`) | CEL (`engine.zip`) | Cap. 4 usaba un unit-test harness liviano; ahora se promueve al **multi-step reasoner completo con Planner y Executor** para metas multi-agente complejas. |
| **Agent registry** (`registry.py`) | CEL (`engine.zip`) | Cap. 4 importaba los agentes manualmente en un unit-test; ahora se **enterprise-enable** registrando los agentes Oracle y agregando sus capacidades al description string del planner → el engine estático interactúa con la converged DB vía interfaz estándar. |
| **Sovereign agents** (`agent_archivist.py`, `agent_recruiter.py`) | `unit_library.zip` (cap. 4) | Antes testeados en aislamiento (MCP compliance); ahora **integrados como especialistas**, ya no invocados manualmente sino **seleccionados dinámicamente por el Planner** según el goal. |
| **Infrastructure layer** (`oracle_lib.py`) | `unit_library.zip` (cap. 4) | El singleton `OracleManager` se vuelve **dependencia core** del setup del engine; garantiza que cuando el Registry llama un agente Oracle ya hay conexión segura establecida. |
| **System helpers** (`helpers.py`) | CEL (`engine.zip`) | Backbone funcional del cap. 4 (LLM interaction, MCP message creation); ahora **utilidad compartida** con sanitización y moderación enterprise. |

## Step 1: Actualizar `registry.py`

El registry es el orchestrator central del context engine: permite operar **sin conocimiento directo de los drivers de base**, manteniendo separación limpia entre la lógica de razonamiento y las estructuras de datos.

![[05-fig-5.2.png]]
*Figure 5.2 Incorporating enterprise agent modules*

**Imports** — traer los agentes soberanos del cap. 4:

```python
# === Imports ===
import logging
# Import Standard LLM Agents (No Database Dependency)
from agents import agent_writer, agent_summarizer
# Import Enterprise Oracle Agents
import agent_archivist
import agent_recruiter
```

**Global toolkit registry** — el dict que mapea títulos de agentes a sus funciones ejecutables. Se registran las capacidades enterprise para invocación dinámica:

```python
class AgentRegistry:
    def __init__(self):
        self.registry = {
            # --- Standard LLM Agents ---
            "Writer": agent_writer,
            "Summarizer": agent_summarizer,
            # --- Enterprise Oracle Agents ---
            "OracleArchivist": agent_archivist.agent_oracle_archivist,
            "OracleRecruiter": agent_recruiter.agent_oracle_recruiter,
        }
```

**Architectural prompt engineering para el Planner** — el Planner depende de una descripción de texto estructurada para entender cómo usar las tools. Se actualiza `get_capabilities_description()` con los roles e inputs requeridos de los agentes Oracle (el **manual de instrucciones** del reasoning engine):

```python
    def get_capabilities_description(self):
        """Returns a structured description of the agents for the Planner LLM."""
        return """
Available Agents and their required inputs:
...
3. AGENT: OracleArchivist
   ROLE: Retrieves unstructured documents (logs, reports) from the Oracle Database.
   INPUTS:
     - "intent_query": (String) A descriptive search query.

4. AGENT: OracleRecruiter
   ROLE: Retrieves job candidates from Oracle Database using Hybrid Search (Structured + Semantic).
   INPUTS:
     - "intent_query": (String) Semantic skill description.
     - "constraints": (Dict) {"max_salary": integer, "min_experience": integer}
"""
```

> [!tip] El poder del architectural prompt engineering
> Refinar estas descripciones **guía al Planner** a elegir el Recruiter cuando el usuario menciona salary o experience constraints, asegurando que el sistema respete la lógica de negocio mientras hace búsquedas semánticas. La intención del sistema se define **en el registry, no en el engine**.

Output al inicializar: `INFO - Agent Registry initialized with Oracle Enterprise capabilities.` — el registry ya rutea requests a stores estructurados y no estructurados.

## Step 2: Configurar el runtime híbrido

En `Universal_Context_Engine_Converged_Edition.ipynb` se ensambla el entorno de ejecución que mergea **helpers cloud (OpenAI) + infraestructura enterprise (la converged DB)**.

![[05-fig-5.3.png]]
*Figure 5.3: Building a hybrid architecture*

El bloque **Oracle Manager** = gateway persistente a data estructurada/no estructurada; **OpenAI** = inteligencia generativa; **Notebook configuration** = asegura que ambos sistemas dispares estén activos y autenticados **antes de cualquier planning agéntico**. Los assets vienen de `unit_library.zip` (cap. 4) y `engine.zip` (CEL).

**Dependencias universales** — versiones pineadas (clave: `pydantic>=2.0` resuelve conflictos en Colab que de otro modo impedirían al Planner parsear los execution plans):

```python
!pip install --upgrade --force-reinstall \
    openai==2.26.0 pydantic==2.11.9 tiktoken==0.12.0 tqdm==4.67.1 \
    tenacity==9.1.2 oracledb==3.4.1 requests==2.32.4 \
    cryptography==43.0.3 pyOpenSSL==24.2.1 --quiet
print("✅ Oracle 23ai and Frozen LLM dependencies installed.")
```

**Cloud clients** — montar el drive (para el Oracle Wallet) e inicializar OpenAI (OpenAI es **mandatorio**: sin él el Planner no funciona, por eso se hace `raise`):

```python
import os
from google.colab import drive, userdata
from openai import OpenAI

if not os.path.exists('/content/drive'):
    drive.mount('/content/drive')
try:
    openai_key = userdata.get("API_KEY")
    if not openai_key:
        raise ValueError("OpenAI API_KEY not found in Secrets.")
    os.environ["OPENAI_API_KEY"] = openai_key
    client = OpenAI()
    print("✅ OpenAI initialized (Mandatory).")
except Exception as e:
    print(f"❌ Critical Error: OpenAI initialization failed. {e}")
    raise e   # OpenAI is required for the Planner to work at all
```

**Enterprise connection** — activar el singleton `OracleManager` del cap. 4 (gateway persistente a Oracle 23ai para el recruiter y el archivist); `initialize()` carga credenciales y hace el heartbeat check:

```python
from oracle_lib import OracleManager
try:
    OracleManager.initialize()
    print("✅ Oracle 23ai Connection established.")
except Exception as e:
    print(f"⚠️ Oracle Connection Failed: {e}")
```

El runtime híbrido queda **operativo**: base estable para que el engine ejecute sus primeras tareas.

## Step 3: Deployar escenarios de convergencia

Un **plan de ejecución cross-domain universal** que prueba que el Planner puede **conmutar dinámicamente entre modalidades de datos**, eligiendo el especialista correcto para una meta compleja que combina constraints de negocio estructurados + búsqueda semántica.

> [!warning] Outputs estocásticos
> Los sistemas de IA generativa son **estocásticos**: los outputs pueden variar entre runs por sus algoritmos probabilísticos.

**El execution wrapper** — `run_oracle_recruitment_scenario` abstrae la complejidad de la llamada al engine (`run_universal_engine`), tomando un goal en lenguaje natural y formateando el output con `textwrap.fill` (preservando párrafos):

```python
import textwrap

def run_oracle_recruitment_scenario(role: str, width: int = 80):
    """Encapsulates Scenario A: Recruitment using the OracleRecruiter agent.
    Returns: tuple: (result, trace)"""
    recruitment_goal = role
    print("-" * width)
    print(f"Goal: {recruitment_goal}")
    print("-" * width)
    try:
        result, trace = run_universal_engine(recruitment_goal)   # Execute the engine
        print("\n--- FINAL RECRUITMENT EMAILS ---\n")
        if result:
            for line in str(result).splitlines():
                if line.strip():
                    print(textwrap.fill(line, width=width))
                else:
                    print()   # Preserve empty lines
        else:
            print("No result returned.")
        print("-" * width)
        return result, trace
    except NameError:
        print("Error: `run_universal_engine` is not defined.")
        return None, None
```

### Scenario A — búsqueda semántica pura

Búsqueda semántica amplia **sin constraints escalares estrictos**: el engine debe rutear al `OracleRecruiter`, que hace vector similarity search para candidatos cuyos resumes matchean *experienced Python developer*.

```python
res, log = run_oracle_recruitment_scenario("Find Experienced Python Developers with experience")
```

El Planner identificó correctamente el intent como recruitment search y activó el Recruiter, que usó vector embeddings para entender que **Alex V.** (12 años, microservices) es un match semántico fuerte, **Quinn R.** (8 años, ahora más management) match medio, y **Riley S.** (Java/PL/SQL) un match débil para un rol Python. El engine **sintetizó las filas crudas de la base en perfiles coherentes** que resaltan el *fit* para el rol:

```text
**Candidate: Alex V.** — Python experience: 12 years; scalable microservices, cloud-native, Go, AWS, CI/CD, led team of 10. → Strong fit (senior backend/microservices, cloud-native, high-scale).
**Candidate: Quinn R.** — 8 years (historically strong, currently less hands-on); engineering management, mentorship. → Best for senior/lead role combining Python oversight + people management.
**Candidate: Riley S.** — Limited Python (Java-focused); Java, PL/SQL, banking, Oracle DB. → Not a strong fit for a senior Python position.
```

### Scenario B — multi-step reasoning con constraints estructurados

Escala la complejidad: request **híbrido** que requiere entendimiento semántico (*"Python Developer"*) **+** filtrado estructurado (max salary 250,000) **+** una tarea generativa (escribir un email de entrevista). Fuerza una **cadena multi-step**: (1) **Plan**: identificar la necesidad de `OracleRecruiter` (búsqueda) y `Writer` (email); (2) **Filter**: aplicar el constraint de salary máximo $250,000 a la query de base; (3) **Synthesize**: tomar la data del candidato recuperado y generar el email.

```python
res, log = run_oracle_recruitment_scenario(
    "Find Experienced Python Developers with experience and 250000 salary max. "
    "Then, write a job interview email to the top candidate, explicitly using their name.")
```

El output es una invitación de entrevista profesional y completa, desglosable en: **header/salutation** (identifica al top candidate filtrado, **Quinn**, y lo direcciona por nombre: *"Subject: Interview Invitation – Senior Python Engineer Opportunity / Dear Quinn,"*), **value proposition** (elabora el rol referenciando implícitamente el salary tier: *"salary budget capped at 250,000 per year"*), y **CTA** (logística de scheduling con opciones de horarios — formato de correspondencia de negocio).

Esto valida la naturaleza **universal** del context engine: combinó un **constraint estructurado (Salary < 250k)** con una **búsqueda semántica (Python)** y luego pivoteó a una **tarea generativa (el email)**. El prompt engineering del registry aseguró que el Planner supiera **exactamente qué agente** maneja el salary constraint, y el Writer tomó ese contexto para producir un email coherente.

> [!tip] Instrucción al Planner sobre el Writer
> Hay que **instruir al Planner para que le diga al Writer que mire los search results** y extraiga variables específicas (ej. el nombre del candidato). La coordinación entre agentes se programa vía el goal + las descripciones del registry.

## System cleanup

Buena práctica: liberar recursos al terminar. Pero **cuidado de no cerrar la conexión mientras se siguen corriendo escenarios** — se usa un flag:

```python
close=False
if close==True:
    OracleManager.close()
```

El engine logic permanece **estático**, mientras el registry le permite **adaptarse dinámicamente** a data enterprise compleja y metas multi-modales. El cap. 6 enhanceará el Universal Context Engine con nuevos componentes, como una interfaz gráfica de control.

## Citas

> "a reasoning engine should be indifferent to the storage medium."
> "When we add Oracle agents to the registry, we aren't just adding code; we are adding new cognitive dimensions to the engine without modifying its core reasoning loop."
> "By refining the descriptions in the registry, we are essentially writing the instruction manual for the planner."
> "The Registry, therefore, is where the system's intent is defined."
> "the engine logic remains static, while the registry allows it to adapt dynamically to complex enterprise data and multi-modal goals."

## Para aplicar

- **Hacer el reasoning engine indiferente al medio de almacenamiento** — el engine se comunica solo con el registry, aislado de los drivers de base; la misma lógica sirve para vector store o tabla relacional.
- **Centralizar el ruteo en un registry (routing table)** — desacoplar el *How* (ejecución) del *Nature* (estructura de datos); evolucionar de lista de funciones a **dynamic capability provider** plug-and-play.
- **Programar el Planner vía `get_capabilities_description()` (architectural prompt engineering)** — el Planner no tiene tools hard-coded; refinar las descripciones del registry es escribir su manual de instrucciones (rol + inputs de cada agente).
- **Agregar capacidades sin tocar el reasoning loop** — registrar agentes nuevos en el `AGENT_TOOLKIT` y describir sus inputs suma dimensiones cognitivas sin modificar el core del engine.
- **Mergear runtime cloud + enterprise** — inicializar OpenAI (mandatorio para el Planner, `raise` si falla) y el singleton `OracleManager` (heartbeat) antes de cualquier planning; montar el drive para el Wallet.
- **Pinear versiones críticas (incl. `pydantic>=2.0`)** — resolver conflictos de parsing que impedirían al Planner interpretar los execution plans.
- **Dejar que el Planner seleccione especialistas dinámicamente** — describir que el Recruiter maneja `constraints` (salary/experience) hace que el Planner lo elija automáticamente ante una query con esos términos.
- **Encadenar multi-step (retrieve → filter → synthesize)** — combinar búsqueda semántica + constraint SQL + tarea generativa (Writer); instruir al Planner para que el Writer **mire los search results** y extraiga variables (ej. el nombre del candidato).
- **Aprovechar la arquitectura glass-box** — execution traces, token analytics y reasoning flows observables para comportamiento auditable.
- **Cerrar la conexión con un flag controlado** (`close=False`) — no cerrar el pool mientras se re-corren escenarios; liberar recursos solo al final.

## Conexiones

- [[_RAG-Driven Generative AI - Second Edition|Mapa del libro]] — el MOC del libro.
- [[04 - Building Sovereign Enterprise Agents]] — cap. 4: este capítulo **integra** sus agentes soberanos (`agent_archivist`, `agent_recruiter`), el `OracleManager` singleton y la interfaz MCP — ya no invocados manualmente sino seleccionados por el Planner; el unit-test harness se promueve al engine completo.
- [[03 - Building a Live Recruiter Agent]] — cap. 3: el hybrid search del Recruiter (skills + salary/experience) que el Scenario B ejerce vía el engine.
- [[02 - RAG Embeddings in Oracle Vector Stores]] — cap. 2: el vector search del Archivist sobre `KNOWLEDGE_STORE`; el dual RAG que el engine soporta.
- [[01 - Why Retrieval-Augmented Generation]] — cap. 1: el **modular RAG** que orquesta dinámicamente la mejor tool (stepping stone a MAS) se realiza aquí como el Planner eligiendo agentes; MAS-RAG.
- [[06 - Operationalizing the Universal Context Engine]] — cap. 6: enhancer el Universal Context Engine con una **interfaz gráfica de control** (`ipywidgets`), control flow event-driven, trace dashboard glass-box y troubleshooting de producción.
- [[_RAG|RAG]] · [[Enterprise RAG Assistant]] · [[Hybrid Search]] · [[Reranking]] · [[Chunking Strategies]] — el patrón RAG del vault; el engine como MAS-RAG cross-domain.
- [[Function Calling]] · [[Tool Calling]] · [[Orchestrator]] — el registry como tool-routing; el Planner/Executor como orquestador multi-step; los agentes como tools MCP.
- [[Grounding]] · [[Hallucinations]] — glass-box (traces, reasoning observable) para grounding y auditabilidad.
- **Universal Context Engine · Core Engine Library (CEL) · Context engineering · Registry / routing table · Planner-Executor · `get_capabilities_description()` · Architectural prompt engineering · Dynamic capability provider · `AGENT_TOOLKIT` · Glass-box architecture · Execution traces / token analytics · MAS-RAG cross-domain · ReAct · Reflexion** — conceptos/técnicas del capítulo; candidatos a nota propia (varios ya existen en el vault, otros son semillas).
