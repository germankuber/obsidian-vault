---
title: 09 - Evaluating RAG Quantitatively and with Visualizations
libro: Unlocking Data with Generative AI and RAG
autor: Keith Bourne
capitulo: 9
created: 2026-06-12
tags:
  - libros/unlocking-data-with-generative-ai-and-rag
  - type/case-study
  - status/permanent
aliases:
  - Evaluating RAG Quantitatively and with Visualizations
  - Evaluación cuantitativa de RAG
updated: 2026-06-12
---

# 09 - Evaluating RAG Quantitatively and with Visualizations

> [!info] Capítulo 9 · *Unlocking Data with Generative AI and RAG* — Keith Bourne (Packt, ISBN 9781835887905)
> La [[Evaluation|evaluación]] es lo que cierra el ciclo construir → medir → mejorar de un sistema RAG: hay que medir **mientras se construye** (identificar puntos débiles, optimizar y cuantificar el impacto de cada cambio) y **tras el deploy** (donde el rendimiento se degrada con datos desactualizados o consultas que evolucionan). El capítulo recorre los **frameworks estandarizados** para elegir componentes ([[MTEB]] para embeddings, [[ANN-Benchmarks]]/[[BEIR]] para vector search, Artificial Analysis/Open LLM Leaderboard para LLMs), define qué es la [[Ground Truth|ground truth]] y cómo generarla, y aterriza todo en el **Code lab 9.1 con [[ragas]]**: genera una [[Synthetic Data Generation|ground truth sintética]] y compara la **[[Hybrid Search|búsqueda híbrida]]** del cap. 8 contra la dense semantic search original con métricas de retrieval, generación y end-to-end. Navegá: [[_Unlocking Data with Generative AI and RAG|Libro]] · anterior [[08 - Similarity Searching with Vectors]] · siguiente [[10 - Key RAG Components in LangChain]].

## Resumen

El capítulo defiende que **no se puede mejorar un RAG que no se mide**. La evaluación cumple dos roles temporales: mientras **construís** (continuous evaluation revela los trade-offs y limitaciones de cada elección —vector store, algoritmo de retrieval, modelo de generación— y permite experimentar p. ej. con embeddings locales gratis vs APIs de pago, o con distintos modelos de generación como ChatGPT, Llama o Claude) y después de **desplegar** (el rendimiento decae cuando los datos de retrieval quedan obsoletos, cuando el modelo lidia con consultas/dominio que evolucionan, o por fallos de infra). Sin un baseline objetivo es imposible diagnosticar si una falla está en el retrieval, en el prompt o en la respuesta del LLM.

Antes de instalar nada, los **frameworks estandarizados** ayudan a acotar la elección de componentes: el **[[MTEB]] Retrieval Leaderboard** para modelos de embedding, **[[ANN-Benchmarks]]** y **[[BEIR]]** para vector stores y vector search, y los leaderboards de **Artificial Analysis** y **Open LLM** para los LLMs. Pero ningún benchmark genérico captura *tu* pipeline con *tus* inputs y outputs, así que terminás necesitando tu propio framework a medida, y para eso hace falta una **[[Ground Truth|ground truth]]**: los datos que representan las respuestas ideales si el sistema funcionara a su máximo. El capítulo enumera cómo obtenerla (anotación humana, expertos/rule-based, crowdsourcing y generación sintética) y luego ejecuta el **Code lab 9.1 con [[ragas]]**, una plataforma de evaluación hecha para RAG: genera 10 (que resultan en **7**) ejemplos de ground truth sintética con tres tipos de evolución (**simple/reasoning/multi_context**), arma dos cadenas (`rag_chain_similarity` dense vs `rag_chain_hybrid` ensemble del cap. 8) y las compara con seis métricas agrupadas en **retrieval** ([[Context Precision|context precision]], [[Context Recall|context recall]]), **generación** ([[Faithfulness|faithfulness]], [[Answer Relevancy|answer relevancy]]) y **end-to-end** ([[Answer Correctness|answer correctness]], [[Answer Similarity|answer similarity]]). Cierra con los insights del cofundador de ragas (**Shahul Es**) sobre datos sintéticos, métricas de feedback y métricas **[[Reference-free Metrics|reference-free]]** ideales para el deployment, y con técnicas adicionales más allá de ragas: **[[BLEU]]**, **[[ROUGE]]**, **[[Semantic Similarity|similitud semántica]]** y la **evaluación humana**. El código está en github.com/PacktPublishing/Unlocking-Data-with-Generative-AI-and-RAG/tree/main/Chapter_09.

## Evaluar mientras construís

La **continuous evaluation** durante el desarrollo permite identificar dónde el sistema rinde mal, optimizarlo y medir el impacto de cada modificación. Revela los **trade-offs y limitaciones** de las elecciones de componentes: qué [[Vector Store|vector store]], qué algoritmo de retrieval, qué modelo de generación. Es lo que te deja experimentar de forma informada:

- **Modelos de embedding** — comparar un modelo local gratuito contra una API en la nube de pago: ¿la API justifica su costo en *tu* caso?
- **Modelos de generación** — comparar ChatGPT, Llama, Claude y otros sobre tu pipeline real.

> [!note] La evaluación convierte el diseño de la arquitectura en un proceso **iterativo e informado**: cada decisión equilibra eficiencia, escalabilidad y capacidad de generalización contra el costo de cómputo, en lugar de elegir a ciegas.

## Evaluar tras el deploy

Una vez en producción, el rendimiento **puede decaer** por varias causas: datos de retrieval desactualizados o irrelevantes; un modelo que lucha con consultas o un dominio que evolucionan; o fallos de infraestructura.

El capítulo usa el **ejemplo de gestión de patrimonio financiero**: el sistema se construye sobre 5 años de análisis de las grandes firmas, pero eventos macro (catástrofes, inestabilidad política, eventos regionales) que **no están en esos 5 años de datos** degradan su valor. Un usuario pregunta:

> "What impact will the Category 5 hurricane that just occurred have on my portfolio in the next year?"

El sistema no puede responder bien porque le falta la información reciente. La solución es actualizar y monitorear de forma continua (en especial sumar los reportes recientes sobre el impacto del huracán).

> [!tip] Mitigación tras el deploy: monitorear los puntos de falla, **actualizar el corpus de retrieval**, hacer fine-tuning del modelo de generación y optimizar la infra. Establecé un **feedback loop** (reportes/sugerencias de usuarios), monitoreá el uso de la UI, los tiempos de respuesta y la relevancia/utilidad del output, y recogé feedback con encuestas, logs de interacción y métricas de satisfacción.

## Por qué evaluar te hace mejorar

Si no medís **antes y después**, no podés saber qué mejoró ni qué empeoró tras un cambio. Y sin un baseline objetivo es difícil diagnosticar las fallas: ¿el problema está en el **retrieval**, en el **prompt** o en la **respuesta del LLM**? La evaluación es la medición **sistemática y objetiva** que responde esas preguntas. Incluso al inicio —antes de instalar nada— se pueden usar **frameworks estandarizados** para acotar la elección de componentes.

## Frameworks estandarizados de evaluación

Cada componente del pipeline tiene su propio benchmark de referencia.

### Embeddings — el MTEB Retrieval Leaderboard

El **[[MTEB]] (Massive Text Embedding Benchmark) Retrieval Leaderboard** (huggingface.co/spaces/mteb/leaderboard) rankea modelos de embedding por el promedio de su rendimiento en muchas tareas. Hacé clic en las pestañas **Retrieval** y **Retrieval w/Instructions**. Podés ordenar por cualquier métrica individual (p. ej. **FiQA2018** si tu caso es QA financiero). Los datasets que componen el leaderboard de retrieval:

| Dataset | Tarea |
|---|---|
| **ArguAna** | Argument retrieval |
| **ClimateFEVER** | Climate fact retrieval |
| **CQADupstackRetrieval** | Duplicate question retrieval |
| **DBPedia** | Entity retrieval |
| **FEVER** | Fact extraction & verification |
| **FiQA2018** | Financial QA |
| **HotpotQA** | Multi-hop QA |
| **MSMARCO** | Passage/document ranking |
| **NFCorpus** | Fact-checking |
| **NQ** | Open-domain QA |
| **QuoraRetrieval** | Duplicate-question detection |
| **SCIDOCS** | Scientific document retrieval |
| **SciFact** | Scientific claim verification |
| **Touche2020** | Argument retrieval |
| **TRECCOVID** | COVID-19 info retrieval |

> [!tip] No mires solo el promedio: ordená el leaderboard por el dataset más parecido a *tu* dominio (FiQA2018 para finanzas, SciFact para ciencia, etc.).

### Vector stores y vector search — ANN-Benchmarks y BEIR

- **[[ANN-Benchmarks]]** — evalúa la precisión, velocidad y uso de memoria de las herramientas de ANN ([[FAISS]], [[ANNOY]], [[HNSW]] — los algoritmos vistos en [[08 - Similarity Searching with Vectors]]).
- **[[BEIR]] (Benchmarking IR)** — un benchmark de IR **heterogéneo y zero-shot** que cubre múltiples dominios (QA, fact-checking, entity retrieval).

> [!note] **[[Zero-shot]]**: consultas para las que no se incluye ningún ejemplo previo. Es lo común en RAG, y se trata más a fondo en [[13 - Using Prompt Engineering to Improve RAG Efforts]].

Los datasets nombrados de BEIR:

- **MSMARCO** — consultas/respuestas reales a gran escala para deep-learning search y QA.
- **HotpotQA** — preguntas naturales **multi-hop** con supervisión de supporting facts → QA explicable.
- **CQADupStack** — benchmark de community QA (cQA) construido sobre 12 subforos de Stack Exchange, anotado con información de preguntas duplicadas.

### Benchmarks de LLMs

El **Artificial Analysis LLM Performance Leaderboard** (artificialanalysis.ai) cubre modelos open y propietarios (ChatGPT, Claude, Llama). Sus sub-leaderboards de **calidad**:

- **General ability** = Chatbot Arena.
- **Reasoning & knowledge** = **[[MMLU]]** (Massive Multitask Language Understanding) + **MT Bench** (Multi-turn Benchmark).

También trackea **velocidad** y **precio**, y un leaderboard de aspectos técnicos: throughput (tokens/seg), latency (tiempo de respuesta), memory footprint y scaling efficiency.

El **Open LLM Leaderboard** (modelos open source) usa estos benchmarks:

- **[[ARC]] (AI2 Reasoning Challenge)** — razonamiento científico.
- **HellaSwag** — sentido común.
- **MMLU** — conocimiento específico de dominio.
- **TruthfulQA** — respuestas veraces/informativas.
- **WinoGrande** — sentido común vía desambiguación de pronombres.
- **[[GSM8K]] (Grade School Math 8K)** — razonamiento matemático.

> [!tip] Los benchmarks estandarizados son un buen **punto de partida** para seleccionar componentes (sumá eficiencia de cómputo y facilidad de integración), pero **pueden NO capturar tu pipeline específico** con tus inputs/outputs → construí tu propio framework de evaluación a medida.

## ¿Qué es la ground truth?

La **[[Ground Truth|ground truth]]** son los datos que representan las respuestas **ideales** si el sistema RAG operara a su máximo rendimiento. Ejemplo de **investigación veterinaria de cáncer**: las preguntas son sobre la última investigación en cáncer canino, los datos son papers de PubMed, y la ground truth son pares Q&A realistas que tu audiencia preguntaría junto con las respuestas ideales esperadas (algo subjetivo, pero crítico).

### Cómo usarla

Es un **benchmark** para medir el rendimiento: comparás el output del RAG contra la ground truth → evaluás la relevancia del retrieval y la exactitud/coherencia de la respuesta, cuantificás distintos enfoques y encontrás áreas de mejora.

### Cómo generarla

Crearla manualmente lleva mucho tiempo; si tu empresa ya tiene datasets, úsalos. Las técnicas:

- **Anotación humana** — anotadores crean manualmente las respuestas ideales. Alta calidad, pero costoso y lento a escala.
- **Conocimiento experto** — SMEs (subject-matter experts) aportan las respuestas de ground truth; bueno para dominios especializados/técnicos. Incluye la **generación basada en reglas**: definir reglas/plantillas que los SMEs completan. Ejemplo de soporte de teléfonos móviles con la plantilla `To resolve [issue], you can try [solution]`, con `issue = battery drain` y `solution = reducing screen brightness and closing background apps`, que hidratada da:

> "To resolve [battery drain], you can try [reducing screen brightness and closing background apps]"

- **Crowdsourcing** — Amazon Mechanical Turk, Figure Eight: tercerizar a un pool de trabajadores con instrucciones claras y control de calidad.
- **Ground truth sintética** — cuando los datos reales son inviables: **modelos de lenguaje fine-tuneados** (fine-tunear un LLM sobre ejemplos de alta calidad para que genere respuestas similares) y **métodos basados en retrieval** (recuperar pasajes relevantes de un corpus de alta calidad como proxy de la ground truth).

## Code lab 9.1 – ragas

**[[ragas]]** (Retrieval-Augmented Generation Assessment) es una plataforma de evaluación construida específicamente para RAG. El lab: implementar ragas, generar una ground truth sintética, configurar las métricas y evaluar el impacto de la búsqueda híbrida del cap. 8 vs la dense semantic search original ([[08 - Similarity Searching with Vectors]]). ragas evoluciona rápido → consultá docs.ragas.io. El código continúa desde el `EnsembleRetriever` del cap. 8 (Code lab 8.3).

La instalación:

```bash
pip install ragas
pip install tqdm -q –user
pip install matplotlib
```

`tqdm` da las barras de progreso que usa ragas; `matplotlib` es para las visualizaciones de las métricas.

Los imports:

```python
import tqdm as notebook_tqdm
import pandas as pd
import matplotlib.pyplot as plt
from datasets import Dataset
from ragas import evaluate
from ragas.testset.generator import TestsetGenerator
from ragas.testset.evolutions import (
    simple, reasoning, multi_context)
from ragas.metrics import (
    answer_relevancy,
    faithfulness,
    context_recall,
    context_precision,
    answer_correctness,
    answer_similarity
)
```

- `datasets` (Hugging Face) → aporta la interfaz `Dataset`.
- `evaluate` → corre la evaluación y devuelve un `Result` con los scores por métrica.
- `TestsetGenerator` → genera Q-A + contexts de ground truth sintética, con distribución personalizable vía `distributions`; funciona con loaders de LangChain y LlamaIndex.
- Las **3 evolutions**: **simple** (preguntas directas), **reasoning** (preguntas que requieren razonamiento), **multi_context** (preguntas que requieren info de múltiples chunks).
- Las **6 métricas** (answer_relevancy, faithfulness, context_recall, context_precision, answer_correctness, answer_similarity); hay dos más component-wise (context relevancy, context entity recall) que acá se omiten.

### Setup de LLMs y embeddings

```python
embedding_ada = "text-embedding-ada-002"
model_gpt35 = "gpt-3.5-turbo"
model_gpt4 = "gpt-4o-mini"
embedding_function = OpenAIEmbeddings(model=embedding_ada, openai_api_key=openai.api_key)
llm = ChatOpenAI(model=model_gpt35, openai_api_key=openai.api_key, temperature=0.0)
generator_llm = ChatOpenAI(model=model_gpt35, openai_api_key=openai.api_key, temperature=0.0)
critic_llm = ChatOpenAI(model=model_gpt4, openai_api_key=openai.api_key, temperature=0.0)
```

Además del `llm` primario, se definen dos LLMs específicos para la evaluación: **generator_llm** (gpt-3.5-turbo, genera los ejemplos) y **critic_llm** (gpt-4o-mini, el más avanzado → mejor para evaluar). Se elimina el viejo `llm = ChatOpenAI(model_name="gpt-4o-mini", temperature=0)`.

### Las dos cadenas

Se nombran dos cadenas RAG: `rag_chain_similarity` (sobre `dense_retriever`) y `rag_chain_hybrid` (sobre `ensemble_retriever`, renombrada desde `rag_chain_with_source`).

```python
rag_chain_similarity = RunnableParallel(
    {"context": dense_retriever,
    "question": RunnablePassthrough()
}).assign(answer=rag_chain_from_docs)
```

```python
rag_chain_hybrid = RunnableParallel(
    {"context": ensemble_retriever,
    "question": RunnablePassthrough()
}).assign(answer=rag_chain_from_docs)
```

Cada cadena se invoca con la misma query (`user_query = "What are Google's environmental initiatives?"`) e imprime el relevance_score, el final_answer y los docs recuperados (id + source). Como cada retriever devuelve un set distinto de documentos, las respuestas enfatizan cosas diferentes.

Final Answer de la cadena **similarity** (dense):

> "Google's environmental initiatives include empowering individuals to take action through features like eco-friendly routing in Google Maps, energy efficiency features in Google Nest thermostats, and carbon emissions information in Google Flights. They also work with partners and customers to reduce carbon emissions by providing them with technology, products, and services. Additionally, Google is involved in various sustainability efforts such as working with coalitions and organizations like iMasons Climate Accord, ReFED, and The Nature Conservancy to drive systemic change and reduce environmental impact."

Final Answer de la cadena **hybrid**:

> "Google's environmental initiatives include engaging with suppliers to reduce their energy consumption and greenhouse gas emissions, reporting environmental data, and assessing environmental criteria. They are also involved in public policy and advocacy efforts to support climate action, working with coalitions such as the World Business Council for Sustainable Development and the World Resources Institute. Additionally, Google is focused on using technology and platforms to organize information about the planet and make it actionable to help partners and customers create a positive impact."

### Generar la ground truth sintética

> [!warning] **El costo de la API.** ragas usa tu API de LLM de forma **extensiva** (LLM-assisted evaluation): cada ejemplo de ground truth generado/evaluado llama a un LLM (a veces varias veces por métrica). 100 ejemplos × 6 métricas → **miles** de llamadas → costo real. El autor incurrió en **~$2–$2.50** por corrida completa con apenas **10 ground examples y 6 métricas**; subir el `test_size` aumenta el costo sustancialmente.

```python
generator = TestsetGenerator.from_langchain(
    generator_llm,
    critic_llm,
    embedding_function
)
documents = [Document(page_content=chunk) for chunk in splits]
testset = generator.generate_with_langchain_docs(
    documents,
    test_size=10,
    distributions={
        simple: 0.5,
        reasoning: 0.25,
        multi_context: 0.25
    }
)
testset_df = testset.to_pandas()
testset_df.to_csv(os.path.join('testset_data.csv'), index=False)
print("testset DataFrame saved successfully in the local directory.")
```

`test_size=10`, con distribución **50% simple / 25% reasoning / 25% multi_context**. Se guarda a CSV (correr una vez → reutilizar). Carga y vista:

```python
saved_testset_df = pd.read_csv(os.path.join('testset_data.csv'))
print("testset DataFrame loaded successfully from local directory.")
saved_testset_df.head(5)
```

El output es la **Figura 9.1** (preguntas + sus respuestas `ground_truth`):

![[09-fig-9.1-ground-truth-dataframe.jpg]]
*Figure 9.1 – DataFrame showing synthesized ground-truth data*

> [!warning] La generación **puede fallar** para algunos ejemplos → terminás con menos de los pedidos. Acá, de `test_size=10` resultaron solo **7 ejemplos**, aceptado para mantener el costo bajo en este ejemplo simple (en pruebas reales conviene **mucho más de 10**).

Preparación del dataset:

```python
saved_testing_data = saved_testset_df.astype(str).to_dict(orient='list')
saved_testing_dataset = Dataset.from_dict(saved_testing_data)
saved_testing_dataset_sm = saved_testing_dataset.remove_columns(["evolution_type", "episode_done"])
```

Al inspeccionar `saved_testing_dataset_sm`:

```
Dataset({
    features: ['question', 'contexts', 'ground_truth', 'metadata'],
    num_rows: 7
})
```

La función `generate_answer` (flexible, acepta cualquiera de las dos cadenas):

```python
def generate_answer(question, ground_truth, rag_chain):
    result = rag_chain.invoke(question)
    return {
        "question": question,
        "answer": result["answer"]["final_answer"],
        "contexts": [doc.page_content for doc in result["context"]],
        "ground_truth": ground_truth
    }
```

Se mapea sobre ambas cadenas:

```python
testing_dataset_similarity = saved_testing_dataset_sm.map(
    lambda x: generate_answer(x["question"], x["ground_truth"], rag_chain_similarity),
    remove_columns=saved_testing_dataset_sm.column_names)
testing_dataset_hybrid = saved_testing_dataset_sm.map(
    lambda x: generate_answer(x["question"], x["ground_truth"], rag_chain_hybrid),
    remove_columns=saved_testing_dataset_sm.column_names)
```

Se corre `evaluate` de ragas sobre ambos datasets con las 6 métricas:

```python
score_similarity = evaluate(
    testing_dataset_similarity,
    metrics=[faithfulness, answer_relevancy, context_precision, context_recall, answer_correctness, answer_similarity]
)
similarity_df = score_similarity.to_pandas()
```

```python
score_hybrid = evaluate(
    testing_dataset_hybrid,
    metrics=[faithfulness, answer_relevancy, context_precision, context_recall, answer_correctness, answer_similarity]
)
hybrid_df = score_hybrid.to_pandas()
```

### Las 6 métricas de ragas

| Métrica | Etapa | Qué mide | Rango / dirección |
|---|---|---|---|
| **[[Context Precision\|context_precision]]** | Retrieval | Signal-to-noise del contexto recuperado; si los ítems relevantes según la ground truth rankean más arriba (usa question + ground_truth + contexts) | 0–1, mayor mejor |
| **[[Context Recall\|context_recall]]** | Retrieval | Si recupera **toda** la info relevante; cuán bien el contexto recuperado se alinea con la respuesta anotada (ground truth) | 0–1, mayor mejor |
| **[[Faithfulness\|faithfulness]]** | Generación | Exactitud/consistencia factual de la respuesta vs el contexto dado (usa answer + retrieved context) | 0–1, mayor mejor |
| **[[Answer Relevancy\|answer_relevancy]]** | Generación | Cuán pertinente es la respuesta al prompt; baja con respuestas incompletas/redundantes (usa question + context + answer) | 0–1, mayor mejor |
| **[[Answer Correctness\|answer_correctness]]** | End-to-end | Exactitud de la respuesta vs la ground truth (usa ground truth + answer) | 0–1, mayor mejor |
| **[[Answer Similarity\|answer_similarity]]** | End-to-end | Semejanza semántica entre la respuesta y la ground truth | 0–1, mayor mejor |

### Analizar los resultados

```python
key_columns = ['faithfulness', 'answer_relevancy', 'context_precision', 'context_recall', 'answer_correctness', 'answer_similarity']
similarity_means = similarity_df[key_columns].mean()
hybrid_means = hybrid_df[key_columns].mean()
comparison_df = pd.DataFrame({'Similarity Run': similarity_means, 'Hybrid Run': hybrid_means})
comparison_df['Difference'] = comparison_df['Similarity Run'] - comparison_df['Hybrid Run']
similarity_df.to_csv(os.path.join('similarity_run_data.csv'), index=False)
hybrid_df.to_csv(os.path.join('hybrid_run_data.csv'), index=False)
comparison_df.to_csv(os.path.join('comparison_data.csv'), index=True)
print("Dataframes saved successfully in the local directory.")
```

> [!tip] Guardá los resultados de la evaluación a **CSV** para no tener que volver a correr ragas (y volver a pagar la API) cada vez que querés analizar los números.

Luego se cargan y se imprimen por etapa:

```python
sem_df = pd.read_csv(os.path.join('similarity_run_data.csv'))
rec_df = pd.read_csv(os.path.join('hybrid_run_data.csv'))
comparison_df = pd.read_csv(os.path.join('comparison_data.csv'), index_col=0)
print("Dataframes loaded successfully from the local directory.")
print("Performance Comparison:")
print("\n**Retrieval**:")
print(comparison_df.loc[['context_precision', 'context_recall']])
print("\n**Generation**:")
print(comparison_df.loc[['faithfulness', 'answer_relevancy']])
print("\n**End-to-end evaluation**:")
print(comparison_df.loc[['answer_correctness', 'answer_similarity']])
```

Y se grafica con matplotlib (un subplot por etapa; barras de similarity en `#D51900` y de hybrid en `#992111`, con etiquetas `f'{height:.1%}'`):

```python
fig, axes = plt.subplots(3, 1, figsize=(12, 18), sharex=False)
bar_width = 0.35
categories = ['Retrieval', 'Generation', 'End-to-end']
metrics = {
    'Retrieval': ['context_precision', 'context_recall'],
    'Generation': ['faithfulness', 'answer_relevancy'],
    'End-to-end': ['answer_correctness', 'answer_similarity']
}
for ax, category in zip(axes, categories):
    metric_names = metrics[category]
    x = range(len(metric_names))
    similarity_bars = ax.bar(
        [i - bar_width/2 for i in x],
        comparison_df.loc[metric_names, 'Similarity Run'],
        bar_width, label='Similarity Run', color='#D51900')
    hybrid_bars = ax.bar(
        [i + bar_width/2 for i in x],
        comparison_df.loc[metric_names, 'Hybrid Run'],
        bar_width, label='Hybrid Run', color='#992111')
    for bars in (similarity_bars, hybrid_bars):
        for bar in bars:
            height = bar.get_height()
            ax.annotate(f'{height:.1%}',
                        xy=(bar.get_x() + bar.get_width() / 2, height),
                        xytext=(0, 3), textcoords="offset points",
                        ha='center', va='bottom')
    ax.set_title(f'{category} Evaluation')
    ax.set_xticks(list(x))
    ax.set_xticklabels(metric_names)
    ax.legend()
fig.text(0.5, 0.04, 'Metrics', ha='center')
fig.suptitle('RAG Performance Comparison: Similarity vs Hybrid')
plt.tight_layout()
plt.subplots_adjust(top=0.95)
plt.show()
```

> [!warning] El dataset es **muy chico (7 ejemplos)** → no leas demasiado en los números absolutos. Sirven para ilustrar el flujo, no para concluir cuál enfoque es mejor en general.

### Evaluación de retrieval (Fig 9.2)

Métricas: **context_precision** + **context_recall**. Output:

```
                  Similarity Run  Hybrid Run  Difference
context_precision        0.906113    0.841267    0.064846
context_recall           0.950000    0.925000    0.025000
```

> [!note] **context_precision** = relación señal/ruido del contexto recuperado: si los ítems relevantes según la ground truth aparecen rankeados más arriba. **context_recall** = si logra recuperar toda la info relevante, alineando el contexto con la respuesta anotada. Se relacionan con la precision/recall tradicionales (precision = proporción de lo recuperado que es relevante; recall = proporción de lo relevante que se recupera), pero ragas además considera el **ranking** y la **alineación**, diseñado para QA.

![[09-fig-9.2-retrieval-comparison.jpg]]
*Figure 9.2 – Chart showing retrieval performance comparison between similarity search and hybrid search*

### Evaluación de generación (Fig 9.3)

Métricas: **faithfulness** + **answer_relevancy**. Output:

```
                Similarity Run  Hybrid Run  Difference
faithfulness          0.977500    0.945833    0.031667
answer_relevancy      0.968222    0.965247    0.002976
```

> [!note] **faithfulness** = exactitud/consistencia factual de la respuesta respecto del contexto dado (de answer + retrieved context). **answer_relevancy** = cuán pertinente es la respuesta al prompt; penaliza respuestas incompletas o redundantes (de question + context + answer).

![[09-fig-9.3-generation-comparison.jpg]]
*Figure 9.3 – Chart showing generation performance comparison between similarity search and hybrid search*

### Evaluación end-to-end (Fig 9.4)

Métricas: **answer_correctness** + **answer_similarity**. Output:

```
                   Similarity Run  Hybrid Run  Difference
answer_correctness       0.776018    0.717365    0.058653
answer_similarity        0.969899    0.969170    0.000729
```

> [!note] **answer_correctness** = exactitud de la respuesta vs la ground truth (de ground truth + answer). **answer_similarity** = semejanza semántica entre la respuesta y la ground truth. Ambas 0–1, mayor mejor.

![[09-fig-9.4-end-to-end-comparison.jpg]]
*Figure 9.4 – Chart showing end-to-end performance comparison between similarity search and hybrid search*

### Tabla 9.x — Comparación completa (Similarity vs Hybrid)

| Métrica | Similarity Run | Hybrid Run | Difference |
|---|---|---|---|
| context_precision | **0.906113** | **0.841267** | 0.064846 |
| context_recall | **0.950000** | **0.925000** | 0.025000 |
| faithfulness | **0.977500** | **0.945833** | 0.031667 |
| answer_relevancy | **0.968222** | **0.965247** | 0.002976 |
| answer_correctness | **0.776018** | **0.717365** | 0.058653 |
| answer_similarity | **0.969899** | **0.969170** | 0.000729 |

> [!tip] La tabla deja claro que, sobre estos 7 ejemplos, la dense semantic search supera ligeramente a la híbrida en las seis métricas — pero el dataset es demasiado chico para sacar conclusiones generales; el valor del ejercicio es el **método**, no el veredicto.

### Otras métricas component-wise

Más allá de las 6 usadas, ragas ofrece:

- **Context relevancy** — relevancia del contexto recuperado (de question + contexts; 0–1).
- **Context entity recall** — recall de las entidades presentes tanto en `ground_truth` como en `contexts`, relativo solo a la ground_truth; útil en casos basados en hechos como un help desk de turismo o Q&A histórico.
- **Aspect critique** — evalúa las respuestas sobre aspectos predefinidos (harmlessness, correctness) o definidos por el usuario; output **binario**; usa `answer` como input.

### Insights del fundador

El autor habló con el cofundador de ragas, **Shahul Es**. Sus cuatro insights:

- **Synthetic data generation** — el roadblock número uno es **no tener suficientes datos de test de ground truth**; la generación sintética de ragas cubre tipos de preguntas variados, pero hay que **revisar la ground truth generada y descartar las preguntas que no correspondan**.
- **Feedback metrics** — incorporar feedback loops, tanto **explícito** ("algo salió mal") como **implícito** (satisfacción, thumbs up/down); el implícito puede ser ruidoso pero igual útil.
- **Reference vs reference-free metrics** — las métricas con referencia necesitan ground truth; ragas enfatiza las **[[Reference-free Metrics|reference-free]]** (ver el paper de ragas, arxiv.org/abs/2309.15217); faithfulness y answer relevance son reference-free.
- **Deployment evaluation** — las métricas reference-free son **ideales para el deployment**, donde es improbable que exista ground truth.

> [!note] Una métrica **reference-free** no requiere ground truth para computarse → es la única viable en producción, donde casi nunca tenés la respuesta ideal a mano. Por eso faithfulness y answer relevancy son las candidatas naturales para monitorear un RAG ya desplegado.

## Técnicas adicionales de evaluación

Más allá de ragas:

- **[[BLEU]] (Bilingual Evaluation Understudy)** — overlap de **n-gramas** entre la respuesta generada y la ground truth; mide similitud **superficial** (elección de palabras/fraseo), puede perder el significado semántico.
- **[[ROUGE]] (Recall-Oriented Understudy for Gisting Evaluation)** — basada en **recall**: cuánto de la ground truth queda capturado; buena para respuestas largas/detalladas (overlap de información, no de palabras exactas).
- **[[Semantic Similarity|Similitud semántica]]** — cosine similarity o **STS (Semantic Textual Similarity)**; captura el significado más allá de las palabras exactas; ideal cuando las respuestas usan palabras distintas pero el mismo sentido.
- **Evaluación humana** — raters juzgan coherencia, fluidez, relevancia, exactitud factual, claridad, tono, inconsistencias y UX; complementa las métricas automáticas.

> [!tip] Combiná las técnicas (BLEU + ROUGE + similitud semántica + evaluación humana) para una visión **holística**, y alineá la elección con las prioridades de tu app (exactitud factual vs fluidez/coherencia).

## Citas

> "What impact will the Category 5 hurricane that just occurred have on my portfolio in the next year?"

> "To resolve [battery drain], you can try [reducing screen brightness and closing background apps]"

## Para aplicar

- **Medí antes y después de cada cambio** — sin baseline objetivo no podés saber qué mejoró ni diagnosticar si la falla está en retrieval, prompt o respuesta del LLM.
- **Arrancá por los benchmarks estandarizados** — [[MTEB]] (ordenado por el dataset de tu dominio) para embeddings, [[ANN-Benchmarks]]/[[BEIR]] para vector search, Artificial Analysis/Open LLM Leaderboard para LLMs; luego construí tu framework a medida.
- **Generá ground truth sintética con ragas** y **revisala**: descartá las preguntas que no correspondan a tu caso.
- **Cuidá el costo de la API** — ragas hace miles de llamadas; ~$2–$2.50 con solo 10 ejemplos × 6 métricas. Empezá chico, guardá resultados a CSV para no re-correr.
- **Usá las 6 métricas agrupadas** — retrieval (context precision/recall), generación (faithfulness, answer relevancy), end-to-end (answer correctness/similarity).
- **Para el deployment, priorizá métricas reference-free** (faithfulness, answer relevancy), que no necesitan ground truth.
- **Combiná técnicas** (BLEU/ROUGE/STS/humana) según si te importa más la exactitud factual o la fluidez.

## Conexiones

- [[_Unlocking Data with Generative AI and RAG|Libro]] — el MOC del libro.
- [[08 - Similarity Searching with Vectors]] — capítulo anterior · provee la búsqueda híbrida ([[EnsembleRetriever]], [[BM25]], [[Reciprocal Rank Fusion|RRF]]) y los algoritmos [[FAISS]]/[[ANNOY]]/[[HNSW]] que acá se evalúan objetivamente.
- [[10 - Key RAG Components in LangChain]] — capítulo siguiente · usar [[LangChain]] de forma efectiva con los componentes de RAG.
- [[13 - Using Prompt Engineering to Improve RAG Efforts]] — donde se profundiza el [[Zero-shot|zero-shot]] mencionado en BEIR.
- [[Evaluation]] · [[ragas]] · [[Ground Truth]] · [[Synthetic Data Generation]] — el núcleo de la evaluación de RAG.
- [[MTEB]] · [[ANN-Benchmarks]] · [[BEIR]] · [[ARC]] · [[MMLU]] · [[GSM8K]] · [[Zero-shot]] — benchmarks estandarizados de componentes.
- [[Context Precision]] · [[Context Recall]] · [[Faithfulness]] · [[Answer Relevancy]] · [[Answer Correctness]] · [[Answer Similarity]] · [[Reference-free Metrics]] — las métricas de ragas.
- [[BLEU]] · [[ROUGE]] · [[Semantic Similarity]] — técnicas adicionales de evaluación.
- [[Hybrid Search]] · [[FAISS]] · [[ANNOY]] · [[HNSW]] · [[LangChain]] — componentes evaluados (candidatos a nota propia en **negrita** si aún no existen: **[[ANN-Benchmarks]]**, **[[BEIR]]**, **[[ragas]]**, **[[Reference-free Metrics]]**).
