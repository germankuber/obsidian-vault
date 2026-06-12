---
title: 12 - Combining RAG with the Power of AI Agents and LangGraph
libro: Unlocking Data with Generative AI and RAG
autor: Keith Bourne
capitulo: 12
created: 2026-06-12
tags:
  - libros/unlocking-data-with-generative-ai-and-rag
  - type/case-study
  - status/permanent
aliases:
  - Combining RAG with the Power of AI Agents and LangGraph
  - RAG agéntico con LangGraph
updated: 2026-06-12
---

# 12 - Combining RAG with the Power of AI Agents and LangGraph

> [!info] Capítulo 12 · *Unlocking Data with Generative AI and RAG* — Keith Bourne (Packt, ISBN 9781835887905)
> Una llamada a un [[LLM]] es potente; un **loop** que itera hacia un objetivo es lo que convierte a ese LLM en un **[[AI Agent|agente de IA]]**. El capítulo demuestra que **un agente, en su forma más básica, es el mismo LLM + un loop que termina cuando la tarea está lista**, y presenta **[[LangGraph]]** (2024, sobre [[LCEL]]) como la forma recomendada de orquestar agentes como **[[Cyclical Graph|grafos cíclicos]]** ([[Node|nodos]], [[Edge|edges]], [[Conditional Edge|conditional edges]]) con **[[Persistence|memoria/persistencia]]** y **[[Human-in-the-loop|human-in-the-loop]]**, ilustrando el patrón **[[ReAct]]** (reason + act). El **code lab 12.1** construye un agente de recuperación que decide entre buscar en un índice o en la web. Navegá: [[_Unlocking Data with Generative AI and RAG|Libro]] · anterior [[11 - Using LangChain to Get More from RAG]] · siguiente [[13 - Using Prompt Engineering to Improve RAG Efforts]]. Código: github.com/PacktPublishing/Unlocking-Data-with-Generative-AI-and-RAG/tree/main/Chapter_12.

## Resumen

El capítulo cierra la parte de técnicas avanzadas del libro combinando RAG con **agentes de IA**. La intuición central es desmitificadora: una sola llamada a un LLM es poderosa, pero darle un **loop que itera hacia un objetivo** lo transforma en un **[[AI Agent|agente]]**. El foco reciente de [[LangChain]] son justamente los **flujos de trabajo agénticos**, y su nueva librería **[[LangGraph]]** se empareja con los agentes para mejorar RAG. El capítulo recorre los **fundamentos de los agentes de IA y su integración con RAG**, los **grafos / LangGraph** y el patrón **[[ReAct]]**, y luego un **code lab** que agrega un agente de recuperación LangGraph a RAG (con sus **[[Tool|tools]] y [[Toolkit|toolkits]]**, su **[[Agent State|agent state]]** y los nodos del grafo), para terminar en la **teoría de grafos** (nodos, edges, conditional edges) y el armado del **grafo cíclico** compilado.

La tesis operativa: el loop es más potente que la llamada directa porque aprovecha la capacidad del LLM de **razonar y descomponer tareas** en sub-tareas más simples; mientras itera, el agente dispone de **funciones llamadas tools** y el LLM razona cuál usar, cómo y con qué datos. [[LangGraph]] aporta dos cosas que el viejo `AgentExecutor` no daba fácilmente: definir **ciclos (grafos cíclicos)** y **memoria incorporada** ([[Persistence|persistencia]]). El code lab materializa todo en un agente que, dada una consulta, decide entre el **retriever** (índice del cap. 8) o un **web search** ([[Tavily]]), puntúa la relevancia de lo recuperado y, según eso, **genera** la respuesta o **mejora** la pregunta y reintenta — el clásico loop ReAct expresado como grafo cíclico.

## Fundamentos de los agentes de IA y su integración con RAG

La definición desmitificada del capítulo es deliberadamente simple: un **[[AI Agent|agente de IA]]** en su forma más básica es **el mismo LLM + un loop que termina cuando la tarea está hecha**. La Figura 12.1 es exactamente el **loop del agente RAG** que se construye en el code lab.

> [!note] "It's just a loop folks!" — un agente de IA, en su forma más básica, es el mismo LLM más un **loop** que itera hasta completar la tarea. No hay magia: hay un bucle de razonamiento.

![[12-fig-12.1-agent-control-flow.jpg]]
*Figure 12.1 – Graph of the agent's control flow*

En el grafo, los términos clave: las cajas ovaladas (p. ej. *agent*, *retrieve*) son **[[Node|nodos]]**; las líneas son **[[Edge|edges]]**; las líneas punteadas son **[[Conditional Edge|conditional edges]]** (edges que además son puntos de decisión). El loop es más potente que el uso directo del LLM porque **explota la capacidad del LLM de razonar y de partir tareas complejas en otras más simples**. Mientras está en el loop, se le dan al agente funciones llamadas **[[Tool|tools]]**; el LLM razona qué tool usar, cómo usarla y qué datos pasarle. Esto escala: múltiples agentes, muchas tools, knowledge graphs, frameworks y arquitecturas.

### Vivir en un mundo de agentes

Los LLMs no quedan obsoletos: los agentes aprovechan una **versión más potente** del LLM, en la que el LLM es el **"cerebro"**. El agente es una **capa entre el usuario y el LLM** que empuja al LLM a recorrer soluciones de múltiples pasos, obteniendo mejores resultados. Esto refleja la resolución de problemas del mundo real: cadenas de observación, razonamiento y ajuste.

### El LLM como cerebro de los agentes

Conviene poner el LLM **más inteligente** disponible como cerebro, porque afecta directamente el razonamiento, las decisiones y los resultados. La metáfora se rompe — pero de forma buena: se puede **intercambiar** el cerebro LLM, o darle **varios cerebros LLM** que se chequeen entre sí, ganando flexibilidad.

> [!tip] Mezclá y cambiá LLMs por tarea: usá el más inteligente como cerebro del agente y modelos más baratos/rápidos para sub-tareas simples; podés incluso darle varios cerebros que se validen entre sí.

## Grafos, agentes de IA y LangGraph

[[LangChain]] introdujo **[[LangGraph]] en 2024** como una extensión sobre **[[LCEL]]** para cargas de trabajo agénticas componibles y personalizables, apoyándose en la **[[Graph Theory|teoría de grafos]]** (nodos / edges) para gestionar agentes. La clase más vieja `AgentExecutor` sigue existiendo, pero **LangGraph es ahora la forma recomendada** de construir agentes; provee un objeto pre-construido equivalente a `AgentExecutor`. LangGraph aporta dos cosas: (1) definir fácilmente **ciclos (grafos cíclicos)**; (2) **memoria incorporada**.

A lo largo del tiempo emergieron enfoques de agentes — **orchestration agents, ReAct agents, self-refine agents, multi-agent frameworks** — con un tema común: un **grafo cíclico** para el control de flujo del agente. Las implementaciones se vuelven obsoletas, pero los conceptos persisten en LangGraph, que da **flujos controlados** y así evita agentes "rogue".

> [!warning] Sin un control de flujo explícito, los agentes pueden volverse **rogue** (irse por la tangente o resolver la tarea equivocada). LangGraph ataca ese problema temprano de los agentes dándoles un grafo de control bien definido.

### ReAct = reason + act

El patrón **[[ReAct]]** combina **razonar + actuar**: el LLM piensa qué hacer → decide una acción → la acción se ejecuta en un entorno → se devuelve una **observación** → se repite hasta cumplir el objetivo (Figura 12.2). LangGraph representa estos loops como **grafos cíclicos** (nodos + edges) — la columna vertebral del framework de agentes; ese foco en el control de flujo aborda los desafíos tempranos de los agentes (agentes rogue / tarea equivocada).

![[12-fig-12.2-react-cyclical-graph.jpg]]
*Figure 12.2 – ReAct cyclical graph representation*

> [!note] **ReAct (reason + act)**: piensa → decide una acción → la acción se ejecuta en el entorno → vuelve una observación → repite hasta lograr el objetivo. El loop es el corazón del agente.

LangGraph además tiene **[[Persistence|persistencia]]** = mantiene la **memoria del agente** (el componente *OBSERVE* del ciclo), lo que habilita **múltiples conversaciones simultáneas**, recordar iteraciones previas y funcionalidades de **[[Human-in-the-loop|human-in-the-loop]]**. Paper de ReAct: arxiv.org/abs/2210.03629.

> [!note] **Persistencia / memoria**: LangGraph mantiene el estado del agente (el *OBSERVE* de ReAct), permitiendo varias conversaciones a la vez, recordar iteraciones anteriores e insertar un humano en el loop (human-in-the-loop).

## Code lab 12.1 – Un agente LangGraph para RAG

El objetivo: un agente que **decide si recupera de un índice O usa una búsqueda web**, muestra sus "pensamientos" internos y funciona como sesión de chat.

Instalación:

```bash
%pip install tiktoken
%pip install langgraph
```

`tiktoken` = el tokenizador de OpenAI; `langgraph` = el paquete propiamente dicho ([[Tiktoken]], [[LangGraph]]).

### Setup: dos LLMs

```python
llm = ChatOpenAI(model_name="gpt-4o-mini", temperature=0, streaming=True)
agent_llm = ChatOpenAI(model_name="gpt-4o-mini", temperature=0, streaming=True)
```

`agent_llm` = el **cerebro del agente** (razonamiento / ejecución); `llm` = tareas generales del LLM. Conviene experimentar con LLMs distintos por tarea (más baratos/rápidos para tareas simples como `improve` o `score_documents`). `streaming=True` encaja con un agente que hace varias llamadas (a veces en paralelo).

> [!tip] Usá `streaming=True` para agentes: hacen muchas llamadas (a veces paralelas) y conviene transmitir resultados a medida que llegan.

### Tools y toolkits

Una **[[Tool|tool]]** = una acción puesta a disposición del agente. La tool de búsqueda web:

```python
from langchain_community.tools.tavily_search import TavilySearchResults
_ = load_dotenv(dotenv_path='env.txt')
os.environ['TAVILY_API_KEY'] = os.getenv('TAVILY_API_KEY')
!export TAVILY_API_KEY=os.environ['TAVILY_API_KEY']
web_search = TavilySearchResults(max_results=4)
web_search_name = web_search.name
```

Hay que obtener una API key de **[[Tavily]]** (tavily.com) y agregarla a `env.txt`. `max_results=4`. Se puede correr directo con `web_search.invoke(user_query)` → devuelve una lista de dicts con `'url'` + `'content'`. Ejemplo de salida (truncado):

```python
[{'url': 'https://sustainability.google/...',
  'content': '... Google Maps ... Google Shopping ... Google Flights ... Nest ...'},
 {'url': '...',
  'content': '2023 Environmental Report ...'}]
```

> [!warning] Necesitás una API key de **[[Tavily]]** (tavily.com) cargada en `env.txt` para que la tool de web search funcione.

La tool de retriever:

```python
from langchain.tools.retriever import create_retriever_tool
retriever_tool = create_retriever_tool(
    ensemble_retriever,
    "retrieve_google_environmental_question_answers",
    "Extensive information about Google environmental efforts from 2023.",
)
retriever_tool_name = retriever_tool.name
```

El web search se importa de `langchain_community.tools.tavily_search` (tercero); la tool de retriever, de `langchain.tools.retriever` (core de LangChain). Usa el **`ensemble_retriever`** del code lab 8.3 ([[08 - Similarity Searching with Vectors]]). El segundo campo es el nombre que **ve el agente**: `retrieve_google_environmental_question_answers` — usá **nombres verbosos** para que el agente entienda las tools.

> [!tip] Poné **nombres y descripciones verbosos** a las tools: el agente decide qué tool llamar leyendo su nombre/descripción. Cuanto más claros, mejor razona.

La lista de tools:

```python
tools = [web_search, retriever_tool]
```

Hay **cientos** de tools en LangChain. El LLM debe ser bueno razonando + haciendo **tool calling** (los chat models están fine-tuneados para tool calling; los no-chat pueden no manejar tools complejas/múltiples). Los **[[Toolkit|toolkits]]** son grupos convenientes de tools. Cita textual de LangChain:

> "For many common tasks, an agent will need a set of related tools. For this LangChain provides the concept of toolkits - groups of around 3-5 tools needed to accomplish specific objectives. For example, the GitHub toolkit has a tool for searching through GitHub issues, a tool for reading a file, a tool for commenting, etc."

Ejemplos: toolkit de pandas DataFrame, integración con Salesforce.

> [!tip] Preferí LLMs **chat** (fine-tuneados para tool calling) cuando el agente usa tools complejas o múltiples; los no-chat pueden fallar. Y explorá los **toolkits** (grupos de ~3-5 tools por objetivo) antes de armar tools sueltas.

### Agent state

El **[[Agent State|agent state]]** es un componente clave: la clase `AgentState` rastrea el "estado" a lo largo del tiempo, está disponible para todas las partes del grafo y puede guardarse en una capa de persistencia.

```python
from typing import Annotated, Literal, Sequence, TypedDict
from langchain_core.messages import BaseMessage
from langgraph.graph.message import add_messages
class AgentState(TypedDict):
    messages: Annotated[Sequence[BaseMessage], add_messages]
```

`BaseMessage` = clase base de los mensajes de conversación; acá el estado = una **lista de "messages"**. Más imports:

```python
from langchain_core.messages import HumanMessage
from langchain_core.pydantic_v1 import BaseModel, Field
from langgraph.prebuilt import tools_condition
```

`HumanMessage` = un mensaje del usuario; `BaseModel`/`Field` = modelos de datos/validación de [[Pydantic]]; `tools_condition` = función pre-construida de LangGraph que evalúa la decisión del agente de usar tools según el estado de la conversación.

El prompt de generación (reemplaza al viejo `prompt = hub.pull("jclemens24/rag-prompt")`, renombrado a `generation_prompt`):

```python
generation_prompt = PromptTemplate.from_template(
    """You are an assistant for question-answering tasks.
    Use the following pieces of retrieved context to answer
    the question. If you don't know the answer, just say
    that you don't know. Provide a thorough description to
    fully answer the question, utilizing any relevant
    information you find.
    Question: {question}
    Context: {context}
    Answer:"""
)
```

## Teoría de grafos

Los **[[Graph Theory|grafos]]** son estructuras matemáticas para representar relaciones; los **[[Node|nodos]]** son los objetos y los **[[Edge|edges]]** son las relaciones (líneas). Un **[[Conditional Edge|conditional edge]]** es una decisión sobre a qué nodo ir a continuación (en ReAct también se le llama **action edge** — reason + action).

![[12-fig-12.3-basic-graph.jpg]]
*Figure 12.3 – Basic graph representing our RAG application*

La Figura 12.3 es un grafo cíclico básico con nodos: **start, agent, retrieve tool, generation, observation, end**. Los edges clave: dónde el LLM **decide qué tool usar**, observa si los datos recuperados son suficientes → empuja a **generation**, o devuelve la observación al **agent** para reintentar (esos son los conditional edges).

> [!note] **Nodo** = un objeto del grafo. **Edge** = una relación/transición entre nodos. **Conditional edge** (action edge en ReAct) = un edge que además es un **punto de decisión** sobre a qué nodo ir.

## Nodos y edges del agente

Los **tres componentes clave** de un grafo de RAG agéntico: **[[Agent State|state]]** (ya hecho), **[[Node|nodos]]** (agregan/actualizan el estado) y **[[Conditional Edge|conditional edges]]** (deciden el próximo nodo).

| Componente | Qué hace |
|---|---|
| **State** (`AgentState`) | Mantiene el estado (la lista de `messages`) disponible a todo el grafo, persistible. |
| **Nodos** | Agregan a / actualizan el estado (p. ej. `agent`, `retrieve`, `improve`, `generate`). |
| **Conditional edges** | Deciden a qué nodo ir según el estado (p. ej. `tools_condition`, `score_documents`). |

> [!note] Los **tres componentes** de un grafo RAG agéntico: **state** (qué se sabe), **nodos** (qué se hace, mutando el state) y **conditional edges** (qué se decide a continuación).

### score_documents (conditional edge)

```python
def score_documents(state) -> Literal["generate", "improve"]:
    class scoring(BaseModel):
        binary_score: str = Field(description="Relevance score 'yes' or 'no'")
    llm_with_tool = llm.with_structured_output(scoring)
    prompt = PromptTemplate(
        template="""You are assessing relevance of a retrieved document to a user question with a binary grade. Here is the retrieved document:
{context}
Here is the user question: {question}
If the document contains keyword(s) or semantic meaning related to the user question, grade it as relevant. Give a binary score 'yes' or 'no' score to indicate whether the document is relevant to the question.""",
        input_variables=["context", "question"],
    )
    chain = prompt | llm_with_tool
    messages = state["messages"]
    last_message = messages[-1]
    question = messages[0].content
    docs = last_message.content
    scored_result = chain.invoke({"question": question, "context": docs})
    score = scored_result.binary_score
    if score == "yes":
        print("---DECISION: DOCS RELEVANT---")
        return "generate"
    else:
        print("---DECISION: DOCS NOT RELEVANT---")
        print(score)
        return "improve"
```

Un modelo [[Pydantic]] `scoring` con `binary_score` (`'yes'`/`'no'`); `llm.with_structured_output(scoring)` fuerza salida estructurada y validada; el prompt evalúa la relevancia del documento; la cadena [[LCEL]] `prompt | llm_with_tool`; se extraen `messages`/`last_message`/`question`/`docs`; se invoca → `binary_score`; si `"yes"` → `"generate"`, si no → `"improve"`. Es un punto de decisión (conditional edge): **generar** la respuesta o **mejorar/reescribir** la pregunta.

### agent

```python
def agent(state):
    print("---CALL AGENT---")
    messages = state["messages"]
    llm = llm.bind_tools(tools)
    response = llm.invoke(messages)
    return {"messages": [response]}
```

Invoca el modelo del agente (vincula las tools con `bind_tools`) → `response`.

> [!warning] El snippet del libro **rebinda `llm`** aunque el texto describe usar `agent_llm`; se reproduce fiel al libro tal cual aparece.

### improve

```python
def improve(state):
    print("---TRANSFORM QUERY---")
    messages = state["messages"]
    question = messages[0].content
    msg = [
        HumanMessage(content=f"""\n
        Look at the input and try to reason about
        the underlying semantic intent / meaning.
        \n
        Here is the initial question:
        \n ------- \n
        {question}
        \n ------- \n
        Formulate an improved question:
        """,
        )
    ]
    response = llm.invoke(msg)
    return {"messages": [response]}
```

Razona sobre la **intención semántica subyacente** → formula una pregunta mejorada.

### generate

```python
def generate(state):
    print("---GENERATE---")
    messages = state["messages"]
    question = messages[0].content
    last_message = messages[-1]
    question = messages[0].content
    docs = last_message.content
    rag_chain = generation_prompt | llm | str_output_parser
    response = rag_chain.invoke({"context": docs, "question": question})
    return {"messages": [response]}
```

Arma `rag_chain = generation_prompt | llm | str_output_parser` (el `str_output_parser` del cap. 11 — ver [[10 - Key RAG Components in LangChain]] y [[11 - Using LangChain to Get More from RAG]]), lo hidrata con `context` (docs) + `question` → `response`.

## Armar el grafo cíclico

Imports:

```python
from langgraph.graph import END, StateGraph
from langgraph.prebuilt import ToolNode
```

`END` = nodo de fin de workflow; `StateGraph` = define el grafo de estado ([[StateGraph]]); `ToolNode` = un nodo para una tool/acción ([[ToolNode]]).

### Nodos y edges

```python
workflow = StateGraph(AgentState)
workflow.add_node("agent", agent)  # agent
retrieve = ToolNode(tools)
workflow.add_node("retrieve", retrieve)  # retrieval from web and or retriever
workflow.add_node("improve", improve)  # Improving the question for better retrieval
workflow.add_node("generate", generate)  # Generating a response after we know the documents are relevant
```

Entry point + conditional edges:

```python
workflow.set_entry_point("agent")
```

```python
workflow.add_conditional_edges("agent", tools_condition,
    {
        "tools": "retrieve",
        END: END,
    },
)
```

`tools_condition` decide: ir a `"retrieve"` o `END`.

```python
workflow.add_conditional_edges("retrieve", score_documents)
workflow.add_edge("generate", END)
workflow.add_edge("improve", "agent")
```

Tras `"retrieve"` → `score_documents` decide el próximo; `"generate"` → `END`; `"improve"` → vuelve a `"agent"` (el **loop**).

### Compilar y correr

```python
graph = workflow.compile()
```

Visualización:

```python
from IPython.display import Image, display
try:
    display(Image(graph.get_graph(xray=True).draw_mermaid_png()))
except:
    pass
```

Correr el agente:

```python
import pprint
inputs = {
    "messages": [
        ("user", user_query),
    ]
}
```

```python
final_answer = ''
```

```python
for output in graph.stream(inputs):
    for key, value in output.items():
        pprint.pprint(f"Output from node '{key}':")
        pprint.pprint("---")
        pprint.pprint(value, indent=2, width=80, depth=None)
        final_answer = value
```

`graph.stream(inputs)` transmite las salidas; imprime la salida de cada nodo (los "pensamientos" del agente); `final_answer` = salida del último nodo.

### La salida del agente

La ejecución muestra el agente razonando paso a paso (excerpts verbatim):

```
---CALL AGENT---
"Output from node 'agent':"
"---"
# AIMessage con tool_calls -> elige la tool
# 'retrieve_google_environmental_question_answers'
```

```
---CHECK RELEVANCE---
---DECISION: DOCS RELEVANT---
"Output from node 'retrieve':"
"---"
# ToolMessage (truncado): iMasons Climate Accord ... ReFED ...
```

```
---GENERATE---
"Output from node 'generate':"
"---"
"Google has a comprehensive and multifaceted approach to environmental sustainability ... 1. **Carbon Reduction and Renewable Energy** ..."
```

El mensaje final vía `final_answer['messages'][0]` (excerpt verbatim):

```
Google has a comprehensive and multifaceted approach ...
1. **Carbon Reduction and Renewable Energy**:
   - **iMasons Climate Accord** ...
   - **Net-Zero Carbon** ...
   all-electric, net water-positive Bay View campus ...
```

## Citas

> "For many common tasks, an agent will need a set of related tools. For this LangChain provides the concept of toolkits - groups of around 3-5 tools needed to accomplish specific objectives. For example, the GitHub toolkit has a tool for searching through GitHub issues, a tool for reading a file, a tool for commenting, etc."

> "It's just a loop folks!"

## Para aplicar

- **Pensá el agente como LLM + loop** — antes de adoptar un framework pesado, recordá que un agente es un LLM iterando hacia un objetivo; el valor está en el control de flujo, no en magia.
- **Usá LangGraph, no `AgentExecutor`** — es la forma recomendada hoy; te da ciclos (grafos cíclicos) y memoria/persistencia listas para usar.
- **Nombres y descripciones verbosos en las tools** — el agente elige la tool leyendo su nombre/descripción (`retrieve_google_environmental_question_answers`).
- **Elegí chat models fine-tuneados para tool calling** — los no-chat pueden fallar con tools complejas o múltiples.
- **`streaming=True` para agentes** — hacen muchas llamadas (a veces paralelas).
- **Mezclá/cambiá LLMs por tarea** — el más inteligente de cerebro, los más baratos para sub-tareas (`improve`, `score_documents`).
- **Conseguí una API key de Tavily** (tavily.com) y cargala en `env.txt` para el web search.
- **Explorá los toolkits** — grupos de ~3-5 tools por objetivo (pandas DataFrame, GitHub, Salesforce).
- **Estructurá las decisiones con conditional edges + structured output** — `score_documents` usa un modelo Pydantic (`'yes'`/`'no'`) para enrutar entre `generate` e `improve`.

## Conexiones

- [[_Unlocking Data with Generative AI and RAG|Libro]] — el MOC del libro.
- [[11 - Using LangChain to Get More from RAG]] — capítulo anterior (loaders, splitters, output parsers; el `str_output_parser` que reusa `generate`) · [[13 - Using Prompt Engineering to Improve RAG Efforts]] — capítulo siguiente (prompt engineering para mejorar RAG).
- [[08 - Similarity Searching with Vectors]] — de ahí viene el `ensemble_retriever` (code lab 8.3) que envuelve la tool de retriever.
- [[10 - Key RAG Components in LangChain]] — los componentes de LangChain (retrievers, `str_output_parser`) que la cadena reutiliza.
- **[[AI Agent]]** — LLM + loop que razona y descompone tareas (candidato a nota propia).
- **[[LangGraph]]** — extensión sobre [[LCEL]] (2024) para agentes como grafos cíclicos con memoria.
- **[[ReAct]]** — reason + act; el ciclo piensa→actúa→observa→repite.
- **[[Tool]]** · **[[Toolkit]]** — acciones disponibles para el agente / grupos de ~3-5 tools.
- **[[Agent State]]** — la clase `AgentState` (lista de `messages`) que cruza todo el grafo.
- **[[Graph Theory]]** · **[[Node]]** · **[[Edge]]** · **[[Conditional Edge]]** · **[[Cyclical Graph]]** — la teoría de grafos detrás de LangGraph.
- **[[Persistence]]** · **[[Human-in-the-loop]]** — memoria del agente y la inserción de un humano en el loop.
- **[[Tavily]]** — el servicio de web search usado como tool.
- **[[StateGraph]]** · **[[ToolNode]]** — las clases de LangGraph para definir el grafo y los nodos-tool.
- [[Pydantic]] — modelos de datos/validación (`scoring`, structured output).
- [[LCEL]] · [[EnsembleRetriever]] · [[LangChain]] · [[Tiktoken]] · [[LLM]] — piezas reutilizadas del resto del libro.
