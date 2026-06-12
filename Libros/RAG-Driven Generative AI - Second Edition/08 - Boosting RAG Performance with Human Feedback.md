---
title: 08 - Boosting RAG Performance with Human Feedback
libro: RAG-Driven Generative AI - Second Edition
autor: Denis Rothman
capitulo: 8
created: 2026-06-11
tags:
  - libros/rag-driven-generative-ai
  - type/case-study
  - status/permanent
aliases:
  - Boosting RAG Performance with Human Feedback
  - Hybrid-Adaptive RAG
updated: 2026-06-11
---

# 08 - Boosting RAG Performance with Human Feedback

> [!info] Capítulo 8 · *RAG-Driven Generative AI - Second Edition* — Denis Rothman
> En 2026 conectar a un LLM es commodity, pero el barrier de *valor* subió: los usuarios rechazan respuestas genéricas aunque sean correctas. El **human feedback (HF)** lleva el modelo de competencia genérica a **alineación experta**. Se construye un **hybrid-adaptive RAG** desde cero (Retriever/Generator/Evaluator) con un **ranking simulation (1-5)** que conmuta dinámicamente la estrategia: rank 1-2 sin RAG (respuesta nativa), 3-4 inyecta feedback experto, 5 RAG estándar. Caso de uso: *Company C* (soporte de C-phone). Navegá: [[_RAG-Driven Generative AI - Second Edition|Mapa del libro]] · anterior [[07 - Empowering AI Models by Fine-Tuning RAG Data]] · siguiente [[09 - Building a Conversational RAG Agent]].

## Resumen

Para 2026 conectar a un LLM es **commodity** (la barrera técnica colapsó), pero la **barrera de valor subió**: los usuarios **ya no aceptan respuestas genéricas de IA** aunque sean factualmente correctas. Modelos como GPT-5.1 tienen razonamiento inmenso pero carecen del **intent específico, el contexto no dicho y el "vibe"** de los expertos humanos. Ahí entra el **human feedback (HF)**: inyectando feedback domain-specific en la generación, lleva el modelo de **competencia genérica a alineación experta**, y hace del humano un **jugador activo de la arquitectura** — una dimensión nueva: **hybrid-adaptive RAG** (adaptativo = ajusta dinámicamente su estrategia según performance y satisfacción del usuario).

El problema en 2026 **rara vez es que el modelo esté factualmente equivocado** — es que **suena a libro de texto y no a experto del dominio**. El RAG estándar alimenta documentos pertinentes, pero si esos documentos carecen del tono o contexto no dicho de la empresa, el resultado se siente impersonal. El **adaptive RAG** introduce feedback humano pragmático que dirige el ecosistema de genérico a alineado.

Se construye un POC desde cero (sin framework existente) con tres partes separables (que en un proyecto real podrían ser agentes/equipos distintos): **Retriever** (scraping de Wikipedia + labels/índices → ground truth), **Generator** (selección adaptive RAG según ranking, input, generación con GPT-5.1), **Evaluator** (response time, cosine similarity, **human user rating**, **human-expert evaluation**). El núcleo es un **Simulation Ranking (1-5)** — un router dinámico: **rank 1-2 (raw)** sin RAG (respuesta nativa del modelo, RAG desactivado hasta mejorarlo); **rank 3-4 (expert)** detecta un "vibe mismatch" e **inyecta notas de expertos humanos** (un `human_feedback.txt`, sin retrieval de documentos); **rank 5 (research)** RAG estándar keyword-search (+ HF previo si hace falta). El caso: *Company C* quiere un agente conversacional que explique qué es un LLM para que sus equipos lo entiendan y mejoren el soporte de la serie C-phone. Cierra con un **feedback loop visual** (thumbs up/down como en los copilots) que captura el matiz experto en tiempo real y lo persiste en `expert_feedback.txt`.

![[08-fig-8.1.png]]
*Figure 8.1: The adaptive RAG ecosystem zoomed in*

## Arquitectura del adaptive RAG

**RAG no resuelve todos los problemas**: aunque ancla las respuestas en documentos factuales, igual puede producir output irrelevante o genérico. El conocimiento core del modelo es **parametric (pesos)**; en RAG el control de la data externa suele ser autónomo (caps. previos) → un **proceso de HF es muy recomendable** para cerrar la brecha entre *simplemente correcto* y *alineado*. Los componentes del ecosistema (mapeados a D·G·E·T del cap. 1):

- **Retriever** — **D1**: collect+process domain data (artículos de Wikipedia sobre LLMs), fetch+clean; **D4**: retrieval query contra el dataset "Ground Truth".
- **Generator** — **G1**: input del end-user; **G2**: augmented input con HF que inyecta guía experta cuando se detecta necesidad de matiz; **G4**: generation+output con GPT-5.1.
- **Evaluator** — **E1**: métricas (cosine similarity, chequeo semántico); **E2**: **Ranking Simulation (1-5)**, el router dinámico que decide la estrategia (bajo 1-4 dispara inyección de feedback experto; alto 5 confía en RAG automatizado estándar).

> [!note] Hybrid + Adaptive
> **Hybrid**: se integra HF *dentro* del proceso de retrieval — no solo se parsean documentos, se **inyectan archivos con feedback humano** (`human_feedback.txt`) cuando el retrieval genérico es insuficiente; el programa conmuta entre data cruda del dominio y notas de expertos → sistema RAG dinámico human-machine. **Adaptive**: el **Simulation Ranking** adapta la estrategia según un score (1-5). Lema pragmático del ML engineering: *what works, works!* — ranking simulations, classifier agents, feedback loops, lo que sirva.

> [!tip] Por qué un POC (proof of concept)
> Probar que la IA funciona **antes de escalar/invertir**; mostrar que se puede customizar para tono y matices específicos; desarrollar skills ground-up; construir la **data governance** y el control de la empresa sobre la IA; sentar bases sólidas para escalar resolviendo los problemas que surjan en el POC.

## Building hybrid-adaptive RAG en Python

Programa dividido en **Retriever, Generator, Evaluator** (separables como agentes/equipos en paralelo). Se construye desde cero, sin framework, para entender el proceso adaptativo.

### Retriever — entorno y dataset

Cuatro paquetes (se construye desde cero): `requests`, `beautifulsoup4==4.14.3`, `openai==2.26.0`, `scikit-learn==1.8.0`.

**Dataset**: documentos de Wikipedia scrapeados por URL, con **labels** que mapean keyword → URL (primer paso hacia indexar documentos — de naïve RAG/keyword a búsqueda por índices/labels):

```python
import requests
from bs4 import BeautifulSoup
import re
urls = {
    "prompt engineering": "https://en.wikipedia.org/wiki/Prompt_engineering",
    "artificial intelligence": "https://en.wikipedia.org/wiki/Artificial_intelligence",
    "llm": "https://en.wikipedia.org/wiki/Large_language_model",
    "llms": "https://en.wikipedia.org/wiki/Large_language_model"
}
```

**`fetch_and_clean`** — scraping + limpieza en 6 pasos: (1) **User-Agent** que imita Chrome (sin él, Wikipedia bloquea los requests como bots); (2) fetch con chequeo de **HTTP status** (200 OK procede; 403/429 devuelve error en vez de crashear); (3) parsear con BeautifulSoup buscando el `div.mw-parser-output` (con **fallback** a `#bodyContent` si cambió el layout); (4) **eliminar clutter** (References, Bibliography, External links, See also — header + contenido); (5) extraer solo los `<p>` (ignora menús/sidebars/imágenes), unir, y strip de citation markers `[1]` con regex; (6) **verificación** de `cleaned_text` no vacío (catch de páginas con solo imágenes/tablas).

```python
headers = {'User-Agent': 'Mozilla/5.0 ... Chrome/91.0.4472.124 Safari/537.36'}
response = requests.get(url, headers=headers)
if response.status_code != 200:
    return f"Error: Server returned status code {response.status_code}..."
soup = BeautifulSoup(response.content, 'html.parser')
content = soup.find('div', {'class': 'mw-parser-output'})
if content is None:
    content = soup.find('div', {'id': 'bodyContent'})   # Fallback
# ... remove References/Bibliography/External links/See also ...
paragraphs = content.find_all('p')
cleaned_text = ' '.join(paragraph.get_text(...) for paragraph in paragraphs)
cleaned_text = re.sub(r'\[\d+\]', '', cleaned_text)   # strip [1], [42]
if not cleaned_text:
    return "Error: Page was found but no text..."
```

> [!note] Por qué scraping en vez de subir a Oracle
> Este approach (fetch por URL/label) satisface ciertos proyectos que **no quieren subir data a la Oracle DB corporativa hasta que esté aprobada**. El ML engineer debe cuidar **no sobrecargar** el sistema con funciones costosas/no rentables; labelear URLs guía el Retriever a las ubicaciones correctas.

**`process_query`** — identifica un keyword en el input del usuario, dispara `fetch_and_clean`, y limita a `num_words` palabras (chunking básico; para escenarios complejos se recomienda embeber a vectores):

```python
import textwrap
def process_query(user_input, num_words):
    user_input = user_input.lower()
    matched_keyword = next((keyword for keyword in urls if keyword in user_input), None)
    if matched_keyword:
        print(f"Fetching data from: {urls[matched_keyword]}")
        cleaned_text = fetch_and_clean(urls[matched_keyword])
        words = cleaned_text.split()
        first_n_words = ' '.join(words[:num_words])
        # ... format + build prompt "Summarize the following information about {matched_keyword}: ..." ...
        return first_n_words
    else:
        print("No relevant keywords found. Please enter a query related to 'LLM', 'LLMs', or 'Prompt Engineering'.")
        return None
```

### Generator — integrar human feedback

El sistema **adaptive RAG selection** usa scores de HF para determinar la estrategia óptima. Evaluadores humanos asignan **mean scores 1-5** que disparan modos distintos:

![[08-fig-8.2.png]]
*Figure 8.2: Automated RAG triggers based on human feedback scores*

| Score | Modo | Comportamiento |
|---|---|---|
| **1-2** | No RAG | El RAG no tiene capacidad compensatoria → necesita mantenimiento o fine-tuning; **RAG desactivado** temporalmente; el input se procesa **sin retrieval**. |
| **3-4** | Human-expert feedback | Augmentation **solo con feedback de experto humano** (flashcards/snippets); RAG documental **desactivado**, pero la data de expertos augmenta el input. |
| **5** | RAG estándar | Keyword-search RAG (+ HF previo si hace falta); el usuario **no** provee feedback nuevo. |

> [!warning] El scoring es un escenario, no una verdad universal
> Este programa implementa **uno de muchos** escenarios; el sistema de scoring, los niveles y los triggers **varían por proyecto**. Se recomienda **organizar workshops con un panel de usuarios** para decidir cómo implementar el adaptive RAG. *Los usuarios pueden estar insatisfechos sin importar lo que digan las métricas.*

**Input** del usuario de Company C: `user_input = input("Enter your query: ").lower()` → *"What is an LLM?"*.

**Simulating the ranking mechanism** — se asume que el panel viene evaluando hace tiempo; el mean de los ratings se guarda en `ranking`. Se inicializa `text_input=[]` (reinicializar al cambiar de escenario):

```python
ranking = 1   # Select a score between 1 and 5 to run the simulation
text_input = []   # reinitialize when switching scenarios
```

**Ranking 1-2 (No RAG)** — `text_input = user_input` (sin retrieval). GPT-5.1 da una respuesta correcta pero **genérica** ("A Large Language Model (LLM) is an advanced AI system...") que **no satisface a Company C**: no la pueden relacionar con sus issues de customer service, aunque sea state-of-the-art. *No está relacionada con su empresa.*

```python
if ranking >= 1 and ranking < 3:
    text_input = user_input
```

**Ranking 3-4 (Human expert feedback RAG)** — disparado por ratings pobres con RAG automatizado. El panel experto llenó una **flashcard** guardada como documento RAG de nivel experto; se carga `human_feedback.txt`, se limpia y se asigna a `text_input`:

```python
hf = False
if ranking >= 3 and ranking < 5:
    hf = True

import os
if hf == True:
    efile = os.path.exists('human_feedback.txt')
    if efile:
        with open('human_feedback.txt', 'r') as file:
            content = file.read().replace('\n', ' ').replace('#', '')
        text_input = content
        print(text_input)
    else:
        print("File not found")
        hf = False
```

El `human_feedback.txt` explica **qué es un LLM Y cómo ayuda a Company C** (instant support 24/7, asistencia técnica del C-phone, recolección de feedback, smart escalation a humanos, personalización). El output resultante es **mucho mejor**: define LLMs y muestra cómo mejorar el customer service de la serie C-phone — alineado con el dominio.

**Ranking 5 (RAG sin HF docs)** — para usuarios que no necesitan flashcards de expertos (ej. software engineers). Limita a `max_words=100` (optimizar costos de API) y usa `process_query`:

```python
if ranking >= 5:
    max_words = 100   # Limit the size of the data added to the input
    rdata = process_query(user_input, max_words)
    if rdata:
        rdata_clean = rdata.replace('\n', ' ').replace('#', '')
        rdata_sentences = rdata_clean.split('. ')
    text_input = rdata
    print(text_input)
```

Output técnico que los software engineers relacionan con su negocio (LLM = neural language model con self-supervised learning, GPTs, fine-tuning/prompt engineering, pero hereda biases/gaps de su training data → governance).

### Generación de contenido

API key vía Colab secrets (mismo patrón con `raise`). **Separar Retriever y Generator** (equipos/servidores/momentos distintos). Modelo **GPT-5.1**, con `time` para medir velocidad:

```python
from openai import OpenAI
import time
client = OpenAI()
gptmodel = "gpt-5.1"   # 2026 Update: default to gpt-5.1 for production
start_time = time.time()

def call_genai_with_full_text(itext):
    text_input = '\n'.join(itext) if isinstance(itext, list) else itext
    prompt = f"Please summarize and refine the following content for a professional 2026 audience:\n{text_input}"
    try:
        response = client.chat.completions.create(
            model=gptmodel,
            messages=[
                {"role": "system", "content": "You are an advanced 2026 Domain Specialist. Address Users expectancy for high density, zero fluff, and perfect alignment with provided context."},
                {"role": "user", "content": prompt}
            ],
            temperature=0.1
        )
        return response.choices[0].message.content.strip()
    except Exception as e:
        return f"Error calling {gptmodel}: {str(e)}"

genai_response = call_genai_with_full_text(text_input)
response_time = time.time() - start_time
```

## Evaluator

Dos métricas automáticas (**response time**, **cosine similarity**) + dos interactivas (**human user rating**, **human-expert evaluation**). *Se pueden implementar tantas funciones de evaluación matemática y humana como haga falta.*

**Response time** — `time.time() - start_time` (varía según conectividad/servidores de OpenAI; ej. 4.40s). Si es suficiente = decisión de management.

**Cosine similarity** — mide el coseno del ángulo entre dos vectores **TF-IDF**: el `text_input` (input de GPT-5.1) y la respuesta del modelo se tratan como dos "documentos"; cuantifica el **overlap temático y léxico**. 1 = idénticos, 0 = sin relación:

```python
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.metrics.pairwise import cosine_similarity

def calculate_cosine_similarity(text1, text2):
    if not text1 or not text2:
        return 0.0
    vectorizer = TfidfVectorizer()
    tfidf = vectorizer.fit_transform([text1, text2])
    similarity = cosine_similarity(tfidf[0:1], tfidf[1:2])
    return similarity[0][0]

similarity_score = calculate_cosine_similarity(text_input, genai_response)   # -> 0.510
```

Un score de **0.510** indica similitud moderada — pero **la métrica es vaga y difícil de interpretar** entre escenarios. ¿Cómo lo calificará un humano?

**Human user rating** — feedback de usuario (panel de developers testeando). Parámetros: `counter=20` (ratings ya ingresados), `score_history=60` (score total), `threshold=4` (mean mínimo `score_history/counter` para no disparar feedback experto). `evaluate_response` pide un score 1-5 (recursivo si es inválido) y computa el mean:

```python
counter = 20         # number of feedback queries
score_history = 60   # human feedback total
threshold = 4        # minimum rankings to trigger human expert feedback

import numpy as np
def evaluate_response(response):
    print("\nGenerated Response:")
    print(response)
    print("\nPlease evaluate the response based on the following criteria:")
    print("1 - Poor, 2 - Fair, 3 - Good, 4 - Very Good, 5 - Excellent")
    score = input("Enter the relevance and coherence score (1-5): ")
    try:
        score = int(score)
        if 1 <= score <= 5:
            return score
        else:
            print("Invalid score. Please enter a number between 1 and 5.")
            return evaluate_response(response)   # Recursive if invalid
    except ValueError:
        print("Invalid input. Please enter a numerical value.")
        return evaluate_response(response)

# ... counter += 1; score_history += score; mean_score = round(np.mean(score_history/counter), 2) ...
```

Con un score de **3** → `Rankings: 21, Score history: 3.0`. Aunque el cosine fue positivo, el mean (3) está **bajo el threshold (4)** → se **dispara la human-expert evaluation**. *Las métricas miden similitud y velocidad, pero no la accuracy profunda ni el "vibe".*

### Human-expert evaluation

![[08-fig-8.3.png]]
*Figure 8.3: Feedback icons*

Cosine similarity mide similitud **pero no accuracy en profundidad**; el response time tampoco. Si el rating es muy bajo, **¿por qué? Porque el usuario no está satisfecho.** Se descargan iconos thumbs-up/down y se dispara la feedback loop con `counter_threshold=10` y `score_threshold=4`:

```python
if counter > counter_threshold and score_history <= score_threshold:
    print("Human expert evaluation is required for the feedback loop.")
```

Se muestra una **interfaz HTML** (en una celda Python) con los iconos; al presionar **thumbs-down**, un `prompt()` JS pide el matiz y `save_feedback` lo persiste en `expert_feedback.txt` vía `output.register_callback`. Usa **UUIDs únicos** por ejecución para evitar conflictos de DOM:

```python
import base64, uuid
from google.colab import output
from IPython.display import display, HTML

def image_to_data_uri(file_path):
    with open(file_path, 'rb') as image_file:
        encoded_string = base64.b64encode(image_file.read()).decode()
        return f'data:image/png;base64,{encoded_string}'

# display_icons() builds HTML with thumbs up/down (unique up_id/down_id),
# binds onclick -> google.colab.kernel.invokeFunction('notebook.save_feedback', [...])
# thumbs-down triggers a JS prompt("Please specify the domain nuance missed:")

def save_feedback(feedback):
    with open('/content/expert_feedback.txt', 'w') as f:
        f.write(feedback)
    print("Feedback captured for fine-tuning loop.")
    if feedback != "POSITIVE ALIGNMENT":
        print(f"Nuance Note: {feedback}")

output.register_callback('notebook.save_feedback', save_feedback)
display_icons()
```

![[08-fig-8.4.png]]
*Figure 8.4: "Enter feedback" prompt bar*

El experto presiona thumbs-down y escribe el matiz, que revela la **inexactitud real** que las métricas no detectaron:

> *Nuance Note: The response should contain more specific references to our company and the possibility of augmenting the productivity of the marketing department such as automatic marketing campaign filers.*

Este feedback experto se usa luego para **mejorar el dataset RAG** — cerrando el loop entre intención humana y output de máquina.

## Guía del hybrid-adaptive RAG (demo para usuarios)

El notebook sirve como **demo interactiva** para presentar a usuarios antes de desarrollar la app completa — se comporta menos como chatbot estático y más como un sistema **dirigible** entre estrategias de respuesta:

1. **Ask the question** — correr el cell Input y entrar *"What is an LLM?"*.
2. **Choose your mode** (variable `ranking`):
   - **1-2 → Basic Chat**: la IA responde **solo de su training** (GPT-5.1), sin retrieval ni notas; útil para ver la capacidad cruda del modelo.
   - **3-4 → Expert Align**: la IA lee un "cheat sheet" de HF en archivos subidos antes de responder → refleja el tono/conocimiento/opinión experta de la organización. Es **human-in-the-loop alignment**.
   - **5 → Researcher**: la IA **recupera info externa** (Wikipedia/data live) y resume; útil cuando se requiere grounding factual, citas o info actualizada.
3. **Grade the result** — el Evaluator pide un score 1-5 (5 = perfecto, 1 = genérico/fluff). Scores bajos consistentes (1-2) desbloquean el Step 4.
4. **The "Fix It" Loop** — aparecen iconos thumbs up/down; click thumbs-down → un prompt; **enseñarle** exactamente qué faltó (ej. *"You forgot to mention that LLMs in 2026 are mostly used for copilots, not just chatbots"*); se guarda en `expert_feedback.txt` y **la próxima corrida en Mode 4 la IA lee tu nota y se corrige**.

**Checklist**: (1) Input query, (2) Set ranking (1 raw / 4 expert / 5 research), (3) Rate (1-5), (4) Fix it (thumbs-down + explicar).

## Citas

> "Users no longer accept generic AI-sounding answers, even if they are factually correct."
> "In 2026, the issue is rarely that the model like GPT-5.1 is factually wrong. The issue is that the model sounds like a textbook rather than a domain expert."
> "In real-life AI, what works, works!"
> "This chapter wasn't just about learning; it was about building a system that listens."
> "effectively closing the loop between human intent and machine output."

## Para aplicar

- **Usar HF para pasar de competencia genérica a alineación experta** — el problema en 2026 no es la corrección factual sino el *vibe*/tono/contexto no dicho; inyectar feedback domain-specific hace del humano un jugador activo de la arquitectura.
- **Implementar un Simulation Ranking (1-5) como router adaptativo** — rank 1-2 sin RAG (ver el modelo crudo), 3-4 inyectar notas de expertos (`human_feedback.txt`), 5 RAG estándar; la estrategia se adapta al score de satisfacción, no es estática.
- **Hacer el sistema hybrid (human-machine)** — conmutar entre data cruda del dominio y notas de expertos dentro del retrieval; *what works, works* (ranking sims, classifiers, feedback loops).
- **Construir un POC antes de escalar** — probar valor, mostrar customización de tono, ground-up skills, data governance; resolver los problemas del POC antes de invertir.
- **Separar Retriever / Generator / Evaluator** — agentes/equipos/servidores/momentos distintos; instalar el entorno generativo aparte del de retrieval.
- **Scrapear con User-Agent + status check + fallback + limpieza** — imitar browser (evita bloqueo de bots), chequear HTTP status, fallback de selector, eliminar References/clutter, strip de `[1]`, verificar texto no vacío.
- **Labelear URLs como índices** — primer paso de naïve (keyword) a búsqueda por índices sin subir data a la DB corporativa hasta aprobarla; cuidar no sobrecargar con funciones costosas.
- **No confiar solo en métricas automáticas** — response time y cosine similarity (TF-IDF) miden similitud/velocidad, **no accuracy profunda ni vibe**; un cosine 0.510 "positivo" con rating humano 3 → disparar feedback experto.
- **Disparar HF experto por threshold** (`counter > counter_threshold and score_history <= score_threshold`) — cuando suficientes usuarios califican bajo, pedir al experto el matiz.
- **Capturar feedback con UI visual (thumbs up/down) y persistirlo** — `expert_feedback.txt` con UUIDs únicos por run; el matiz se reinyecta en la próxima corrida (human-in-the-loop alignment) y mejora el dataset RAG.
- **Diseñar la UI de feedback con un workshop de usuarios** — no en aislamiento por el equipo de ingeniería; el scoring/niveles/triggers varían por proyecto.
- **Default `gpt-5.1` con `temperature=0.1`** para tareas de dominio (alta densidad, zero fluff, alineación con el contexto provisto).

## Conexiones

- [[_RAG-Driven Generative AI - Second Edition|Mapa del libro]] — el MOC del libro.
- [[07 - Empowering AI Models by Fine-Tuning RAG Data]] — cap. 7: el `expert_feedback.txt` capturado aquí alimenta un *"fine-tuning loop"*; el human feedback (E2) que el cap. 7 anticipaba en SciQ se implementa interactivo.
- [[01 - Why Retrieval-Augmented Generation]] — cap. 1: el **ecosistema D·G·E·T** (D1/D4, G1/G2/G4, E1/E2) explícito; el **modular/adaptive RAG** que conmuta dinámicamente la estrategia es el modular RAG del cap. 1 llevado a HF; cosine similarity con TF-IDF reaparece.
- [[05 - Building a Universal Context Engine]] — cap. 5: el Evaluator (E2, human feedback) y el "Researcher" mode (mode 5) como contraparte del retrieval.
- [[06 - Operationalizing the Universal Context Engine]] — cap. 6: el feedback loop visual y el workshop UX (decidir con stakeholders, no en aislamiento) reaparecen; HTML interactivo en celda.
- [[09 - Building a Conversational RAG Agent]] — cap. 9: multimodal RAG (video); la metadata de los videos se **refina con este mismo loop de human feedback (RLHF)** antes de vectorizarse.
- [[_RAG|RAG]] · [[Enterprise RAG Assistant]] · [[Reranking]] · [[Chunking Strategies]] — el patrón RAG del vault; adaptive/hybrid RAG como evolución.
- [[Grounding]] · [[Hallucinations]] · [[Evals]] · [[LLM as Judge]] · [[Ground Truth]] — el "Researcher mode" para grounding factual; cosine similarity como eval limic; el ground truth de Wikipedia.
- [[RLHF]] · [[_MLOps|MLOps]] — human feedback in-the-loop (no en los pesos, sino en el contexto); el `expert_feedback.txt` como señal para mejorar el dataset.
- [[Cosine Similarity]] · [[TF-IDF]] · [[Embeddings]] — las métricas semánticas del Evaluator (candidatos a nota propia).
- **Human feedback (HF) · Hybrid-adaptive RAG · Adaptive RAG · Simulation Ranking (1-5) · Human-in-the-loop alignment · Domain alignment / vibe · Ground truth (Wikipedia scraping) · BeautifulSoup / User-Agent · Cosine similarity / TF-IDF · `expert_feedback.txt` / flashcards · Thumbs up/down feedback UI · Company C / C-phone (caso) · Researcher / Expert Align / Basic Chat modes** — conceptos/técnicas del capítulo; candidatos a nota propia (varios ya existen en el vault, otros son semillas).
