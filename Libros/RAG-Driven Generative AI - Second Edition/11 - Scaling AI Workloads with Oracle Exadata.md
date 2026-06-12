---
title: 11 - Scaling AI Workloads with Oracle Exadata
libro: RAG-Driven Generative AI - Second Edition
autor: Denis Rothman
capitulo: 11
created: 2026-06-11
tags:
  - libros/rag-driven-generative-ai
  - type/case-study
  - status/permanent
aliases:
  - Scaling AI Workloads with Oracle Exadata
  - RAGOps / Exadata Scaling
updated: 2026-06-11
---

# 11 - Scaling AI Workloads with Oracle Exadata

> [!info] Capítulo 11 · *RAG-Driven Generative AI - Second Edition* — Denis Rothman
> De diseño de aplicación a **production engineering / RAGOps**: el RAG enterprise depende de la **velocidad de retrieval**. Con **OCI + Oracle Exadata** (AI Smart Scan offloadea la matemática de vectores a las storage cells, eliminando el *data movement tax*) se mantiene sub-segundo a escala. Journey en 4 etapas: ingesta de 50k filas (idempotente, `executemany`), baseline (stress test CPU), scaling elástico (OCI Python SDK, OCPUs), verificación (1.8x más rápido). Cierra probando que la carga CPU es la misma para data random que meaningful. Navegá: [[_RAG-Driven Generative AI - Second Edition|Mapa del libro]] · anterior [[10 - Building an Agent with Spatial-RAG and GraphRAG]] · siguiente [[12 - The Autonomous Database Architect]].

## Resumen

Hasta ahora el foco fue la **capa cognitiva** (prompts, agents, MAS). Pero un sistema vale tanto como el **ecosistema donde corre**. Este capítulo transiciona de application design a **production engineering y RAGOps**, enfocándose en la **capa de infraestructura** que hace viable la GenAI en un entorno enterprise mission-critical. El desafío del RAG enterprise es simple pero formidable: **depende altamente de la velocidad de retrieval**. Al crecer la data de miles a **millones de vectores high-dimensional**, una base estándar se vuelve un cuello de botella — si el retrieval tarda 5 segundos, el usuario espera 5 segundos por más potente que sea el LLM.

La solución: la **sinergia entre Oracle Cloud Infrastructure (OCI) y la arquitectura Oracle Exadata**. Una converged database sobre Exadata **no es mero storage sino un motor de alta performance**: con **AI Smart Scan** se **offloadea la carga matemática pesada (cálculos de vector distance) directo a los storage servers**, eliminando el **data movement tax** que frena las arquitecturas tradicionales. Los storage servers tienen su propio CPU y **NVMe flash** para procesar la matemática **donde viven los datos**, devolviendo solo los resultados finales a los nodos de la base → retrieval instantáneo de miles a millones de embeddings. (Los benchmarks hands-on corren en **Autonomous Database Free Tier**, pero los patrones de scaling y comportamientos de performance son la base de producción donde Exadata está disponible.)

El journey en **4 etapas (de prototipo a producción)**: (1) **Ingestion** — poblar la tabla benchmark con **50.000 filas sintéticas** vía bulk inserts idempotentes (`executemany`, alto throughput); (2) **Baselining** — un stress test CPU-intensive (full table scan que combina vector math + agregaciones SQL relacionales) para medir la duración inicial; (3) **Scaling** — el **OCI Python SDK** para ajustar programáticamente los recursos de compute (OCPUs) en tiempo real; (4) **Verification** — re-correr el benchmark sobre la infra escalada y **probar empíricamente que más OCPUs reduce la latencia** (1.8x más rápido). Se cierra con un principio contraintuitivo clave: **la CPU consume el mismo tiempo/energía calculando vector distances sin importar el significado semántico de la data** — un vector es solo un array de floats; esto valida usar **data sintética para predecir cargas reales** sin ingerir millones de documentos propietarios.

![[11-fig-11.1.png]]
*Figure 11.1: The architecture of Oracle Cloud and Exadata*

## Arquitectura de AI escalable con Oracle Cloud y Exadata

Se "construyó el auto" (la lógica AI); ahora hay que **ingenierizar la autopista**. Para que un agente siga viable en producción debe mantener **retrieval sub-segundo** aun buscando en millones de vectores → foco en la capa de infraestructura: **OCI (la autopista/substrato de ejecución) + Exadata (el motor de alta performance)**.

### El substrato de ejecución: OCI

Oracle Cloud Infrastructure provee el substrato fundacional, optimizado para apps enterprise data-intensive vía **bare metal compute de alta performance** y un **network fabric de baja latencia**. Es el espacio global donde residen la base, los modelos AI y la lógica de orquestación; provee el **isolation y la seguridad** para que la data soberana esté protegida mientras la procesan los LLMs.

### El motor de base: Oracle Exadata

Si el cloud es la autopista, **Exadata es el motor de alta performance tuneado para la base**. Es un **engineered system**: integración vertical de compute, storage y networking + software especializado diseñado para **eliminar la fase de movimiento de datos**. Las arquitecturas tradicionales se frenan porque **deben traer masas de vector data del storage a memoria** para calcular.

> [!note] AI Smart Scan — traer la matemática a los datos
> Exadata resuelve esto con **AI Smart Scan**: en vez de mover data, **shipea el procesamiento SQL (incluidos los cálculos de vector distance) directo a las storage cells**. Los storage servers tienen su propio **CPU y NVMe flash** para procesar la matemática **donde vive la data**, procesando millones de vectores en paralelo y devolviendo solo los resultados finales. Así el retrieval se mantiene instantáneo al crecer el dataset. (**NVMe flash** = SSD de muy baja latencia bajo el estándar NVMe; no es tecnología Oracle-only, es el uso que Oracle hace de NVMe, ej. Smart Flash Cache.)

### Journey de prototipo a producción

Se arranca con el **Autonomous Database Free Tier** (entorno pre-configurado con guardrails; converged database = vectores + SQL relacional clásico sin tuning manual). Las 4 etapas: **Ingestion** (50k filas sintéticas, bulk idempotente), **Baselining** (full table scan CPU-intensive: vector math + agregaciones SQL), **Scaling** (OCI Python SDK pide más compute), **Verification** (re-correr el benchmark y probar que más potencia = menos latencia — patrón que se extiende a producción Exadata). De *building logic* a gestionar **RAGOps** → infraestructura que nunca tiene un *traffic jam*.

## Extender el DBA console para AI scaling

Se construye sobre el control plane unificado de los caps. 9-10 (sin scripts aislados). Pocos updates: registrar las tablas benchmark sintéticas en el registry externo, registrar el scope, ejecutar el create.

**Schema registry** — se agrega `CHAPTER_11` al `DDL_CATALOG` con la tabla `WORKLOAD_BENCHMARK`:

```python
"CHAPTER_11": [
    {
        "table_name": "WORKLOAD_BENCHMARK",
        "description": "Stores synthetic benchmark data for scaling performance tests.",
        "sql": """
        CREATE TABLE WORKLOAD_BENCHMARK (
            session_id VARCHAR2(50),
            complexity_score NUMBER,
            operation_vector VECTOR(3, FLOAT32),
            payload_data CLOB
        )
        """
    }
]
```

> [!tip] Por qué `VECTOR(3, FLOAT32)` (low-dimensional)
> Se usa intencionalmente un vector **3D** para ejecutar millones de operaciones matemáticas rápido y **estresar la CPU minimizando el storage overhead**. Como la matemática de vector distance **escala linealmente con la dimensionalidad ($O(d)$)**, las curvas de performance y patrones de scaling son **arquitectónicamente idénticos a producción 1536D** (la dimensionalidad estándar de embeddings enterprise como `text-embedding-3-small`); **solo cambia el tiempo absoluto de ejecución**.

**`TABLE_REGISTRY`** y **dropdown UI** — se agrega `'CHAPTER_11': ['WORKLOAD_BENCHMARK']` (context-aware: VERIFY/DROP solo esta tabla) y `'CHAPTER_11'` a las options del `scope_dropdown` (default). **Crear + verificar** vía la consola (recarga `create_script.py`, idempotente): output `✅ Created` luego `WORKLOAD_BENCHMARK | Rows: 0 | ✅ [EMPTY]`. *Sin SQL ad-hoc.*

## Scaling the workflow (RAGOps)

Del control plane a la capa de ejecución. **RAGOps** = production engineering para hacer viable la GenAI en el mundo real: gestionar retrieval speed, manejar escala masiva, arquitecturas cost-efficient apalancando la **elasticidad** del Autonomous Database en OCI.

### Setup — el OCI SDK y el circuit breaker

Además del `oracledb` se instala el **`oci` Python SDK** (controla programáticamente la infra cloud, escalando CPU sin intervención manual en la web console):

```python
!pip install oci   # -> Successfully installed circuitbreaker-2.1.3 oci-2.165.0
```

![[11-fig-11.5.png]]
*Figure 11.5: The interaction between the Python environment and the Oracle Cloud control plane*

Dos componentes clave: **OCI SDK** (un "traductor profesional" — el código habla Python, el cloud habla web requests complejos; el SDK permite comandos simples para operaciones pesadas como aumentar los **OCPUs**, manejando auth, formatting y el handshake seguro). **`circuitbreaker`** (instalado como dependencia): como el de tu casa — si un cloud service tiene alta latencia o es inalcanzable, la app podría desperdiciar recursos reintentando hasta crashear; el circuit breaker **monitorea las calls, trip-ea el circuito si detecta demasiados fallos** y frena los requests un rato → un hiccup de red no se vuelve un fallo total. **Automatizado + resiliente.**

### Etapa 1 — ingesta idempotente (50k filas)

50.000 filas sintéticas, cada una con complexity score + vector 3D. **Idempotente**: primero chequea si la tabla ya está poblada para evitar trabajo redundante:

```python
def ingest_benchmark_data(cursor, num_rows=50000):
    # 1. Idempotency Check
    cursor.execute("SELECT count(*) FROM WORKLOAD_BENCHMARK")
    count = cursor.fetchone()[0]
    if count > 0:
        print(f"✅ Table already contains {count} rows. Skipping ingestion.")
        return

    # Generate Synthetic Data
    data_batch = []
    for i in range(num_rows):
        complexity = random.randint(1, 100)
        session_id = f"SES_{random.randint(1000, 9999)}"
        payload = f"Simulated payload data for transaction {i}" * 5
        vec = array.array('f', [random.random(), random.random(), random.random()])
        data_batch.append((session_id, complexity, vec, payload))

    # 2. Bulk Insert (single round-trip via executemany)
    cursor.executemany("""
        INSERT INTO WORKLOAD_BENCHMARK (session_id, complexity_score, operation_vector, payload_data)
        VALUES (:1, :2, :3, :4)
    """, data_batch)
    connection.commit()

ingest_benchmark_data(cursor)
```

Cada tupla mapea a las 4 columnas (session_id string, complexity int 1-100, operation_vector array 3D float, payload_data string repetido). **`executemany`** streamea el batch entero en **un solo round-trip** (vs uno por uno, ineficiente). Output: **`✅ Successfully inserted 50000 rows in 2.12 seconds`** — de 0 a 50k en ~2s demuestra el alto throughput del Autonomous Database.

### Etapa 2 — baseline (stress test CPU)

`run_stress_test()` fuerza la CPU al límite: no un lookup simple sino una operación CPU-intensive que combina vector math + agregación aritmética sobre 50k filas, **sin índice (full table scan forzado)** para utilizar cada core OCPU al máximo:

```python
def run_stress_test(label):
    # CPU-intensive: VECTOR_DISTANCE per row + arithmetic aggregation, NO index (full scan)
    sql = """
    SELECT
        AVG(complexity_score),
        SUM(VECTOR_DISTANCE(operation_vector, :target_vec, COSINE))
    FROM WORKLOAD_BENCHMARK
    """
    target_vec = array.array('f', [0.5, 0.5, 0.5])
    start_time = time.time()
    cursor.execute(sql, [target_vec])
    result = cursor.fetchone()
    duration = time.time() - start_time
    print(f"   ⏱️ Duration: {duration:.4f} seconds")
    return duration

baseline_time = run_stress_test("Baseline (Before Scaling)")
```

Output baseline: `Results processed: (50.5098, 5352.49...)`, **`Duration: 0.2115 seconds`** — el punto de partida.

### Etapa 3 — scaling elástico (OCI SDK)

El verdadero poder del cloud es la **elasticidad**: el OCI SDK escala dinámicamente el compute (subir potencia en peak, bajar para ahorrar costos en idle). Lógica robusta con **simulation fallback** (para free tier / local):

```python
import oci
from google.colab import files
simulate_scaling = True
try:
    uploaded = files.upload()   # upload 'config' + .pem private key for real scaling
    if 'config' in uploaded:
        simulate_scaling = False
except:
    print("⚠️ No config uploaded. Proceeding in Simulation Mode.")

def scale_cpu(target_cpu_count):
    if simulate_scaling:
        print(f"[SIMULATION] Scaling database to {target_cpu_count} OCPUs...")
        time.sleep(2)
        return
    # --- REAL OCI SCALING ---
    config = oci.config.from_file(file_location="./config")
    adb_client = oci.database.DatabaseClient(config)
    adb_id = "ocid1.autonomousdatabase.oc1..."   # your OCID
    update_details = oci.database.models.UpdateAutonomousDatabaseDetails(cpu_core_count=target_cpu_count)
    adb_client.update_autonomous_database(adb_id, update_details)
    oci.wait_until(adb_client, adb_client.get_autonomous_database(adb_id), 'lifecycle_state', 'AVAILABLE')
    print("✅ Database Scaled Successfully!")

scale_cpu(2)   # Scale up to 2 OCPUs
```

Real o simulado, el workflow es idéntico: **definir target state → aplicar el cambio vía API → esperar a que la infra estabilice** (`lifecycle_state = AVAILABLE`). (El `google.colab` es específico de Colab para config/wallet uploads; en otros entornos sustituir por el file management correspondiente.)

### Etapa 4 — verificación (1.8x más rápido)

Se re-corre el mismo stress test sobre la infra escalada. En Exadata real la base distribuiría la vector math across los nuevos cores. En simulación se reduce el tiempo artificialmente para demostrar el concepto:

```python
scaled_time = run_stress_test("Scaled (2 OCPU)")
if simulate_scaling:
    scaled_time = baseline_time * 0.55
print(f"Baseline Time: {baseline_time:.4f}s")
print(f"Scaled Time:   {scaled_time:.4f}s")
print(f"Improvement:   {baseline_time / scaled_time:.1f}x faster")
```

Output: **Baseline 0.2115s → Scaled 0.1163s → Improvement 1.8x faster**. Al crecer la data RAG, se mantiene retrieval sub-segundo **simplemente ajustando el dial de la infraestructura** (en producción, acelerable aún más por Exadata).

## Carga CPU: data random vs meaningful

![[11-fig-11.7.png]]
*Figure 11.7: The mathematical equivalence of meaningful and random noise contexts*

**Concepto core (contraintuitivo)**: la CPU consume **el mismo tiempo/energía** calculando vector distances **sin importar el significado semántico** de la data. Para un humano, un vector de un soneto de Shakespeare se siente "más pesado" que ruido aleatorio; **para la CPU, ambos son arrays idénticos de floats**. Un vector 3D siempre son 3 valores float; calcular la distancia entre dos requiere un **número fijo de FLOPS (floating-point operations)**.

Prueba empírica con dos casos (Case A meaningful, Case B random, ambos `np.float32` → mismo memory footprint), corriendo el **dot product** (núcleo de cosine similarity) **1 millón de veces** cada uno:

```python
import numpy as np, time
def prove_cpu_equivalence(iterations=1000000):
    vec_a_meaningful = np.array([0.123, 0.456, 0.789], dtype=np.float32)
    vec_b_meaningful = np.array([0.321, 0.654, 0.987], dtype=np.float32)
    vec_a_random = np.array([0.1, 0.2, 0.3], dtype=np.float32)
    vec_b_random = np.array([0.9, 0.8, 0.7], dtype=np.float32)
    def run_math_load(v1, v2):
        start = time.time()
        for _ in range(iterations):
            np.dot(v1, v2)
        return time.time() - start
    time_meaningful = run_math_load(vec_a_meaningful, vec_b_meaningful)
    time_random = run_math_load(vec_a_random, vec_b_random)
```

Output: `Time for 'Meaningful' Data: 3.8347s`, `Time for Random Data: 1.2422s`, `Difference: 2.5925s (Negligible)`, `✅ Conclusion: The CPU workload is identical.`

> [!note] La diferencia es un artefacto del entorno, no de complejidad matemática
> La naturaleza del procesador es **determinista**: para un vector 3D el dot product **siempre son 3 multiplicaciones y 2 sumas**; los hardware gates no diferencian un número que representa una palabra de uno aleatorio (ambos = patrones de 32 bits idénticos en los registros). La variación de 2.59s es **overhead del primer loop high-iteration** (memory allocation / library caching), no mayor complejidad. Como el total de FLOPS es idéntico, el workload es **científicamente equivalente**.

Esto valida usar **data sintética para predecir cargas enterprise reales** sin ingerir millones de documentos propietarios — *asegurar que la autopista está lista antes de que el primer auto entre*.

## Citas

> "If retrieval takes five seconds, users must wait five seconds or more for an answer, no matter how powerful the LLM may be."
> "Instead of moving data, it ships the SQL processing, including vector distance calculations, directly to the storage cells."
> "To a human, a vector representing a profound Shakespearean sonnet feels heavier or more significant than a vector of random noise. To a CPU, however, both are merely identical arrays of floating-point numbers."
> "we moved beyond building the car (the AI agent) and focused on engineering the highway (the database infrastructure)."

## Para aplicar

- **Tratar la velocidad de retrieval como requisito de producción (RAGOps)** — el RAG enterprise depende de retrieval sub-segundo a escala; la infraestructura es tan crítica como la lógica AI.
- **Apalancar AI Smart Scan / Exadata para evitar el data movement tax** — offloadear los cálculos de vector distance a las storage cells (CPU+NVMe donde vive la data); procesar millones de vectores en paralelo, devolver solo resultados.
- **Extender el DBA console para benchmarking** — agregar el scope al registry/`TABLE_REGISTRY`/dropdown, sin SQL ad-hoc; mantener todo version-controlled.
- **Usar vectores low-dimensional para benchmarks CPU** — `VECTOR(3, FLOAT32)` estresa la CPU minimizando storage; como la matemática es $O(d)$, los patrones de scaling son idénticos a 1536D (solo cambia el tiempo absoluto).
- **Ingerir en bulk idempotente con `executemany`** — chequear count > 0 antes de poblar; un solo round-trip para 50k filas (~2s) vs inserts uno por uno.
- **Forzar full table scan para el baseline** — sin índice, combinando `VECTOR_DISTANCE(COSINE)` + agregaciones, para utilizar cada OCPU al máximo y medir la duración real.
- **Escalar programáticamente con el OCI Python SDK** — modificar `cpu_core_count` (OCPUs) vía `UpdateAutonomousDatabaseDetails`, esperar `lifecycle_state = AVAILABLE`; subir en peak, bajar en idle (cost-efficient).
- **Incluir un simulation fallback y circuit breaker** — que el notebook funcione sin OCI config (free tier/local); el circuit breaker frena reintentos ante fallos cloud para no crashear la app.
- **Probar empíricamente que más OCPUs = menos latencia** — baseline vs scaled, reportar el speedup (1.8x); el dial de infraestructura mantiene el sub-segundo al crecer la data.
- **Usar data sintética para predecir cargas reales** — la CPU no distingue significado (mismos FLOPS para meaningful y random); stress-testear el scaling sin ingerir documentos propietarios.

## Conexiones

- [[_RAG-Driven Generative AI - Second Edition|Mapa del libro]] — el MOC del libro.
- [[10 - Building an Agent with Spatial-RAG and GraphRAG]] — cap. 10: extiende el mismo **DBA console / schema registry** (5 updates → CHAPTER_11); la converged DB (vector+spatial+graph) ahora se escala en compute.
- [[09 - Building a Conversational RAG Agent]] — cap. 9: el patrón idempotente (`user_tables`/count) y `executemany` reusados para la ingesta masiva.
- [[02 - RAG Embeddings in Oracle Vector Stores]] — cap. 2: `VECTOR`, `VECTOR_DISTANCE`, HNSW/IVF; aquí se mide la performance del kernel sobre ese tipo nativo.
- [[01 - Why Retrieval-Augmented Generation]] — cap. 1: el ecosistema RAG (D2/D3 processing/storage thresholds); la infraestructura que sostiene el retrieval.
- [[12 - The Autonomous Database Architect]] — cap. 12 (final): la frontera — un agente que diseña y construye sus propias tablas (ejecución determinística); el patrón idempotente y `user_tab_columns` reusados.
- [[_RAG|RAG]] · [[Enterprise RAG Assistant]] · [[Hybrid Search]] · [[Reranking]] · [[Chunking Strategies]] — el patrón RAG del vault; RAGOps como la capa de producción.
- [[_MLOps|MLOps]] — RAGOps como pariente de MLOps; benchmarking, elasticidad, cost-efficiency, infra version-controlled.
- [[Grounding]] · [[Hallucinations]] — sin relación directa, pero la velocidad de retrieval condiciona la UX del grounding.
- **RAGOps · Oracle Exadata · AI Smart Scan · Data movement tax · OCI (Oracle Cloud Infrastructure) · OCI Python SDK · OCPUs · Elasticity / programmatic scaling · NVMe flash · Engineered system · Autonomous Database Free Tier · `executemany` (bulk insert) · Full table scan · `VECTOR(3, FLOAT32)` · Dimensionality $O(d)$ · FLOPS · CPU equivalence · Circuit breaker · `cpu_core_count` / `UpdateAutonomousDatabaseDetails`** — conceptos/técnicas del capítulo; candidatos a nota propia (varios ya existen en el vault, otros son semillas).
