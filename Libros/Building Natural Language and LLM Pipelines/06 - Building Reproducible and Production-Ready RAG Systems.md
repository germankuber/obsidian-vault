---
title: 06 - Building Reproducible and Production-Ready RAG Systems
libro: Building Natural Language and LLM Pipelines
autor: deepset / Packt
capitulo: 6
created: 2026-06-12
tags:
  - libros/building-natural-language-and-llm-pipelines
  - type/case-study
  - status/permanent
aliases:
  - Building Reproducible and Production-Ready RAG Systems
  - Cap 6 - Production-Ready RAG
updated: 2026-06-12
---

# 06 - Building Reproducible and Production-Ready RAG Systems

> [!info] Capítulo 6 · *Building Natural Language and LLM Pipelines* — Packt (ISBN 9781835467992)
> El capítulo que completa la transición de **RAG developer a RAG architect**: pasar de notebooks a un **proyecto Python reproducible** estructurado por roles MLOps, blindar la arquitectura con dos decisiones clave (la **vector space singularity** y la **dual-Elasticsearch architecture**), evaluar cuantitativamente [[RAG]] naive vs hybrid con [[Ragas]] (4 métricas), agregar **observabilidad + FinOps** con [[Weights and Biases]] (`WeaveConnector` + `rag_analytics.py`) y resolver el trade-off cost-performance de [[Embeddings]] (`text-embedding-3-small` vs `large`). Navegá: [[_Building Natural Language and LLM Pipelines|Building NL & LLM Pipelines]] · anterior [[05 - Haystack Pipeline Development with Custom Components]] · siguiente [[07 - Deploying Haystack-Based Applications]].

## Resumen

Tras generar el dataset ground-truth en el [[05 - Haystack Pipeline Development with Custom Components|Cap 5]], este capítulo cierra la construcción de la tool layer transformando un RAG funcional en un **sistema reproducible y production-ready** — el salto de **RAG developer a RAG architect**. El arco tiene cinco bloques. Primero, **reproducibilidad**: salir de los notebooks exploratorios hacia un proyecto Python estructurado en `scripts/`, con el blueprint del [[SuperComponent]] (un `@super_component` que encapsula los dos pipelines de un Q&A system — indexing y retrieval/generation — detrás de una interfaz limpia con `input_mapping`/`output_mapping`), una estructura de carpetas mapeada a roles MLOps (NLP, QA/test, DevOps), y tooling de producción (**`uv`**, y de in-memory a **Elasticsearch persistente** vía `docker-compose.yml` con dual document store).

Segundo, las **justificaciones arquitectónicas — el "porqué"**: la **vector space singularity** (usar el mismo embedding model en indexing, synthetic data generation y querying, porque cada modelo crea su propio espacio vectorial y mezclarlos da resultados sin sentido — *"a map of Paris to navigate Tokyo"*) y el **decoupling de document stores** en una **dual-Elasticsearch architecture** que habilita A/B testing cost-performance e independent resource allocation. Tercero, la **evaluación sistemática con [[Ragas]]**: usar el golden test set del Cap 5 para comparar `NaiveRAGSuperComponent` (single-path dense) contra `HybridRAGSuperComponent` (sparse [[BM25]] + dense en paralelo + fusión + reranking [[cross-encoder]]) sobre 4 métricas (faithfulness, context precision, context recall, answer relevancy), con el Hybrid RAG ganando en todas (Tabla 6.1).

Cuarto, la **observabilidad production-grade con [[Weights and Biases]]**: distinguir evaluación (test estático one-off pre-deploy, [[Ragas]]) de observabilidad (monitoreo continuo en producción, W&B), integrar el `WeaveConnector` y, con `rag_analytics.py`, loguear token counts y costos para convertir W&B en un **[[FinOps]] dashboard** (Tabla 6.2). Quinto, el **trade-off cost-performance de embeddings**: `text-embedding-3-small` vs `text-embedding-3-large` en MTEB y en costo — un **6.5x cost increase** por solo ~2.3 puntos de benchmark (Tablas 6.3 y 6.4), con la conclusión de que el small es el default óptimo y el large una specialty tool para dominios high-stakes. El hilo: la arquitectura decoupled permite *elegir* dentro de un proyecto unificado.

## Setting up a reproducible RAG project

Un sistema de **question-answering (Q&A)** RAG se compone de **dos pipelines** distintos (Figura 6.1), un patrón conocido como **vanilla RAG** o **naive RAG**. El **indexing pipeline** prepara el conocimiento offline: data extraction → data wrangling → chunking → **embedding model** → vector storage. El **retrieval and generation pipeline** responde online: question input → query encoding (el mismo embedding model) → data retrieval → data ranking (si hay un [[cross-encoder]]) → processing of the final answer.

![[06-fig-6.1-qa-system-pipelines.png]]
*Figure 6.1 – The two pipelines of a Q&A system*

> [!note] **Vanilla / naive RAG.** La arquitectura base de dos pipelines (indexing + retrieval/generation) es el punto de partida. El capítulo la evoluciona hacia hybrid RAG y hacia un proyecto reproducible, pero el esqueleto conceptual son estos dos flujos: uno que ingiere y vectoriza el corpus, otro que recupera y genera.

El primer paso hacia la reproducibilidad es **abandonar los notebooks exploratorios** y migrar a un **proyecto Python estructurado** en `scripts/`. El corazón de ese proyecto es el blueprint del [[SuperComponent]]: encapsular el pipeline completo detrás de una interfaz limpia.

### El blueprint del SuperComponent

El decorador `@super_component` envuelve una clase cuyo `__init__` arma el pipeline interno y cuyo `_build_pipeline` lo construye en pasos:

```python
@super_component
class NaiveRAGSuperComponent:
    def __init__(self, document_store, generator):
        self.document_store = document_store
        self.generator = generator
        self.pipeline = self._build_pipeline()

    def _build_pipeline(self):
        pipe = Pipeline()
        pipe.add_component("text_embedder", text_embedder)
        pipe.add_component("retriever", retriever)
        pipe.add_component("prompt_builder", prompt_builder)
        pipe.add_component("llm", self.generator)
        # connect the five steps...
        return pipe
```

### Input / output mapping

La abstracción se completa con el mapeo de entradas y salidas: una sola `query` de usuario se rutea a los tres sockets internos que la necesitan, y las salidas internas se renombran a una interfaz pública limpia:

```python
input_mapping = {
    "query": ["text_embedder.text", "retriever.query", "prompt_builder.question"],
}
output_mapping = {
    "llm.replies": "replies",
    "retriever.documents": "documents",
}
```

> [!tip] **Ventajas del SuperComponent blueprint.** (1) **Interface abstraction** — quien usa el RAG ve un único `query` in / `replies`+`documents` out, sin conocer los cinco componentes internos. (2) **Easy substitution** — se puede cambiar el retriever, el generator o el embedder sin tocar la interfaz pública. (3) **Parallel processing** — distintos SuperComponents (naive, hybrid) corren la misma query en paralelo para comparar.

### Estructura de carpetas mapeada a roles MLOps

La organización de `scripts/` no es arbitraria: cada subcarpeta corresponde a un **rol de un equipo MLOps**, dejando claro quién es dueño de qué.

| Carpeta | Archivos clave | Rol MLOps |
|---|---|---|
| `scripts/rag/` | `indexing.py`, `naiverag.py`, `hybridrag.py` | NLP engineer |
| `scripts/synthetic_data_generation/` | `knowledge_graph_component.py`, `synthetic_test_components.py`, `SDGGenerator` / `synthetic_data_generator_supercomponent.py` | QA / test engineer |
| `scripts/ragas_evaluation/` | `ragasevalsupercomponent.py` (3 componentes: `CSVReaderComponent`, `RAGDataAugmenterComponent`, `RagasEvaluationComponent`; abstraíbles en `RAGEvaluationSuperComponent`) | QA / test engineer |
| `scripts/wandb_experiments/` | `rag_analytics.py` | DevOps / MLOps / LLMOps engineer |

> [!note] La estructura comunica el **ownership** del sistema: el NLP engineer construye los pipelines RAG, el QA/test engineer genera datos sintéticos y evalúa con Ragas, y el DevOps/MLOps/LLMOps engineer instrumenta la observabilidad y los costos. El código refleja el organigrama.

### Tooling de producción

- **`uv`** — el package/dependency manager moderno usado para gestionar el entorno reproducible del proyecto.
- **De in-memory a Elasticsearch persistente** — se reemplaza el `InMemoryDocumentStore` exploratorio por **Elasticsearch persistente** levantado vía `docker-compose.yml`, con un **dual document store** (dos clusters independientes, ver más abajo).
- **Requisitos** — **Python 3.12**, **Docker Desktop**, y una **API key gratuita de [[Weights and Biases]]**.

## Key architectural justifications: the "why"

Dos decisiones arquitectónicas explican por qué el sistema reproducible se construye como se construye. No son detalles de implementación: son lo que separa a un developer de un architect.

### The vector space singularity

> [!warning] **Regla del espacio vectorial — uno de los failure modes más comunes y críticos.** Hay que usar **exactamente el mismo embedding model** en las tres etapas: indexing, synthetic data generation y querying. Mezclar modelos rompe el sistema en silencio.

La razón es matemática: cada **embedding model** crea un **espacio vectorial único**. En ese espacio, el **query vector** funciona como una *brújula* y los **document vectors** como un *mapa*; ambos tienen que estar dibujados en la misma proyección para que la búsqueda de similitud signifique algo. Si el índice se construyó con un modelo y la query se codifica con otro, brújula y mapa pertenecen a geometrías distintas — el resultado son *"completely irrelevant, nonsensical results"*. Jabloun lo resume como **"the equivalent of using a map of Paris to navigate the streets of Tokyo"**.

> [!note] Un **vector space mismatch** no lanza una excepción: el pipeline corre, devuelve documentos, y simplemente son los equivocados. Por eso es uno de los bugs más insidiosos del RAG — falla silenciosamente. La misma regla aparece en [[01 - Introduction to Natural Language Processing Pipelines]] y [[02 - Diving Deep into Large Language Models]].

### Decoupling document stores: dual-Elasticsearch architecture

El `docker-compose.yml` levanta **dos clusters Elasticsearch independientes** para distintas etapas/pipelines en vez de uno solo compartido. Este **decoupling** habilita tres cosas: **A/B testing**, **independent resource allocation** y **cost-performance optimization**. Tiene dos ventajas concretas.

**(a) Blueprint para cost-performance A/B testing.** Cada pipeline apunta a su propio store con su propio embedding model:

- **Pipeline A (small)** — usa `ES_SMALL_URL` con `text-embedding-3-small`.
- **Pipeline B (large)** — usa `ES_LARGE_URL` con `text-embedding-3-large`.

Se manda **la misma query a ambos** y se loguea la performance (con [[Ragas]]) y el cost-per-query (con `rag_analytics.py`), obteniendo una comparación apples-to-apples del trade-off costo/calidad.

**(b) Independent resource allocation.** Los dos modelos tienen demandas de recursos muy distintas, así que sus clusters se dimensionan por separado:

| Cluster | Heap size | Embedding | Dimensiones |
|---|---|---|---|
| Small instance | 512MB – 1GB | `text-embedding-3-small` | **1,536** |
| Large instance | 2 – 4GB | `text-embedding-3-large` | **3,072** |

> [!tip] El embedding large produce vectores de **3,072 dimensiones** contra **1,536** del small — el doble de storage y RAM por vector. El decoupling permite hostear el cluster large en una máquina high-memory (cara) y el small en una más barata, en lugar de pagar el hardware del peor caso para todo. Es la base del [[FinOps]] del capítulo.

## Systematic pipeline evaluation with Ragas

Con el **golden test set generado en el [[05 - Haystack Pipeline Development with Custom Components|Cap 5]]** se evalúa cuantitativamente `NaiveRAGSuperComponent` contra `HybridRAGSuperComponent`. La pregunta concreta: ¿vale la complejidad extra del hybrid?

- **Naive RAG** — single-path **dense retriever** (`ElasticsearchEmbeddingRetriever`). Bueno en preguntas conceptuales, pero **débil en keyword-specific**: sufre **vocabulary mismatch** y falla con error codes, SKUs y jargon (términos exactos que la búsqueda semántica diluye).
- **Hybrid RAG** — **multi-path**: corre `ElasticsearchBM25Retriever` (sparse, [[BM25]]) y `ElasticsearchEmbeddingRetriever` (dense) **en paralelo**, fusiona con `DocumentJoiner`, y agrega un **reranker** `SentenceTransformersSimilarityRanker` (un [[cross-encoder]]) que reordena con alta precisión — *"dramatically improving recall performance"* (Pinecone). Combina la precisión léxica de BM25 con la comprensión semántica de los embeddings.

### Las 4 métricas de Ragas

> [!note] **Las cuatro dimensiones que mide [[Ragas]].**
> - **Faithfulness** — consistencia factual de la respuesta contra el contexto recuperado (medida directa de **hallucination**).
> - **Context precision** — el ratio **signal-to-noise** del contexto recuperado (¿los chunks son relevantes?).
> - **Context recall** — si el retriever encontró **toda** la información necesaria para responder.
> - **Answer relevancy** — la relevancia end-to-end de la respuesta a la pregunta del usuario.

El componente `RagasEvaluationComponent.run()` espera cuatro inputs — `query`, `ground_truth`, `retrieved_documents`, `generated_answer` — y devuelve un **dict con los scores**. Los notebooks de referencia son `get_started_rag_evaluation_with_ragas.ipynb` y `ragas_evaluation_with_custom_components.ipynb`.

### Executing the evaluation

La ejecución es un loop sobre el dataset sintético, en cuatro pasos por cada par pregunta-respuesta:

1. **Run pipeline one (Naive)** — correr `NaiveRAGSuperComponent` sobre la query.
2. **Run pipeline two (Hybrid)** — correr `HybridRAGSuperComponent` sobre la misma query.
3. **Evaluate** — pasar query, ground_truth, retrieved_documents y generated_answer al `RagasEvaluationComponent`.
4. **Aggregate** — acumular los scores de ambos pipelines a lo largo del dataset.

### Tabla 6.1 — Ragas evaluation results

| Metric | Naive RAG | Hybrid RAG | Improvement (%) | Better system |
|---|---|---|---|---|
| Faithfulness | 0.6411 | 0.9626 | 50.16 | Hybrid RAG |
| Answer relevancy | 0.6678 | 0.7374 | 10.42 | Hybrid RAG |
| Context recall | 0.6800 | 0.7633 | 12.25 | Hybrid RAG |
| Factual correctness (F1) | 0.3556 | 0.4090 | 15.03 | Hybrid RAG |

*Table 6.1 – Ragas evaluation results across 10 question-answer pairs*

> [!tip] El **Hybrid RAG gana en las cuatro métricas**, y de forma dramática en **faithfulness** (+50%): el reranking [[cross-encoder]] sobre la fusión sparse+dense reduce fuertemente la hallucination al traer contexto más preciso. La evidencia cuantitativa justifica la complejidad extra del hybrid sobre el naive.

## Adding production-grade observability with Weights & Biases

> [!note] **Evaluación (Ragas) ≠ Observabilidad (W&B).** La **evaluación** es un **test estático one-off** contra un golden dataset, ejecutado **antes del deploy** — responde "¿mi pipeline es bueno?". La **observabilidad** es **monitoreo continuo en tiempo real en producción** sobre **queries reales de usuarios** — responde "¿mi pipeline *sigue* siendo bueno?": detecta performance drift, costos y fallos en vivo. Son complementarias, no intercambiables.

La integración con [[Weights and Biases]] se hace con el componente **`WeaveConnector`** (la integración nativa de Haystack para W&B). A diferencia de otros componentes, **no requiere `.connect()`**: intercepta los traces automáticamente a través de dos variables de entorno, `HAYSTACK_CONTENT_TRACING_ENABLED` y `WANDB_API_KEY`. Solo hay que agregarlo al pipeline:

```python
from haystack_integrations.components.connectors import WeaveConnector

connector = WeaveConnector(pipeline_name="rag_pipeline")
pipe.add_component("weave", connector)
```

### De observabilidad a FinOps

El script **`rag_analytics.py`** loguea los **token counts** de cada run y, combinándolos con el pricing de `gpt-4o-mini` (el LLM) y de los embedding models, calcula el **dollar cost por run**. Esto convierte el dashboard de W&B en un **[[FinOps]] dashboard**.

![[06-fig-6.2-wandb-cost-dashboard.png]]
*Figure 6.2 – Cost per embedding model dashboard in Weights & Biases*

### Tabla 6.2 — Cost of large vs small embedding model

| Embedding model | Total cost | LLM cost | Embedding cost | Average cost/query |
|---|---|---|---|---|
| Small embedding | 0.009311 | 0.008356 | 0.0009549 | 0.000931 |
| Large embedding | 0.01445 | 0.008398 | 0.006053 | 0.001445 |

*Table 6.2 – Cost of using a large versus a small embedding model*

> [!tip] El **LLM cost es casi idéntico** entre ambos (~0.0084) — lo que cambia es el **embedding cost**, que se dispara ~6.3x en el large (0.006053 vs 0.0009549). El costo del retrieval vectorial, no del LLM, es la palanca de FinOps acá.

> [!note] **Preguntas de negocio que el dashboard responde.** ¿Cuánto costó ayer en total? ¿Cuál es el costo promedio por query? ¿Qué pipeline es más cost-effective? ¿Hay spikes de costo asociados a un tipo de query? El FinOps dashboard convierte tokens y latencias en decisiones de presupuesto.

## Exploring cost-performance trade-offs by analyzing embedding models

La pregunta final: ¿conviene `text-embedding-3-large` o alcanza con `text-embedding-3-small`? Se mide en dos ejes — benchmark de calidad y costo — usando la dual-store para comparar ambos en el mismo proyecto.

- **Calidad (MTEB)** — en el **Massive Text Embedding Benchmark**, el large promedia **64.6%** contra **62.3%** del small: una mejora de solo **2.3 puntos** (~3.7% relativo).
- **Costo** — small a **$0.02 / 1M tokens**, large a **$0.13 / 1M tokens** = un **6.5x cost increase** solo en el paso de embedding.

> [!warning] **Diminishing returns.** Pagar **6.5x** el costo por ganar **2.3 puntos** de MTEB es un trade-off de rendimientos decrecientes para el caso general. El embedding large no es "mejor" en abstracto: es más caro por una mejora marginal.

### Tabla 6.3 — Cost between large and small embedding

| Metric | Small embedding | Large embedding |
|---|---|---|
| Total cost ($) | 0.009396 | 0.014277 |
| LLM cost ($) | 0.008441 | 0.008224 |
| Embedding cost ($) | 0.000955 | 0.006053 |
| Avg cost/query ($) | 0.000940 | 0.001428 |

*Table 6.3 – Cost between large and small embedding*

### Tabla 6.4 — Ragas metric performance between large and small embedding

| Metric | Small embedding | Large embedding |
|---|---|---|
| Faithfulness | 0.804444 | 0.747778 |
| Context recall | 0.926667 | 0.946667 |
| Factual correctness | 0.599 | 0.578 |
| Response relevancy | 0.869054 | 0.868255 |
| Context entity recall | 0.356603 | 0.415104 |

*Table 6.4 – Ragas metric performance between large and small embedding*

> [!tip] Las tablas 6.3 y 6.4 cuentan la historia completa: el large cuesta ~52% más por query y, sobre el dataset evaluado, **no gana de forma consistente** — incluso pierde en faithfulness, factual correctness y response relevancy, ganando solo en context recall y context entity recall. Más caro no es más preciso por defecto.

> [!note] **Conclusión.** `text-embedding-3-small` es el **ganador y default óptimo** para la mayoría de RAG general-purpose. `text-embedding-3-large` es una **specialty tool** reservada para dominios **high-stakes** (legal, financial, medical), donde cada punto de precisión justifica el costo. La clave del **dual-store decoupled** es que permite **elegir**: small como default cost-effective + large solo para los pipelines high-value, todo dentro de un mismo proyecto unificado.

## Citas

> "the equivalent of using a map of Paris to navigate the streets of Tokyo"
> "completely irrelevant, nonsensical results"
> "This represents a 6.5x cost increase for the large model, just for the embedding step."
> "In this chapter, we completed the critical transition from RAG developer to RAG architect."
> "We are no longer just developers; we are architects."

## Para aplicar

- **Setup del proyecto reproducible** — **Python 3.12** (pyenv), **Docker Desktop**, **`uv`** para dependencias, una **API key gratuita de [[Weights and Biases]]**, y `docker-compose.yml` con la **dual-Elasticsearch** (dos clusters independientes).
- **Salir de los notebooks** — estructurar el código en `scripts/` con subcarpetas mapeadas a roles MLOps (`rag/` NLP, `synthetic_data_generation/` + `ragas_evaluation/` QA, `wandb_experiments/` DevOps).
- **Encapsular con SuperComponent** — envolver cada RAG en un `@super_component` con `input_mapping` (rutear `query` a `text_embedder.text` + `retriever.query` + `prompt_builder.question`) y `output_mapping` (`llm.replies`→`replies`, `retriever.documents`→`documents`).
- **Respetar la vector space singularity** — usar **el mismo embedding model** en indexing, synthetic data generation y querying. Nunca mezclar modelos: el mismatch falla en silencio.
- **Usar el dual-store para A/B testing** — Pipeline A (`ES_SMALL_URL` + small) y Pipeline B (`ES_LARGE_URL` + large); misma query a ambos, loguear performance (Ragas) y cost-per-query (`rag_analytics.py`); dimensionar el heap por cluster (small 512MB–1GB, large 2–4GB).
- **Evaluar naive vs hybrid con Ragas** — usar el golden set del Cap 5; loop por par: run naive → run hybrid → `RagasEvaluationComponent` (inputs `query`, `ground_truth`, `retrieved_documents`, `generated_answer`) → aggregate; mirar faithfulness, context precision, context recall, answer relevancy.
- **Preferir Hybrid RAG para keyword-specific** — sparse [[BM25]] + dense en paralelo + `DocumentJoiner` + reranker [[cross-encoder]] (`SentenceTransformersSimilarityRanker`) cuando el corpus tiene error codes, SKUs o jargon.
- **Agregar observabilidad + FinOps** — sumar `WeaveConnector` al pipeline (sin `.connect()`, solo env vars `HAYSTACK_CONTENT_TRACING_ENABLED` y `WANDB_API_KEY`) y `rag_analytics.py` para convertir W&B en un dashboard de costos.
- **Elegir embeddings con criterio** — `text-embedding-3-small` como **default** (mejor cost-performance); reservar `text-embedding-3-large` solo para dominios high-stakes (legal/financial/medical).

## Conexiones

- [[_Building Natural Language and LLM Pipelines|Building NL & LLM Pipelines]] — el MOC del libro.
- [[05 - Haystack Pipeline Development with Custom Components]] — capítulo anterior; genera el **golden dataset** ground-truth con knowledge graph + [[Ragas]] que este capítulo consume para evaluar.
- [[07 - Deploying Haystack-Based Applications]] — capítulo siguiente; despliega productivamente (Docker / [[Hayhooks]]) el sistema reproducible preparado aquí.
- [[04 - Bringing Components Together - Haystack Pipelines for Different Use Cases]] — donde se construyeron originalmente los pipelines naive y hybrid RAG que aquí se evalúan.
- [[Ragas]] — el framework de evaluación cuantitativa (4 métricas) usado para comparar naive vs hybrid.
- [[Weights and Biases]] · [[FinOps]] — observabilidad continua en producción y análisis de costos (`WeaveConnector` + `rag_analytics.py`).
- [[RAG]] · [[Hybrid Search]] · [[BM25]] · [[cross-encoder]] · [[Embeddings]] · [[Reciprocal Rank Fusion (RRF)]] — el stack de retrieval evaluado.
- [[Haystack 2.0]] · [[SuperComponent]] · [[Hayhooks]] — el framework y la abstracción usados.
- **vector space singularity** · **dual-Elasticsearch architecture** · **MTEB (Massive Text Embedding Benchmark)** · **WeaveConnector** · **observability vs evaluation** — conceptos clave del capítulo; candidatos a nota propia.
