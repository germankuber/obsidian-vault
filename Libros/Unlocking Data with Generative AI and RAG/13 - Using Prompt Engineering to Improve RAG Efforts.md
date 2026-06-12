---
title: 13 - Using Prompt Engineering to Improve RAG Efforts
libro: Unlocking Data with Generative AI and RAG
autor: Keith Bourne
capitulo: 13
created: 2026-06-12
tags:
  - libros/unlocking-data-with-generative-ai-and-rag
  - type/case-study
  - status/permanent
aliases:
  - Using Prompt Engineering to Improve RAG Efforts
  - Prompt Engineering para RAG
updated: 2026-06-12
---

# 13 - Using Prompt Engineering to Improve RAG Efforts

> [!info] Capítulo 13 · *Unlocking Data with Generative AI and RAG* — Keith Bourne (Packt, ISBN 9781835887905)
> El [[Prompt|prompt]] es **THE key element** de cualquier app de IA generativa / RAG, y el [[Prompt Engineering|prompt engineering]] es la formulación y refinamiento estratégico de los prompts para mejorar la recuperación y la generación. El capítulo recorre los **parámetros del prompt** ([[Temperature|temperature]], [[Top-p|top-p]], [[Seed|seed]]), los enfoques **no/single/few/multi-shot**, las 8 técnicas de [[Prompt Design|diseño]] y 6 fundamentos, cómo **adaptar prompts a distintos LLMs** ([[Claude]] XML, [[Llama 3]] SYS/INST) y dos code labs (un [[Prompt Template|PromptTemplate]] custom + 13 prompts que iteran, resumen, infieren, transforman y expanden). Navegá: [[_Unlocking Data with Generative AI and RAG|Libro]] · anterior [[12 - Combining RAG with the Power of AI Agents and LangGraph]] · siguiente [[14 - Advanced RAG-Related Techniques for Improving Results]].

## Resumen

El capítulo parte de una premisa fuerte: el **prompt es el elemento clave** de toda aplicación de IA generativa y, por extensión, de todo RAG — porque la calidad de lo que recupera y genera el sistema depende directamente de cómo se le pide. El **[[Prompt Engineering|prompt engineering]]** se define como la **formulación y el refinamiento estratégico de los prompts de entrada** para mejorar tanto el retrieval como la generation. Sobre esa base, el capítulo se estructura en cuatro grandes bloques. Primero, los **parámetros del prompt** que más impactan en RAG: **[[Temperature|temperature]]** (grado de aleatoriedad sobre la distribución de probabilidad del siguiente token), **[[Top-p|top-p]]** (aleatoriedad focalizada sobre una porción de la distribución) y **[[Seed|seed]]** (reproducibilidad de la secuencia aleatoria, junto al campo de respuesta `system_fingerprint`). Segundo, los fundamentos conceptuales: los enfoques **no-shot / single-shot / few-shot / multi-shot** (un "shot" = un ejemplo que se le da al LLM), y la distinción entre **prompting**, **prompt design** y **prompt engineering** retomada del [[01 - What Is Retrieval-Augmented Generation (RAG)|capítulo 1]]. Tercero, las **8 técnicas de diseño** (shot design, chain-of-thought, personas, chain of density, tree of thoughts, graph prompting, knowledge augmentation, show-me/tell-me) y los **6 fundamentos del diseño** (ser conciso y específico, una tarea por vez, convertir tareas generativas en clasificación, incluir ejemplos, empezar simple e iterar, poner las instrucciones al inicio), más la advertencia de que los prompts **no se transfieren sin más entre LLMs** ([[Claude]] prefiere XML, [[Llama 3]] usa SYS/INST). Cuarto, dos code labs prácticos: el **13.1** construye un [[Prompt Template|PromptTemplate]] custom (con un patrón de [[Personas|personas]]) reemplazando el prompt estándar del [[LangChain Hub]], y el **13.2** recorre cinco conceptos de prompting — **iterar, resumir, inferir, transformar y expandir** — sobre un escenario de marketing, con 13 plantillas y sus salidas. El capítulo cierra señalando que el siguiente y **último** capítulo aborda técnicas más avanzadas para mejorar RAG.

## Parámetros del prompt

Hay tres parámetros que impactan directamente en cómo se comporta el LLM dentro de un sistema RAG: **temperature**, **top-p** y **seed**. Los tres regulan el balance entre **determinismo** y **aleatoriedad** de la salida.

### Temperature

Un [[LLM|LLM]] predice el siguiente token a partir de una **distribución de probabilidad**. La **[[Temperature|temperature]]** dicta qué tan probable es que el modelo elija palabras más abajo en esa distribución, es decir, el **grado de aleatoriedad**. Es opcional, su valor por defecto es **1**, y su rango va de **0 a 2**: más alto = más aleatorio, más bajo = menos.

> [!note] Simple temperature example
> Para la frase `The dog ran`, la distribución del siguiente token podría ser:
> `P("next word" | "The dog ran") = {"down": 0.4, "to" : 0.3, "with": 0.2, "away": 0.1}`
> (suma 1). La palabra más probable es `down`, la segunda `to`; pero `away` igual puede aparecer. Con **temperature 0** el modelo solo elige la palabra más probable; con **temperature 2** es mucho más probable que elija palabras menos probables.

Desde el principio del libro veníamos usando `temperature=0` para obtener resultados de laboratorio **predecibles y reproducibles**:

```python
llm = ChatOpenAI(model_name="gpt-4o-mini", temperature=0)
```

> [!tip] Subí la temperature cuando quieras salida más **creativa** (brainstorming, redacción), y bajala (hacia 0) cuando necesites respuestas factuales, deterministas y reproducibles — lo habitual en RAG.

### Top-p

El **[[Top-p|top-p]]** también introduce aleatoriedad, pero **apunta a una parte específica** de la distribución. Con la misma distribución del ejemplo anterior, un **top-p 0.7** considera solo las primeras dos palabras (0.4 + 0.3 = 0.7 de la distribución). Es opcional y su valor por defecto es **1** (todas las opciones).

> [!warning] Se recomienda usar **o temperature o top-p, NO ambos**. Combinarlos produce una complejidad impredecible.

### Tabla 13.1 — Showing the type of outcomes from each LLM parameter

| Parámetro | Tipo de resultado |
|---|---|
| Temperature | General randomness |
| Top-p | Focused randomness |
| Temperature + top-p | Unpredictable complexity |

> [!note] La tabla resume el porqué de la recomendación: temperature da aleatoriedad **general**, top-p la **focaliza**, y combinarlos lleva a **complejidad impredecible**.

El `top_p` se pasa vía `model_kwargs`:

```python
llm = ChatOpenAI(model_name="gpt-4o-mini", model_kwargs={"top_p": 0.5})
```

`model_kwargs` es la forma de pasar parámetros que **no están integrados en LangChain** pero sí existen en la API subyacente del LLM. Conviene revisar la documentación de cada API para saber qué parámetros admite.

### Seed

Por defecto, las respuestas de un LLM son **no deterministas**. OpenAI (y otros) ofrecen el parámetro **[[Seed|seed]]** más el campo de respuesta **`system_fingerprint`** para obtener salidas (mayormente) deterministas. Un seed produce la **misma secuencia aleatoria** cada vez; se le asigna cualquier entero y funciona incluso junto con temperature o top-p.

> [!note] Los resultados **igual pueden diferir** aunque uses el mismo seed: la API/servicio cambia con el tiempo. Para diagnosticarlo, compará el `system_fingerprint` entre llamadas — si cambió (con el mismo seed), la salida puede diferir por cambios del sistema de OpenAI.

También se pasa vía `model_kwargs`:

```python
optional_params = {
  "top_p": 0.5, "seed": 42
}
llm = ChatOpenAI(model_name="gpt-4o-mini", model_kwargs=optional_params)
```

## Take your shot

Un **"shot"** es **un EJEMPLO** que le das al LLM para guiar su respuesta. Según cuántos ejemplos des:

- **no-shot** — ningún ejemplo.
- **single-shot** — un ejemplo.
- **few-shot / multi-shot** — más de un ejemplo.

Cada shot equivale a un ejemplo. El ejemplo de **single-shot**:

```
"Give me a joke that uses an animal and some action that animal takes that is funny.
Use this example to guide the joke you provide:
Joke-question: Why did the chicken cross the road?
Joke-answer: To get to the other side."
```

En RAG con frecuencia provees ejemplos **en el contexto** — aunque no siempre, porque a veces el contexto es solo data. Si das ejemplos de pregunta/respuesta para dirigir al LLM, eso es un enfoque de shot; algunas apps de RAG siguen de cerca el patrón **multi-shot**.

## Prompting, prompt design y prompt engineering (revisitado)

Retomando el vocabulario del [[01 - What Is Retrieval-Augmented Generation (RAG)|capítulo 1]]:

- **[[Prompting|Prompting]]** — enviar una query/prompt a un LLM.
- **[[Prompt Design|Prompt design]]** — la **estrategia** para diseñar el prompt.
- **[[Prompt Engineering|Prompt engineering]]** — los **aspectos técnicos** para mejorar las salidas (por ejemplo, partir una query compleja en 2–3 interacciones con el LLM).

El foco del capítulo está en el **design** y el **engineering**.

## Diseño vs engineering: las 8 técnicas

Los enfoques de shot pertenecen al **prompt design**; hidratar el prompt template con pregunta + contexto pertenece al **prompt engineering**. Hay mucho solapamiento y a menudo se usan de forma intercambiable.

> [!note] La distinción del autor: el **prompt engineering es más amplio** — engloba el prompt design **MÁS** la optimización / fine-tuning de toda la interacción usuario↔modelo.

Las técnicas de **[[Prompt Design|diseño]] de prompts**:

- **[[Shot Design|Shot design]]** — el punto de partida; se elabora el prompt inicial con ejemplos y se mezcla con otros patrones.
- **[[Chain-of-thought|Chain-of-thought prompting]]** — partir problemas complejos en pasos más chicos, pidiendo razonamiento intermedio en cada paso → respuestas mejores y más precisas.
- **[[Personas|Personas (role prompting)]]** — un personaje ficticio con nombre, ocupación, demografía, historia y pain points → salida relevante, útil y consistente con la audiencia, con más personalidad y estilo.
- **[[Chain of Density|Chain of density (summarization)]]** — asegurar una summarización correcta: que no se deje afuera información vital pero que sea concisa; usa **densidad de entidades** mientras el LLM itera, garantizando que se incluyan las entidades más importantes.
- **[[Tree of Thoughts|Tree of thoughts (exploration over thoughts)]]** — del prompt inicial salen múltiples opciones de pensamiento; iterativamente se selecciona la mejor para generar la siguiente ronda → exploración diversa y comprehensiva.
- **[[Graph Prompting|Graph prompting]]** — framework nuevo para data con estructura de grafo; entiende/genera en base a las relaciones entre entidades de un grafo.
- **[[Knowledge Augmentation|Knowledge augmentation]]** — aumentar los prompts con información relevante adicional; por ejemplo vía RAG, incorporando conocimiento externo.
- **[[Show Me vs Tell Me|Show Me vs Tell Me prompts]]** — *Show Me* = ejemplos/demostraciones; *Tell Me* = instrucciones/documentación explícitas; usar **ambos** = flexibilidad + precisión.

## Fundamentos del diseño de prompts

Seis fundamentos prácticos, cada uno con su antes/después:

- **Be concise and specific** — bad / good:

```
bad:  Please analyze the given context and provide an answer to the question, taking into account all the relevant information and details
good: Based on the context provided, answer the following question: [specific question]
```

- **Ask one task at a time** — en vez de un prompt que hace tres cosas, partilo en tres prompts encadenados:

```
bad:    Summarize the main points of the context, identify the key entities mentioned, and then answer the given question

better:
1. Summarize the main points of the following context: [context]
2. Identify the key entities mentioned in the following summary: [summary from previous prompt]
3. Using the context and entities identified, answer the following question: [specific question]
```

> [!tip] **Una tarea por prompt.** Encadenar prompts simples produce salidas más controlables y precisas que un único prompt sobrecargado.

- **Turn generative tasks into classification tasks** — acotá la salida a categorías:

```
bad:  Based on the context, what is the sentiment expressed towards the topic?
good: Based on the context, classify the sentiment expressed towards the topic as either positive, negative, or neutral
```

- **Improve response quality by including examples** — bad / good:

```
bad:  Answer the following question based on the provided context
good: Using the examples below as a guide, answer the following question based on the provided context:
      Example 1: [question] [context] [answer]
      Example 2: [question] [context] [answer]
      Current question: [question]
      Context: [context]
```

- **Start simple and iterate gradually** — en vez de un prompt gigante, iterá:

```
Iteration 1: Summarize the main points of the following article: [article text]
Iteration 2: Summarize the main points and identify key entities in the following article: [article text]
Iteration 3: Based on the summary and key entities, answer the following question: [question] Article: [article text]
```

- **Place instructions at the beginning of the prompt** — usá separadores claros como `###`:

```
bad:  [Context] Please use the above context to answer the following question: [question]. Provide your answer in a concise manner
good: Instructions: Using the context provided below, answer the question in a concise manner.
      Context: [context]
      Question: [question]
```

> [!tip] Poné las **instrucciones al inicio** y separalas del contexto con marcadores claros (`###`). Le da al modelo una estructura inequívoca de qué hacer antes de leer la data.

## Adaptar prompts a distintos LLMs

No todo es OpenAI: están **[[Claude|Anthropic Claude]]** (ventanas de contexto largas), los modelos de Google, y open source como **[[Llama 3|Llama]]**.

> [!warning] Los prompts **NO se transfieren sin más** entre LLMs. Un prompt afinado para OpenAI puede rendir distinto en otro modelo.

- **[[Claude|Claude-3]]** prefiere **encoding XML**.
- **[[Llama 3|Llama3]]** usa la sintaxis **SYS** e **INST**.

Ejemplo para Llama:

```
<SYS> You are an AI assistant created to provide helpful and informative responses to user questions. </SYS>
<INST> Analyze the user's question below and provide a clear, concise answer using your knowledge base. If the question is unclear, ask for clarification.
User's question: "What are the main advantages of using renewable energy sources compared to fossil fuels?" </INST>
```

Donde **SYS** = system / system message e **INST** = instructions.

> [!tip] Adaptá el prompt al LLM destino: XML para Claude, SYS/INST para Llama. No reutilices ciegamente un prompt de un proveedor en otro.

## Code lab 13.1 – Template custom

Notebook `CHAPTER13-1_PROMPT_TEMPLATES.ipynb` (construye sobre el lab 8.3 del [[08 - Similarity Searching with Vectors|capítulo 8]]). La clase **[[Prompt Template|PromptTemplate]]** gestiona los prompts en [[LangChain]]. El template más usado a lo largo del libro es el del [[LangChain Hub]]:

```python
prompt = hub.pull("jclemens24/rag-prompt")
```

Al imprimirlo, el texto es:

```
You are an assistant for question-answering tasks. Use the following pieces of retrieved context to answer the question. If you don't know the answer, just say that you don't know.
Question: {question}
Context: {context}
Answer:
```

Y el objeto completo:

```python
ChatPromptTemplate(input_variables=['context', 'question'], metadata={'lc_hub_owner': 'jclemens24', 'lc_hub_repo': 'rag-prompt', 'lc_hub_commit_hash': '1a1f3ccb9a5a92363310e3b130843dfb2540239366ebe712ddd94982acc06734'}, messages=[HumanMessagePromptTemplate(prompt=PromptTemplate(input_variables=['context', 'question'], template="You are an assistant for question-answering tasks. Use the following pieces of retrieved context to answer the question. If you don't know the answer, just say that you don't know.\nQuestion: {question} \nContext: {context} \nAnswer:"))])
```

Es un **`ChatPromptTemplate`** (pensado para escenarios de chat); sus `input_variables` son `context` + `question`. Para crear el custom se importa:

```python
from langchain_core.prompts import PromptTemplate
```

Y se reemplaza `prompt = hub.pull("jclemens24/rag-prompt")` por un template propio (usa un patrón de **[[Personas|personas]]**, un "environment expert"):

```python
prompt = PromptTemplate.from_template(
    """
    You are an environment expert assisting others in
    understanding what large companies are doing to
    improve the environment. Use the following pieces
    of retrieved context with information about what
    a particular company is doing to improve the
    environment to answer the question.
    If you don't know the answer, just say that you don't know.
    Question: {question}
    Context: {context}
    Answer:"""
)
```

Los prompt templates toman un **dict** (cada clave = una variable a rellenar); la salida es un **`PromptValue`** (que se pasa a un LLM/ChatModel o como paso de una cadena [[LCEL]]). `print(prompt)` muestra los `input_variables` + el `template`; `print(prompt.template)` muestra solo el texto.

El prompt de relevancia ya existente (del patrón "segundo LLM" del [[05 - Managing Security in RAG Applications|capítulo 5]]):

```python
relevance_prompt_template = PromptTemplate.from_template(
    """
    Given the following question and retrieved context, determine if the context is relevant to the question.
    Provide a score from 1 to 5, where 1 is not at all relevant and 5 is highly relevant.
    Return ONLY the numeric score, without any additional text or explanation.
    Question: {question}
    Retrieved Context: {retrieved_context}
    Relevance Score:"""
)
```

> [!note] Esto es un **String prompt template** (string plano + placeholders). La alternativa es el **`ChatPromptTemplate`**, que formatea una lista de mensajes.

## Code lab 13.2 – Opciones de prompting

Notebook `CHAPTER13-2_PROMPT_OPTIONS.ipynb`. Cubre cinco conceptos — **iterar, resumir, transformar, expandir** (+ inferencia) — sobre un escenario del departamento de **marketing**. A continuación las 13 plantillas (prompt2–prompt13) y sus salidas.

### Iterar

Rara vez el primer prompt es el mejor.

**Iterar el tono** — `prompt2` (descripción de marketing, contexto entre triple backticks):

```python
prompt2 = PromptTemplate.from_template(
    """Your task is to help a marketing team create a
    description for the website about the environmental
    initiatives our clients are promoting.
    Write a marketing description based on the information
    provided in the context delimited by triple backticks.
    If you don't know the answer, just say that you don't know.
    Question: {question}
    Context: ```{context}```
    Answer:"""
)
```

Para usarlo, se cambia el prompt de la cadena a `prompt2` (en `rag_chain_from_docs`):

```python
    | prompt2 # <- update here
```

Salida (descripción orientada a marketing; excerpt):

```
Google is at the forefront of environmental sustainability… Empowering Individuals: …1 billion people… eco-friendly routing in Google Maps… By 2030… 1 gigaton… [TRUNCATED]… a more sustainable future.
```

Pero debe caber en una caja de **50 words** en el sitio.

**Acortar el largo** — `prompt3` (agrega `Use at most 50 words.`):

```python
prompt3 = PromptTemplate.from_template(
    """Your task is to help a marketing team create a
    description for the website about the environmental
    initiatives our clients are promoting.
    Write a marketing description based on the information
    provided in the context delimited by triple backticks.
    If you don't know the answer, just say that you don't know.
    Use at most 50 words.
    Question: {question}
    Context: ```{context}```
    Answer:"""
)
```

Salida:

```
Google's environmental initiatives include promoting electric vehicles, sustainable agriculture, net-zero carbon operations, water stewardship, and a circular economy. They aim to help individuals and partners reduce carbon emissions, optimize resource use, and support climate action through technology and data-driven solutions.
```

**Cambiar el foco** — `prompt4` (audiencia técnica, sin límite de largo):

```python
prompt4 = PromptTemplate.from_template(
    """Your task is to help a marketing team create a
    description for the website about the environmental
    initiatives our clients are promoting.
    Write a marketing description based on the information
    provided in the context delimited by triple backticks.
    The description is intended for a technology audience,
    so this should focus on only the aspects of the
    company's efforts that relate to using technology. If
    you don't know the answer, just say that you don't
    know.
    Question: {question}
    Context: ```{context}```
    Answer:"""
)
```

Salida (excerpt):

```
Google is at the forefront of leveraging technology… Eco-Friendly Product Features: Google Maps… Google Nest… Google Flights… [TRUNCATED]… Sustainability-Focused Accelerators: Google for Startups Accelerator…
```

### Resumir

**Summarizing** — `prompt5` (resumen, a lo sumo 30 words):

```python
prompt5 = PromptTemplate.from_template(
    """Your task is to generate a short summary of what a
    company is doing to improve the environment.
    Summarize the retrieved context below, delimited by
    triple backticks, in at most 30 words.
    If you don't know the answer, just say that you don't
    know.
    Question: {question}
    Context: ```{context}```
    Answer:"""
)
```

Salida:

```
Google's environmental initiatives include achieving net-zero carbon, promoting water stewardship, supporting a circular economy, and leveraging technology to help partners reduce emissions.
```

**Resumir con un foco** — `prompt6` (foco en la eco-friendliness de los productos):

```python
prompt6 = PromptTemplate.from_template(
    """Your task is to generate a short summary of what a
    company is doing to improve the environment.
    Summarize the retrieved context below, delimited by
    triple backticks, in at most 30 words, and focusing
    on any aspects that mention the eco-friendliness of
    their products.
    If you don't know the answer, just say that you don't
    know.
    Question: {question}
    Context: ```{context}```
    Answer:"""
)
```

Salida:

```
Google's environmental initiatives include eco-friendly routing in Google Maps, energy-efficient Google Nest thermostats, and carbon emissions information in Google Flights.
```

**Extraer en vez de resumir** — `prompt7` (usar *extract* en vez de *summarize* para evitar info no deseada):

```python
prompt7 = PromptTemplate.from_template(
    """Your task is to generate a short summary of what a
    company is doing to improve the environment.
    From the retrieved context below, delimited by
    triple backticks, extract the information focusing
    on any aspects that mention the eco-friendliness of
    their products. Limit to 30 words.
    If you don't know the answer, just say that you don't
    know.
    Question: {question}
    Context: ```{context}```
    Answer:"""
)
```

Salida:

```
Google's environmental initiatives include eco-friendly routing in Google Maps, energy efficiency features in Google Nest thermostats, and carbon emissions information in Google Flights to help users make sustainable choices.
```

> [!tip] Cambiar **"summarize" por "extract"** acota la salida a lo que está literalmente en el contexto y evita que el modelo agregue información de su propio conocimiento.

### Inferir

**Inference** — `prompt8` (extract + sentiment booleano positive/negative):

```python
prompt8 = PromptTemplate.from_template(
    """Your task is to generate a short summary of what a
    company is doing to improve the environment.
    From the retrieved context below, delimited by
    triple backticks, extract the information focusing
    on any aspects that mention the eco-friendliness of
    their products. Limit to 30 words.
    After this summary, determine what the sentiment
    of context is, providing your answer as a single word,
    either "positive" or "negative". If you don't know the
    answer, just say that you don't know.
    Question: {question}
    Context: ```{context}```
    Answer:"""
)
```

Salida:

```
Google is enhancing eco-friendliness through features like eco-friendly routing in Maps, energy-efficient Nest thermostats, and carbon emissions data in Flights, aiming to reduce emissions significantly.
Sentiment: positive
```

**Extraer data clave** — `prompt9` (lista de productos relacionados con `Related products: `):

```python
prompt9 = PromptTemplate.from_template(
    """Your task is to generate a short summary of what a
    company is doing to improve the environment.
    From the retrieved context below, delimited by
    triple backticks, extract the information focusing
    on any aspects that mention the eco-friendliness of
    their products. Limit to 30 words.
    After this summary, determine any specific products
    that are identified in the context below, delimited
    by triple backticks.  Indicate that this is a list
    of related products with the words 'Related products: '
    and then list those product names after those words.
    If you don't know the answer, just say that you don't
    know.
    Question: {question}
    Context: ```{context}```
    Answer:"""
)
```

Salida:

```
Google is enhancing eco-friendliness through products like eco-friendly routing in Google Maps, energy efficiency features in Google Nest thermostats, and carbon emissions information in Google Flights.
Related products: Google Maps, Google Nest thermostats, Google Flights
```

**Inferir topics** — `prompt10` (ocho topics, con `Related topics: `):

```python
prompt10 = PromptTemplate.from_template(
    """Your task is to generate a short summary of what a
    company is doing to improve the environment.
    From the retrieved context below, delimited by
    triple backticks, extract the information focusing
    on any aspects that mention the eco-friendliness of
    their products. Limit to 30 words.
    After this summary, determine eight topics that are
    being discussed in the context below delimited
    by triple backticks.
    Make each item one or two words long.
    Indicate that this is a list of related topics
    with the words 'Related topics: '
    and then list those topics after those words.
    If you don't know the answer, just say that you don't
    know.
    Question: {question}
    Context: ```{context}```
    Answer:"""
)
```

Salida (el summary +):

```
Related topics:
1. Electric vehicles
2. Net-zero carbon
3. Water stewardship
4. Circular economy
5. Supplier engagement
6. Climate resilience
7. Renewable energy
8. AI for sustainability
```

### Transformar

**Transformation**: tomar data y llevarla a otro estado/formato (traducción, JSON/HTML, ortografía/gramática).

**Transformación de lenguaje (traducción)** — `prompt11` (Spanish, French, English Pirate):

```python
prompt11 = PromptTemplate.from_template(
    """Your task is to generate a short summary of what a
    company is doing to improve the environment.
    From the retrieved context below, delimited by
    triple backticks, extract the information focusing
    on any aspects that mention the eco-friendliness of
    their products. Limit to 30 words.
    Translate the summary into three additional languages,
    Spanish, French, and English Pirate:
    labeling each language with a format like this:
    English: [summary]
    Spanish: [summary]
    French: [summary]
    English pirate: [summary]
    If you don't know the answer, just say that you don't
    know.
    Question: {question}
    Context: ```{context}```
    Answer:"""
)
```

Salida (las 4 líneas, verbatim para English y English pirate):

```
English: Google enhances eco-friendliness through features like eco-friendly routing in Maps, energy-efficient Nest thermostats, and carbon emissions info in Flights, helping reduce carbon emissions significantly.
Spanish: [summary en español]
French: [summary en francés]
English pirate: Google be makin' things greener with eco-routes in Maps, energy-savin' Nest thermostats, and carbon info in Flights, helpin' to cut down on carbon emissions mightily.
```

**Transformación de tono** — `prompt12` (email, tono friendly/casual):

```python
prompt12 = PromptTemplate.from_template(
    """Your task is to generate a short summary of what a
    company is doing to improve the environment.
    From the retrieved context below, delimited by
    triple backticks, extract the information focusing
    on any aspects that mention the eco-friendliness of
    their products. Limit to 30 words.
    After providing the summary, translate the summary
    into an email format with a more friendly and
    casual tone. If you don't know the answer, just say
    that you don't know.
    Question: {question}
    Context: ```{context}```
    Answer:"""
)
```

Salida (summary + excerpt del email):

```
Email Format:
Subject: Exciting Eco-Friendly Features from Google!
Hi [Recipient's Name],
I hope you're doing well!… Best, [Your Name]
```

### Expandir

**Expansion**: el reverso de la summarización — tomar poca data y expandirla.

**Expandir un texto corto** — `prompt13` (expandir el summary para inversores, usando SOLO el summary como base):

```python
prompt13 = PromptTemplate.from_template(
    """Your task is to generate a short summary of what a
    company is doing to improve the environment.
    From the retrieved context below, delimited by
    triple backticks, extract the information focusing
    on any aspects that mention the eco-friendliness of
    their products. Limit to 30 words.
    After providing the summary, provide a much longer
    description of what the company is doing to improve
    the environment, using only the summary you have
    generated as the basis for this description. If you
    don't know the answer, just say that you don't know.
    Question: {question}
    Context: ```{context}```
    Answer:"""
)
```

Salida (excerpt):

```
Summary: Google offers eco-friendly routing in Google Maps…
Broader Description: Google is actively enhancing the eco-friendliness of its products… For investors, this focus on sustainability can be a significant value proposition… positions Google as a leader in environmental responsibility…
```

> [!note] Los cinco conceptos de prompt design del lab 13.2: **iteration, summarization, inference, transformation, expansion**.

## Citas

> "Give me a joke that uses an animal and some action that animal takes that is funny. Use this example to guide the joke you provide: Joke-question: Why did the chicken cross the road? Joke-answer: To get to the other side."

> "P("next word" | "The dog ran") = {"down": 0.4, "to" : 0.3, "with": 0.2, "away": 0.1}"

> "Google be makin' things greener with eco-routes in Maps, energy-savin' Nest thermostats, and carbon info in Flights, helpin' to cut down on carbon emissions mightily."

## Para aplicar

- **Elegí UN parámetro de aleatoriedad** — usá [[Temperature|temperature]] **o** [[Top-p|top-p]], nunca ambos (Tabla 13.1). Para RAG factual, `temperature=0`.
- **Reproducibilidad con seed** — pasá `seed` por `model_kwargs` y compará el **`system_fingerprint`** entre corridas para detectar cambios del sistema del proveedor.
- **Una tarea por prompt** — partí los prompts complejos en pasos encadenados; empezá simple e iterá.
- **Instrucciones al inicio con separadores** (`###`) — y convertí tareas generativas en clasificación cuando puedas acotar la salida.
- **"extract" > "summarize"** — usá *extract* para que el modelo no agregue conocimiento propio.
- **Adaptá el prompt al LLM** — XML para [[Claude]], SYS/INST para [[Llama 3]]; los prompts no se transfieren sin más.
- **Personalizá el [[Prompt Template|PromptTemplate]]** — reemplazá el prompt genérico del [[LangChain Hub]] por uno con [[Personas|personas]] adaptado a tu dominio.
- **Caja de herramientas de prompting** — iterar / resumir / inferir / transformar / expandir como repertorio para refinar la salida de RAG.

## Conexiones

- [[_Unlocking Data with Generative AI and RAG|Libro]] — el MOC del libro.
- [[12 - Combining RAG with the Power of AI Agents and LangGraph]] — capítulo anterior (RAG agéntico con LangGraph) · [[14 - Advanced RAG-Related Techniques for Improving Results]] — capítulo siguiente y **final** (técnicas avanzadas / modelos multimodales).
- [[01 - What Is Retrieval-Augmented Generation (RAG)]] — de ahí viene el vocabulario prompting / prompt design / prompt engineering.
- [[08 - Similarity Searching with Vectors]] — el lab 13.1 construye sobre su lab 8.3.
- [[11 - Using LangChain to Get More from RAG]] — PromptTemplate y output parsers.
- [[05 - Managing Security in RAG Applications]] — el patrón del relevance prompt (segundo LLM que puntúa relevancia).
- **[[Prompt]]** · **[[Prompt Engineering]]** · **[[Prompt Design]]** · **[[Prompting]]** · **[[Prompt Template]]** — el núcleo del capítulo.
- **[[Temperature]]** · **[[Top-p]]** · **[[Seed]]** — los parámetros de aleatoriedad/determinismo.
- **[[Few-shot]]** · **[[Shot Design]]** · **[[Chain-of-thought]]** · **[[Personas]]** · **[[Chain of Density]]** · **[[Tree of Thoughts]]** · **[[Graph Prompting]]** · **[[Knowledge Augmentation]]** · **[[Show Me vs Tell Me]]** — las técnicas de diseño.
- **[[Summarization]]** · **[[Inference]]** · **[[Transformation]]** — los conceptos del lab 13.2.
- [[Claude]] · [[Llama 3]] · [[LLM]] · [[LCEL]] · [[LangChain Hub]] · [[LangChain]] — modelos y orquestación tocados en el capítulo.
