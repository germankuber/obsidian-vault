---
title: "12 - The Autonomous Database Architect"
libro: RAG-Driven Generative AI - Second Edition
autor: Denis Rothman
capitulo: 12
created: 2026-06-11
tags:
  - libros/rag-driven-generative-ai
  - type/case-study
  - status/permanent
aliases:
  - The Autonomous Database Architect
  - Self-Evolving Database
---

# 12 - The Autonomous Database Architect

> [!info] Capítulo 12 (final) · *RAG-Driven Generative AI - Second Edition* — Denis Rothman
> De retrieval **pasivo/probabilístico** a **ejecución determinística**: un agente con privilegios de DBA y skills de data engineer que **percibe data cruda, diseña un schema, ejecuta el DDL y puebla sus propias tablas**. La cuarta era de la interacción con datos: el LLM no como chatbot sino como *fuzzy-logic compiler*. Loop **Perceive-Plan-Act-Audit** (Architect + ETL Worker + Governance), con tres leyes de seguridad (isolation, auditability, reversibility) y schemas self-healing. La **infraestructura self-evolving del futuro**. Navegá: [[_RAG-Driven Generative AI - Second Edition|Mapa del libro]] · anterior [[11 - Scaling AI Workloads with Oracle Exadata]].

## Resumen

Etapa final del journey. Hasta ahora se construyó un MAS capaz de **ver, oír y leer** la empresa (knowledge graphs, video, vector stores), pero la arquitectura seguía **fundamentalmente pasiva**: los agentes **observaban el estado del mundo pero no podían cambiarlo**. Esto se vuelve crítico ante el caos del mundo real: data valiosa en formatos crudos no estructurados (logs desordenados, cadenas de emails reenviados, streams de sensores) **que nunca fue modelada en una tabla relacional**. Un RAG estándar solo puede *adivinar* las respuestas ocultas en ese caos — no puede consultarlas, sumarlas ni filtrarlas determinísticamente **porque la tabla simplemente no existe**.

Este capítulo rompe esa barrera final: de **retrieval probabilístico a ejecución determinística** construyendo un **Autonomous Architect agent** con los **privilegios de un DBA y los skills de un data engineer**. En vez de esperar a que humanos diseñen schemas, el agente **percibe la data, planifica una estructura óptima, ejecuta el DDL para construirla, y puebla sus propias tablas** con información. Es la **cuarta era** de la interacción con datos (2026+): el LLM no como escritor sino como **fuzzy-logic compiler** (*lógica probabilística constreñida por reglas determinísticas*). La base deja de ser una bóveda y se vuelve un **taller**; la IA deja de ser bibliotecario y se vuelve **arquitecto**.

El núcleo es el loop **Perceive-Plan-Act-Audit** sobre tres componentes: el **Architect** (diseña el schema vía LLM → DDL), el **ETL Worker** (extrae, transforma y carga la data cruda en `INSERT`s, usando **schema reflection** para no alucinar columnas), y la **Governance layer** (un `AGENT_SCHEMA_REGISTRY` que loguea cada creación con su *user goal* justificante — nunca una black box). Se valida con dos escenarios (logs de IT → `AI_INCIDENT_DOWNTIME`; crisis de supply chain → `AI_SCM_CRISIS_LOG`), se profundiza en la **anatomía del prompt DDL** (persona, JSON mode, type inference, safety constraints), y se establecen las **tres leyes de seguridad autónoma** (schema isolation/sandbox, forensic auditability, lifecycle reversibility) + las operaciones **Day-2 / self-healing** (reactive schema evolution: catch error → analyze → evolve `ALTER TABLE` → retry). El resultado: una **categoría nueva de software — la self-evolving database**.

![[12-fig-12.1.png]]
*Figure 12.1: The evolution from human-gated SQL to autonomous AI data engineering*

## Evolución de la interacción con datos

Durante décadas, la distancia entre una pregunta de negocio y la respuesta de la base estuvo definida por **fricción, latencia y estructura rígida**. Cuatro eras, según cuánta distancia había entre la *intención* del usuario y la *ejecución* de la query:

- **Phase 1 — Era del "SQL priesthood" (1980s–2010s)**: el RDBMS priorizaba integridad y storage sobre accesibilidad; **el schema era rey** (decidido en el diseño, difícil de cambiar en producción). Un manager no podía preguntar a la base: enviaba un **ticket al IT**; un **DBA/analyst** interpretaba el request vago y craftaba la SQL a mano. Resultados precisos pero **latencia de días/semanas** — la base era una **bóveda** y el DBA el **gatekeeper**.
- **Phase 2 — Era de la abstracción de aplicación (2010s–2023)**: BI tools, dashboards, ORMs para **remover la necesidad de saber SQL**. Los devs pre-anticipaban cada pregunta con views/widgets rígidos. Más accesible pero menos flexible → el problema del **walled garden** (acceso restringido, sin exportar/combinar libremente, locked al vendor). Una pregunta no anticipada (*"correlacioná el inventario de Rotterdam con el clima del Mar del Norte"*) **rompía el sistema** → de vuelta al ciclo de latencia de la Phase 1.
- **Phase 3 — Era del retrieval probabilístico (2023–2025)**: LLMs + RAG (los primeros 11 capítulos). Por primera vez se puede consultar data **nunca modelada en una tabla** (PDFs, emails, transcripts). Pero introduce la **trampa probabilística**: un RAG puede *resumir* mil logs (*"los servers parecen inestables por el calor"*) pero **no puede hacer agregación determinística** — si le pedís *"calculá la suma exacta de minutos de downtime de todos los incidentes Severity 1"* lucha, **alucina un número** o se pierde un log, porque recupera chunks por similitud semántica, no ejecuta un cálculo de filas. **El agente puede leer pero no puede contar.**
- **Phase 4 — Era del autonomous architect (2026+)**: de retrieval probabilístico a **ejecución determinística**. No se fuerza a la IA a *adivinar* la respuesta del texto crudo: se la empodera a **construir la herramienta que puede responder**. Se le dan privilegios de DBA + skills de data engineer: analiza el caos crudo, **diseña un schema Oracle 23ai compliant, crea la tabla con DDL, carga la data con DML, y corre una SQL estándar** para la respuesta. **Combina la flexibilidad infinita del lenguaje natural (Phase 3) con la precisión matemática del relacional (Phase 1).** El schema deja de ser estático → **smart agent fluido que evoluciona en tiempo real**.

> [!note] El gap fundamental del RAG tradicional
> El RAG previo era un **pipeline unidireccional** (data estática → embedding → vector store → retrieval), que **asume que la data necesaria ya existe estructurada/indexable**. Pero a menudo la respuesta vive en data cruda **nunca modelada**: *"analizá las tendencias de downtime de los logs de la última semana"* falla si los logs son archivos de texto desordenados, no filas. Hay que **invertir la relación tradicional entre aplicación y base.**

## Arquitectura del Autonomous Database Architect

La IA ya no se limita a *leer* data: se vuelve capaz de **diseñar las estructuras** para almacenarla, organizarla y consultarla → de **RAG probabilístico** (adivina del texto crudo) a **RAG determinístico** (construye el framework relacional para *computar* respuestas con precisión matemática). Tres componentes cooperando, en tres zonas operacionales (design / construction / governance plane):

![[12-fig-12.2.png]]
*Figure 12.2: The workflow of the self-evolving system*

### Architect agent (el diseñador) — el cerebro cognitivo

Implementa el loop **Perceive-Plan-Act-Audit**: percibe la intención del usuario, planifica un schema para los hechos necesarios, actúa para crear y poblar el schema, audita sus propias acciones. Traduce una **intención humana vaga en una especificación técnica precisa** — a diferencia de un chatbot que crea texto, **este agente crea estructura**. Recibe un goal (*"track project risks"*) + sample de data cruda, hace **análisis semántico** para identificar tipos de datos (severity → string, date → date, impact → numérico) **derivándolos del contexto, no adivinando**. Su output **no es una respuesta conversacional sino un JSON válido con un `CREATE TABLE`** que se ejecuta directo contra el kernel → la app **modifica su propia estructura de memoria persistente en tiempo real**, curando su "data blindness".

### ETL Worker agent (el constructor) — las manos

Una vez existe el schema, extrae/transforma/carga. Opera vía **schema reflection**: **antes** de procesar una línea, **consulta el data dictionary (`user_tab_columns`)** para inspeccionar la tabla creada por el Architect, verificando nombres y tipos exactos → **previene el error de alucinación** donde la IA inserta en una columna que nunca se creó. Luego itera los chunks de texto crudo, extrae las entidades, las mapea a las columnas verificadas, genera `INSERT`s y los ejecuta → convierte el **significado efímero del texto en registros relacionales persistentes** consultables determinísticamente.

### Governance layer (la red de seguridad)

El componente más crítico para adopción enterprise: dejar a una IA crear/modificar tablas sin control es un riesgo significativo. Se embebe un **audit trail mandatorio** en la arquitectura vía la tabla **`AGENT_SCHEMA_REGISTRY`**: cada vez que el Architect diseña una tabla, **debe simultáneamente insertar un registro** con timestamp, nombre de la tabla, el DDL exacto, y **crucialmente el user goal que justificó la acción**. El sistema **nunca es una black box** — un DBA puede consultar el registry para entender qué construyeron los agentes y por qué. **Separación de duties**: el agente construye tablas de trabajo efímeras, el sistema mantiene un audit log persistente → experimentación segura (se borran las tablas temporales tras el análisis, se mantienen los records de governance para siempre).

## Implementación

### El Architect — `call_genai_json` + `autonomous_data_architect`

Helper que **fuerza output JSON** (`response_format={"type": "json_object"}`) para que el LLM "hable el idioma de las máquinas", con `temperature=0.1` (minimiza creatividad, maximiza adherencia determinística a la sintaxis):

```python
def call_genai_json(system_prompt, user_prompt, client, model="gpt-5.2"):
    """Enforces JSON output — essential for machine-readable SQL structures."""
    response = client.chat.completions.create(
        model=model,
        messages=[{"role": "system", "content": system_prompt},
                  {"role": "user", "content": user_prompt}],
        response_format={"type": "json_object"},   # Force JSON mode
        temperature=0.1
    )
    return response.choices[0].message.content.strip()
```

La función Architect: **system prompt** que define la persona (Oracle 23ai Database Architect) y constriñe el output a un JSON con `table_name` (debe empezar con `AI_`) y `ddl_statement` (tipos Oracle: `VARCHAR2`, `CLOB`); **user prompt** con el goal + los primeros 1000 chars de data cruda:

```python
def autonomous_data_architect(goal, raw_text_sample, client):
    # --- Phase 1: Design (Generative) ---
    system_prompt = """
    You are an Oracle 23ai Database Architect.
    Task: Create a SQL table schema (DDL) to structure the provided raw text based on the user's goal.
    Output Format: JSON only.
    {
        "table_name": "AI_<meaningful_name>",
        "ddl_statement": "CREATE TABLE AI_... (...)"
    }
    """
    user_prompt = f"GOAL: {goal}\nSAMPLE RAW DATA:\n{raw_text_sample[:1000]}\nGenerate the JSON DDL now."
    response_json = call_genai_json(system_prompt, user_prompt, client)
    design = json.loads(response_json)
    table_name = design['table_name']
    ddl_script = design['ddl_statement']
```

**Phase 2 — Build (acción)**: con la conexión activa, dropea la tabla si existe (idempotencia del demo) y ejecuta el DDL generado por la IA. **Phase 3 — Audit**: inserta en `agent_schema_registry`, con **self-healing** — si el registry no existe (`ORA-00942`), lo crea on-the-fly antes de loguear:

```python
    # --- Phase 2: Build ---
    with connection.cursor() as cursor:
        try:
            cursor.execute(f"DROP TABLE {table_name}")   # cleanup (idempotency)
        except: pass
        cursor.execute(ddl_script)   # Execute the AI's DDL
        # --- Phase 3: Audit (Governance Log) ---
        try:
            cursor.execute("""
                INSERT INTO agent_schema_registry (user_goal, target_table_name, ddl_script)
                VALUES (:goal, :tbl, :ddl)
            """, [goal, table_name, ddl_script])
        except Exception as registry_error:
            if "ORA-00942" in str(registry_error):   # Self-healing: create registry if missing
                # ... [create AGENT_SCHEMA_REGISTRY] ... retry insert
```

### El ETL Worker — `autonomous_etl_worker`

La innovación crítica es **schema reflection**: el agente no asume la estructura, **consulta `user_tab_columns`** para los nombres/tipos exactos → lo *aterriza en la realidad* y previene alucinaciones:

```python
def autonomous_etl_worker(table_name, raw_data_list, client):
    with connection.cursor() as cursor:
        # SUB-STEP 1: Schema Reflection (the "Eyes")
        cursor.execute("""
            SELECT column_name, data_type FROM user_tab_columns WHERE table_name = :t
        """, [table_name.upper()])
        columns = cursor.fetchall()
        schema_desc = ", ".join([f"{c[0]} ({c[1]})" for c in columns])

        # SUB-STEP 2: Prompt (Oracle Data Engineer, ONLY use listed columns)
        system_prompt = f"""
        You are an Oracle 23ai Data Engineer.
        Task: Convert the provided Unstructured Data into a SQL INSERT statement for '{table_name}'.
        CRITICAL: You must ONLY use the columns listed in the target schema below. Do not invent columns.
        TARGET SCHEMA: {table_name} ({schema_desc})
        Output: JSON only {{ "sql": "INSERT INTO {table_name} ..." }}
        """
        # SUB-STEP 3: Loop and Load
        for i, text_chunk in enumerate(raw_data_list):
            response_json = call_genai_json(system_prompt, f"DATA SEGMENT #{i+1}: {text_chunk}", client)
            insert_sql = json.loads(response_json)['sql']
            cursor.execute(insert_sql)
        connection.commit()
```

### Scenario A — IT operations (logs de incidentes)

Lista de `raw_server_logs` desordenados + goal *"Analyze downtime duration and severity..."*. Se dispara el Architect (`autonomous_data_architect(goal, raw_server_logs[0], client)`) → propone **`AI_INCIDENT_DOWNTIME`**, ejecuta el DDL, loguea. Luego el ETL Worker detecta el schema (`INCIDENT_ID NUMBER`, `DOWNTIME_MINUTES NUMBER`, `SEVERITY VARCHAR2`) y convierte cada segmento en una fila. La verificación `SELECT * FROM AI_INCIDENT_DOWNTIME` prueba que ahora se puede hacer **análisis determinístico** sobre lo que antes era solo un string — *technical chaos → structured order*.

> [!tip] Anatomía de un prompt DDL — el LLM como fuzzy-logic compiler
> El éxito **no depende del Python sino del prompt engineering** del system prompt (generar SQL válido es notoriamente difícil; un LLM tiende a ser *chatty*). Cuatro constraints: **(1) Persona ("who")** — *"You are an Oracle 23ai Database Architect"* biasea el latent space hacia sintaxis Oracle (`VARCHAR2`, `CLOB`, `SYSDATE`) en vez de SQL genérico (`TEXT`, `DATETIME`) → evita `ORA-00902`. **(2) Format ("how")** — *JSON mode enforcement*: de "creative writing" a "data serialization"; parseable con `json.loads()`, el preámbulo conversacional lanza excepción **antes** de tocar la base. **(3) Type inference ("reasoning")** — *"Incident #101"* parece número pero tiene `#` → es identificador, no cantidad → `VARCHAR2(50)` (evita el integer overflow con seriales `SN-998`). **(4) Safety ("what not to do")** — no usar reserved words como columnas, no crear FKs a tablas no provistas. *Probabilistic logic constrained by deterministic rules.*

### Scenario B — Supply chain crisis (extensión teórica / pseudocódigo)

Un SME de SCM dice: *"mi problema no es server uptime — tengo 5000 emails urgentes de port authorities/customs/transport, la data que necesito está atrapada en emails mientras mi ERP muestra todo verde porque la data estructurada no se actualizó manualmente"*. Se ingiere un **email chain crudo** sobre un bloqueo en el Puerto de Rotterdam (timestamps, urgency flags, container IDs, vessels, narrativas) con goal *"Identify all blocked containers, status, reasons, priority for air freight needs"*.

El Architect identifica que `CNT-77281` es un primary-key-equivalent, que `Status` necesita un string categórico (no booleano: `BLOCKED`/`AWAITING_TRANSPORT`/`STUCK_ON_BOARD`), y muestra **enterprise awareness** (infiere que los Container IDs corresponden a una master table `SHIPMENTS` y en producción crearía un FK). Su JSON PLAN incluye un `reasoning_trace` y diseña **`AI_SCM_CRISIS_LOG`** eligiendo `CLOB` para `DELAY_REASON` (texto variable) y `VARCHAR2(50)` para `CONTAINER_ID` (formato ISO). El ETL Worker hace schema reflection y genera los `INSERT`s. **Business result**: el SME ignora los emails y corre una query determinística:

```sql
SELECT container_id, delay_reason, estimated_impact
FROM ai_scm_crisis_log
WHERE priority_level = 'High'
AND status IN ('BLOCKED', 'STUCK_ON_BOARD');
```

→ una narrativa free-text convertida en filas consultables que **disparan un alert high-priority** (en producción, un PL/SQL stored procedure que reserva air freight automáticamente). **La lección**: primero *ancla tus afirmaciones en un POC de agente codeado y funcionando* (`The_Autonomous_Database_Architect.ipynb`), después extrapolá blueprints a cualquier dominio.

## Las tres leyes de la seguridad autónoma

Que una IA modifique el schema estructural elicita dos reacciones: el dev ve posibilidad infinita, el DBA ve **riesgo catastrófico** (¿qué evita que sobrescriba la master `SHIPMENTS`? ¿que la base se vuelva un cementerio de tablas temporales abandonadas?). Para pasar de notebook a producción, un framework de governance rígido — **hard architectural constraints**, no guidelines:

1. **Schema isolation (the sandbox)** — el agente **nunca opera en el schema primario del core**. Requiere el privilegio poderoso de crear/dropear tablas → dárselo al app user viola *least privilege*. Solución: un **sandbox schema** (usuario distinto, quotas en un tablespace dedicado, permiso de crear **solo en su namespace**, explícitamente negado alterar/dropear el core). Aunque el LLM alucine un comando destructivo, el **blast radius queda contenido** — lo peor es que corrompe su propia memoria temporal.
2. **Forensic auditability (el "why" detrás del "what")** — cada creación debe acompañarse de un **registro de justificación**. El auditing estándar dice *quién* (el agente) y *qué* (`create table X`) pero **no el porqué** (¿`AI_ANALYSIS_T44` fue para analizar ventas o una alucinación de un prompt injection malicioso?). Se impone **registración mandatoria transaccional** linkeando el comando al user goal → de black box a **engine transparente** donde cada cambio es atribuible a una intención humana.
3. **Lifecycle reversibility (the clean slate)** — las estructuras autónomas son **efímeras por default, persistentes solo por excepción**. El riesgo es el **schema drift**: sin lifecycle management la base acumula miles de tablas temporales (*data detritus* que degrada performance, consume storage, confunde a los humanos). Las tablas del Architect son **disposable memory** ligada a la task/session → al terminar el análisis, la tabla física se marca para borrado. Pero **la data es disposable, el knowledge se preserva**: se borra la tabla, se retiene el entry en el Audit Registry → base lean y high-performance + record histórico permanente.

## Day-2 operations: self-healing schemas (conceptual / pseudocódigo)

Day-1 (de cero a uno) está demostrado; **Day-2 es donde se pelea la batalla**, definido por la **entropía**: en cuanto se define un schema, la realidad empieza a *driftear*. Ej.: el lunes `AI_INCIDENT_DOWNTIME` tiene 3 columnas; el martes DevOps deploya un update y los logs emiten un campo nuevo `ERROR_CODE_HEX` (`0x404`). En un ETL tradicional esto es un **fallo catastrófico** (el `INSERT` hard-coded falla por mismatch de columnas, crash a las 3 AM, ticket, `ALTER TABLE` manual). En la arquitectura autónoma: **reactive schema evolution**.

![[12-fig-12.3.png]]
*Figure 12.3: The reactive-schema-evolution cycle handling data drift*

> [!note] El error de base no es señal de stop, es un prompt nuevo
> Cuando el ETL Worker inserta los logs del martes en la tabla del lunes, Oracle devuelve `ORA-00913: too many values` o `ORA-00904: invalid identifier`. En vez de crashear, el agente **catchea la excepción** y entra a un subrutina self-healing: **(1) Catch** (intercepta el ORA error) → **(2) Analyze** (compara la data nueva contra el schema existente vía schema reflection) → **(3) Plan** (identifica el delta: *"la data trae 'ERROR_CODE_HEX' pero la tabla no tiene esa columna"*) → **(4) Evolve** (`ALTER TABLE AI_INCIDENT_DOWNTIME ADD (ERROR_CODE_HEX VARCHAR2(20))`) → **(5) Retry** (re-corre el `INSERT`, que ahora funciona). La base deja de ser un container estático que se rompe → **organismo vivo que se adapta**.

**Límite de la evolución (el immutable core)**: el schema fluido aplica **solo al *edge*** (tablas de análisis temporales de los agentes), **nunca al *core*** (master `CUSTOMERS`/`FINANCE`). Esto crea una **arquitectura IT bimodal** dentro de Oracle: **The Core** (gestionado por humanos, altamente gobernado, lento, optimizado para estabilidad) + **The Edge** (gestionado por IA, altamente adaptativo, cambia al instante, optimizado para agilidad). Aislar los agentes a su sandbox (Law 1) permite la evolución rápida sin desestabilizar los systems of record.

## Citas

> "Our agents could observe the state of the world, but they could not change it."
> "The agent can read, but it cannot count. It provides a probabilistic guess rather than a deterministic fact."
> "We give the agent the privileges of a database administrator and the skills of a data engineer."
> "We are using the LLM not as a writer, but as a fuzzy-logic compiler."
> "We have not just built a tool; we have defined a new category of software: the self-evolving database."

## Para aplicar

- **Pasar de retrieval probabilístico a ejecución determinística** — cuando la respuesta exige agregación/conteo exacto sobre data no modelada, no hacer que el LLM adivine: que **construya la tabla** (DDL) y corra SQL estándar.
- **Implementar el loop Perceive-Plan-Act-Audit** — percibir la intención, planificar el schema, actuar (crear+poblar), auditar; el agente como data engineer activo, no retriever pasivo.
- **Forzar JSON mode + temperature baja (0.1) para generar SQL** — `response_format={"type": "json_object"}` convierte "creative writing" en "data serialization" parseable; el preámbulo conversacional falla antes de tocar la base.
- **Usar schema reflection antes de insertar** — consultar `user_tab_columns` para los nombres/tipos reales; instruir *"ONLY use these columns, do not invent"* → previene alucinaciones de columnas inexistentes.
- **Diseñar el prompt DDL con las 4 constraints** — persona (sintaxis del vendor correcto), JSON format, type inference semántico (`#`→identificador→`VARCHAR2`), safety (no reserved words, no FKs a tablas no provistas).
- **Embeber un audit trail mandatorio (`AGENT_SCHEMA_REGISTRY`)** — loguear cada creación con su **user goal justificante**; el sistema nunca es black box; self-healing si el registry no existe.
- **Aplicar las 3 leyes de seguridad** — **isolation** (sandbox schema, least privilege, blast radius contenido), **auditability** (linkear comando ↔ intención humana, no solo quién/qué sino por qué), **reversibility** (tablas efímeras por default; borrar la data, retener el knowledge).
- **Separar core (humano, estable) de edge (IA, ágil)** — arquitectura bimodal; la evolución autónoma solo en el edge/tablas temporales, nunca en las master tables.
- **Tratar el error de base como un prompt nuevo (self-healing)** — catch `ORA-xxxxx` → analyze delta → `ALTER TABLE ADD` → retry; el schema se adapta al data drift en vez de crashear.
- **Anclar las afirmaciones en un POC codeado y funcionando** — primero el agente real (el notebook), después extrapolar blueprints conceptuales a cualquier dominio.

## Conexiones

- [[_RAG-Driven Generative AI - Second Edition|Mapa del libro]] — el MOC del libro.
- [[11 - Scaling AI Workloads with Oracle Exadata]] — cap. 11: la infraestructura/RAGOps que sostiene; el patrón idempotente y `user_tab_columns` reusados.
- [[09 - Building a Conversational RAG Agent]] · [[10 - Building an Agent with Spatial-RAG and GraphRAG]] — caps. 9-10: el DBA console / schema registry que aquí se vuelve **auto-gestionado por el agente**; `user_tab_columns` para reflection.
- [[01 - Why Retrieval-Augmented Generation]] — cap. 1: la distinción **parametric vs non-parametric**; aquí se cierra el arco — el agente crea estructura non-parametric determinística on-demand. Modular RAG → autonomous architecture.
- [[08 - Boosting RAG Performance with Human Feedback]] — cap. 8: el human-in-the-loop / governance reaparece como las 3 leyes; el RAG no resuelve todo (no puede contar) → ejecución determinística.
- [[06 - Operationalizing the Universal Context Engine]] — cap. 6: el business rules engine / governance y la auditabilidad (glass-box) reaparecen como audit trail.
- [[_RAG|RAG]] · [[Enterprise RAG Assistant]] · [[Hybrid Search]] — el patrón RAG del vault; la frontera: de RAG a ejecución determinística.
- [[Function Calling]] · [[Tool Calling]] · [[Orchestrator]] — el agente que ejecuta DDL/DML como tools; Architect+ETL Worker orquestados.
- [[Grounding]] · [[Hallucinations]] — schema reflection como grounding (previene columnas alucinadas); JSON mode + persona constrain la probabilidad; las 3 leyes como guardrails.
- **Autonomous Database Architect · Self-evolving database · Perceive-Plan-Act-Audit · Deterministic vs probabilistic RAG · DDL/DML autonomy · `AGENT_SCHEMA_REGISTRY` / governance layer · Schema reflection (`user_tab_columns`) · JSON mode enforcement · Type inference · Fuzzy-logic compiler · Three laws (isolation/auditability/reversibility) · Sandbox schema · Reactive schema evolution / self-healing · Schema drift · Bimodal IT (core/edge) · `ALTER TABLE` / `ORA-00913` / `ORA-00904` / `ORA-00942`** — conceptos/técnicas del capítulo; candidatos a nota propia (varios ya existen en el vault, otros son semillas).
