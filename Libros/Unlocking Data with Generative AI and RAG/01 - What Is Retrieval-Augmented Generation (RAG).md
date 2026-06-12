---
title: 01 - What Is Retrieval-Augmented Generation (RAG)
libro: Unlocking Data with Generative AI and RAG
autor: Keith Bourne
capitulo: 1
created: 2026-06-12
tags:
  - libros/unlocking-data-with-generative-ai-and-rag
  - type/case-study
  - status/permanent
aliases:
  - What Is Retrieval-Augmented Generation (RAG)
  - Qué es RAG
updated: 2026-06-12
---

# 01 - What Is Retrieval-Augmented Generation (RAG)

> [!info] Capítulo 1 · *Unlocking Data with Generative AI and RAG* — Packt (ISBN 9781835887905) · Keith Bourne
> [[RAG]] (Retrieval-Augmented Generation) conecta un [[LLM]] con los datos **privados, internos o recientes** de una empresa para extraer valor estratégico: el modelo nunca vio esos datos durante su entrenamiento, así que RAG los **recupera** y los inyecta como contexto antes de generar la respuesta. Este capítulo define qué es RAG, sus **5 ventajas** y **7 desafíos**, sus aplicaciones empresariales, lo enfrenta a la **IA generativa convencional** y al **[[Fine-tuning]]**, fija el vocabulario base ([[Context Window]], [[Embeddings]], vector stores) y desarma su **arquitectura de tres etapas** ([[Indexing]] · [[Retrieval]] · [[Generation]]). Navegá: [[_Unlocking Data with Generative AI and RAG|Libro]] · siguiente [[02 - Code Lab - An Entire RAG Pipeline]].

## Resumen

El capítulo arranca con la motivación central de **RAG**: la **IA generativa** (los **modelos generativos** que producen texto, código, imágenes, audio o video) es enormemente potente, pero un **[[LLM]]** solo conoce lo que vio durante su entrenamiento. No conoce los **datos privados** de tu empresa (documentos internos, bases de conocimiento, registros), y tampoco los **datos públicos recientes** publicados después de su corte de entrenamiento. RAG resuelve exactamente esa brecha: **recupera** información relevante de una fuente externa y la **augmenta** al prompt para que el modelo **genere** una respuesta fundamentada en esos datos, en vez de inventar. La frase que resume la diferencia frente a la IA generativa sola es que el modelo **"no sabe lo que no sabe"**: sin RAG, responde con confianza incluso cuando carece de la información, produciendo **[[Hallucinations|alucinaciones]]**.

A partir de ahí el capítulo recorre las **cinco ventajas** de RAG (precisión y relevancia, personalización, flexibilidad, expansión del conocimiento más allá del entrenamiento, y reducción de alucinaciones) y sus **siete desafíos** (dependencia de la calidad de los datos, necesidad de manipular y limpiar datos, sobrecarga computacional, explosión y complejidad del almacenamiento e integración, sobrecarga de información, alucinaciones residuales, y la alta complejidad de los componentes de RAG). Luego enumera **siete aplicaciones empresariales** reales, fija el **vocabulario de RAG** (LLM, prompting/prompt design/prompt engineering, inference, [[Context Window]], fine-tuning con sus variantes FMFT/PEFT/LoRA/quantization, vector store vs vector database, y los [[Embeddings|vectores]]), profundiza en los **vectores**, y dedica dos comparaciones clave: **RAG vs IA generativa convencional** y **RAG vs fine-tuning**, con la analogía de memoria a largo plazo (fine-tuning) vs memoria a corto plazo (input/RAG). Cierra desarmando la **arquitectura de RAG** en tres etapas técnicas — **indexing** (previo a la consulta), **retrieval** y **generation** — con los siete pasos detallados que ocurren cuando llega la query.

> [!note] **RAG = Retrieval-Augmented Generation.** Un patrón que toma la consulta del usuario, **recupera** (retrieval) datos relevantes de una fuente externa, los **augmenta** al prompt, y deja que el LLM **genere** la respuesta fundamentada en esos datos recuperados.

## ¿Qué es RAG? La motivación

La **IA generativa** abarca los modelos que generan contenido nuevo: texto, código, imágenes, audio y video (los modelos generativos de imágenes/audio/video se tratan en [[14 - Advanced RAG-Related Techniques for Improving Results]]). El motor de la generación de texto es el **[[LLM]]** (Large Language Model). El problema: un LLM se entrena sobre un corpus fijo y, por más grande que sea, **hay dos clases de información que nunca vio**:

- **Datos privados / internos** — los documentos, bases de conocimiento, tickets, contratos y registros propios de tu organización, que jamás formaron parte del entrenamiento del modelo.
- **Datos públicos recientes** — información publicada *después* del corte de entrenamiento del modelo, que tampoco conoce.

> [!note] RAG existe para cerrar esa brecha: le da al LLM acceso, en tiempo de consulta, a datos que **no estaban en su entrenamiento**, recuperándolos y pasándolos como contexto en el prompt.

## Ventajas de RAG

El capítulo identifica **cinco ventajas** centrales de adoptar RAG:

- **Mejora de la precisión y la relevancia (improved accuracy & relevance)** — al fundamentar la respuesta en datos recuperados específicos para la consulta, las respuestas son más exactas y pertinentes que las que el modelo generaría de memoria.
- **Personalización (customization)** — permite adaptar el comportamiento del sistema a un dominio, una empresa o un usuario concretos, conectándolo a sus propias fuentes de datos.
- **Flexibilidad (flexibility)** — se pueden cambiar, ampliar o actualizar las fuentes de datos sin reentrenar el modelo; el mismo LLM sirve para distintos cuerpos de conocimiento.
- **Expansión del conocimiento más allá de los datos de entrenamiento (expanding the model's knowledge beyond the training data)** — el modelo puede responder sobre información que nunca vio durante el entrenamiento (datos privados o posteriores al corte).
- **Eliminación de alucinaciones (removing hallucinations)** — al apoyar la respuesta en evidencia recuperada, se reduce drásticamente la tendencia del modelo a inventar hechos.

> [!tip] La combinación de **precisión + conocimiento expandido + menos alucinaciones** es lo que convierte a RAG en el patrón por defecto para fundamentar IA generativa sobre datos empresariales.

## Desafíos de RAG

Contra esas ventajas, el capítulo enumera **siete desafíos** que hay que gestionar:

- **Dependencia de la calidad de los datos (dependency on data quality)** — RAG es tan bueno como los datos que recupera; datos malos producen respuestas malas.
- **Necesidad de manipulación y limpieza de datos (need for data manipulation & cleaning)** — los datos de origen casi siempre requieren preparación, normalización y limpieza antes de ser útiles.
- **Sobrecarga computacional (computational overhead)** — vectorizar, indexar, buscar y volver a generar tiene un coste de cómputo significativo.
- **Explosión del almacenamiento de datos / complejidad en integración y mantenimiento (data storage explosion / complexity in integration & maintenance)** — guardar los datos originales más sus representaciones vectoriales multiplica el almacenamiento, y mantener e integrar todos los componentes es complejo.
- **Potencial de sobrecarga de información (potential for information overload)** — recuperar demasiado contexto puede saturar al modelo y degradar la respuesta.
- **Alucinaciones (hallucinations)** — RAG las reduce pero **no las elimina por completo**; siguen siendo un riesgo residual.
- **Altos niveles de complejidad dentro de los componentes de RAG (high levels of complexity within RAG components)** — cada pieza (vectorización, vector store, búsqueda, generación) añade complejidad propia al sistema.

> [!warning] **"Garbage in, garbage out".** La calidad del output de RAG está topada por la calidad de los datos que se indexan y recuperan: si entra basura, sale basura. La limpieza y curación de datos no es opcional.

## Vocabulario de RAG

Antes de avanzar, el capítulo fija los términos que se usan a lo largo del libro.

> [!note] **[[LLM]] (Large Language Model)** — el modelo de lenguaje grande que genera el texto; el motor de la etapa de generación de RAG.

**Prompting, prompt design y prompt engineering** son tres niveles distintos:

- **Prompting** — simplemente pasarle una entrada (un prompt) al modelo.
- **Prompt design** — diseñar deliberadamente la estructura y el contenido del prompt para obtener mejores resultados.
- **Prompt engineering** — la disciplina más amplia y sistemática de construir, probar y optimizar prompts. (El libro profundiza en prompt design y prompt engineering en [[13 - Using Prompt Engineering to Improve RAG Efforts]].)

> [!note] **Inference** — el proceso de ejecutar el modelo ya entrenado para producir una salida a partir de una entrada (es decir, "usar" el modelo, no entrenarlo).

**[[LangChain]]** y **[[LlamaIndex]]** son los dos frameworks de orquestación más usados para construir aplicaciones RAG (encadenan recuperación, prompting y generación).

### Context window

> [!note] **[[Context Window]]** — el número máximo de **tokens** que un LLM puede procesar en una sola pasada (input + output). Queda **fijado en el momento del entrenamiento** y no se puede ampliar después.

Para textos más largos que la ventana, se recurre a una **sliding window** (ventana deslizante) o a **truncation** (truncamiento), descartando parte del texto. Un valor de ejemplo típico es una ventana de **4.096 tokens**. Las ventanas más grandes permiten manejar más contexto pero implican **mayor coste** por consulta.

> [!warning] **El recuento de tokens ≠ recuento de palabras.** Los tokens no son palabras: en la mayoría de los LLMs `"ice cream"` son **2 tokens**, y los tokens también incluyen puntuación, símbolos y números. Hay que estimar en tokens, no en palabras.

> [!note] **Dato interesante (Interesting fact).** **Gemini 1.5** alcanza **1.000.000 de tokens** de ventana de contexto, equivalentes a **más de 1.000 páginas** de texto.

Equivalencias aproximadas de tokens a páginas que da el capítulo:

- **4.096 tokens ≈ 5 páginas**
- **8.192 tokens ≈ 10 páginas**
- **32.768 tokens ≈ 40 páginas**

### Tabla 1.1 — Different context windows for LLMs

| Modelo | Proveedor | Ventana de contexto (tokens) |
|---|---|---|
| ChatGPT-3.5 Turbo 0613 | OpenAI | 4.096 |
| Llama 2 | Meta | 4.096 |
| Llama 3 | Meta | 8.000 |
| ChatGPT-4 | OpenAI | 8.192 |
| ChatGPT-3.5 Turbo 0125 | OpenAI | 16.385 |
| ChatGPT-4.0-32k | OpenAI | 32.000 |
| Mistral | Mistral AI | 32.000 |
| Mixtral | Mistral AI | 32.000 |
| DBRX | Databricks | 32.000 |
| Gemini 1.0 Pro | Google | 32.000 |
| ChatGPT-4.0 Turbo | OpenAI | 128.000 |
| ChatGPT-4o | OpenAI | 128.000 |
| Claude 2.1 | Anthropic | 200.000 |
| Claude 3 | Anthropic | 200.000 |
| Gemini 1.5 Pro | Google | 1.000.000 |

> [!tip] La tendencia es a ventanas cada vez más grandes: los modelos más antiguos (los de la derecha en la Figura 1.1) tenían ventanas más pequeñas, y las nuevas generaciones las amplían hasta el millón de tokens.

![[01-fig-1.1-context-windows.jpg|559]]
*Figure 1.1 – Different context windows for LLMs*

### Fine-tuning y sus variantes

> [!note] **[[Fine-tuning]]** — reentrenar un modelo base sobre datos adicionales para ajustar sus **pesos y biases**. Es un cambio **permanente** en el modelo.

El capítulo nombra las variantes:

- **FMFT (full-model fine-tuning)** — fine-tuning del modelo completo, ajustando todos sus parámetros.
- **PEFT (parameter-efficient fine-tuning)** — fine-tuning eficiente en parámetros, que ajusta solo una fracción de los pesos.
- **[[LoRA]]** — una técnica concreta de PEFT.
- **Quantization** — reducir la precisión numérica de los pesos para abaratar el cómputo y la memoria.
- **Representative fine-tuning** — fine-tuning sobre un subconjunto representativo de datos.

### Vector store vs vector database, y vectores

> [!note] **Vector store vs [[Vector Database]]** — un *vector store* es un almacén de vectores; una *vector database* es una base de datos especializada en almacenar y buscar vectores con funcionalidad completa (indexado, escalado, consultas). El capítulo distingue ambos términos. Se tratan en profundidad en [[07 - The Key Role Vectors and Vector Stores Play in RAG]].

> [!note] **[[Embeddings|Vectores / embeddings]]** — representaciones numéricas (listas de números) del significado de un dato, que permiten comparar similitud semántica. Son la columna vertebral de la recuperación en RAG.

## Vectores (en profundidad)

Un **vector** (o **embedding**) es una lista de números que captura el significado de un fragmento de datos. Un ejemplo de vector de **4 dimensiones**:

```text
[0.123, 0.321, 0.312, 0.231]
```

La **dimensionalidad** es la cantidad de números en el vector. En la práctica los embeddings tienen **cientos o miles de dimensiones**: a **mayor dimensionalidad, mayor detalle semántico** capturado.

> [!note] En la práctica los vectores se manejan como **NumPy arrays**, no como listas de Python: son **más rápidos** y son el **estándar de facto** en todo el ecosistema científico/ML — **SciPy, pandas, scikit-learn, TensorFlow, Keras y PyTorch** — porque permiten **matemática vectorizada** (operar sobre arrays enteros de una sola vez).

> [!tip] La búsqueda por similitud (similarity search) sobre estos vectores es el corazón de la etapa de retrieval; el libro la desarrolla en [[08 - Similarity Searching with Vectors]].

## Aplicaciones de RAG en la empresa

El capítulo lista **siete aplicaciones empresariales** de RAG (que se desarrollan en [[03 - Practical Applications of RAG]]):

- **Soporte al cliente y chatbots (customer support & chatbots)** — asistentes que responden con la base de conocimiento real de la empresa.
- **Soporte técnico (technical support)** — resolución de problemas técnicos fundamentada en documentación y manuales internos.
- **Generación automatizada de reportes (automated reporting)** — producir informes a partir de datos internos recuperados.
- **Soporte de e-commerce (e-commerce support)** — asistencia sobre catálogo, pedidos y políticas.
- **Aprovechamiento de bases de conocimiento (utilizing knowledge bases)** — sobre dominios como **legal, compliance, research (investigación), medical (médico), academia, patents (patentes) y technical (técnico)**.
- **Innovation scouting (búsqueda de innovación)** — rastrear y sintetizar información para detectar oportunidades de innovación.
- **Formación y educación (training & education)** — material y tutoría apoyados en el corpus de la organización.

## RAG vs IA generativa convencional

> [!note] La IA generativa convencional **"no sabe lo que no sabe" (does not know what it does not know)**: responde con la misma confianza tenga o no la información, lo que la lleva a **alucinar** cuando le falta el dato.

RAG corrige esto inyectando datos recuperados y verificables en el prompt: en vez de depender solo de lo que el modelo memorizó, le da el material concreto sobre el que basar la respuesta, y le permite **declarar que no sabe** cuando el contexto no contiene la respuesta.

## RAG vs fine-tuning

Hay **dos formas de adaptar un LLM** a tus necesidades:

- **Fine-tuning** — **ajustar los pesos y biases** del modelo con datos adicionales. Es un cambio **permanente** en el modelo.
- **Input / prompts (RAG)** — **usar** el modelo tal cual, pasándole la información en el prompt en tiempo de consulta.

> [!note] **Analogía de memoria.** El **fine-tuning** es como la **memoria a largo plazo**: el conocimiento queda incorporado de forma permanente en el modelo. El **input/RAG** es como la **memoria a corto plazo**: la información está disponible solo durante esa consulta, en el contexto.

Cada enfoque tiene su terreno:

- **Fine-tuning** es mejor para **tareas especializadas** y para fijar un **estilo/voz** del modelo, pero es **menos fiable para la recuperación factual** (recordar hechos exactos).
- **RAG** es superior para datos **factuales, privados y nuevos**: información que cambia, que es propia de la empresa, o que apareció después del entrenamiento.

> [!warning] **Trade-off del input/RAG:** la cantidad de información que se puede pasar por prompt está **limitada por la [[Context Window|ventana de contexto]]**. Y al hacer fine-tuning hay que cuidar el **overfitting**. Ambos enfoques **no son excluyentes** y conviene tener presentes a la vez los límites de overfitting (fine-tuning) y de context window (RAG).

## La arquitectura de los sistemas RAG

Desde la **perspectiva del usuario**, RAG son **tres pasos**: el usuario hace una pregunta, el sistema recupera información relevante, y el modelo responde usando esa información. Por debajo, son **tres etapas técnicas**:

> [!note] Las tres etapas de RAG: **[[Indexing]]** · **[[Retrieval]]** · **[[Generation]]**. El **indexing se hace ANTES** de que llegue la consulta (se preparan y vectorizan los datos por adelantado); retrieval y generation ocurren en el momento de la query. (Las tres etapas se desarrollan con código en [[04 - Components of a RAG System]].)

![[01-fig-1.2-three-stages-of-rag.jpg]]
*Figure 1.2 – The three stages of RAG*

Una vez que llega la consulta, el flujo técnico detallado son **siete pasos**:

1. **Vectorizar la consulta** — convertir la query del usuario en un vector/embedding.
2. **Búsqueda vectorial (vector search)** — buscar en el vector store los vectores más similares al de la consulta.
3. **Devolver los resultados relevantes + claves únicas** — la búsqueda retorna los resultados relevantes junto con sus **claves únicas** identificadoras.
4. **Recuperar los datos originales por sus claves** — usar esas claves para traer el dato original completo (el texto fuente, no solo el vector).
5. **Filtrar / post-procesar** — filtrar y post-procesar los resultados recuperados.
6. **Pasar al LLM con el prompt de asistente** — entregar la consulta + la información recuperada al LLM dentro de un prompt como el siguiente.
7. **El LLM responde** — el modelo genera la respuesta fundamentada en la información provista.

El prompt de asistente que el capítulo reproduce:

```text
You are a helpful assistant for question-answering tasks. Take the following question (the user query) and use this helpful information (the data retrieved in the similarity search) to answer it. If you don't know the answer based on the information provided, just say you don't know.
```

> [!tip] Ese prompt encapsula la disciplina anti-alucinación de RAG: usar **solo** la información recuperada y **admitir el desconocimiento** ("just say you don't know") cuando el contexto no alcanza.

## Pruebas de recuperación en contexto largo

El capítulo menciona dos pruebas usadas para evaluar qué tan bien un LLM aprovecha su ventana de contexto:

- **Needle-in-a-haystack / multiple-needles** — insertar un dato (o varios) en un texto largo y comprobar si el modelo lo recupera.

> [!warning] **"Lost in the middle".** Aunque la ventana de contexto sea grande, los detalles ubicados **en la mitad** de un contexto largo tienden a **perderse**: el modelo presta más atención al principio y al final que al centro. Una ventana grande no garantiza que se use bien toda la información.

## Citas

> "You are a helpful assistant for question-answering tasks. Take the following question (the user query) and use this helpful information (the data retrieved in the similarity search) to answer it. If you don't know the answer based on the information provided, just say you don't know."

## Para aplicar

- **Elegí RAG vs fine-tuning según el tipo de dato** — RAG para hechos, datos privados y datos nuevos/cambiantes; fine-tuning para tareas especializadas y voz/estilo. No son excluyentes.
- **Curá los datos antes de indexarlos** — la calidad del output está topada por la calidad de los datos ("garbage in, garbage out"); planificá limpieza y manipulación.
- **Estimá en tokens, no en palabras** — usá las equivalencias (4.096≈5 págs, 8.192≈10 págs, 32.768≈40 págs) y recordá que `"ice cream"` = 2 tokens.
- **Dimensioná la context window** — más grande = más contexto pero más coste; y cuidado con el "lost in the middle" en contextos largos.
- **Usá NumPy arrays para los vectores** — son el estándar de facto y habilitan matemática vectorizada más rápida que las listas de Python.
- **Diseñá el prompt anti-alucinación** — instruí explícitamente al LLM a usar solo la información recuperada y a decir "no lo sé" cuando no alcance.

## Conexiones

- [[_Unlocking Data with Generative AI and RAG|Libro]] — el MOC del libro.
- [[02 - Code Lab - An Entire RAG Pipeline]] — capítulo siguiente: el walkthrough de código que construye un pipeline RAG básico.
- [[03 - Practical Applications of RAG]] — profundiza en las 7 aplicaciones empresariales.
- [[04 - Components of a RAG System]] — las etapas indexing/retrieval/generation con código.
- [[07 - The Key Role Vectors and Vector Stores Play in RAG]] — vectores, embeddings y vector databases en profundidad.
- [[08 - Similarity Searching with Vectors]] — la búsqueda por similitud que sustenta el retrieval.
- [[13 - Using Prompt Engineering to Improve RAG Efforts]] — prompt design y prompt engineering.
- [[14 - Advanced RAG-Related Techniques for Improving Results]] — modelos generativos de imágenes, audio y video.
- [[RAG]] · [[LLM]] · [[Embeddings]] · [[Context Window]] · [[Fine-tuning]] · [[Vector Database]] · [[Hallucinations]] · [[LangChain]] · [[LlamaIndex]] · [[Indexing]] · [[Retrieval]] · [[Generation]] · [[LoRA]] — conceptos núcleo del capítulo (candidatos a nota propia si aún no existen en el vault).
